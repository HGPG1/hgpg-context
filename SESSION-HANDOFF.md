<!-- Last Updated: 2026-05-16 -->

# Session Handoff

## Last session: 2026-05-16 (late) — CMA packet narrative persistence ACTUALLY fixed 🟢

### What got built (commit chain on hgpg-cma-tool main)
- `cd36dee` — first attempt: maxDuration=120, force-dynamic, server-side persist using SUPABASE_URL/SERVICE_ROLE. **Didn't work** — those env vars on cma-tool point at HGPG Listing Reports + MLS (set up for mls-sync cron), not HGPG Core where cma_reports lives.
- `f0a7683`, `b1c435d` — diagnostic versions logging full PostgREST error fields. Still couldn't see them due to Vercel log UI truncation.
- `f01119e` — standalone `/api/debug/persist-test` GET endpoint that returns env-check + supabase-js error + raw fetch result + read-back. **This is what cracked it.** Returned PGRST205 "Could not find table cma_reports in schema cache" pointing at the wrong project.
- `8572dea` — **the fix.** Switched persist to `NEXT_PUBLIC_SUPABASE_URL` + `NEXT_PUBLIC_SUPABASE_ANON_KEY` (those target Core, where cma_reports + cma_reports_open_all RLS policy live). Anon key against RLS-open table works identically to service role for this write.

### Verification (2026-05-16 02:51-02:55 UTC / 10:51-10:55 PM ET)
Two consecutive successful runs on mobile against new Candlestick draft `b755f9cd`:
- narrative_generated_at populated
- recommended_price $930,000
- has_narrative=true

Server-side persist confirmed working on iPhone Safari mid-session, which was the original failure mode (mobile fetch abort dropping the client-side useSaveNarrative hook).

### Why the previous "shipped" handoff was wrong
My prior SESSION-HANDOFF write claimed the fix landed with commit cd36dee. It didn't — cd36dee was using the wrong Supabase project's env vars. The brain got an over-confident report before real-world verification. **Lesson: don't write "shipped" to the brain until at least one real mobile retry actually persists.** I am rewriting this handoff after Brian confirmed it worked.

### Architecture gotcha now documented
**The cma-tool Vercel project has TWO Supabase target patterns:**
- `SUPABASE_URL` + `SUPABASE_SERVICE_ROLE_KEY` → HGPG Listing Reports + MLS (wdheejgmrqzqxvgjvfee). Used by `/api/cron/mls-sync`, `/api/cron/team-media-sync`. Reads/writes `mls_property`, `team_media`, etc.
- `NEXT_PUBLIC_SUPABASE_URL` + `NEXT_PUBLIC_SUPABASE_ANON_KEY` → HGPG Core (ioypqogunwsoucgsnmla). Used by `lib/supabase-browser.ts`, `app/api/packet/seller/route.ts` (post-fix). Reads/writes `cma_reports`.

`lib/supabase.ts` (server, anon, no NEXT_PUBLIC prefix) is **vestigial** — points at SUPABASE_URL which is HGPG Listing Reports + MLS. Don't use it for cma_reports writes. Worth deleting on the next cleanup pass.

### Diagnostic infrastructure still in place
- `/api/debug/persist-test` GET — still deployed on cma-tool. Health-check endpoint that exercises a test cma_reports UPDATE and returns full env + error context. Useful for future Supabase-target debugging. Should be removed once we're confident the fix is stable.
- `_cma_packet_debug` table in HGPG Core — created during diagnosis, never received a row because all the failed writes were going to the wrong project. Worth dropping next session.

### Brain App / commit endpoint gotcha discovered
Commit `1d6b19b` via `/api/external/commit` didn't auto-trigger Vercel build. Pushing the same file content with a one-byte trailing-newline change as commit `f01119e` did trigger. Unclear why — possibly the GitHub App webhook silently dropped that specific push event. Workaround when an expected deploy doesn't appear: re-commit with any byte-level difference.

### Open / parked
- **PR-mode endpoint for the brain commit API** — still parked. Tonight three commits went straight to main on a production tool. Worked out, but the next bigger change should not. Roughly: create branch ref, commit file to branch, open PR. ~70 lines, one new file at `app/api/external/pr/route.ts` on brain-app. Self-hosts via existing commit endpoint.
- Drop `/api/debug/persist-test` route + `_cma_packet_debug` table once the fix has a week of stable use.
- Delete `lib/supabase.ts` (vestigial, points at wrong project) and any leftover imports.
- **Original Candlestick anchor drop** — NOT A BUG, captured in prior handoff. Auto-Find pulled fresh MLS data; two pendings (Spelman, Regal) closed lower than their pending list prices. Anchor moved $1,026,037 → $929,748 (-9.4%). Math correct. May 12 baseline `7cb7ef75` still in /history.
- Stack consolidation (parked 2-3 weeks from late April): dead concierge repo, deals → transactions migration, Title Case vs snake_case status mismatch, absorb tc-concierge into TM as /intake.
- MLS Grid API token from Canopy (Bridgett Bouvier, data@canopyrealtors.com).
- Sellers Guide Meta ads launch.
- TM deferred: addendum body block injection, per-party messaging capability tracking, calendar gate logic backfill.
- Agent TM onboarding Phase 3 (Ashley, Taylor, Brenda).

### Prior session (2026-05-06): Brain App MVP shipped 🟢
- New Vercel project `brain-app`, repo HGPG1/brain-app, live at brain.homegrownpropertygroup.com
- Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link), single-user lock
- Resend custom SMTP wired into HGPG Core Supabase, 30/hr (was 2/hr default), affects ALL apps on this Supabase: TM, CMA, TC Concierge, brain-app
- Supabase project renames for hygiene (Core, FUB Integration, Listing Reports + MLS, Signature + Relocation)
