<!-- Last Updated: 2026-05-15 -->

# Session Handoff

## Last session: 2026-05-15 (evening) — Team dashboard hero photo investigation + brain token rotation

### What shipped

- `projects/team-photo-sync.md` — added "Deferred polish: hero photo quality" section documenting why team dashboard hero photos look random (commit `edf57e0`). Root cause: Canopy strips `preferred_photo_yn` from MLS Grid feed so the fallback chain always lands on `order_num=0`, which is whatever the listing agent uploaded first into Matrix (often a render, floor plan, or aerial — not the curated primary photo). Parked, no agent complaints yet. Cheap fix path documented: `long_description` heuristic + `hero_media_key` override column.
- Brain write token persisted in Claude memory entry #12. Future sessions no longer need to ask Brian to paste it.
- Added memory entry #14 that explicitly supersedes the stale "Token in 1Password / Brian pastes" phrasing inside entry #13. Memory entry size limit (500 chars on user-edits) prevented rewriting #13 inline.

### Investigation findings (for context if revisiting)

Hero photo selection in `hgpg-team-dash/src/lib/inventory.ts` ranks `mls_media` rows by:
1. `preferred_photo_yn = true` first
2. then `order_num` ascending
3. then first available

All 5 active team listings have `preferred_photo_yn = null` on every photo. So step 1 is dead weight on Canopy. Examples confirmed via SQL on `wdheejgmrqzqxvgjvfee`:

- CAR267743419: hero = `long_description='Conceptual Render'`
- CAR294800544, CAR298348903, CAR300982442: all `order_num=0`, no description
- CAR293583134: 33 photos, **0 rehosted** — separate gap, will self-resolve as worker grinds or be a one-off
- CAR300982442: 35 photos, only 15 rehosted (also still draining)

### Carry-over from earlier 2026-05-15 work

Already reflected in `CONTEXT.md` and `projects/team-photo-sync.md`:

- Team Photo Sync v1 shipped (1,652/1,653 = 99.94%, all 4 active team listings have hero photos)
- hgpg-team-dash mobile pass shipped (commit `934c2204`, deploy `dpl_HkXATLEPfthmrNhk1pGZQXqMrV3V`)
- `mls_media` RLS policy added (was missing, silent empty result sets on team-dash)
- Resend custom SMTP wired into HGPG Listing Reports + MLS Supabase project (`wdheejgmrqzqxvgjvfee`) — 30/hr, sender `noreply@homegrownpropertygroup.com`. Brian's `last_sign_in_at = 2026-05-15 14:55:43 UTC` via the new path. `infrastructure.md` already updated.

### What got done in the afternoon cleanup pass

- ✅ Dropped unused `public.team_media_sync_candidates(text, integer)` RPC from HGPG Listing Reports + MLS (migration `drop_unused_team_media_sync_candidates_rpc`). Verified zero remaining functions matching the name.
- ✅ Verified the 5 orphaned storage objects at `mls-media-rehosted/CAR118472288/*.jpg` (top-level, no `carolina/` prefix) ARE genuinely orphaned. DB `mls_media` rows all point to the `carolina/`-prefixed dupes, which also exist in storage. Safe to delete from the Mac via Storage API (raw SQL blocked by `storage.protect_delete` trigger).
- ⚠️ `test.md` accidentally committed to root during API probe of `/api/external/write` shape — the endpoint doesn't support delete. File is neutralized to a TODO placeholder; needs `gh rm test.md` from Mac.

### Sellers Guide Meta campaigns launched (Ads project, 2026-05-15 PM)

Documented in detail in `marketing.md` under "Sellers Guide campaign (launched 2026-05-15)".

- TRAF campaign `52506271902963` ($10/day CBO) and CONV campaign `52506270109163` ($15/day CBO) live
- 12 ads total: Concept A + D in TRAF, Concept B + C in CONV, all 3 sizes per concept
- Pixel `861295553661596` + CAPI confirmed firing Lead events. Custom Conversion `HGPG - Sellers Guide Lead` (ID `972661382159071`) active.
- 5-zip geo radius across NC/SC border (Waxhaw, S. Charlotte, Indian Land, Fort Mill, Lancaster)
- HGPG - Ads project instructions revised to mirror Tech & Builds pattern (brain write API explicit, token embedded, session start/end discipline). Brian to paste into project settings.

