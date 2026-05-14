<!-- Last Updated: 2026-05-14 -->

# Charlotte New Construction

- **Status:** 🟡 Live. Ads were PAUSED 2026-05-11 for lead-quality fix; Variants A and B killed, Variant D built 2026-05-13 with corrected tagging fixed 2026-05-13. NeverBounce email validation also live. Ad spend resumption pending one open item: Automation 2.0 enrollment verification for Lead Flow 54. Pixel/CAPI plumbing is load-bearing for ad spend.
- **URL:** newconstruction.homegrownpropertygroup.com
- **Repo:** HGPG1/charlotte-new-construction-nextjs
- **Default branch:** `main`
- **Stack:** Next.js (server-rendered), Vercel deployment
- **Vercel team:** `team_FietQPKCmnyioG2n0FdteQCV`

## Watch item

Periodic Pixel/CAPI health checks in Meta Events Manager warranted, especially after any deploy. Silent break = wasted ad budget + bad attribution. Check every couple weeks.

## Purpose

Lead-gen funnel for buyers interested in new-construction homes in the Charlotte metro area. Drives Meta paid traffic to landing pages, captures leads with progressive disclosure, syncs to FUB.

## Routes shipped

| Route | Purpose | Lead capture |
|---|---|---|
| `/` | Main landing | Yes |
| `/incentives` | Builder incentive comparison + lead form | Yes (primary funnel) |
| `/how-we-work` | Service explainer | Yes |
| `/quiz` | Buyer match quiz | Yes |

All 4 routes have FUB lead capture wired.

## Meta Pixel + CAPI

- Pixel ID: `1880396459290092`
- CAPI live with browser+server dedup
- All 6 funnel events firing (PageView, ViewContent, Lead, etc.)
- Pattern documented in `META-PIXEL-CAPI-PLAYBOOK.md` in repo root, reusable for other sites; each site needs its own Pixel ID

## FUB integration

Lead form on every route POSTs to a server-side route handler that pushes to FUB with:
- Source: `Meta — New Construction Incentives` (sourceId 113, Lead Flow ID 54)
- UTM custom fields populated when present in URL
- Lead routing per FUB Automations 2.0 rules (NOT Action Plans, those are deprecated)
- FUB email template IDs: Guide Delivery=1147, Guide Day 1=1148, Calculator=1149, Quiz=1150, Builder Intro=1151

## 2026-05-11 Lead quality audit (Apr 11 - May 10 ads cycle)

**Headline finding: ~94% of leads with known location are out of market.**

Spent ~$318 across 1 campaign / 1 ad set / 3 active variants. Meta reports 26 leads at $12 blended CPL. Pulled all 16 FUB-side leads via MCP and ran a geo audit using FUB social data enrichment:

| Location signal | Count |
|---|---|
| Charlotte metro (in market) | 1 (Tammy Flores, FUB ID 31833) |
| Salisbury NC (marginal, ~1hr north) | 1 (Marsha) |
| Out of market (NJ, VA, NY, PA, TX, CA) | 7 |
| No social data available | 7 |

Real in-market CPL = $318 / 1 confirmed = ~$318, not $12.

### Variant performance (Meta side)

| Variant | Spend | Leads | CPL | CTR | LP-to-Lead | Rankings |
|---|---|---|---|---|---|---|
| A "Dollar" ($50K off only) | $223 | 14 | $15.93 | 1.73% | 15.6% | - |
| B "Rate" (low rate only) | $53 | 3 | $17.82 | 2.36% | 6.1% | - |
| C "Choice" ($50K OR rate) | $42 | 9 | $4.63 | 2.60% | 33.3% | Above avg / Above avg |

**Variant C wins decisively on every metric.** Choice architecture is doing the work, the "either/or" framing self-selects engaged buyers who fit one of the two financial profiles. A and B are static offers that don't qualify.

Ad set flagged "Learning Limited", explained by total volume being below Meta's 50-conversions-per-week threshold, not a quality complaint.

### Root cause of out-of-market traffic

Campaign is correctly flagged as **Special Ad Category: Housing**, which strips standard targeting levers (age, gender, ZIP, interests, behaviors, life-events, lookalikes from housing data). With no targeting refinement layer available, Meta defaults to "cheapest form-filler anywhere geo-adjacent," and the default "People living in OR recently in this location" type lets travelers / layover-airport-wifi users into the audience.

### Engagement signal

0 of 16 leads marked "Contacted: Yes" in FUB. 0 email replies received. Pre-May-6 leads got manual emails from Brian (tagged `Backfill-Manual-Handled-2026-05-04`). Post-May-6 leads should be in Automation 2.0 sequence but FUB MCP `/events` endpoint only returned form-submission events, no automation enrollment or email-send events visible. Either the automation isn't connected to Lead Flow 54, or email events live in a separate FUB object the MCP doesn't expose. Needs verification.

