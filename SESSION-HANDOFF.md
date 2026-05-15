# SESSION-HANDOFF — 2026-05-15 (final)

## Status: ✅ Team Photo Sync v1 fully shipped

**Production state (11:32 UTC):**
- **1,652 of 1,653 team Media records rehosted (99.94%)**
- 1 error remaining: a single photo on `CAR4311537` (Closed) whose MLS Grid signed URL was expired by the time the cron tried it. Will retry naturally at 12:30 UTC and succeed. Not on dashboard so doesn't matter for UX.
- **All 4 active team listings have hero photos visible** (`CAR4222460`, `CAR4336731`, `CAR4341869`, `CAR4374686`)
- Team dashboard `team.homegrownpropertygroup.com/inventory` is fully populated with real photos via `mls_media.media_url_rehosted`

**Production cron:** `dpl_EXCVhxSChdrJz9hfRXaDmGy21qbe` (commit `5c4ad6f`).
- `/api/cron/team-media-sync` every 30 min
- 150ms throttle between rehost downloads
- `MAX_REHOSTS_PER_RUN=200` steady state
- Phase A correctly preserves `rehost_attempted_at` across runs
- Debug endpoints neutralized to 404 stubs

---

## What shipped this session

**Team Photo Sync v1** — full architecture in `projects/team-photo-sync.md`. Multi-day debugging through MLS Grid v2 quirks, PostgREST limits, TS errors, Vercel build SIGTERMs, and self-inflicted rate-limit bursts. Now in steady-state production.

**Commits (hgpg-cma-tool):**
- `aa628cf`, `9858872`, `cd8c630`, `3755b75`, `03be600`, `214828b`, `e8ecef2`, `2cceaa6`, `f286811`, `5c4ad6f`

**Frontend integration (hgpg-team-dash):**
- `f36f333` — hero photo fallback in `src/lib/inventory.ts`

**Brain docs:**
- `projects/team-photo-sync.md` — created
- `CONTEXT.md` — Team Photo Sync added to active, MLS Grid lessons added to standing rules
- `SESSION-HANDOFF.md` — this file

---

## Pickup for next session (low-priority cleanup)

1. **Investigate the 1 `download 400` row** at `media_key=68f0c902bf3305147c2ee8f2` (listing `CAR4311537`). Should self-resolve at 12:30 UTC. If not, manually trigger a refresh or mark as `permanent_fail`.

2. **Delete debug route files from the Mac via `gh`** — brain-app's `/api/external/commit` can only stub, not delete. Files:
   - `app/api/cron/team-media-debug/route.ts`
   - `app/api/admin/team-media-debug/route.ts`

3. **5 orphaned storage objects** at `mls-media-rehosted/CAR118472288/*.jpg` (no `carolina/` prefix, ~2 MB total). Delete via Supabase Storage API from Mac.

4. **Drop unused RPC** `public.team_media_sync_candidates` if no other callers.

5. **Update `rehost_error` preservation** — Phase A currently wipes `rehost_error` to NULL when restoring an error row. Should preserve it so debugging is easier. Edit `lib/mls/mediaSync.ts` finalRows logic to copy `rehost_error` from existingState. (This is what made me chase the post-mortem earlier — without it I couldn't tell whether an error row was 429 or 400.)

---

## Parked items (from earlier sessions)

- **Tammy Flores (FUB 31833) + Andrew Broughton (FUB 31885) outreach** — drafted, Brian to send
- **NC SMS Speed-to-Lead live e2e test** — parked since 2026-05-13
- **Sellers Guide FUB Automation 2.0** — verify on first real ad lead
- **Buyer Alerts code patch** — DRAFTED in `projects/buyer-alerts.md`, NOT committed
- **TM $395 fee toggle spec** — ready, awaiting go-ahead
- **Don's TM feedback** ID `84a2761a-f5b9-40a7-ba3a-f7923e9f1b3f` (5/13)
- **DocuSign migration** — build deadline 2026-06-13
- **1Password backup** of GitHub App private key

---

## Key learnings (carry forward)

1. **MLS Grid v2 has many undocumented limits.** See `projects/team-photo-sync.md` for the full catalog. Highlights: `$expand=Media` needs `$select` + `$top=50`. Property `$filter` only accepts 7 specific fields. Nested Media has wrong `ResourceRecordKey` and missing `OriginatingSystemName` — always override from parent context.

2. **MLS Grid media CDN (`media.mlsgrid.com`) has its own rate limit** (~7-10 RPS). Always throttle downloads (150ms is enough).

3. **MLS Grid signed URLs have ~24h TTL** but the timestamp is set at SOURCE time, not at fetch time. A photo that hasn't been re-indexed by MLS Grid in 24h will hand you an already-expired URL. The next Phase A refresh re-mints it.

4. **PostgREST `.in()` has URL length limits.** Chunk at 200 keys.

5. **Supabase `upsert()` clobbers ALL columns by default.** Preserve via lookup-and-merge for any field that must survive.

6. **Vercel build SIGTERM** = a route doing outbound HTTP at build time during prerender. Add `export const dynamic = 'force-dynamic';` everywhere.

7. **Burst-debugging crons is a foot-gun.** Hitting an endpoint 13x in 60s to "drain faster" recreates rate-limit problems. Trust the production schedule.

8. **Brain App's `/api/external/commit` cannot delete files** — only stub them. Real deletions need `gh` from the Mac.

9. **Python edit-scripts can silently fail to update sections** that don't exactly match the `old` string. Always verify with a fresh re-read after edits, especially when chaining multiple edits to the same file.
