<!-- Last Updated: 2026-05-11 -->

# Session Handoff

## Last session: 2026-05-11 — CMA Enhancement 1 shipped (autosave on /seller/adjust + presented flip on PDF export)

### What got built
- `/seller/adjust` now autosaves on every edit (rate, feature rate, line toggle, condition tier, agent-include, swap). 500ms debounce, 3-attempt exponential-backoff retry (250/500/1000ms), re-arms on the next edit if all retries are exhausted.
- New `Draft — Saving / Saved HH:MM / Save failed` banner under the page eyebrow. Always visible while on /seller/adjust.
- First save on a fresh CMA INSERTs and captures the returned id back into the WorkingSet; subsequent autosaves UPDATE the same row. Continue-to-Packet now flushes the in-flight save instead of running its own duplicate.
- Seller, buyer, and appraiser packet pages each flip `cma_reports.status` from `draft` → `presented` after a successful PDF download. `presented` stays canonical for the post-packet value (matches existing `CmaReportStatus` enum + History UI dropdown).
- No schema migration. `cma_reports` already had `status text NOT NULL DEFAULT 'draft'` and an open RLS policy in HGPG Core (`ioypqogunwsoucgsnmla`) — writes worked already, only the autosave loop was missing.

### Shipped
- PR #34 (`feat-autosave-adjust`) merged. Commit `701ed86`. Vercel deploy `dpl_46pzX7iRwpYmrVETWzWrVcignX6t` READY in 29s. Live at cma.homegrownpropertygroup.com.

### Project-file dual location of cma_reports — record for future sessions
- `cma_reports` lives in HGPG Core Supabase (`ioypqogunwsoucgsnmla`), NOT the MLS project (`wdheejgmrqzqxvgjvfee`). The CMA app uses both: MLS for comp search via `searchComps()`, Core for reports + adjustment defaults. Browser autosave uses the `NEXT_PUBLIC_SUPABASE_URL` env var on Vercel, which points at Core. Local `.env` / `.env.local` set the server-side `SUPABASE_URL` to MLS for the IDX cache; don't be fooled into thinking that's the reports DB.
- RLS on `cma_reports` is wide open (`cma_reports_open_all`, `cmd=ALL`, `USING true`, `WITH CHECK true`). PIN-auth gates access at the app layer.

### Open thread
- Hardcoded test: open Cressingham 5022 draft, bump a slider, confirm banner cycles Saving → Saved within 1-2s, reload, verify the slider sticks. Then click Export PDF and confirm `cma_reports.status` flips draft → presented (visible in History dropdown).
- `beforeunload` prompt is best-effort only. Browsers don't await async work on unload — if a tab is killed mid-debounce (within 500ms of the last edit), the in-progress edit may not land. The autosave covers everything older than 500ms.
- 43 existing rows in `cma_reports` are all `status='draft'` even though many have completed narratives. Per session decision, NOT backfilling them to `presented` — the historical drafts were saved by Continue-to-Packet but never had a PDF exported, so `draft` is honest.

### Pickup notes for next session
- If autosave starts thrashing under network instability, look at `app/seller/adjust/page.tsx` — `inFlightRef`, `retryAttemptRef`, and the `editVersion` counter together gate concurrency.
- The Continue-to-Packet handler now awaits an in-flight save before navigating. If it ever hangs, check the `while (inFlightRef.current)` loop — it sleeps 50ms per tick.
- New code uses `compIdOf` from `lib/cma/adjustments.ts` for keying `pqsByCompId` lookups. Verified identical to the `compIdOf` in `lib/cma/pqs.ts` — both stringify `listingID ?? idxID ?? address`.

---

## Earlier session: 2026-05-11 — Brain drift cleanup (round 2)

### What got done
- **Sherlock 403** on Transaction Manager confirmed resolved (was already marked closed in `projects/transaction-manager.md` on 2026-05-09, but CONTEXT.md still listed it as a blocker). Fix was likely security/auth related — no specifics logged; if it recurs, start with API key scope.
- **Mac Mini GitHub auth / Listing Report Portal** confirmed resolved (project file already had status 🟢 Live, GitHub auth blocker resolved as of 2026-05-06 favicon push). CONTEXT.md still flagged it.
- `CONTEXT.md` updated: both items moved from blockers to recently completed. Last-updated bumped to 2026-05-11.

### Open thread
- `~/Downloads/fix-admin-page.sh` was flagged on the Listing Report Portal project file as possibly having unpushed UI changes (SocialPostsManager dark navy headings, MagicLinkCard regenerate placement, admin section reorder, zero-value stat hiding). Not verified this session. Check next time the portal is touched.

### Pickup notes for next session
- Brain index now in sync with project files. Drift creeps in when project files update but CONTEXT.md does not — periodic reconciliation passes are worth doing.
- Remaining items on the board:
  - **Exposed GitHub PAT rotation** (security hygiene)
  - **`transaction-pdfs` bucket flip to private** — PR not yet opened on branch `claude/transaction-pdfs-private-AqXkA` (see `projects/transaction-manager.md` for full order of operations)
  - **CMA Engine MLS Grid auto-pull** (active build)
  - **NC office routing verification** in ReZEN builder (open item in TM)
  - **.net Google Workspace migration to .com** (do not proactively remind)

---

## Previous session: 2026-05-06 — Brain App MVP shipped 🟢

### What got built
- New Vercel project: `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- New repo: `HGPG1/brain-app` (private)
- Live at: `https://brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Single-user lock: `BRIAN_EMAIL=brian@homegrownpropertygroup.com` allow-list
- GitHub auth: fine-grained PAT scoped to `HGPG1/hgpg-context`, contents:write only
- Round-trip verified: edit file in browser → commit lands on `main` with author `brian@homegrownpropertygroup.com`

### Infra changes that affect other apps
- Resend custom SMTP wired into `HGPG Core` Supabase (project `ioypqogunwsoucgsnmla`)
  - Sender: `noreply@homegrownpropertygroup.com`, name: HGPG
  - API key stored under "Supabase HGPG Core" in Resend
  - Rate limit went from 2/hr (Supabase default) to 30/hr (Resend default), can be raised
  - This affects ALL apps using this Supabase: TM, CMA, TC Concierge, brain-app
- Supabase project renames for hygiene
- Supabase `HGPG Core` redirect URLs added for `brain.homegrownpropertygroup.com/**` and `http://localhost:3000/**`

### Deferred / Phase 2 for brain-app
- iPhone smoke test (CodeMirror + iOS soft keyboard scroll behavior)
- Cooper Hewitt self-hosted (currently falling back to system sans)
- File rename and delete
- Diff view before save
- Cross-file search
