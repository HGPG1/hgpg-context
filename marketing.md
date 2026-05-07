<!-- Last Updated: 2026-05-07 -->

# Marketing

> Account IDs and pixel IDs live in memory.

## Meta Ads

- **Targeting strategy:** NYC-to-Charlotte relocation
- **Budget split (late March 2026 baseline, may have shifted):**
  - Lead Gen campaign: $30/day (CPL ~$22 vs $35 target)
  - Retargeting RT-AD1: $10/day
  - Retargeting RT-AD3: $10/day
  - Traffic BK-AD1: $7.50/day
  - **Total: $57.50/day**
- Two 60-day custom audiences created.
- HGPG brand correction confirmed in account: NO green, NO Cormorant Garamond.

## Meta Pixel + CAPI (multi-site rollout)

Each site has its own dedicated Pixel. Server-side CAPI mirror with event_id deduplication is the standard pattern.

| Site | Pixel ID | Status |
|---|---|---|
| Charlotte New Construction | 1880396459290092 | Live, all 6 funnel events firing |
| Sellers Guide | 861295553661596 | Live, QA in progress, Custom Conversion registration pending |
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
- Pipeline: AI avatar (RSS -> script -> HeyGen -> publish)
- Distribution: Instagram + YouTube
- Newsletter: Beehiiv
- Daily pipeline via `morning_fetch.yml` / `run_morning_fetch_enhanced.py`
- Video normalization for Instagram (H.264, `+faststart`, known-good Reels spec)
- HTML digest email with red failure banner + subject prefix on downstream failures
- HeyGen failure short-circuits IG/YouTube checks to prevent cascade false positives
- Growth levers identified: TikTok expansion, "Comment REPORT" CTA

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