3 soft fails parked for day-30 rebuild (logged in `marketing.md`):
1. CONV ad set optimization stuck on raw Lead pixel event - Meta blocks `promoted_object` after publish
2. CONV attribution stuck at 7-day click only - Meta blocks `attribution_spec` after publish
3. Both campaigns on CBO not ABO - functionally fine with 1 ad set, must switch when adding retargeting

Active Pixel issue (Tech & Builds): Lead event passes `value` (score) without `currency` - "Missing Lead Currency Parameter" warning. Fix: remove `value` from Lead event payload in the sellers guide site's pixel JS.

### Open / parked

**Cleanup leftovers (do from Mac):**

- Delete `test.md` from `hgpg-context` root: `gh rm test.md && git commit -m "Remove accidental test.md from 2026-05-15 API probe" && git push`
- Delete 5 orphaned storage objects via Supabase Storage API (recipe below).
- Delete the 2 debug route stubs in `hgpg-cma-tool` — `app/api/cron/team-media-debug/route.ts` and `app/api/admin/team-media-debug/route.ts`. Currently 404 stubs; can be fully removed.

**Deferred polish (no rush):**

- Hero photo quality on team dashboard — see `projects/team-photo-sync.md` "Deferred polish" section. Revisit when an agent flags a bad photo or doing Tab 1 polish anyway.

**Other parked items:**

- 1 `mls_media` row with download 400 on `CAR4311537` (Closed listing) — self-resolves next Phase A refresh
- Phase A doesn't preserve `rehost_error` across runs — nice-to-have fix in `lib/mls/mediaSync.ts`
- Tammy Flores + Andrew Broughton outreach drafts ready to send
- NC SMS Speed-to-Lead e2e test from Brian's real phone
- Buyer Alerts code patch
- TM $395 fee toggle
- DocuSign migration (6/13 deadline)
- 1Password backup of GitHub App private key
- **Revise sister project prompts** to match new Tech & Builds + Ads pattern (brain write API explicit, token embedded, CONTEXT.md + SESSION-HANDOFF.md fetch at start, SESSION-HANDOFF.md update at end). Pending: HGPG - Marketing & Content, HGPG - CRM & Leads.
- **Sellers Guide day-7 audit** (target 2026-05-22): Brian pulls Ads Manager last-7-days CSV broken out by ad set + creative, plus FUB lead count for `sellers-guide-2026` + `utm:meta`. Run audit prompt from Ads project.
- **Sellers Guide day-30 rebuild** (target 2026-06-14): Rebuild CONV ad set to clear 3 soft fails (optimization event, attribution, CBO->ABO). If audience pool >1,000, spin up retargeting ad set per launch plan.
- **Sellers Guide lander fix** (Tech & Builds owns): Remove `value` parameter from Lead event payload, OR add `currency: 'USD'`. Suppresses "Missing Lead Currency Parameter" warning. Cosmetic, not blocking.

### Storage API delete recipe (run from Mac)

    export SB_URL=https://wdheejgmrqzqxvgjvfee.supabase.co
    export SB_KEY="<service_role_key_for_wdheejgmrqzqxvgjvfee>"
    curl -X POST "$SB_URL/storage/v1/object/mls-media-rehosted" \
      -H "apikey: $SB_KEY" \
      -H "Authorization: Bearer $SB_KEY" \
      -H "Content-Type: application/json" \
      -d '{"prefixes":["CAR118472288/65abd5c90095d833e3795307.jpg","CAR118472288/65abd5c90095d833e3795308.jpg","CAR118472288/65abd5c90095d833e3795309.jpg","CAR118472288/65abd5c90095d833e379530a.jpg","CAR118472288/65abd5c90095d833e379530b.jpg"]}'

### Pickup notes for next session

- Brain is current. CONTEXT.md and infrastructure.md were already updated by an earlier session today — don't blindly rewrite them, fetch and diff first.
- Brain App write endpoint contract: `POST /api/external/write` with body `{"path": "...", "content": "...", "message": "..."}`. Required: `path`, `content`. Optional: `message`. Endpoint does NOT support file deletion — use `gh rm` from Mac. Empty content also rejected.
- Brain write token is now in Claude memory entry #12 — no need to ask Brian. Memory #14 supersedes stale "1Password / Brian pastes" phrasing in #13.
- Two MCP-able cleanups done earlier today; only Storage API delete + `gh rm` of test.md and 2 debug stubs remain manual.
