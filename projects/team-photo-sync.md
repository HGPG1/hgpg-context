<!-- Last Updated: 2026-05-08 -->

# Team Listing Photo Sync (lightweight)

**Status:** 🟡 Designed, not built. Defer until after Team Dashboard Tab 2.

## Approach

Sync-on-demand instead of bulk. Only pull Media for the team's listings (currently ~63 lifetime, ~5 active+UC+pending) instead of the full Canopy MLS feed.

## Trigger

Run on a 30-min cron (or piggyback on the existing 15-min Property sync). For each `mls_property` row where:

- `list_office_key = 'CAR118249224'` AND
- `mls_media` has zero rows for `resource_record_key = listing_key` OR
- `mls_property.modification_timestamp` is newer than the latest `mls_media.last_synced_at` for that listing

...fetch Media via MLS Grid API filtered to that one ListingKey. Insert rows into `mls_media`.

## Compliance

MLS Grid TOS requires rehosting (no hotlinking). Pipeline:

1. Fetch Media metadata from MLS Grid → upsert into `mls_media` with `media_url_original` populated and `rehost_status = 'pending'`
2. Background worker (or same cron tick) downloads each `media_url_original`, uploads to Supabase Storage bucket `mls-media-rehosted`, sets `media_url_rehosted` and `rehost_status = 'ok'`
3. Team Dashboard reads `media_url_rehosted` exclusively (never `media_url_original`)

Volume estimate: 63 lifetime listings × ~30 photos = ~1,900 files total. Negligible storage cost. After initial backfill, only new listings + modified existing listings get fetched, so steady-state is near-zero.

## Implementation files (when ready)

- `lib/mls/mediaSync.ts` — fetch Media from MLS Grid for a single ListingKey
- `lib/mls/mediaRehost.ts` — download + upload to Supabase Storage
- `app/api/cron/team-media-sync/route.ts` — scheduled handler in Team Dashboard repo (or in CMA Engine repo since it owns the MLS Grid pipeline)

## Schema notes

`mls_media` already exists with the right columns (`media_url_original`, `media_url_rehosted`, `rehost_status`, `preferred_photo_yn`, `order_num`, `media_category`). No migration needed.

## Why this beats full Media sync

- 99% reduction in API calls (63 listings × 1 fetch on change vs. 2.57M property records × full media)
- 99% reduction in storage (~1,900 files vs. ~700K+ files for full Canopy feed)
- Compliance scope shrinks proportionally — quarterly audit only checks files we actually keep
- Falls back gracefully when not available (current placeholder UI keeps working)

## When to upgrade to full sync

If Brian wants Tab 2 (Market Intelligence) to show comp photos for non-team properties, then we need the full Media feed. Until then, on-demand is sufficient.
