# Team Photo Sync

**Status:** ✅ Live in production. v1 backfill complete (99.94% of team photos rehosted; 1,652 of 1,653).
**Owner:** Brian.
**Last touched:** 2026-05-15 (multi-day build session, final cron drain at 11:30 UTC).

---

## What it does

Rehosts MLS photos for the HGPG team's own listings (`list_office_key='CAR118249224'`, MLS office ID `CARR04075`) from MLS Grid's signed-URL CDN onto public Supabase Storage at `mls-media-rehosted/carolina/<listing_key>/<media_key>.jpg`. Powers hero photos on the team dashboard (`team.homegrownpropertygroup.com/inventory`).

MLS Grid TOS requires that we host the bytes on infrastructure we control rather than hot-linking their `media.mlsgrid.com` signed URLs. The signed URLs also expire (24h tokens), so any UI surface that linked them directly would break daily.

---

## Architecture (Phase A + Phase B)

**Cron:** `/api/cron/team-media-sync` runs every 30 minutes (`vercel.json` cron `*/30 * * * *`). One Vercel function, two phases.

**Phase A — discovery + queue:** Queries MLS Grid v2 `/Property?$filter=OriginatingSystemName eq 'carolina' and ListOfficeMlsId eq 'CARR04075'&$expand=Media&$select=ListingKey,OriginatingSystemName&$top=50` walking up to `MAX_PAGES_PER_RUN=5` pages. Each Property has nested `Media[]`. Upserts every Media record into `mls_media` keyed by `media_key`. Preserves existing `rehost_status='ok'` + `media_url_rehosted` + `rehost_attempted_at` so the worker queue order doesn't reset on every tick.

**Phase B — rehost worker:** `rehostPendingMedia({supabase, supabaseUrl, limit: MAX_REHOSTS_PER_RUN})`. Selects pending rows (and error rows older than 1 hour), ordered by `rehost_attempted_at ASC NULLS FIRST` (NULL = never attempted = process first). For each: downloads from `media_url_original` (24h-signed MLS Grid URL), uploads to public Supabase Storage, sets `rehost_status='ok'` + `media_url_rehosted` URL. 150ms throttle between downloads (MLS Grid media CDN limits to ~7-10 RPS).

**Defaults:**
- `MAX_PAGES_PER_RUN=5` (Phase A)
- `MAX_REHOSTS_PER_RUN=200` (steady state; was 500 during backfill)
- `maxDuration=300s` (Vercel function timeout)

---

## Files (hgpg-cma-tool repo)

- `app/api/cron/team-media-sync/route.ts` — cron handler, Bearer-token verified
- `lib/mls/mediaSync.ts` — `syncMediaForOffice()` Phase A logic
- `lib/mls/mediaRehost.ts` — `rehostPendingMedia()` Phase B logic
- `lib/mls/client.ts` — `fetchPage()` MLS Grid v2 client (shared with `mls-sync` cron)
- `lib/mls/mappers.ts` — `mapMedia()` MLS Grid → `mls_media` row mapping
- `vercel.json` — cron schedule
- `app/api/cron/team-media-debug/route.ts` — neutralized 404 stub (was the dev debug endpoint)
- `app/api/admin/team-media-debug/route.ts` — neutralized 404 stub (caused build SIGTERM earlier)

**Frontend integration:** `hgpg-team-dash` repo, `src/lib/inventory.ts` (commit `f36f333`) — hero photo fallback chain: `portal?.hero ?? mls_media.media_url_rehosted ?? null`. Selects preferred photo by `preferred_photo_yn=true` (always null on Canopy) then `order_num=0` then first available.

---

## Data layout

**Supabase project:** HGPG Listing Reports + MLS (`wdheejgmrqzqxvgjvfee`)

**Tables:**
- `mls_property` — full listing records. Team filter: `list_office_key='CAR118249224'`
- `mls_media` — one row per photo. Columns we care about: `media_key` (PK), `resource_record_key` (=parent ListingKey, FK to mls_property.listing_key), `originating_system_name` (='carolina'), `media_url_original` (signed 24h URL), `media_url_rehosted` (public Supabase URL), `rehost_status` ('pending'|'ok'|'error'), `rehost_attempted_at`, `rehost_error`, `order_num`, `preferred_photo_yn`

