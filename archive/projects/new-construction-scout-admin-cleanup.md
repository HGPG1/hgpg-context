<!-- Last Updated: 2026-05-13 -->

# New Construction Scout Admin - scout_status Filter

**Status:** 🟢 SHIPPED 2026-05-13
**Repo:** `HGPG1/charlotte-new-construction-nextjs`
**Domain:** `newconstruction.homegrownpropertygroup.com`
**Driver:** Scout admin tab was showing all 23 builders including 4 with persistently failing crawls (404, 403, JS-required). Cleaned it up to show only builders the crawler is actually doing work on.

## What shipped

**DB migration** (Supabase `HGPG Signature + Relocation`, project `fkxgdqfnowskflgbuxhm`):
- Added `nc_builders.scout_status TEXT NOT NULL DEFAULT 'crawl'` with CHECK constraint allowing `crawl | email_only | paused`
- Index on `scout_status`
- Comment documents intended use

**Data fixes (also in the DB):**
- KB Home incentives_url corrected from broken `/new-homes-charlotte-area` to working `/new-homes-south-carolina/indian-land` (Indian Land URL is actually MORE relevant for HGPG-core market than the broader Charlotte page)
- 4 builders flipped to `scout_status = 'email_only'`: Eastwood Homes, Mattamy Homes, Stanley Martin, Taylor Morrison

**Code (PR #3, merge commit 4ae51fc):**
- `src/lib/supabase.ts`: added `ScoutStatus` type, `scout_status` field on BuilderRow
- `src/app/admin/page.tsx`: Scout tab now filters where `scout_status = 'crawl'`. Builders tab shows scout_status badge per row (green Crawl / amber Email Only / gray Paused) and exposes scout_status as editable in the edit form
- `src/app/api/admin/scout/candidates/route.ts`: filter lives in the API route, not just client-side, so email_only builders never enter the Scout payload
- `src/app/api/admin/builders/route.ts`: allows editing scout_status

**ScoutStatusBadge** defensive: accepts null/undefined and falls back to 'crawl' (DB default makes this unreachable in practice).

## Builder breakdown after this change

| scout_status | Count | Names | Why |
|---|---|---|---|
| crawl | 19 | All except the 4 below | URL works, crawler does the work |
| email_only | 4 | Eastwood, Mattamy, Stanley Martin, Taylor Morrison | Persistently failing crawl (403, JS-required). Incentives ingested via Apps Script email pipeline instead |
| paused | 0 | (none today) | Future-use slot for "we're not pulling at all right now" |

## Key learnings (worth keeping)

1. **JS-rendered + WAF-blocked sites can't be crawled with plain HTTP fetch.** Eastwood returns 403 to anything that isn't a real browser. Mattamy / Stanley Martin / Taylor Morrison return "no usable text" because content is JavaScript-rendered. These need either a headless browser (Playwright) or email-pipeline ingestion. We chose email-pipeline because it already exists.
2. **Some builder URLs are wrong, not blocked.** KB Home's `/new-homes-charlotte-area` was a 404 - real Charlotte-metro coverage lives at `/new-homes-south-carolina/indian-land`. When a builder is 404'ing, try alternate URL patterns before flipping to email_only.
3. **Don't deactivate builders to hide them from one admin view.** Adding a column-based filter (scout_status) is cleaner than `is_active=false` because the builder is still real and public on /builders, just not being crawled.
4. **Filter at the API layer, not just the UI.** Email_only builders genuinely never enter the Scout candidates payload now. Cheaper and prevents accidental client-side filter bypass.

## Future considerations (parked, no commitment)

- **Playwright-rendered crawl path** for the JS-rendered builders. Would let Mattamy / Stanley Martin / Taylor Morrison surface in Scout admin. Larger lift - probably not worth it unless email pipeline starts missing incentives for these builders.
- **More builders to add to the active roster.** Brian originally asked for "deep search for missing builders" but the actual ask was hiding non-crawlable ones. If/when the additions question comes back, the broader-Charlotte candidates worth considering: Tri Pointe (already there), Toll Brothers (already there), Stanley Martin (already there - but email_only), Smith Douglas Homes, Century Communities, KB Home (already there). HGPG already has 23 builders which is dense coverage.

## References

- DB migration: `add_scout_status_to_nc_builders` (2026-05-13)
- PR #3: https://github.com/HGPG1/charlotte-new-construction-nextjs/pull/3
- Merge commit: `4ae51fc7e758530aeab56f69d4682994afc8cfe2`
- Prod deploy: `dpl_GLVMxgiz6o8N9zG1rsgJxHjBSgLs` READY
- Related prior work: builders display_order tiering (commit 7oTcYzM), scout review routing (commit 6jLWGug)
