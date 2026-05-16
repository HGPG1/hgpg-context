<!-- Last Updated: 2026-05-15 -->

# Session Handoff

## Last session: 2026-05-15 — CMA packet narrative persistence fix shipped 🟢

### What got built
- Diagnosed mobile "Load failed" issue in CMA packet generation. Two issues conflated:
  1. **Anchor shifted on Candlestick re-run** — NOT A BUG. Auto-Find pulled fresh MLS data; Spelman Drive ($875K pending → $845K closed) and Regal Way ($925K pending → $915K closed) had closed lower than their pending list prices, and McPherson Street + Sturminster Drive joined the comp set. Result: anchor moved from $1,026,037 (May 12 baseline `7cb7ef75`) to $929,748 (new draft `4fffae6f`), -$96,289 / -9.4%. Math doing its job. May 12 baseline still in /history as status=presented.
  2. **Packet "Load failed" was real** — `/api/packet/seller` returned 200 server-side both times Brian retried, but `cma_reports.narrative` and `narrative_generated_at` stayed null. Root cause: persistence happens client-side via `useSaveNarrative` AFTER `await res.json()`. If browser drops connection mid-response (mobile Wi-Fi handoff, tab backgrounded, Safari "Load failed"), narrative never persists. Function also had no `export const maxDuration` so was getting Vercel platform default (60s) while Anthropic client was configured for 120s — likely contributing factor.

### Fix shipped: commit cd36dee on hgpg-cma-tool main
- `app/api/packet/seller/route.ts`:
  - Added `export const maxDuration = 120;` `export const dynamic = "force-dynamic";` `export const runtime = "nodejs";`
  - Added server-side persistence: when `workingSet.reportId` is present, route writes narrative + strategy_prices + recommended_price to `cma_reports` BEFORE returning response, using service-role Supabase client (same env pattern as `/api/cron/mls-sync`). Best-effort: failures logged, client-side useSaveNarrative still fires on successful response. Belt + suspenders.
- Deployed: dpl_DXM8xK7F5Rb2Jae3ydGHrYfa8ArJ, ready 23:39 ET (03:39 UTC)
- Pushed via Brain App `/api/external/commit` endpoint (Claude session has write access to any HGPG1 repo via GitHub App credential)

### Discovered tonight
- Brain App ships THREE external endpoints (not just write):
  - `/api/external/read` — POST or GET, reads any HGPG1 repo file. Optional `ref` for branch/sha.
  - `/api/external/write` — POST, hgpg-context only (legacy compat).
  - `/api/external/commit` — POST, ANY HGPG1 repo. Uses `BRAIN_WRITE_TOKEN`. `repo` + `path` + `content` + optional `message` + optional `branch` (defaults main).
- All three endpoints share one bearer token in env var `BRAIN_WRITE_TOKEN`. Memory entry on the token is in chat history.
- The commit endpoint auto-deploys via Vercel push-to-main → Claude can ship Vercel-deployed code changes end-to-end without Brian's gh CLI for small surgical patches.
- `ALLOWED_REPOS` env var can scope the endpoint to a specific list if needed (currently empty = all HGPG1 repos).

### Pickup notes for next session
- Verify the fix lands narrative on next live Candlestick or fresh CMA run. Expected: `cma_reports.narrative IS NOT NULL` and `narrative_generated_at IS NOT NULL` even on a "Load failed" client experience.
- Vercel runtime logs for `/api/packet/seller` will now include `[/api/packet/seller] server-side narrative persisted for report <id>` log lines on success, or warnings on persistence failure.

### Open / parked
- **PR-mode endpoint for the brain commit API**: tonight's push went straight to main. Decision was tonight's fix is small and surgical enough for direct-main, but for bigger changes (CMA math, TM messaging, anything not 1-file surgical) we want a PR path. Roughly: create branch ref, commit file to branch, open PR. New endpoint at `/api/external/pr` on brain-app. ~70 lines. Next session: draft it, self-host it via the existing commit endpoint (commit it to brain-app via /api/external/commit), Vercel deploys, future Claude sessions default to PR mode for app repos and direct mode for hgpg-context.
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
