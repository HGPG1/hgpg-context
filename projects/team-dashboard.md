<!-- Last Updated: 2026-05-08 -->

# Team Dashboard

**Status:** 🟢 SHIPPED (Tab 1 + Tab 2 live as of 2026-05-08)
**Live URL:** `team.homegrownpropertygroup.com`
**Repo:** `HGPG1/hgpg-team-dash`
**Supabase:** `wdheejgmrqzqxvgjvfee` (HGPG Listing Reports + MLS)
**Vercel project:** `prj_Q4dFcHGUvmUbUPaDydcHhdS2Laxd` on team `team_FietQPKCmnyioG2n0FdteQCV`

## Purpose

Team-facing dashboard with two tabs:

- **Tab 1 — Inventory + Showings.** Live HGPG team listings (active, UC, pending, closed) with showings, ListTrac platform views, feedback, key dates. Filtered by office key `CAR118249224`.
- **Tab 2 — Market Intelligence.** Search any zip / subdivision / address radius in the Charlotte metro. 6-month trend charts (median price, DOM, sold volume), snapshot stats (months of supply, list-to-sold ratio, 30-day movers). Compare two areas side-by-side. Save areas for quick access.

NOT to be confused with:
- `reports.homegrownpropertygroup.com` — seller-facing Listing Report Portal
- `dash.homegrownpropertygroup.com` — Signature listing admin only

## Audience

Brian, Ashley, Brenda, Taylor (agents). Lauren (closings), Vessie Lopez also on the office key.

Don is a TC, not an agent — uses TM and TC Concierge.

## Auth

Magic link via Supabase Auth on `wdheejgmrqzqxvgjvfee` project. Email allow-list via `TEAM_EMAILS` env var (comma-separated). Middleware (`src/proxy.ts`) gates everything except `/auth/*` and rejects users not on allow-list.

## Data backbone

Office key: **`CAR118249224`** (NOT R04075 — that's a Canopy display ID, doesn't appear in the MLS Grid feed).

### Tab 1 query semantics

| Source | Key | Notes |
|---|---|---|
| `mls_property` | filter `list_office_key = 'CAR118249224'` | live MLS Grid feed, 15-min cron |
| `listings` (manual portal table) | join `listings.mls_number = REPLACE(mls_property.listing_id, 'CAR', '')` | only present for ~7 portal-onboarded listings |
| `showings` | `showings.listing_id = listings.id` | requires portal row |
| `weekly_stats` | `weekly_stats.listing_id = listings.id` | ListTrac platform views, requires portal row |
| `mls_media` | `resource_record_key = listing_key AND preferred_photo_yn = true` | ZERO ROWS today — sync deferred (see `team-photo-sync.md`) |

### Tab 2 query semantics

- Reads from FULL `mls_property` (no office filter) — Canopy MLS Brokerage Back Office license covers entire metro
- ALWAYS includes `mlg_can_view = true` (required to hit existing partial indexes on postal_code/comp_search; otherwise 18s+ timeouts)
- Address composition: `TRIM(CONCAT_WS(' ', street_number, street_name, street_suffix))`
- Radius search: bbox query on lat/lon (uses index), then JS haversine refinement to true distance
- Geocoding via Nominatim/OpenStreetMap (free, fair-use, cached)
- Subdivision typeahead via `search_subdivisions(text)` RPC (case-insensitive group, returns canonical+active+lifetime)

## Database migrations applied (in order)

1. `team_dash_read_mls_tables` — RLS SELECT policies for authenticated users on mls_property, mls_member, mls_sync_state
2. `team_dash_office_key_index` — composite index on (list_office_key, standard_status, list_date DESC) and listing_id index
3. `team_dash_watched_areas` — `team_watched_areas` table with user-scoped RLS
4. `team_dash_subdivision_search_rpc` — `search_subdivisions(text)` security-definer function

## Routes

    /                           redirect to /inventory
    /auth/login                 magic link request
    /auth/callback              auth code exchange
    /auth/error                 friendly error states
    /auth/signout               POST sign-out
    /inventory                  Tab 1 grid + filter pills
    /inventory/[listingId]      Tab 1 drilldown
    /market                     Tab 2 with search + comparison
    /api/subdivisions           typeahead JSON
    /api/watched-areas          GET/POST/DELETE saved areas

## Key learnings

- **CAR-prefix strip** required on listing_id to join MLS Grid → portal listings table
- **`mlg_can_view = true`** must be on every Tab 2 query to hit indexes (only 4 of 2.57M rows have false; cost is negligible)
- **PostgREST 1000-row default** silently truncates queries; bumped to `range(0, 9999)` on busy queries
- **Nominatim user-agent header** required to comply with their fair-use policy (set to `HGPG-TeamDash (brian@homegrownpropertygroup.com)`)
- **Comparison via URL params** — `mode2`/`zip2`/`subdivision2`/`address2`/`radius2` mirror primary; presence of `mode2` alone (without secondary value) shows the second search bar so users can fill it in

## Open follow-ups

- **Hero photos** — defer until `team-photo-sync.md` ships (~30 min, on-demand sync for active team listings, rehosted to Supabase Storage)
- **Vessie Lopez** — on office key but role is unclear; her listings show in Tab 1 unconditionally
- **Performance** — busy zips (28277 = 1348 recent rows) hit ~2s page load. Acceptable for search-driven UX, would optimize if browse-heavy usage emerges
- **Mobile responsiveness** — built mobile-first but not tested on actual phones yet

## Future tab ideas

- **Tab 3: Buyer Alerts** — would extend MLS Dashboard Suite framing. Brian's preferred path is a separate tool; reconsider after Buyer Alerts ships standalone
- **Comp photo gallery** in Tab 2 — requires full MLS Grid Media sync (parked)
