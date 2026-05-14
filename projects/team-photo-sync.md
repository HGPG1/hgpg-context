# Team Photo Sync

**Repo:** `HGPG1/hgpg-cma-tool` (lives in CMA Engine repo since it owns MLS Grid pipeline)
**Status:** v1 shipped 2026-05-14, awaiting first successful cron run
**Owner:** Brian

## What it does

Syncs MLS Media records for the team's listings (and, in v1.1, CMA tour subjects) into a public Supabase Storage bucket. The team dashboard reads the rehosted URLs so listing cards always have hero photos, with no MLS Grid TOS exposure (we never hotlink originals).

## v1 scope (shipped)

Team listings only, filtered by `mls_property.list_office_key = 'CAR118249224'`. About 63 lifetime listings (5 currently active).

CMA tour subjects parked for v1.1.

## Architecture

```
*/30 cron
  └─> /api/cron/team-media-sync
       ├─ Phase A: discover candidates (RPC team_media_sync_candidates)
       │            └─ for each: fetch from MLS Grid /Media?$filter=...
       │                 └─ upsert into mls_media (preserves rehost state)
       └─ Phase B: drain rehost queue (up to 100/run)
                    └─ download original → upload to mls-media-rehosted bucket
                       → set rehost_status='ok', media_url_rehosted=...
```

### Files

**hgpg-cma-tool:**
- `lib/mls/mediaSync.ts` — per-ListingKey fetch + upsert
- `lib/mls/mediaRehost.ts` — download → upload → mark rehosted
- `app/api/cron/team-media-sync/route.ts` — Phase A + Phase B handler
- `vercel.json` — cron registered `*/30 * * * *`

**hgpg-team-dash:**
- `src/lib/inventory.ts` — hero photo fallback: `portal?.hero ?? mls_media.media_url_rehosted ?? null`

**Supabase wdheejgmrqzqxvgjvfee:**
- Storage bucket `mls-media-rehosted` (public, 10MB/file, image MIMEs)
- RPC `team_media_sync_candidates(p_list_office_key text, p_limit int)`

### Per-run caps

- `MAX_LISTINGS_TO_SYNC_PER_RUN = 25` — Phase A fetches Media for at most 25 listings
- `MAX_REHOSTS_PER_RUN = 100` — Phase B rehosts at most 100 pending records
- `maxDuration = 300s`

### Photo serving — public bucket

Decided public over signed URLs. MLS photos are already public on Realtor/Zillow/the originating brokerage's site; TOS requires rehosting (not auth). Signed URLs add ~50-150ms per request without changing audit defensibility. If we ever sync non-public listings (off-market, private comps), put those in a separate private bucket.

## Storage layout

```
mls-media-rehosted/{originating_system_name}/{resource_record_key}/{media_key}.{ext}
e.g. mls-media-rehosted/carolina/CAR293583134/CARM4287199.jpg
```

Cache-Control set to 1 year — MLS photos are immutable per MediaKey.

## Build error trail (2026-05-14)

Three sequential failures before the first successful deploy. Worth knowing for future TS work on this repo:

1. **Unused constant lint:** `MAX_ATTEMPTS_BEFORE_GIVING_UP` declared but unused → ESLint blocked the build. Removed.
2. **`Map.entries()` downlevelIteration:** cma-tool's tsconfig requires `Array.from()` around `Map.entries()` for iteration. The brain-app's tsconfig is more lenient.
3. **TS inference cycle on `page`:** `'page' implicitly has type 'any' because it does not have a type annotation and is referenced directly or indirectly in its own initializer.` Fix: explicit annotation `const page: MlsPage<MediaRow> = await fetchPage<MediaRow>(cfg, url)`.

Final commits: c8174ec (mediaSync.ts), e955487 (mediaRehost.ts), 977c69b (cron route), 858e228 (vercel.json). Team-dash patch: f36f333.

## First-run debugging notes

The 17:00 UTC 2026-05-14 first cron run returned 200 but inserted 0 rows. Added diagnostic logging (commits 62d843c, 1bd1a0d) to see:
- whether `team_media_sync_candidates` RPC returns candidates inside the cron (vs my manual SQL test which returned 25)
- the actual MLS Grid URL being hit
- the per-page response from MLS Grid

Next cron run at 17:30 UTC should populate Vercel logs. If MLS Grid returns 0 records for a known team listing key, the most likely cause is:
- Media subscription not included in the Brokerage Back Office MLS Grid license (verify with Bridgett Bouvier at Canopy)
- Or `ResourceName eq 'Property'` filter being too strict — try removing it (MLS Grid may use a different ResourceName value)

## v1.1 scope (parked)

CMA tour subjects → MLS Media sync. Reads `cma_reports` from HGPG Core, resolves `subject_address` (free-form text) to a ListingKey via the existing `addressParse.ts` + `subjectDetect.ts` resolvers in this same repo (do NOT rebuild them), then feeds those keys to the existing `syncMediaForListing` function.

Interface is already plumbed — just need to extend the cron's Phase A to gather subjects from cma_reports in addition to team listings. ~30 min build once v1 is verified working.

## What you'll see when working

- Team dashboard at `team.homegrownpropertygroup.com/inventory` shows hero photos on listing cards
- `mls_media` table grows to ~1,900 rows for team (63 listings × ~30 photos each)
- Storage bucket `mls-media-rehosted` accumulates ~1,900 files at ~500KB each (~1GB total)
- Cron run logs in Vercel show Phase A/B counts every 30 min

## Monitoring queries

```sql
-- Backfill progress
SELECT
  count(*) AS total_rows,
  count(*) FILTER (WHERE rehost_status='ok') AS rehosted,
  count(*) FILTER (WHERE rehost_status='pending') AS pending,
  count(*) FILTER (WHERE rehost_status='error') AS errored,
  count(DISTINCT resource_record_key) AS listings_with_media
FROM mls_media
WHERE resource_record_key IN (
  SELECT listing_key FROM mls_property WHERE list_office_key='CAR118249224'
);

-- Listings still missing photos
SELECT p.listing_key, p.standard_status, p.list_date
FROM mls_property p
WHERE p.list_office_key='CAR118249224'
  AND p.listing_key NOT IN (
    SELECT DISTINCT resource_record_key FROM mls_media WHERE rehost_status='ok'
  )
ORDER BY p.list_date DESC;
```