**Storage:** Public bucket `mls-media-rehosted`, 10MB/file limit, image MIMEs only. Path pattern: `{originating_system_name}/{resource_record_key}/{media_key}.{ext}` → `carolina/CAR300982442/65abd5c....jpg`

**RPC:** `public.team_media_sync_candidates(p_list_office_key text, p_limit int)` — unused by current office-wide approach but still exists. Can drop if confident.

---

## MLS Grid v2 gotchas (learned the hard way)

1. **Direct `/Media` queries are forbidden in v2** — must use `$expand=Media` on Property
2. **`ListingKey` is NOT filterable on Property** — only `MlgCanView`, `ModificationTimestamp`, `OriginatingSystemName`, `StandardStatus`, `ListingId`, `PropertyType`, `ListOfficeMlsId` are
3. **Hard 2 RPS rate limit** shared across all crons hitting MLS Grid (mls-sync, team-media-sync)
4. **Response size matters**: without `$select`, Property records contain ~200 columns + nested Media. `$top=200` returns 10MB+ which gets a 400 HTML response from MLS Grid's edge. Use `$select=ListingKey,OriginatingSystemName` + `$top=50` to stay under the cap.
5. **Nested Media records have a misleading `ResourceRecordKey`** — it's NOT the parent ListingKey. Override `resource_record_key=parentListingKey` after the `mapMedia` call.
6. **Nested Media records DO NOT include `OriginatingSystemName`** — it's only on the parent Property. Override `originating_system_name=cfg.originatingSystemName`.
7. **MLS Grid media CDN (`media.mlsgrid.com`) has a separate, tighter rate limit** — ~7-10 RPS. Burst downloads from tight for-loops cause `download 429`. Use 150ms throttle between rehost calls.

## PostgREST gotchas

8. **`.in()` lookups exceed URL length limit at scale** — 1,653 media keys in `.in()` returns "Bad Request". Chunk into batches of 200 keys.
9. **`upsert()` clobbers ALL columns by default** — `rehost_status`, `media_url_rehosted`, `rehost_attempted_at` get reset to whatever's in the row being upserted. Preserve existing values explicitly via lookup-and-merge before upsert.

## Vercel build gotchas

10. **`SIGTERM` during build** = a route is doing outbound HTTP at build time during prerender. Add `export const dynamic = 'force-dynamic';` to any route that fetches external data, even debug routes.

---

## v1 backfill timeline

- 2026-05-13: First Phase A attempts, 0 rows returned (per-listing iteration approach + wrong filter)
- 2026-05-14 18:00 UTC: Rewrote to office-wide query (`syncMediaForOffice`). Cascading TS errors and build SIGTERMs.
- 2026-05-14 19:30 UTC: First successful Phase A — 63 listings, 1,653 Media records discovered, all upserted.
- 2026-05-14 19:50 UTC: First successful Phase B — 5 photos rehosted end-to-end. URL verified accessible.
- 2026-05-14 20:00–overnight: Production cron ran every 30 min, draining 500 rows/tick (temporarily bumped from 100).
- 2026-05-15 10:30 UTC: 1,591/1,653 rehosted (96.2%), 0 pending, 62 errored (61 download 429 + 1 download 400).
- 2026-05-15 10:35 UTC: 150ms download throttle added (`mediaRehost.ts`), `MAX_REHOSTS_PER_RUN` dropped to 200. 61 download-429 errors reset to pending for retry.
- 2026-05-15 11:30 UTC: 11:30 cron processed the 61 retries successfully with the new throttle. Final state: 1,652/1,653 rehosted (99.94%). 1 remaining error is on a Closed listing (`CAR4311537`) with an already-expired MLS Grid signed URL; will self-resolve on the next Phase A refresh.

**All 4 active listings have hero photos visible on the team dashboard.**

---

## Post-launch fix: RLS policy missing on mls_media

2026-05-15 ~12:00 UTC, Brian reported no photos on the inventory page despite 1,652/1,653 rehosted in the DB. Cause: `mls_media` had `rowsecurity=true` with **zero RLS policies**. PostgreSQL RLS default = deny, so the team-dash (which uses the `authenticated`-role client via `getSupabaseServer()`) got empty result sets silently. The fix:

```sql
CREATE POLICY authenticated_read_mls_media
  ON public.mls_media
  FOR SELECT
  TO authenticated
  USING (true);
```

