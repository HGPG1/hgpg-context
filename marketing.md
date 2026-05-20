<!-- Last Updated: 2026-05-19 -->

# Marketing

> Account IDs and pixel IDs live in memory.

## Meta Ads

- **Active campaigns (May 2026):** Sellers Guide launch + NYC-to-Charlotte relocation continuing + New Construction Incentives Variant E launched 2026-05-18
- HGPG brand correction confirmed in account: NO green, NO Cormorant Garamond
- All housing-related ads run under Special Ad Category: Housing

### New Construction Incentives Variant E (launched 2026-05-18)

- **Angle:** Dollar-anchor — "$10,000+ in closing-cost help" hook, mirrors Variant C's choice-architecture framing but with a single magnetic number
- **Brief:** `variant-e_brief.pdf` (Drive ID `1U6Voq92rqVSFDIrTFiborWCBKFbiLqk3`), shipped 2026-05-18 19:44 UTC
- **Creative package:** 3 sizes, all using IMG_5675 (black polo, kitchen island, warm closed-mouth smile — same photo as Variant D)
  - 1080x1080 square
  - 1080x1350 portrait (4:5)
  - 1080x1920 vertical (9:16)
- **Composition:** photo top with navy gradient, single hero number `$10,000+`, locality strip (Indian Land · Fort Mill · Waxhaw · Lake Wylie), HGPG + Real Broker lockup, CTA pill
- **CTA:** "Learn More" (locked in brief — earlier drafts proposed "Get Offer", launched version uses Learn More)
- **UTM:** `utm_content=variant-e-10k`
- **Audience/placement:** mirrors Variant C
- **Pixel:** 1880396459290092 (same as rest of New Construction funnel)
- **Special Ad Category:** Housing
- **Benchmark:** Variant C ($50K off OR low rate) at $4.63 CPL, 2.60% CTR, 33.3% LP-to-Lead
- **Read schedule:**
  - Day 3-4 first read: 2026-05-21 → 2026-05-22
  - Day 5-7 decision point: 2026-05-23 → 2026-05-25
  - Success bar: E within 1.5x C's CPL at day 3-4 (≤ ~$7 CPL)

### Sellers Guide campaign (launched 2026-05-15)

- **Campaigns:**
  - `HGPG-SellersGuide-TRAF-2026-Q2` (campaign ID 52506271902963), $10/day CBO, Traffic objective, LandingPageView optimization
  - `HGPG-SellersGuide-CONV-2026-Q2` (campaign ID 52506270109163), $15/day CBO, Sales/Leads objective, Lead optimization
- **Single ad set per campaign:**
  - `TRAF-HSS-NCSC-Border` - Concepts A + D, 3 sizes each (6 ads)
  - `CONV-HSS-NCSC-Border` (ad set ID 52506271965563) - Concepts B + C, 3 sizes each (6 ads)
- **Geo:** 15-mile radius around 28173 Waxhaw, 28277 South Charlotte, 29707 Indian Land, 29715 Fort Mill, 29720 Lancaster
- **Targeting:** Homeowner OR Real estate OR Home improvement interests, Advantage+ Audience expansion ON
- **Pixel:** 861295553661596 with confirmed CAPI dedup
- **Custom Conversion:** "HGPG - Sellers Guide Lead" (ID 972661382159071), rule = URL contains sellersguide.homegrownpropertygroup.com, source event = Lead
- **Landing pages:**
  - TRAF -> `sellersguide.homegrownpropertygroup.com/` (root)
  - CONV -> `sellersguide.homegrownpropertygroup.com/home-selling-score/` (direct to score)
- **Target CPL:** $35. Reference: New Construction Lead Gen running ~$22 CPL.
- **Budget reality vs plan:** $25/day total ($750/mo) is above the original $300-500/mo plan envelope. Conscious decision 2026-05-15 to leave it at this level - faster Learning Phase exit (~2 weeks vs 3+ at lower budget), better creative rotation across 12 ads, knowable downside (~$175 worst-case week 1). Revisit at day 7 audit based on actual CPL trend.
- **Test plan:** 4-week launch cadence per `HGPG_Sellers_Guide_Meta_Ads_Plan.pdf`. Day 7 first read, day 14 first cull, day 30 retargeting ad set added if audience pool >1,000.

#### Known soft fails on CONV ad set (revisit at day 30 rebuild)

Three Meta API limitations hit during launch. None affect lead delivery, all affect reporting cleanliness.

1. **Optimization event is raw Lead, not custom conversion.** Meta blocks `promoted_object` changes after ad set publish (error_subcode 3260011). Functionally identical - both fire on the same trigger. Ads Manager will show "Lead" column instead of "HGPG - Sellers Guide Lead". Counts match. Fix at rebuild: duplicate ad set with custom conversion baked in from creation, archive original.
2. **Attribution is 7-day click only, missing 1-day view.** Meta blocks `attribution_spec` changes after ad set publish (error_subcode 1504040). Costs ~10-15% reporting visibility on view-through conversions. Same fix as above.
3. **CBO instead of ABO.** Meta API blocks setting campaign budget to $0 to migrate to ABO. With one ad set per campaign, mathematically equivalent. Becomes a problem when adding the retargeting ad set at day 30 - must switch to ABO at that point (UI allows on live campaigns; only API create flow blocks).

