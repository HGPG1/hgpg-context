<!-- Last Updated: 2026-05-08 -->

# CMA Engine — Adjustment Engine Bugs (Discovered 2026-05-08)

Real bugs found while reviewing 4 production CMAs (215 Tyndale Ct, 504 Redwine St, 5601 Medlin Rd, 6022 Candlestick Ln). The adjustment engine is producing PMVs that are NOT defensible against appraisal review. Two of four reports were unshippable due to math bugs; one (Candlestick) had directionally correct comps but ~$200K overstated PMV due to logic gaps.

**Status: BLOCKING. Do not ship CMAs to clients until these are fixed.** The wrong-zip fallback work currently in flight is unrelated and can ship independently.

## Bug summary (severity-ordered)

### Bug 1 — Subject features adjustment ignores comp parity (CRITICAL)

The adjustment engine adds the full "subject features" delta (currently up to ~$70K for the standard kit: finished-basement, primary-on-main, screened-porch, in-law-suite, chef-kitchen) to **every comp** regardless of whether the comp has those features.

Evidence from 6022 Candlestick Ln (report id `8023175d-37c8-45d7-95da-b9245abe4761`): every adjustment record contains an identical line `"subjectValue": "finished-basement, primary-on-main, screened-porch, in-law-suite, chef-kitchen"` with `"compValue": "—"` and `"dollarAdjustment": 70000`. The MLS remarks for at least two of the comps (7026 Wyngate, 7200 Irongate) explicitly state "finished walk-out basement" — but the engine still applied the full $25K finished-basement adjustment as if subject-only.

Required fix: parity scoring per feature. For each feature in the rate table:
- If both subject and comp have it → $0 adjustment
- If only subject has it → +$rate (subject worth more)
- If only comp has it → -$rate (subject worth less)

Detecting features on comps requires either (a) MLS remarks NLP, (b) explicit hgpg* tags on the comp record (already done for `hgpgConditionTier` and `hgpgPoolTier`), or (c) a manual tagging step in the Adjust UI before the report runs. Recommended: implement (b) — add `hgpgFeatures: string[]` to the listing record, populated by the same inference pass that sets condition tier and pool tier, with conservative thresholds (only tag if remarks are unambiguous).

### Bug 2 — GLA includes basement sqft (CRITICAL, appraisal violation)

The subject sqft input is a single field that gets compared 1:1 against comp sqft at $60/sqft. This violates Fannie Mae URAR Form 1004 and Charlotte appraisal convention.

Appraisal standard:
- **GLA (Gross Living Area) = above-grade finished sqft only.** Basement sqft is NEVER part of GLA, even if finished, even one inch below grade.
- **Below-grade finished area** has its own contributory rate, typically 30-50% of GLA rate (so ~$25-30/sqft for Charlotte metro at the $500K-$900K tier).
- **Below-grade unfinished** is much lower, ~$8-12/sqft.
- **In-law suite / accessory dwelling** can carry an additional contributory premium ($10-30K) if it has separate entrance, kitchenette, full bath, bedroom — beyond the finished basement rate.

Evidence from 6022 Candlestick Ln: subject reported as 5,055 sqft total, with `finished-basement` and `in-law-suite` features. The basement is half-finished/half-unfinished per Brian. So subject GLA is realistically ~3,500-3,800 sqft, with ~750 sqft finished basement (containing the in-law suite) and ~750 sqft unfinished basement. The engine compared 5,055 to comp's 4,955 (which is GLA-only per MLS convention) at full $60/sqft — assigning the subject a premium for sqft it shouldn't get credit for at that rate.

Required fix: separate subject input fields for GLA, finished basement sqft, unfinished basement sqft. Match against MLS columns separately:
- `living_area` or `above_grade_finished_area` → GLA, $60/sqft contributory
- `below_grade_finished_area` → finished basement, ~$25-30/sqft
- (basement total - finished) → unfinished basement, ~$10/sqft
- ADU / in-law premium remains a feature-level adjustment if applicable

Default rates table in adjustment engine needs updating to reflect this structure.

### Bug 3 — Outlier penalty applies inconsistently to similar comps (MAJOR)

The outlier penalty (drops weight to 0.02) fires based on a `percentFromAnchor` threshold, but the threshold is computed against an anchor that itself depends on which comps are weighted vs penalized — circular logic that lets one outlier through while penalizing a near-twin.

Evidence from 6022 Candlestick: 5542 Soft Shell Drive ($610K, 6BR, Walnut Creek) was outlier-penalized to weight 0.02. 5659 Soft Shell Drive ($595K, 6BR, Walnut Creek) was NOT penalized and kept weight 0.08. They tell the same story (Walnut Creek 6BR luxury inventory in low $600s not moving), but the engine treated them differently.

