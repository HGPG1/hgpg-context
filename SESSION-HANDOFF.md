<!-- Last Updated: 2026-05-11 -->

# Session Handoff

## Last session: 2026-05-11 — Sellers Guide ad-readiness QA + 4 bugs fixed 🟢

### Outcome
Sellers Guide is **ad-ready and QA-verified**. End-to-end QA confirmed Pixel + CAPI dedup, NeverBounce, FUB ingestion with full tag set, UTM attribution, fub_event_id round-trip, and canonical domain everywhere. Google Search Console sitemap resubmitted + homepage and /home-selling-score/ manually indexed. Started the session believing the funnel was launch-ready; found and fixed 4 production bugs along the way.

### Four bugs fixed today

1. **Score page had no Pixel init**
   - `/home-selling-score/` is the primary conversion funnel
   - Page had event-firing helpers but never loaded `fbevents.js` or called `fbq('init', ...)`
   - Every browser-side Pixel event was a silent no-op
   - Fix: added Pixel `<head>` block and rewrote `fireFbqEvent` to delegate to `window.hgpgTrack` for browser+CAPI shared event_id

2. **`/api/assessment/submit` FUB call was fire-and-forget**
   - Handler used `fetch(...).catch(...)` without `await`
   - Vercel serverless lambda teardown killed the in-flight FUB POST
   - Every v2 lead returned 200 to user but never reached FUB (all 5 pre-existing rows had `fub_event_id=null`)
   - Fix: awaited the FUB fetch. ~2-3s response time but lead reliably reaches FUB.

3. **`/api/fub-lead` discarded FUB's response body**
   - Returned `{success: true}` even though FUB returned the person object with `id`
   - Caller had no way to record FUB person ID for cross-system lookup
   - Fix: parse FUB response, return `{ success, fub_person_id, fub_person, fub_status }`. `/api/assessment/submit` writes person ID back to `seller_assessments.fub_event_id`.

4. **Stale `sellers.` domain in 8 files**
   - Canonical `<link>`, OG tags, Twitter cards, JSON-LD schemas, sitemap.xml, robots.txt all referenced `https://sellers.homegrownpropertygroup.com/...`
   - That subdomain has no DNS — Google was reading sitemap as canonical and pointing at a void
   - Fix: sed find/replace across all 8 files to canonical `https://sellersguide.homegrownpropertygroup.com/...`
   - Verified live: zero instances of stale domain anywhere on the site

### QA result (single-pass, real submission from phone with utm_source=meta)

- Lead in DB: `schema_version=v2-2026-05-07`, `tier_name=Solid Bones`, `email_validation_status=catchall`, all 4 UTMs + `metaBypass: true` in `utms_json`, `fub_event_id=31928`, `fub_posted_at` populated
- FUB person 31928 landed with tags: `sellers-guide-2026`, `website-lead`, `home-selling-score`, `meta-bypass`, `utm:meta`. Stage=Lead, Source=Sellers Guide 2026.
- Meta Test Events: PageView, ScoreCompleted, Lead processed with shared event_id; CAPI hits in Vercel logs at matching timestamps
- NeverBounce ran and persisted result
- Meta-bypass flow worked (no 6-digit verify step)

### SEO follow-through completed

- Google Search Console: new property `sellersguide.homegrownpropertygroup.com` auto-verified
- Sitemap resubmitted: `sitemap.xml` → Success, 6 pages found
- URL Inspection + Request Indexing fired for:
  - https://sellersguide.homegrownpropertygroup.com/
  - https://sellersguide.homegrownpropertygroup.com/home-selling-score/
- Old `sellers.` property in GSC left alone (no DNS, will self-cleanup over time)

### Data cleanup completed

- Removed 4 QA test rows from `seller_assessments`
- Brian manually deleted FUB persons 31927 and 31928

### Outstanding before scaling ad spend

- **FUB Automation 2.0 not built** on `sellers-guide-2026` tag. Leads land tagged correctly but no automatic drip/assignment fires. Initial ad launch is fine (want manual eyes on first leads anyway), but blocks scaling.

### Meta ads brief prompt drafted

A complete Meta ads campaign brief is in this session's chat history — full context baked in, works for freelancer/agency/GPT recipient. Covers brand, audience (Housing Special Ad Category compliance), creative direction, budget split ($300-500/mo: 60% Conversions / 40% Traffic), 4-week test plan, KPIs. Reuse when ready to launch ads.

### Pickup notes for next session

- **Build FUB Automation 2.0** on `sellers-guide-2026` tag — drip cadence + agent assignment. Unblocks ad scale.
- Set up Meta ad campaigns using the brief from chat history. Destination URLs must bake `?utm_source=meta&utm_medium=paid-social&utm_campaign=<name>` into ad creative.
- Phase 1 ads test markers in code (per 5/5 Pixel commit) — clean up post-launch
- `marketing.md` in brain repo was NOT updated this session — when ads go live, document campaign IDs, ad set structure, and budget there