### NYC-to-Charlotte relocation (continuing)

- **Targeting strategy:** NYC-to-Charlotte relocation
- **Budget split (late March 2026 baseline, may have shifted):**
  - Lead Gen campaign: $30/day (CPL ~$22 vs $35 target)
  - Retargeting RT-AD1: $10/day
  - Retargeting RT-AD3: $10/day
  - Traffic BK-AD1: $7.50/day
  - **Total: $57.50/day**
- Two 60-day custom audiences created.
- Retargeting RT-AD frequency hit 8.68 by 2026-05-15 - audience saturated, refresh needed soon.

## Meta Pixel + CAPI (multi-site rollout)

Each site has its own dedicated Pixel. Server-side CAPI mirror with event_id deduplication is the standard pattern.

| Site | Pixel ID | Status |
|---|---|---|
| Charlotte New Construction | 1880396459290092 | Live, all 6 funnel events firing |
| Sellers Guide | 861295553661596 | Live, Lead event firing confirmed 2026-05-15, Custom Conversion "HGPG - Sellers Guide Lead" (ID 972661382159071) active. Active issue: "Missing Lead Currency Parameter" warning - Lead event passes `value` (score) without `currency`. Cosmetic, not blocking. Fix: remove `value` from Lead event payload or add `currency: 'USD'`. Owned by HGPG - Tech & Builds. |
| TM, marketing analyzer, signature | n/a | Playbook ready, ~30 min/site, not yet rolled out |

**Reusable playbook:** `META-PIXEL-CAPI-PLAYBOOK.md` in charlotte-new-construction-nextjs repo root.

## Lead Re-engagement Campaign

- Eight-variant email campaign for FUB lead database:
  - Four variants for broad segment
  - Four variants for hotter pipeline segment
- **Batch send strategy:** Three batches of ~1,000, Tuesday through Thursday, monitor deliverability after first batch.

### FUB automation re-trigger workarounds

- Re-fire the trigger
- OR clone automation with new trigger tag to bypass duplicate protection
- **Mass Actions do NOT fire automations.**

## South Charlotte Report (content brand)

- Handle: `@southcharlottereport`
- Pipeline (two tracks):
  - **Daily news (Track 1):** RSS + S3-scraped government/social -> 2-stage brand-safety filter -> Claude script -> HeyGen avatar video -> Instagram Reel + WordPress blog post. Runs `morning_fetch.yml` / `run_morning_fetch_enhanced.py` at 10:00 UTC (5 AM EST / 6 AM EDT).
  - **Monthly HGPG market video (Track 2):** triggered by `south-charlotte-market-report` repo via `repository_dispatch: market-report-ready`. Publishes to HGPG YouTube only. Runs `hgpg_market_video.yml` / `generate_hgpg_video.py`.
- Daily distribution: **WordPress + Instagram Reels only** (SCR YouTube Shorts is NOT in use per repo HANDOFF).
- Monthly distribution: HGPG YouTube only (higher editorial bar).
- Daily digest email to Brian via **Gmail SMTP** (`GMAIL_SENDER_EMAIL` + `GMAIL_APP_PASSWORD`). Not Resend, not Beehiiv. If a Beehiiv subscriber newsletter exists, it's outside the workflow — needs Brian clarification.
- Video normalization for Instagram (H.264, `+faststart`, known-good Reels spec).
- HTML digest email with red failure banner + subject prefix on downstream failures.
- HeyGen failure short-circuits IG checks to prevent cascade false positives.
- Growth levers identified: TikTok expansion, "Comment REPORT" CTA.
- Full project state in `projects/south-charlotte-report.md`.

## IDX Broker

- Migration from Ylopo complete. Showcase IDX cancelled.
- Hosted on WordPress at SiteGround: `search.homegrownpropertygroup.com`
- Realty Candy handles IDX header fixes.
- **CMA Engine note:** IDX `/mls/search` endpoint not enabled on current tier. CMA Engine reads MLS data from Supabase via MLS Grid replication (Canopy MLS Brokerage Back Office license). IDX is retained as a fallback flag (`MLS_USE_IDX_FALLBACK`).

## Messaging platforms

- **Sendblue: SELECTED for FUB AI Agent.** Decision locked 2026-05-06. See `projects/fub-ai-agent.md`.
- **LoopMessage: SELECTED for Transaction Manager.** Sole iMessage path for TM. Owns Apple ID/sender identity, group creation, routing.
- **Twilio: FULLY DEPRECATED.** Removed from TM 2026-04. Blocked on TCR resubmission for any future use (need SMS consent checkbox on contact forms).
- Linq, Project Blue evaluated and not selected. Project Blue prohibits mass outbound blasts and is HighLevel-native.

## URL shortener idea

Raw IDX search URLs are too long for Safari. Options for sharing with buyers:
1. Send IDX login link (shows all saved searches)
2. Use a URL shortener
3. Use IDX "Email this Search" feature from lead profile

Potential build: lightweight URL shortener on `hgpg.link` subdomain (Vercel + Supabase, with click tracking).