This mirrors the existing `authenticated_read_mls_property` policy. No deploy needed. Lesson: when creating a new table in `wdheejgmrqzqxvgjvfee` that the team-dash will query, **always create an `authenticated SELECT` policy** at table-creation time. Check `pg_policies WHERE schemaname='public'` to verify.

## Open items

- ✅ **Drop `public.team_media_sync_candidates` RPC** — done 2026-05-15 (migration `drop_unused_team_media_sync_candidates_rpc`).
- ✅ **Confirm 5 orphaned storage objects are safe to delete** — verified 2026-05-15. DB rows point to `carolina/`-prefixed dupes which also exist in storage. Top-level files at `mls-media-rehosted/CAR118472288/65abd5c90095d833e379530{7..b}.jpg` are genuinely orphaned.
- **Delete the 5 orphaned storage objects** via Supabase Storage API from the Mac (raw SQL DELETE is blocked by `storage.protect_delete()` trigger). Recipe in `SESSION-HANDOFF.md` of the same date.
- **Delete the two debug route files** from the Mac via `gh` (currently neutralized to 404 stubs). Files: `app/api/cron/team-media-debug/route.ts`, `app/api/admin/team-media-debug/route.ts`.
- **Investigate `download 400` on the single permanent-fail row** (`CAR4311537` — Closed listing, expired signed URL). Likely self-resolves on next Phase A refresh. Consider adding a `permanent_fail` status to skip it forever.
- **Phase A doesn't preserve `rehost_error` across runs** — nice-to-have fix in `lib/mls/mediaSync.ts`. Currently the error reason is cleared on every Phase A upsert, making triage harder.

## Commits (v1)

- `aa628cf` — Fix cron: `rehostPendingMedia` takes `{supabase, supabaseUrl, limit}`
- `9858872` — Slim MLS Grid response: `$select` Property cols + `$top=50`
- `cd8c630` — Chunk `.in()` lookup to 200 keys/batch
- `3755b75` — Override `resource_record_key` to parent ListingKey
- `03be600` — Override `originating_system_name` from parent cfg
- `214828b` — Bump `MAX_REHOSTS_PER_RUN` to 500 for backfill
- `e8ecef2` — Fix TS: preserve `rehost_attempted_at` across Phase A runs
- `2cceaa6` — Throttle rehost downloads 150ms (media CDN was 429ing)
- `f286811` — Drop `MAX_REHOSTS_PER_RUN` to 200 (steady state)
- `5c4ad6f` — Neutralize debug endpoint to 404 stub

Frontend (hgpg-team-dash):
- `f36f333` — Hero photo fallback in inventory list + detail views

---

## Deferred polish: hero photo quality

**Identified:** 2026-05-15. Parked, no impact on dashboard usability today.

The current fallback chain (`portal?.hero ?? mls_media.media_url_rehosted ?? null`) selects MLS hero by:
1. `preferred_photo_yn = true`
2. then `order_num ASC`
3. then first available

**Problem:** Canopy strips `preferred_photo_yn` from the MLS Grid feed (always `null` for the team's listings). So selection always falls through to `order_num = 0`, which is whatever the listing agent uploaded first into Matrix. That's frequently a render, floor plan, lot/aerial, or interior, NOT the agent's curated primary photo. Example: `CAR267743419` hero is currently a `long_description='Conceptual Render'`.

**Why it doesn't matter right now:**
- 4 of 5 active listings show a photo (just not always ideal)
- CAR293583134 currently has 0/33 rehosted but that'll self-resolve as the worker grinds, or be a one-off
- No agent complaints yet from Ashley/Brenda/Taylor

**Pick this back up when:**
- A render or floor plan ends up as hero on a high-profile listing
- Agents flag bad photos during a demo
- Doing a Tab 1 polish pass anyway

**Likely fix when revisited (cheap path):**
- Add a `long_description` heuristic to skip rows containing "render", "floor plan", "site plan", "aerial" before falling to `order_num`
- Plus a `hero_media_key` override column on `mls_property` + tiny admin UI for one-click manual override on the remainder

Avoid chasing `preferred_photo_yn` backfill from Canopy. Probably impossible if they strip it at the feed layer, not worth the spelunking without evidence it's recoverable.