### Builder interest distribution (signal worth keeping)

Card_modal submissions self-identify a builder. Useful for future routing / follow-up:
- David Weekley: 3
- Empire Homes: 2
- M/I Homes: 1
- Dream Finders: 1

## Open / pending

### Pre-restart blockers (must resolve before re-enabling ad spend)
- 🟡 **Verify Automation 2.0 enrollment** for Lead Flow 54 → New Construction Incentives source. FUB API does not expose Automations to outside integrators (`/v1/automations` returns 403), so verification must be UI-based. Open any post-May-6 lead in FUB UI and confirm enrollment + email send events on timeline.

### Manual outreach pending (high-priority leads from paused campaign)
- 🟡 **Personal touch on Tammy Flores (FUB ID 31833)** — only confirmed in-market lead from the paused campaign. David Weekley card_modal submit. Original draft was in a CRM thread but staleness risk grows daily; outreach copy now drafted 2026-05-14 (see below).
- 🟡 **Personal touch on Andrew Broughton (FUB ID 31885)** — submitted 3 builder modals (Empire $50K, Empire $25K, DW $30K) in 2 minutes. Highest engagement signal in the batch even with unknown location. Outreach copy now drafted 2026-05-14 (see below).

### Done / shipped since 2026-05-11
- ✅ **Killed Variants A and B** (single-offer creatives that didn't qualify intent)
- ✅ **Built Variant D** (2026-05-13) with corrected tagging fix (2026-05-13)
- ✅ **NeverBounce email validation** live on `/incentives` form (see `projects/neverbounce-validation.md`)

### Backlog (not ad-restart-blocking)
- **Phase 2 ad creative** in production with Sami — current scope NOT captured in brain; need to write up what's being made, what KPI it targets, and what Phase 1 (A/B/C/D) baseline it's compared against. Treat as a brain gap.
- **Builder data** for the incentive comparison page is hardcoded. When the third or fourth builder gets added, move to CMS or JSON config with rotation logic.
- **Quiz scoring logic calibration** — `/quiz` scoring buckets are functional but haven't been validated against actual lead quality. Plan was to wait for first 50 leads of data; with the campaign paused, that bar moves. Treat as a brain gap: the brain doesn't describe what the scoring logic actually does, what the buckets are, or what "calibration" would mean.

### Brain gaps flagged 2026-05-14
- **Phase 2 ad creative** — only documented as "in production with Sami". Needs: scope, deliverable list, target KPI, comparison baseline, expected ship date.
- **Quiz scoring logic** — only documented as "functional but uncalibrated". Needs: what the buckets are, what inputs feed the score, what the score outputs decide (routing? auto-response? lead value?), and what "calibration" would change.

## Key learnings

- **Housing Special Ad Category requires a different playbook than standard Meta ads.** You can't target your way to quality, the standard demographic / interest / behavior levers are gone. Creative copy + placements + exclusion audiences are the only real levers left. Geo-stamp your headlines and imagery so out-of-market viewers self-deselect.
- **CPL is a lying metric on Housing campaigns.** Always cross-reference Meta's reported CPL against actual in-market lead count from CRM. Meta will happily report $12 CPL while 90%+ of those leads are unsellable.
- **FUB social data enrichment is a fast quality audit tool.** Pulling `socialData.location` across a batch of new leads surfaces geo-quality problems in minutes. Worth running on any new ad campaign after first 15-20 leads.
- **Choice-architecture creative outperforms single-offer creative for incentive funnels.** Variant C ($50K off OR low rate) beats both A and B at every level, CTR, LP-to-lead conversion, ranking. The user picks their own offer, which qualifies intent before the click.
- Meta Pixel + CAPI playbook in repo root is the reusable template for any other HGPG site that needs the same setup. Each site needs its own Pixel ID (you cannot share one Pixel across multiple domains effectively)
- Server-side rendering (Next.js App Router) means client-side `fbq()` events fire in tight succession on initial render, important to make sure dedup `event_id` is generated correctly so the CAPI mirror doesn't double-count

## Pickup notes

- This is one of the actively-marketed funnels, Meta ad budget runs through here weekly. PAUSED 2026-05-11 → 2026-05-14: A/B killed, D built, tagging fixed, NeverBounce live. Pending one verification (Automation 2.0 enrollment, see Open / pending) before re-enabling spend.
- Repo has a `META-PIXEL-CAPI-PLAYBOOK.md` at root that should be referenced (not duplicated) by future sites adopting the pattern
- ✅ NeverBounce validation shipped and live on `/incentives` form (per spec in `projects/neverbounce-validation.md`)
- Ad targeting + creative work continues in HGPG - Ads project as of 2026-05-11
- Lead pipeline work + automation verification continues in HGPG - CRM & Leads project
