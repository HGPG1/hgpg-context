<!-- Last Updated: 2026-05-14 -->

# Quiz Scoring Logic (Charlotte New Construction)

Documents the scoring function in `src/app/quiz/QuizClient.tsx` (lines ~105-128) on `HGPG1/charlotte-new-construction-nextjs`.

## What it is

A **community ranker**, not a buyer-quality scorer. It sorts 6 hardcoded Charlotte-area communities against a buyer's quiz answers and surfaces the top 3 as recommendations on the results screen. The score never leaves the page — not posted to FUB, not used for routing, not compared against close rate.

## Inputs (5 quiz steps)

| Step | Field | Type | Used in scoring? |
|---|---|---|---|
| 1 | `budgetMin` / `budgetMax` | numeric range | ✅ Yes |
| 2 | `priorities` | multi-select: schools, commute, amenities, walkability, newest, value | ✅ Yes |
| 3 | `commute` | uptown / south-charlotte / fort-mill / work-from-home | ❌ NOT used |
| 4 | `state` | nc / sc / either | ✅ Yes |
| 5 | `firstTimeBuyer` | yes / no / owned-before | ❌ NOT used |

**Two of the five answers (commute, FTB) are dead weight for scoring purposes.** They get captured to FUB on lead submit, but don't influence which communities the user sees.

## The 6 communities (hardcoded in `getMatches`)

| Community | Area | Price band | Tags |
|---|---|---|---|
| Ballantyne | Charlotte | $400K-$800K | schools, amenities, walkability |
| Fort Mill | SC | $350K-$650K | schools, value, commute |
| Indian Land | SC | $300K-$550K | value, newest, commute |
| Waxhaw | NC | $380K-$650K | schools, walkability, amenities |
| Lancaster | SC | $280K-$450K | value, newest |
| Steele Creek | Charlotte | $320K-$550K | value, commute, amenities |

Notably **missing**: Tega Cay, Rock Hill, Huntersville, Matthews, Pineville, Concord, Mooresville. Brand-relevant gaps.

## Scoring formula

For each community:

| Condition | Points |
|---|---|
| Community's price band overlaps buyer's budget range at all | +3 |
| Each matching priority tag (multiplied) | +2 per match |
| State match: `nc` lead + non-SC community | +2 |
| State match: `sc` lead + SC community | +2 |
| State indifference: `either` | +1 |

Top 3 by score are returned.

## Known weaknesses (calibration backlog)

1. **Binary budget match.** Any price overlap gets +3. A buyer at $400-500K and a community spanning $300-550K get the same +3 as a buyer at $400-500K and a community spanning $400-800K. Should weight by overlap quality (overlap / range or similar).
2. **Uniform priority weights.** Every matching priority = +2. In practice, "schools" likely correlates more with conversion than "newest construction" for the target buyer (move-up family). Worth calibrating against real lead data.
3. **State scoring inverted for "either"-leaning buyers.** `state==='either'` gets +1 across the board, which under-rewards Charlotte/NC communities for buyers who genuinely don't care. The Charlotte/NC vs. SC asymmetry isn't symmetric in the lead's mind.
4. **No tie-breaker.** If three communities tie at the top score, the order is non-deterministic (whatever `.sort` settles on).
5. **Hardcoded community list.** Brand expansion or a new community launch requires a code deploy. Same gotcha noted in `charlotte-new-construction.md` for builder data.

## What "calibration" would actually mean

The brain originally noted "buckets haven't been calibrated against actual lead quality yet (need data from first 50 leads)" — but the current code has no buckets to calibrate, just a sum. Plausible calibration approaches if/when the campaign generates 50+ qualified leads:

- **Re-weight priorities** by which signals correlate with downstream pipeline progression (assigned → contacted → appointment-set → contract).
- **Switch budget to weighted overlap** rather than binary.
- **Add commute and FTB to scoring** instead of letting them sit unused.
- **Validate community list** against where in-market leads actually buy (lead might say "Fort Mill" but close in Indian Land — which communities have the lowest "intent vs. outcome" gap?).

## Pickup notes for future work

- Code lives in `src/app/quiz/QuizClient.tsx`. Single file, no separate scoring module.
- The score is local state in the component; no API call returns it. Server-side has no awareness of the score the user saw.
- If we want the score (or the top 3) sent to FUB, the lead capture call inside the same component (`LeadCaptureModal` from `@/components/LeadCaptureModal`) would need to be extended.
- Ad campaign is currently paused (see `charlotte-new-construction.md`), so the 50-leads bar for calibration is dormant. Resume after Variant D + Automation 2.0 verification clears.
