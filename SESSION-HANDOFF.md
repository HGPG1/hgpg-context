<!-- Last Updated: 2026-05-20 -->

# Session Handoff

## Last session: 2026-05-20 (evening) — NC phone-capture review + 3 fixes shipped 🟢

### TL;DR

Ran the parked New Construction Phone Capture Rate Review. Headline: phone
capture rate was 0/19 (0%) over 5/12-5/20 — but it was a STRUCTURAL zero, not
a UX problem. The /incentives page (sole destination for all Meta ad traffic)
routed to /api/incentives-lead, which used postLightLeadToFub — a helper that
by design cannot save a phone. The four forms that DO support phone capture
(builder/guide/quiz/calculator) got zero submissions in the window. Three
separate bugs found, all three fixed and shipped to charlotte-new-construction-nextjs main.

### What the review found

19 NC leads since 5/12, every one through /incentives, every one tagged
Email-Only Lead. Three bugs:

1. Incentives form had NO phone field at all. IncomingBody in the route had no
   phone key; postLightLeadToFub has no phone param. Email-only by construction.
2. UTM custom fields never written. fubLight stuffed UTMs into the FUB note body
   but never set customUTMSource/Medium/Campaign/Content/Term — all 19 leads have
   those fields null. Attribution dashboards keyed on them were blind.
