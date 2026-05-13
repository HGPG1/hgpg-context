<!-- Last Updated: 2026-05-13 -->

# New Construction - Phone Capture Rate Review (Parked Follow-up)

**Status:** 📋 PARKED - run between 2026-05-19 and 2026-05-26
**Driver:** Phone capture shipped 2026-05-12, SMS speed-to-lead shipped 2026-05-13. After 7-14 days of post-ship data, evaluate whether the optional-phone nudges are working hard enough or need iteration.

## Why this matters

Today's work (SMS speed-to-lead) capitalizes on each captured phone but doesn't change the capture rate. The phone-capture PR from 2026-05-12 is what changes the rate. We need real numbers to know if the optional-phone nudges (text guide link, text matches, text breakdown) are pulling weight or just decoration.

If Brian is also driving paid Meta ads at this site, capture rate is directly tied to ROAS - every uncaptured phone is wasted ad spend.

## What to query

Pull from FUB (custom approach since FUB Lead Flow can't filter custom fields - use the People API directly).

### Step 1: Get all new construction contacts since 2026-05-12

```
curl -s -u "fka_0cyqBULqHHKqrZiEJH6Njl7jqDkBBNTUYJ:" \
  "https://api.followupboss.com/v1/people?limit=100&sort=created&order=desc" \
  | python3 -m json.tool > /tmp/fub_recent_leads.json
```

May need pagination if more than 100 leads. Check the `_metadata` for total count.

### Step 2: Calculate rate by source

For each contact created since 2026-05-12, categorize by:
- **Source** (`Website - New Construction` = Builder Intro, `Website - New Construction Guide Download` = Guide, etc. - one source per form)
- **Has phone?** (truthy `phones` array)
- **SMS consent?** (`customSmsConsent` custom field = YES)

Then compute:
- Phone capture rate per form (% of contacts with phone, by source)
- SMS consent rate per form (% of phone-having contacts that checked consent)
- Builder Intro absolute volume (since it's required, this is the volume floor for guaranteed-phone leads)

### Step 3: Benchmarks to evaluate against

| Form | Expected capture rate | If below | If above |
|---|---|---|---|
| Builder Intro | 100% (required) | Bug in code or form | Working as designed |
| Guide Delivery | 20-40% (optional with nudge) | Test new nudge copy or remove field entirely | Nudge is working |
| Quiz Complete | 25-45% (warm intent) | Same as Guide | Working |
| Calculator | 30-50% (high intent - they did math) | Same as Guide | Working |

Anything materially below the low end on optional forms means the benefit-led nudges aren't pulling weight. Anything materially above the high end is bonus.

## What to do based on the data

**If optional capture rates are healthy (≥ benchmark low end):**
- Document the rates as a baseline in the brain
- Move on. Next iteration is when ad-spend changes the funnel mix.

**If optional capture rates are below benchmark:**
- A/B test alternative nudge copy. Three candidates worth testing:
  1. Different value prop: "Text me my matches" (active voice) vs "We'll text you matches" (passive - current)
  2. Different verb: "Send" vs "Text" - "Send" feels less spammy
  3. Drop "(optional)" from label - some studies show this reduces capture because users read it as "don't bother"
- Test one variable at a time. Don't ship 3 changes simultaneously and lose attribution.

**If Builder Intro capture is anything other than 100%:**
- Bug. Either the validation isn't enforcing required, or contacts are being created without phones somehow (impossible per current code, but worth checking the actual data).

**If SMS consent rate on phone-having contacts is very low (under 50%):**
- The consent checkbox is the bottleneck, not the phone field. Means people are typing a phone but balking at the SMS commitment. Worth softening checkbox copy or considering uncheck-by-default vs check-by-default.

## Cross-reference with Meta ad data

If Brian's running Meta ads at this site in the parked-launch state:
- Pull Meta Ads Manager → Lead conversions for the campaign
- Compare to FUB count - confirm Meta CAPI deduplication is working (counts should be close, not 2x)
- Look at CPM and CTR by ad creative if the funnel mix shifts based on which creatives are landing
- The `ph` parameter being added to Lead events (shipped 2026-05-12) should improve event match quality - confirm in Meta Events Manager → Diagnostics

## What's parked, what's not

**This review is parked:** decision-making depends on having data, and the data needs 7-14 days minimum.

**NOT parked (still owed):**
- Live end-to-end test of SMS speed-to-lead (Brian had to jet before running it 2026-05-13)
- Confirming Stage 1 + Stage 2 work end-to-end with a real submission

## References

- Phone capture project: `projects/new-construction-phone-capture.md`
- SMS speed-to-lead project: `projects/new-construction-sms-speed-to-lead.md`
- Conversion of `customSmsConsent` field happened 2026-05-13 (verify API name: `customSmsConsent`, label `Sms Consent`)
- Meta Pixel ID for this site: `1880396459290092`
