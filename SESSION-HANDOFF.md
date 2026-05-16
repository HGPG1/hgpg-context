<!-- Last Updated: 2026-05-16 -->

# Session Handoff

## Latest session: 2026-05-16 (morning) — PR-mode endpoint shipped 🟢

### What got built
- **`/api/external/pr` on brain-app** — multi-file PR open/update endpoint for any HGPG1 repo. Same bearer auth + same guardrails as `/api/external/commit`. Auto-generates branch name `claude/YYYY-MM-DD-<slug>` from PR title if not specified. If the branch + open PR already exist, additional commits append to it. Live at https://brain.homegrownpropertygroup.com/api/external/pr (commit `a9d057f` on brain-app main).
- **GitHub App permission flip**: HGPG Brain Commit App granted `Pull requests: Read and write` (was missing). App settings: https://github.com/settings/apps/hgpg-brain-commit. Brian approved the permission update.
- **Smoke test passed**: PR #45 on hgpg-cma-tool opened end-to-end via the endpoint. https://github.com/HGPG1/hgpg-cma-tool/pull/45 (close + delete branch when convenient — both throwaway).

### Endpoint shape

    POST https://brain.homegrownpropertygroup.com/api/external/pr
    Authorization: Bearer <BRAIN_WRITE_TOKEN>
    Content-Type: application/json
    {
      "repo": "hgpg-cma-tool",
      "title": "Fix packet narrative persist",
      "body": "Markdown PR description",
      "files": [
        { "path": "app/api/foo/route.ts", "content": "..." },
        { "path": "lib/foo.ts", "content": "..." }
      ],
      "branch": "claude/2026-05-16-my-fix",   // optional, auto-derived from title
      "base": "main",                          // optional, defaults to main
      "commitMessage": "..."                   // optional, defaults to title
    }

Response includes `prNumber`, `prUrl` (tappable on phone), `prState`, `branchCreated`, `created` (true if PR opened by this call, false if appended), and `commits[]`.

### New default workflow for Claude sessions

**For hgpg-context (brain repo)**: keep using `/api/external/write` (direct to main). Brain notes don't need a review gate.

**For app repos (hgpg-cma-tool, hgpg-transaction-manager, charlotte-sellers-guide-vercel, etc.)**: default to `/api/external/pr`. Direct-to-main via `/api/external/commit` is now reserved for:
- One-byte retriggers when Vercel misses a webhook
- Brian explicitly asks for direct push
- Emergency hotfix that can't wait for Brian to review on phone

Multi-commit debug cycles (like last night's 5-commit chase) should now all land on a single review branch instead of polluting production with diagnostic commits.

### Gotcha tracked tonight
**brain-app builds went ERROR x3** before I figured out the issue. TypeScript strict mode rejected `app.data.owner.login` because the Octokit response type has `owner` as a union (`SimpleUser | Enterprise`) where one variant lacks `login`. Stub-replaced the diagnostic route to unblock the build. Brain-app is back to green; `/api/external/app-info` returns 410. **Lesson: every brain-app endpoint Claude adds needs to pass `tsc` against the strict tsconfig, not just look right. If unsure, lean on `unknown` casts at the response boundary instead of indexing into Octokit response types.**

### Cleanup outstanding (tap-on-phone tasks for Brian)
- Close PR #45 on hgpg-cma-tool and delete the test branch (`claude/2026-05-16-test-pr-endpoint-safe-to-close`). Both safe to remove.
- Optional: clean up the test file from the branch first if you'd rather keep the branch for future reference. Either way works.

### Open / parked
- Drop `/api/debug/persist-test` route + `_cma_packet_debug` table once the packet fix has a week of stable use.
- Delete `lib/supabase.ts` (vestigial, points at wrong project) on the cma-tool — combine with the diag-route removal as one PR.
- **Original Candlestick anchor drop** — NOT A BUG, captured below. Auto-Find pulled fresh MLS data; two pendings (Spelman, Regal) closed lower than their pending list prices. Anchor moved $1,026,037 → $929,748 (-9.4%). Math correct. May 12 baseline `7cb7ef75` still in /history.
- Stack consolidation (parked 2-3 weeks from late April): dead concierge repo, deals → transactions migration, Title Case vs snake_case status mismatch, absorb tc-concierge into TM as /intake.
- MLS Grid API token from Canopy (Bridgett Bouvier, data@canopyrealtors.com).
- Sellers Guide Meta ads launch.
- TM deferred: addendum body block injection, per-party messaging capability tracking, calendar gate logic backfill.
- Agent TM onboarding Phase 3 (Ashley, Taylor, Brenda).

---

## Prior session: 2026-05-16 (late night) — CMA packet narrative persistence ACTUALLY fixed 🟢

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

### Architecture gotcha now documented
**The cma-tool Vercel project has TWO Supabase target patterns:**
- `SUPABASE_URL` + `SUPABASE_SERVICE_ROLE_KEY` → HGPG Listing Reports + MLS (wdheejgmrqzqxvgjvfee). Used by `/api/cron/mls-sync`, `/api/cron/team-media-sync`. Reads/writes `mls_property`, `team_media`, etc.
- `NEXT_PUBLIC_SUPABASE_URL` + `NEXT_PUBLIC_SUPABASE_ANON_KEY` → HGPG Core (ioypqogunwsoucgsnmla). Used by `lib/supabase-browser.ts`, `app/api/packet/seller/route.ts` (post-fix). Reads/writes `cma_reports`.

`lib/supabase.ts` (server, anon, no NEXT_PUBLIC prefix) is **vestigial** — points at SUPABASE_URL which is HGPG Listing Reports + MLS. Don't use it for cma_reports writes. Worth deleting on the next cleanup pass.

### Brain App / commit endpoint gotcha discovered
Commit `1d6b19b` via `/api/external/commit` didn't auto-trigger Vercel build. Pushing the same file content with a one-byte trailing-newline change as commit `f01119e` did trigger. Unclear why — possibly the GitHub App webhook silently dropped that specific push event. Workaround when an expected deploy doesn't appear: re-commit with any byte-level difference.

---

## Prior session (2026-05-06): Brain App MVP shipped 🟢
- New Vercel project `brain-app`, repo HGPG1/brain-app, live at brain.homegrownpropertygroup.com
- Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link), single-user lock
- Resend custom SMTP wired into HGPG Core Supabase, 30/hr (was 2/hr default), affects ALL apps on this Supabase: TM, CMA, TC Concierge, brain-app
- Supabase project renames for hygiene (Core, FUB Integration, Listing Reports + MLS, Signature + Relocation)
