# SESSION-HANDOFF — 2026-05-15

## What shipped this session

**Team Photo Sync v1 is live** (`projects/team-photo-sync.md`). After a multi-day debugging odyssey through cascading MLS Grid v2 quirks, PostgREST limits, TS errors, Vercel build SIGTERMs, and self-inflicted rate-limit bursts, the system is now in steady-state production.

**Production state at handoff (2026-05-15 10:41 UTC):**
- 1,591 of 1,653 team Media records rehosted (96.2%)
- 0 pending (all 61 download-429 errors reset to pending; 11:00 UTC cron will retry)
- 1 permanent error (`download 400` on a single broken MLS Grid URL)
- 3 of 4 active team listings have hero photos (`CAR4222460`, `CAR4341869`, `CAR4374686`)
- `CAR4336731` is the one without hero — all 33 photos errored on 429 bursts, all reset to pending

**Architecture:**
- `/api/cron/team-media-sync` every 30 min, two-phase (sync → rehost)
- 150ms throttle between rehost downloads (MLS Grid media CDN limits ~7-10 RPS)
- `MAX_REHOSTS_PER_RUN=200` for steady state (was 500 during backfill)
- Public Supabase Storage at `mls-media-rehosted/carolina/<listing_key>/<media_key>.jpg`

**Commits this session (hgpg-cma-tool):**
- `aa628cf`, `9858872`, `cd8c630`, `3755b75`, `03be600`, `214828b`, `e8ecef2`, `2cceaa6`, `f286811`, `5c4ad6f`
- Full details in `projects/team-photo-sync.md`

**Frontend integration:** `hgpg-team-dash` `f36f333` — hero photo fallback in `src/lib/inventory.ts` (list + detail views).

---

## Pickup for next session

### Immediate (verify backfill completion)

1. **Verify all 4 active listings have hero photos.** Hit `team.homegrownpropertygroup.com/inventory`. After the 11:00 UTC cron (post-handoff), `CAR4336731` should have its 33 photos rehosted with the new 150ms throttle preventing 429s. If not, check the 11:30 UTC cron.

2. **Investigate the 1 permanent `download 400`** in `mls_media` (`CAR288651827` listing). Likely a broken/expired URL. Could add a `permanent_fail` status or just leave it as `error` forever (it'll never retry after the first error since `rehost_error='download 400'` is preserved).

### Cleanup (low priority)

3. **Delete debug route files from the Mac via `gh`** — the brain-app commit endpoint can only stub them, not remove. Files:
   - `app/api/cron/team-media-debug/route.ts` (currently a 404 stub)
   - `app/api/admin/team-media-debug/route.ts` (currently a 404 stub)

4. **5 orphaned storage objects** at `mls-media-rehosted/CAR118472288/*.jpg` (~2 MB, no `carolina/` prefix). From the early failed-path bug. Delete via Supabase Storage API from the Mac (SQL DELETE is blocked by `storage.protect_delete()` trigger).

5. **Drop unused RPC** `public.team_media_sync_candidates(p_list_office_key text, p_limit int)` if not referenced anywhere.

### On the horizon (from earlier sessions, still parked)

- **Tammy Flores (FUB 31833) + Andrew Broughton (FUB 31885) outreach** — drafted, Brian to send
- **NC SMS Speed-to-Lead live e2e test** — parked since 2026-05-13, can resume anytime
- **Sellers Guide FUB Automation 2.0** — verify on first real ad lead
- **Buyer Alerts code patch** — DRAFTED in `projects/buyer-alerts.md`, NOT committed. Replace `sendLoopMessage` in `src/app/api/cron/buyer-alerts/route.ts` to match TM canonical `lib/loopBridge.ts`.
- **TM $395 fee toggle spec** — ready, awaiting Brian's go-ahead
- **Don's TM feedback ID** `84a2761a-f5b9-40a7-ba3a-f7923e9f1b3f` — 2026-05-13, mid-deal seller signing 5/26 3PM Lando Law without changing closing date
- **DocuSign migration** — build deadline 2026-06-13
- **1Password backup** of the `HGPG Brain Commit` GitHub App private key (currently at `~/.hgpg-secrets/github-app-private-key.pem` on Brian's Mac + Vercel env)

---

## Key learnings to internalize

1. **MLS Grid v2 has many undocumented limits.** `$expand=Media` needs `$select` to stay under response-size cap. Property `$filter` only accepts ~7 specific fields. Nested Media has wrong `ResourceRecordKey` and missing `OriginatingSystemName` — always override from parent context.

2. **MLS Grid media CDN is a separate service with its own rate limit** (~7-10 RPS). Tight for-loops doing downloads will trigger 429s. Always throttle between downloads.

3. **PostgREST `.in()` has a URL length limit.** Chunk lookups at 200 keys for safety.

4. **Supabase `upsert()` clobbers all columns.** Preserve existing state via lookup-and-merge before upserting if any columns need to survive.

5. **Vercel build SIGTERM** = a route is doing outbound HTTP at build time during prerender. Add `export const dynamic = 'force-dynamic';` to every route that fetches external data, including debug routes.

6. **Burst-debugging is its own foot-gun.** Hitting a cron endpoint 13 times in a minute to "drain faster" recreated the exact rate-limit problem I was trying to debug. Trust the production cron's natural pacing.

7. **Brain App's GitHub-App-backed `/api/external/commit` endpoint cannot delete files** — only stub them. Real deletions need `gh` from the Mac.

8. **Don't try to fix while context is running thin.** I attempted to fix the preservation logic late in the session when the build error was a simple TS type miss. A clean session would have caught it in the first attempt.