Required fix: outlier penalty should consider clusters, not just individual percentFromAnchor. If 2+ comps within ±5% of each other and >25% from anchor, treat the cluster as legitimate counter-evidence to the anchor, not as outliers. Simpler alternative: penalize ALL comps where percentFromAnchor exceeds threshold OR adjusted price differs from nearest sibling by less than $30K — symmetric penalty.

### Bug 4 — PMV anchor sanity check missing (MAJOR)

The weighted PMV can land in a value that no individual comp supports — i.e., the average of two distant clusters with no data point near the average.

Evidence from Candlestick: PMV anchor $1,172,236. Closest comp adjusted prices: $991,500 (Wyngate) and $1,385,500 (Mill Race). NO comp adjusted to anywhere near $1.17M. The anchor is a synthetic average of two different submarkets.

Required fix: after computing weighted PMV, check that at least one comp's adjusted price falls within ±$50K (or ±5%) of the anchor. If not, set `confidence: "low"` and surface a warning in the report ("Weighted PMV does not correspond to any individual comp — manual review required"). Optionally block report generation until agent confirms.

### Bug 5 — Confidence range collapses to single point (CRITICAL math bug)

Two of four reports (504 Redwine St, 5601 Medlin Rd) shipped with `Low` and `High` PMV bounds equal to each other — and in the Medlin case, the PMV anchor sat $45K *above* both bounds. Logically impossible.

Evidence:
- Redwine: PMV $374,029 / Low $381,760 / High $381,760
- Medlin: PMV $376,389 / Low $331,500 / High $331,500 (anchor above both bounds)

Likely cause: when comp set has 1 closed-status comp at weight 1.00 and 5 active-status comps at weight 0.08 each, the bounds calc collapses on the dominant weight and ignores the spread of the actives. Or the bounds are computed from a percentile filter that returns identical values when comp count is too small.

Required fix: bounds should always span at minimum the min/max of weighted-contribution adjusted prices, with a sanity check that `low <= anchor <= high`. If the calc produces invalid bounds, fall back to `low = anchor * 0.93, high = anchor * 1.08` and label confidence as `low`.

### Bug 6 — Active comp weighting too generous when Sold count is thin (MODERATE)

Currently Active comps get weight 0.08 each. With 1 Sold + 5 Actives, the Actives collectively contribute 0.40 weight (29% of total weighted contribution). Anchor gets pulled toward Active asking prices, not actual sales.

In the Redwine case (1 Sold + 5 Actives), the Sold adjusted to $381K but the PMV anchor came in at $374K — pulled DOWN by Active asking prices that aren't the same listings actually selling. In the Medlin case, similar setup, anchor pulled UP by overpriced Actives that have been sitting 100+ days.

Required fix: scale down Active weights when Sold count is thin. Suggested:
- 0 Sold comps → ~~run report~~ block report (insufficient data)
- 1 Sold comp → Active weight = 0.03 (collective 0.15 max for 5 actives)
- 2+ Sold comps → Active weight = 0.08 (current)

OR: only use Actives to inform `dom` and market tempo signals, never let them contribute to PMV calculation. They're the wrong instrument for valuation.

## Test cases for the fixes

Use these production reports as regression tests once fixes are shipped. Each should produce materially different (and better) output:

- `8023175d-37c8-45d7-95da-b9245abe4761` (6022 Candlestick Ln, Lancaster) — feature parity bug, basement-in-GLA bug, outlier asymmetry
- (Redwine and Medlin report IDs not yet captured — pull from `cma_reports` by subject_address before fixing)

After fixes, expected behavior on Candlestick:
- 7026 Wyngate adjusted price drops from $991,500 to ~$925-940K (basement parity, primary-on-main parity)
- 5124 Mill Race weight drops to <0.5 due to far + pool + submarket mismatch
- PMV anchor lands $880K-$960K, NOT $1.17M
- Aspirational/PMV/Event spread tightens accordingly

## Why this matters

If a seller listed at the engine's recommended Event price ($1.063M for Candlestick) and an appraiser ran proper URAR Form 1004 logic, the appraisal would come in at $920-940K. That kills the financing on a sale. We'd be the team that talked the seller into a price the bank wouldn't back. That's a deal we don't want to learn about the hard way.

Until these are fixed, agents should manually review every CMA against the comp set before sending — especially the feature parity and GLA-vs-basement adjustments.

## Recommended ship order

1. Bug 5 first (confidence collapse) — purely a math fix, low risk, unblocks Redwine/Medlin-style reports
2. Bug 1 (feature parity) — biggest dollar impact, core defensibility
3. Bug 2 (GLA/basement separation) — requires schema additions + UI, biggest LOE
4. Bug 6 (active weighting) — small change, high impact on thin comp sets
5. Bug 4 (anchor sanity check) — guardrail, fast win
6. Bug 3 (outlier symmetry) — refinement, ship last

Bugs 1, 2, 5, 6 should land before any CMA goes to a real seller again.