3. Variant E traffic mis-bucketed. variantTagFromUtmContent switch had no case for
   variant-e-10k (Variant E's utm_content). 6 leads from 5/19-5/20 fell through to
   Variant-Unknown. Variant E day 3-4 read window opened 5/21 with mis-bucketed data.

### What shipped (3 commits, charlotte-new-construction-nextjs, main)

- 18bb2b16 — src/lib/fubLight.ts: added Variant-E-10K mapping (accepts both
  underscore and hyphen utm_content styles), writes UTM custom fields +
  customFacebookClickID to FUB, and accepts an OPTIONAL phone + smsConsent.
  Phone present -> writes phones[] + customSmsConsent + 'Phone Lead' tag.
  Phone absent -> byte-identical to old email-only behavior. Exports
  normalizeLightPhone(). NOTE: this single commit supersedes two earlier
  commits this session (c3285b68, f84e95b6) — same fixes, plus phone.
- 853505c7 — src/app/api/incentives-lead/route.ts: accepts + normalizes
  optional phone, rejects typed-but-invalid phone (blank sails through),
  extracts fbclid from the fbc cookie, passes phone into CAPI user_data
  (better Meta match quality), adds lead_type to Pixel/CAPI custom_data.
- 5b56316d — src/app/incentives/IncentivesClient.tsx: optional phone capture
  on all 3 incentives forms (hero/secondary/card_modal). Option C — field
  labeled just "Phone" with no "(optional)" suffix (the word suppresses
  capture); per-form benefit nudge; SMS consent checkbox revealed only once a
  digit is typed; phone never blocks submit. Consent copy matched VERBATIM to
  the string already live in src/components/LeadCaptureModal.tsx (the other
  4 forms) so A2P language is identical site-wide.

### Open / parked

- Live test on /incentives after Vercel deploy: one email-only submission
  (should look unchanged) + one with a phone (consent box appears, lead lands
  in FUB with phones[] + customSmsConsent: YES + 'Phone Lead' tag).
- DONE: 6 Variant-Unknown leads (5/19-5/20) re-tagged to Variant-E-10K. All 6
  verified via note body as genuine variant-e-10k traffic (no organic/direct
  contamination). UTM custom fields backfilled on all 6 too
  (source/medium/campaign/content/term). FUB IDs: 32106, 32121, 32122, 32124,
  32129, 32130. Tomorrow's Variant E day 3-4 read is now fully clean.
- Variant E day 3-4 read (window opened 5/21): any lead AFTER the deploy is
  cleanly bucketed; leads before are not.
- Phone capture rate should now be re-measured ~7 days post-deploy to see if
  the Option C nudge pulls weight (the original brief's actual question).

### Brain / project-prompt hygiene fix (same session)

- Root-caused the recurring "stale brain" false alarm: it was web_fetch's
  internal cache serving weeks-old content, NOT raw.githubusercontent.com's
  CDN (which is only ~5 min). operations.md updated (commit 6bde9667) with an
  explicit "read brain/repo files via bash+curl, never web_fetch" rule. The
  HGPG Tech & Builds project prompt was also rewritten to match — Brian pasted
  the new version into project settings.

### Pickup notes for next session

- JWT rotation (projects/legacy-jwt-rotation.md) is the priority — planned to
  attack 5/21. Nothing executed yet. Still the only thing genuinely "on fire."
- For the phone-capture re-measure, the query is the same as
  projects/new-construction-phone-capture-rate-review.md but now there should
  be non-zero phone counts and Variant-E-10K should appear as a real bucket.

---

## Previous session: 2026-05-20 (latest) — /api/meta-insights leads inflation fixed (4x bug) 🟢

### TL;DR

Ads-side correctly identified the closings.homegrownpropertygroup.com `/api/meta-insights` endpoint as the source of the SG-2026 "4 leads" inflation. Their direct Meta Graph API audit returned `lead=1, fb_pixel_lead=1, onsite_web_lead=1` for CONV ad set 52506271965563 — three aliases reporting the SAME single submission. Endpoint's `sumLeadActions()` was doing substring match `t.includes("lead")` which summed all aliases into one inflated leads count. Fixed in commit `79f89c7`, deployed to main, awaiting Vercel auto-deploy + 60s cache TTL bust.

**Every CPL and lead count from earlier today (incl. headline findings on Variant E, Variant C, Sellers Guide CPL etc.) came from this endpoint and is corrupted by roughly 3-4x.** Relative ordering between variants probably survives (inflation is roughly uniform). Absolute thresholds in the Variant E day 5-7 decision tree DO NOT survive and need re-derivation from post-fix endpoint reads.

### The bug

File: `hgpg-transaction-manager/app/api/meta-insights/route.ts`

Old `sumLeadActions()`:

    if (t.includes("lead") || t === "complete_registration" || t === "submit_application") {
      total += Number(a?.value) || 0;
    }

`t.includes("lead")` matches `lead`, `fb_pixel_lead`, `onsite_web_lead`, `offsite_conversion.fb_pixel_lead`, `leadgen.other`. Meta returns the same event under 3-4 aliases. They all get summed. Daniela = 1 real submission = 4 reported leads.

New version (committed):

    if (t === "lead") {
      total += Number(a?.value) || 0;
    }

`complete_registration` and `submit_application` also removed from the fold — those are separate Meta standard events, not lead aliases.

### What this fix touches and what it doesn't

- AFFECTED: `leads` count, `cpl` (= spend / leads). Both were inflated 3-4x.
- NOT AFFECTED: `spend`, `impressions`, `clicks`, `ctr`, `cpc`. Pulled as scalars directly from Meta row, never went through sumLeadActions.

### Validation Ads needs to run post-deploy

Wait 90s after commit `79f89c7` for Vercel deploy. Endpoint has 60s response cache so the first hit after deploy may still serve stale; either wait 60s more or change the `days=` value to bust the cache key.

    curl -H "Authorization: Bearer $META_INSIGHTS_TOKEN" \
      "https://closings.homegrownpropertygroup.com/api/meta-insights?days=7&level=adset&parent_id=52506270109163&parent_kind=campaign"

Expected after fix: `totals.leads = 1`, `totals.cpl = ~83.42`, matching Meta Graph API direct read.

### Corrected approximate CPLs (will re-baseline against endpoint post-deploy)

Using rough 4x correction factor (will refine after re-pull):

- Account 7d: $608.87 / ~28 / ~$22 CPL (was reported $5.54)
- Variant E 9:16: $84.17 / ~9 / ~$9 CPL (was reported $2.27)
- Variant C: $240.60 / ~14 / ~$17 CPL (was reported $4.22)
- Variant D: $125.66 / ~3 / ~$42 CPL (was reported $10.47)
- Sellers Guide CONV: $83.42 / 1 / $83.42 CPL (was reported $20.86)

Note that absolute CPL values shift dramatically. Variant E still outperforms Variant C, Variant D still worst. Sellers Guide CONV no longer beats its $35 target — it's well above at $83/lead with 1 lead in 7d.

### Other unrelated SG-2026 facts confirmed earlier today (still valid)

- Pixel `Lead` event fires only inside `btn-submit-lead` handler's `if (result.ok)` branch. Audited live HTML across all 7 sellersguide.* pages. Form-submit only. Quiz events fire as trackCustom, do not count as Meta Lead.
- Hostname gate live (`location.hostname === 'sellersguide.homegrownpropertygroup.com'`). Confirmed on `/api/meta/capi` server side and across all browser pixel emissions.
- FUB `sellers-guide-2026` tag: exactly 1 person (Daniela Portillo, FUB id 32123). Confirmed matches Meta Graph API's actual single lead.
- `seller_assessments` rows 2026-05-13 → 2026-05-20: 3 rows (Daniela + 2 Brian tests with `utm_campaign=test`/`test2` that don't attribute to the CONV campaign).
- SG-2026 Nurture templates 1158-1163 exist and are shared. Wiring status via FUB API: 1158/1159/1160/1161/1163 have `automations: []`. 1162 and 1164 wired only to Automation 332 (Cold Nurture, complete). Confirms Send Email steps in the nurture shell are unwired — React Save bug per Ads handoff is real.
- FUB API blocks `/v1/automations` with 403. The Send Email step rebuild cannot be done from API. Must be done in FUB UI by Brian or Viktor. Use one-step-save-refresh workaround for the React Save bug.

### Open / parked

- Ads-side re-pulls endpoint at 7d/14d/30d after deploy confirms, re-baselines CPL across all active campaigns
- SG-2026 Nurture rebuild in FUB UI (Brian/Viktor) — unblocked, build spec below
- IDXRE-B2 sellers + buyers automation activation — gated on (1) nurture rebuild + (2) endpoint re-baseline confirmation

### SG-2026 Nurture build spec (for FUB UI rebuild)

    Trigger: tag added > sellers-guide-2026
      Day 0:  Email -> Template 1158 (Guide Delivery)
      Day 2:  Condition: has tag home-selling-score?
                Yes -> Email Template 1159 (Tier Means)
                No  -> Email Template 1160 (Didn't Finish)
      Day 5:  Email Template 1161 (One Thing Most Sellers Get Wrong)
      Day 10: Email Template 1162 (Local Market Snapshot)
      Day 17: Task (review + personal outreach) + Email Template 1163 (Live Walkthrough CTA)
      Day 45: Condition: Agent = Brian?
                Yes -> Email Template 1164 (Monthly Check-In)
    Run every time: ON
    Auto-pause on reply: ON

React Save workaround: add ONE step at a time, save the automation, refresh the page, then add the next step. Saving multiple Send Email steps in a single edit pass has been failing per Ads handoff.

### Pickup notes for next session

- Post-deploy endpoint validation is the very first thing to run
- Re-baseline all campaign CPLs before any further variant scale/pause decisions
- Endpoint `sumLeadActions` change is permanent — if a future session widens the match again (e.g. "leads look low, let me add fb_pixel_lead"), they'll be reintroducing this bug. The new function has a header comment explaining why; respect it.

---

## Prior session: 2026-05-20 (later) — SG-2026 attribution gap investigated, no gap found 🟢

### TL;DR for Ads-side pickup

The "Meta reports 4 leads vs FUB 1" attribution gap **does not exist in the underlying data**. Pulled Meta Insights API directly for CONV ad set 52506271965563 at 7d, 14d, and 30d windows — all return exactly **1 lead** (`offsite_conversion.fb_pixel_lead: 1`, `lead: 1`, `onsite_web_lead: 1`). Matches Daniela Portillo (FUB id 32123) exactly. Pixel wiring is correct. The "4 leads" figure originated somewhere upstream of the API and needs to be re-sourced before any pixel/webhook work happens.

### ✅ RESOLVED — Lead count discrepancy was an endpoint bug, now fixed (added 2026-05-20 late evening)

Tech-side root-caused and shipped fix in commit 79f89c7 on hgpg-transaction-manager main, file `app/api/meta-insights/route.ts`. Function `sumLeadActions()` was doing substring match `t.includes("lead")` which summed every Meta action_type alias for the same single event (`lead`, `fb_pixel_lead`, `onsite_web_lead`, `offsite_conversion.fb_pixel_lead`). One real submission was being counted 3-4 times. Fix: hard match `action_type === "lead"` only. `complete_registration` and `submit_application` also removed (separate Meta standard events, not lead aliases).

Affected: `leads` and `cpl` (~4x inflated and ~4x understated respectively).
NOT affected: `spend`, `impressions`, `clicks`, `ctr`, `cpc` — pulled as scalars.

Validation pass at 7d adset level for SG CONV ad set 52506271965563:
- Endpoint now returns leads=1, cpl=$83.48 — matches Meta Graph API direct exactly.

**POST-FIX BASELINE (re-pulled 2026-05-20 late evening):**

Account totals:
- 7d:  $620.87 / 26 leads / **$23.88 CPL** (was reported $5.54, 4.3x understated)
- 14d: $868.43 / 38 leads / **$22.85 CPL** (was reported $5.29)
- 30d: $1210.83 / 57 leads / **$21.24 CPL** (was reported $4.74)

New Construction 7d ad level (sample sizes are small now):
- Variant C (LocalPride, ACTIVE): $240.63 / 13 leads / $18.51 CPL / 2.94% CTR
- Variant D (Incentives, PAUSED): $125.73 / 3 leads / $41.91 CPL / 3.26% CTR — worst, stay paused
- Variant E - 9:16 vertical (ACTIVE): $92.85 / 9 leads / **$10.32 CPL** / 3.11% CTR — still the best, but not the "$2.27 breakout" we celebrated
- Variant E - 1:1 + 4:5 (PAUSED): $5.58 total, 0 delivery, no signal

30d lifetime variant context (best CPL ranking):
- E vertical: $10.32 CPL (newest, smallest sample, only ~3d in market)
- C: $13.49 lifetime CPL — but degrading: 7d window is $18.51, lifetime is $13.49, fatigue starting
- A (paused): $16.16
- B (paused): $17.82
- D (paused): $27.64

Sellers Guide CONV: $83.48 spend, 1 lead, $83.48 CPL — **2.4x over the $35 target**, not under it. The "Daniela = Meta = FUB" alignment is now consistent across all three sources. No attribution gap exists. Pixel and webhook clean per tech-side audit.

**Story holds, story changes:**
- HOLDS: variant ordering (E vertical > C > everything paused). E vertical still the strongest variant, C still degrading.
- HOLDS: pause D was right.
- CHANGES: Sellers Guide CONV is NOT beating its target — it's at $83 CPL, ~2.4x over. Single lead in 7 days. Either the lander needs work or the ad creative does or the audience is wrong, but the previous "we're crushing it" framing was bug-driven.
- CHANGES: Account CPL is ~$22, not ~$5. That's still respectable for lead-gen real estate but the bar is different.

**Recomputed Variant E day 5-7 decision tree (against true CPLs):**

E vertical needs to clear two bars at the day 5-7 read (~May 23-25) for "new control" status:
1. Volume bar: must hold sub-$15 Meta CPL across ≥$250 cumulative spend AND ≥20 leads. (Was sub-$3 / 100 leads against bug numbers.) If CPL drifts past $15, E is still a B-variant but not the new control.
2. Stability bar: CTR must stay ≥3.0% (currently 3.11%). Below 3.0% with rising CPL means creative fatigue is starting on a fresh ad.

Outcomes:
- Both bars cleared → E vertical is the new control, demote C to support, build E square/portrait variants that actually deliver (current ones got no impressions before pause)
- Volume bar only → E is a strong B-variant, keep both at parity
- Neither bar → E was a 2-day fluke at small sample, revert to C, retire E

---

### What got verified (tech side, read-only)

**Pixel firing point (2a):** Pulled the live served HTML at `https://sellersguide.homegrownpropertygroup.com/home-selling-score/`. Meta standard `Lead` event fires exactly once — inside the `btn-submit-lead` async handler, inside the `if (result.ok)` branch, after `submitAssessmentAndLead(v.values)` succeeds (which requires BOTH Supabase write AND FUB forward to succeed). All other events on the page (`AssessmentStarted`, `ScoreCompleted`, `QuizStarted`, `QuizCompleted`) fire as `trackCustom` and do NOT count as Meta Leads. Hostname gate `HGPG_PIXEL_ENABLED = (location.hostname === 'sellersguide.homegrownpropertygroup.com')` confirmed live. Audited all 7 pages — Lead event exists only on `/home-selling-score/`.

**Meta Insights API direct read (2b):** Campaign 52506270109163 (HGPG-SellersGuide-CONV-2026-Q2), ad set 52506271965563 (CONV-HSS-NCSC-Border):
- 7d: 6 ads, total spend $81.54, impressions 2362, clicks 46, landing_page_view 20, **lead 1**, fb_pixel_lead 1, onsite_web_lead 1
- 14d: identical to 7d (campaign launched 5/15)
- 30d: identical to 7d
- Top-spend ad: HGPG-SG-C-4x5 ($60.69, 1626 impr, 33 clicks)

**FUB database read:**
- `seller_assessments` rows 2026-05-13 → 2026-05-20: exactly 3 rows
  - 2026-05-20 03:38 UTC — Daniela Portillo (FUB id 32123, real, `utm_content=C_5categories_4x5`, IP 174.111.145.68)
  - 2026-05-15 19:36 UTC — "test mctest" (FUB id 32043, your test, `utm_campaign=test2`, IP 67.197.10.27)
  - 2026-05-15 19:27 UTC — "Test user" (FUB id 31872, your test, `utm_campaign=test`, IP 67.197.10.27)
- `seller_assessments_v2_summary` shows same 3 rows. No hidden submissions.
- `sellers-guide-2026` tag query via FUB MCP: exactly 1 person (Daniela). The 2 tests used `utm_campaign=test`/`test2` not `sellersguide_2026q2`, so they wouldn't attribute to the CONV ad set in Meta's attribution windows.

**Conclusion:** Meta API = 1, FUB = 1, Supabase = 1 (plus 2 tests excluded by campaign attribution). No gap.

### Where the "4 leads" figure likely came from

Best guesses, in order of likelihood:
1. Ads Manager UI misread — wrong column (Results vs Leads vs Landing Page Views), wrong attribution window, or wrong ad set filter
2. Conflating `landing_page_view: 20` with leads at a glance
3. Stale Meta MCP cache from before the closings.* `/api/meta-insights` endpoint fix
4. Reading from a different account or campaign by mistake

Vercel runtime logs are gone for the 5/13-5/20 window (no Log Drain configured, retention <24h), but the Meta Insights API is canonical and it says 1. No webhook delivery investigation needed.

### Status of templates and automations (read-only via FUB MCP)

| Template | Name | Wired to |
|---|---|---|
| 1158 | SG-2026 - Guide Delivery | (none) |
| 1159 | SG-2026 - What Your Tier Actually Means | (none) |
| 1160 | SG-2026 - Didn't Finish Your Score? | (none) |
| 1161 | SG-2026 - The One Thing Most Sellers Get Wrong | (none) |
| 1162 | SG-2026 - Local Market Snapshot | Automation 332 (Cold Nurture) |
| 1163 | SG-2026 - Want a Live Walkthrough? | (none) |
| 1164 | SG-2026 - Monthly Check-In | Automation 332 (Cold Nurture) |

Confirms the React Save bug: Nurture shell has zero Send Email steps pointed at any of the 7 templates. Cold Nurture (332) is intact. FUB API blocks `/v1/automations` with 403 so the Nurture rebuild MUST happen in the FUB UI — cannot be done programmatically.

### Rebuild spec for FUB UI (one-step-save-refresh to dodge React bug)

Sequence (also delivered to Brian inline this session):

    Trigger: tag added > sellers-guide-2026
    Day 0:  Email -> Template 1158
    Day 2:  Condition: has tag home-selling-score?
              YES -> Email Template 1159
              NO  -> Email Template 1160
    Day 5:  Email Template 1161
    Day 10: Email Template 1162
    Day 17: Task (review + personal outreach) + Email Template 1163
    Day 45: Condition: Assigned Agent = Brian?
              YES -> Email Template 1164
    Run every time: ON
    Auto-pause on reply: ON

### Parked

- SG-2026 Nurture UI rebuild (Brian/Viktor) — spec above, one-step-save-refresh
- Validate rebuild with a fresh test contact (NOT Daniela) tagged `sellers-guide-2026`, watch Day 0 send
- Ads-side: re-source the "4 leads" figure or accept Meta API count of 1
- After (1) and re-source land: activate IDXRE-B2 sellers + buyers in parallel
- Day 5-7 Variant C vs E CPL re-read (~May 23-25)
- Pipeboard Pro renewal decision before ~May 28

### Pickup notes for next session

**For Ads-side Claude rechecking the gap:**
- The Insights API truth is `act_31445287` / campaign `52506270109163` / ad set `52506271965563`
- Pull via Pipeboard MCP: `bulk_get_insights(level="adset", account_ids=["act_31445287"], campaign_ids=["52506270109163"], time_range="last_7d", fields=["spend","impressions","clicks","actions"])`
- Or hit `closings.homegrownpropertygroup.com/api/meta-insights?days=7&level=adset&parent_id=52506270109163&parent_kind=campaign` with `META_INSIGHTS_TOKEN`
- Expected: `lead: 1`, `fb_pixel_lead: 1`. If you see 4, the source you're reading is wrong.

**For Tech-side next session:**
- If Brian comes back with confirmation of where "4 leads" came from and it points to a real ingestion issue (not a misread), revisit. Otherwise the pixel/webhook chain is verified clean.
- Daniela Portillo (32123) enrollment status in rebuilt Nurture is the real validator. Watch her activity feed once Brian/Viktor rebuild.

---

## Earlier session: 2026-05-20 — Meta insights endpoint fixed, Variant E breakout, FUB attribution gap surfaced 🟢🟡



### ⚠️ Lead count discrepancy — closings endpoint suspected unreliable (added 2026-05-20 evening)

Tech-side audited the SG-2026 attribution gap and found Meta Insights API direct pull returns lead=1, fb_pixel_lead=1, onsite_web_lead=1 for CONV ad set 52506271965563 across 7d/14d/30d — matching Daniela Portillo exactly and matching FUB and Supabase. **No gap exists between Meta and FUB.**

The closings.homegrownpropertygroup.com `/api/meta-insights` endpoint returns `leads: 4` at every level (campaign/adset/ad) for the same window. The endpoint disagrees with the underlying Meta API by ~4x.

**Likely cause:** the endpoint is summing or unioning multiple Meta action types (`lead` + `fb_pixel_lead` + `onsite_web_lead` and possibly `submit_application_total`) under one "leads" field instead of deduping to the canonical `lead` action. Tech-side owns the endpoint source and can confirm in 30 seconds.

**Implication for ALL numbers in this session:** every CPL and lead count below came from the same endpoint and is suspect. Relative variant ordering (C > D, E vertical > C) probably still holds because the inflation likely applies uniformly. But absolute CPL figures are not trustworthy.

**Held until tech-side confirms endpoint fix:**
- Variant E day 5-7 decision tree (the $3/$5 CPL thresholds were set against inflated numbers)
- Sellers Guide activation gating (the "Meta says 4, FUB says 1" framing was wrong)
- Any further variant pause/scale decisions

Pixel firing point and webhook delivery are NOT the problem. Tech-side verified pixel fires form-submit only inside `if(result.ok)`, hostname gate live, all quiz/score events are trackCustom and don't count as Meta Leads. The IDXRE-B2 nurture sequence stays blocked on the FUB UI rebuild only (tech-side has the build spec, FUB API blocks /v1/automations with 403).

---

### What got verified
- `/api/meta-insights` on closings.homegrownpropertygroup.com is returning correctly-windowed data. 7d/14d/30d all distinct, no lifetime-cache pollution. Bug fixed, 60s TTL holding.
- Bearer token `META_INSIGHTS_TOKEN` works end-to-end. Stored in HGPG - Ads project instructions.
- Pipeboard Pro trial active through ~2026-05-28. Decision pending after this read.

### Headline findings
**Account totals (pre-D-pause snapshot, locked for tomorrow's delta read):**
- 7d:  $608.87 spend / 110 leads / $5.54 CPL
- 14d: $856.43 spend / 162 leads / $5.29 CPL
- 30d: $1198.83 spend / 253 leads / $4.74 CPL
- CPL drifting up wk/wk pre-pause. Variant D was the drag.

**Variant E (10K hook) is the breakout, day 2:**
- E - 9:16 vertical: $84.17 spend, 37 leads, $2.27 CPL, 3.28% CTR
- E - 1:1 square: $1.60 spend, no delivery
- E - 4:5 portrait: $3.98 spend, no delivery
- Meta auctioned budget into vertical placement. Square + portrait got no delivery and were left running (Meta isn't burning budget on them).
- E vertical beats Variant C (control workhorse) by 1.86x on CPL with only ~18% of total spend.

**Variant C (LocalPride):** $240.60 / 57 leads / $4.22 CPL. Still the workhorse.
**Variant D (Incentives):** $125.66 / 12 leads / $10.47 CPL. PAUSED 2026-05-20 PM.

**Sellers Guide — Meta vs FUB attribution gap surfaced:**
- Meta reports: CONV ad set $83.42 / 4 leads / $20.86 CPL (under $35 target)
- FUB actual: 1 real lead tagged sellers-guide-2026 (Daniela Portillo id 32123, in this morning 03:38 UTC)
- 1 test record (id 32043 Test Mctest) was neutralized to Trash with `DELETE-ME-test-data` tag — Brian to sweep in UI
- True CPL is closer to $83 (1 real lead / $83 spend) than $20.86
- Cause is unknown: pixel firing on quiz-start instead of form-submit, OR webhook dropping leads, OR Meta double-counting attribution. Routed to Tech & Builds for diagnosis.

### Actions taken this session
- Pulled clean 7d/14d/30d campaign-level + ad-level data from fixed endpoint
- Verified FUB sellers-guide-2026 tag lead flow: 2 records, 1 real, 1 test
- Neutralized FUB test record id 32043 (stage=Trash, tags stripped to DELETE-ME-test-data)
- Brian paused Variant D ad (HGPG_New_Con_VariantD_LocalPride_2026-05-11, id 52503410311963)
- Drafted Tech & Builds handoff covering (a) SG-2026 Sellers Guide Nurture automation rebuild and (b) pixel/webhook attribution diagnosis — Brian shipped to T&B

### Blocked / pending
- **SG-2026 Sellers Guide Nurture automation — Send Email steps did not persist (FUB React Save bug).** Templates 1158-1165 are all created and shared. Automation shell exists but is unusable. Sequence to rebuild lives in T&B handoff. **Do NOT activate IDXRE-B2 sellers automation or the buyers automation until this is rebuilt AND pixel/webhook gap is diagnosed.**
- Cold Nurture (Auto #332) is complete and unaffected.
- Stoppers (Unsubscribed, Y_DNC_REGISTRY_TRUE) confirmed working.
- Pixel event trigger needs verification (quiz-start vs form-submit) — T&B owns
- Webhook delivery logs needed for 2026-05-13 through 2026-05-20 — T&B owns

### Day 5-7 decision tree for Variant E (~May 23-25)

E vertical needs to clear two bars at the day 5-7 read for "new control" status:

1. **Volume bar:** must hold sub-$3 Meta CPL across at least 3x current spend (≥$250 cumulative) AND ≥100 leads. If CPL drifts to $3-5 at scale, E is still a strong B-variant but not the new control.
2. **Stability bar:** CTR must stay ≥3.0% (currently 3.28%). Below 3.0% with rising CPL means creative fatigue is already starting on a fresh ad.

Outcomes:
- Both bars cleared → E vertical is the new control, demote C to support, build E variants for square/portrait that actually get delivery (the current ones don't)
- Volume bar only → E is a strong B-variant, keep both at parity in the rotation
- Neither bar → E was a 2-day fluke, revert to C as control, retire E

### Recommendations sent to Brian today
1. ✅ Pause Variant D — DONE
2. ✅ Pause E square (52509166115763) and E portrait (52509170611363) — DONE 2026-05-20 PM. Ad set is now C + E vertical only.
3. ⛔ Do NOT switch Sellers Guide destination URL — CONV is hitting Meta target; FUB gap is upstream attribution, not the lander
4. ⛔ Do NOT activate FUB sellers automation yet — blocked on T&B rebuild + attribution diagnosis
5. Day 5-7 re-read scheduled for ~May 23-25 against the decision tree above

### Pickup notes for next session
- **Tomorrow (2026-05-21) midday pull:** compare against today's account totals snapshot (above) to measure D-pause impact. Specifically watch (a) does E vertical absorb D's freed spend, (b) does account CPL drop back toward $4.74, (c) does C's lead count stay flat or grow.
- Endpoint pattern: `curl -H "Authorization: Bearer $META_INSIGHTS_TOKEN" "https://closings.homegrownpropertygroup.com/api/meta-insights?days=7&level=campaign"` — also takes `level=adset&parent_id=<cid>&parent_kind=campaign` and `level=ad&parent_id=<cid>&parent_kind=campaign`
- Token lives in HGPG - Ads project instructions
- Meta MCP rollout flag still off on all three ad accounts — endpoint is the workaround that actually works now
- Variant E ad IDs for next pull: 52509166115763 (1x1), 52509170611363 (4x5), 52509170629763 (9x16 — the breakout)
- One real Sellers Guide lead in FUB: Daniela Portillo (id 32123). She's the canary for the rebuilt nurture sequence when T&B unblocks it.
- Pipeboard Pro renewal decision before ~May 28. Recommendation: keep it. The fix paid for itself in one session.

### Parked
- Day 5-7 Variant C vs E CPL re-read (~May 23-25) — decision tree above
- T&B: SG-2026 Sellers Guide Nurture automation rebuild
- T&B: pixel event trigger diagnosis + webhook delivery log pull
- FUB IDXRE-B2 sellers and buyers automation activation (BLOCKED on T&B)
- Pipeboard Pro renewal decision (~May 28)
- Geo-farming postcards blocked on Canopy MLS data (May 22 upload deadline — separate workstream)
- Brian to sweep FUB UI for `DELETE-ME-test-data` tagged records when time permits
- Ad set state as of session close: C ACTIVE ($4.22 CPL), E vertical ACTIVE ($2.44 CPL), D + E square + E portrait all PAUSED

---

## Even earlier session: 2026-05-06 — Brain App MVP shipped 🟢

### What got built
- New Vercel project: `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- New repo: `HGPG1/brain-app` (private)
- Live at: `https://brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Single-user lock: `BRIAN_EMAIL=brian@homegrownpropertygroup.com` allow-list
- GitHub auth: fine-grained PAT scoped to `HGPG1/hgpg-context`, contents:write only
- Round-trip verified: edit file in browser → commit lands on `main` with author `brian@homegrownpropertygroup.com`

### Infra changes that affect other apps
- Resend custom SMTP wired into `HGPG Core` Supabase (project `ioypqogunwsoucgsnmla`)
  - Sender: `noreply@homegrownpropertygroup.com`, name: HGPG
  - API key stored under "Supabase HGPG Core" in Resend
  - Rate limit went from 2/hr (Supabase default) to 30/hr (Resend default), can be raised
  - This affects ALL apps using this Supabase: TM, CMA, TC Concierge, brain-app
- Supabase project renames for hygiene:
  - `ioypqogunwsoucgsnmla` → "HGPG Core"
  - `ngdrliyjtqcwhhfrbxao` → "HGPG FUB Integration" (verify)
  - `wdheejgmrqzqxvgjvfee` → "HGPG Listing Reports + MLS" (verify)
  - `fkxgdqfnowskflgbuxhm` → "HGPG Signature + Relocation" (verify)
- Supabase `HGPG Core` redirect URLs added:
  - `https://brain.homegrownpropertygroup.com/**`
  - `http://localhost:3000/**`
  - (Existing tools.hgpg entries left intact)

### Bugs found and fixed mid-session
- Magic link redirected to `tools.homegrownpropertygroup.com` (Supabase Site URL fallback) — fixed by adding `/auth/callback` route handler that was missing from initial scaffold + pointing `emailRedirectTo` at it
- Supabase free SMTP rate limit (2/hr) hit during testing — fixed permanently by switching to Resend custom SMTP

### Project status updates
- `projects/brain-app.md` — status now 🟢 SHIPPED (was 🟡)
- `projects/hgpg-team-tools2.md` — Site URL in Supabase still points here for the broken app's eventual fix
- `projects/transaction-manager.md` — no changes today, but TM benefits from Resend SMTP upgrade

### Deferred / Phase 2 for brain-app
- iPhone smoke test (CodeMirror + iOS soft keyboard scroll behavior)
- Cooper Hewitt self-hosted (currently falling back to system sans)
- File rename and delete
- Diff view before save
- Cross-file search

### Pickup notes for next session
- Brain-app is live and working — use it for any future updates to `hgpg-context`
- Resend API key is in 1Password ("Supabase HGPG Core SMTP")
- Brain-app local dev: `cd ~/brain-app && npm run dev` on Mac mini (work machine)
- Brain-app local on iMac: same setup, repo at `~/Developer/brain-app` if rebuilt, otherwise needs fresh `gh repo clone HGPG1/brain-app` + `npm install` + `cp env.example .env.local`
- The `package-lock.json` may differ between iMac and Mac mini — push from whichever machine you most recently ran `npm install` on




