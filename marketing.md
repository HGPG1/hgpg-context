# Marketing

## Meta Ads

- **Ad Account:** 31445287
- **Pixel:** 4259881614223735
- **Targeting strategy:** NYC-to-Charlotte relocation
- **Budget split (late March 2026 baseline):**
  - Lead Gen campaign: $30/day (CPL ~$22 vs $35 target)
  - Retargeting RT-AD1: $10/day
  - Retargeting RT-AD3: $10/day
  - Traffic BK-AD1: $7.50/day
  - **Total: $57.50/day**
- Two 60-day custom audiences created.
- HGPG brand correction confirmed in account: NO green, NO Cormorant Garamond.

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
- Pipeline: AI avatar (RSS → script → HeyGen → publish)
- Distribution: Instagram + YouTube
- Newsletter: Beehiiv
- Growth levers identified: TikTok expansion, "Comment REPORT" CTA

## IDX Broker

- Migration from Ylopo complete. Showcase IDX cancelled.
- Hosted on WordPress at SiteGround: `search.homegrownpropertygroup.com`
- `/mls/search` endpoint NOT enabled on current tier (relevant for CMA Engine).
- Realty Candy handles IDX header fixes.

## iMessage/SMS platforms (evaluated, none selected)

- **Sendblue, Linq, Project Blue (tryprojectblue.com)** all evaluated as CRM follow-up tools.
- Project Blue prohibits mass outbound blasts and is HighLevel-native (unclear FUB integration).
- All three carry Apple platform risk.
- No platform selected.

## URL shortener idea

Raw IDX search URLs are too long for Safari. Options for sharing with buyers:
1. Send IDX login link (shows all saved searches)
2. Use a URL shortener
3. Use IDX "Email this Search" feature from lead profile

Potential build: lightweight URL shortener on `hgpg.link` subdomain (Vercel + Supabase, with click tracking).
