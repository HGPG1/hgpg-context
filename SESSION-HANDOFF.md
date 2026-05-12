<!-- Last Updated: 2026-05-12 -->

# Session Handoff

## Last session: 2026-05-12 — Buyers Guide instrumentation (NeverBounce shipped, Pixel/CAPI unblocked)

### What got built

Pre-session belief: all three items (Meta Pixel, CAPI, NeverBounce) needed to be built from scratch. Reality on opening the repo:

- **Pixel browser + CAPI server code was already in repo on `main`** since late April: `api/_lib/meta.ts` (full SHA-256 hashed CAPI v21.0 helper), `api/fub-lead.ts` (CAPI fires after FUB success with shared `event_id`), `src/components/MetaPixel.tsx` (fbevents.js loader + PageView on route change), and Lead events wired in `UnlockModal`, `ContactDialog`, `GuideContext`. Live bundle had zero `fbq` references because `VITE_META_PIXEL_ID` was unset at build time → Vite tree-shook the dead path. Pixel + CAPI become live the moment env vars are provisioned and Vercel rebuilds.
- **NeverBounce was genuinely missing.** Shipped in commit `5c233ee` on `claude/buyers-guide-instrumentation-nGCLI` (pushed to origin in-session):
  - `api/validate-email.ts` — server-side proxy to NeverBounce v4 single/check, fails open on infra failures (missing key, upstream timeout, non-success) by returning `result: "unknown"` with a `server_error` tag
  - `src/lib/validateEmail.ts` — client helper, fail-closed on `invalid` / `disposable` only, fail-open on `unknown` / network failure
  - `UnlockModal` + `ContactDialog` updated to call `validateEmail()` BEFORE firing the Pixel Lead event or POSTing to `/api/fub-lead`, with inline error messaging and re-validation on email edit

Also fixed in same commit (pre-existing on `main`, blocking the `npm run build` gate but not Vercel since `vercel.json` runs `npx vite build` only):

- `src/pages/Bonuses.tsx` — `@/contexts/GuideContext` alias → relative path (tsconfig.app.json has no `paths` config)
- `src/pages/Calculator.tsx` — Recharts Tooltip formatter type widening

### Pixel ID + env vars decided this session

- **Pixel ID:** `1449157226505129` (dedicated to Buyers Guide per playbook — one pixel per site)
- **NeverBounce key:** reuse Sellers Guide value
- **Env vars to add on Vercel (Production scope on all):** `VITE_META_PIXEL_ID`, `META_PIXEL_ID`, `META_CAPI_ACCESS_TOKEN`, `META_CAPI_TEST_EVENT_CODE` (QA only, remove before scaling), `NEVERBOUNCE_API_KEY`, plus confirm existing `FUB_API_KEY`

### What's pending (Brian's hands, not the agent's)

1. Open PR at https://github.com/HGPG1/charlotte-buyers-guide/pull/new/claude/buyers-guide-instrumentation-nGCLI and merge (or merge directly to main).
2. Provision the 5-6 Vercel env vars above in Production scope.
3. Trigger a fresh production deploy (env-var change alone won't rebuild a Vite app — `VITE_*` vars bake in at build time).
4. Run post-deploy verification gates (exact curl commands in `projects/buyers-guide.md`).
5. End-to-end smoke: open Meta Events Manager Test Events with `META_CAPI_TEST_EVENT_CODE`, submit a lead, confirm browser + server dedup (single row, not two), confirm FUB person created with source `Charlotte Buyer's Guide`.
6. Once dedup is confirmed in Test Events: REMOVE `META_CAPI_TEST_EVENT_CODE` from Vercel env and redeploy. Leaving it on routes events to Test Events and blocks ad-spend scaling.

### Still pending after Brian's hands (separate workstreams)

- 🟡 **Manus 301 redirect** — original URL is `https://homegrownbg-hyqbjnnt.manus.space/?code=Cgvt4g5HuhfxNgdswHwqb4`. CANNOT be redirected from our Vercel project (the host doesn't resolve to us). Must be configured on Manus's side: in their app dashboard set the destination URL to `https://buyersguide.homegrownpropertygroup.com` and either take the app down with a forwarding URL or leave it as a redirect-only stub.
- 🟡 **Page-parity audit vs. Manus** — Brian raised this mid-session. Deferred because the Manus magic-link gate blocks automated comparison. Needs Brian's eyes on the live Manus site or screenshots. Current Vercel routes documented in `projects/buyers-guide.md`.

### Verification run locally in-session

- `npm run build` clean (after fixing the two pre-existing TS errors)
- `npx tsc --noEmit` clean
- With `VITE_META_PIXEL_ID=1449157226505129` at build time: bundle contains `fbq` (9 hits), `connect.facebook.net` (1), Pixel ID (1), `validate-email` (1). Note: don't grep for `neverbounce` in the client bundle — that string is server-side only; client-side signal is `validate-email`.

### Lessons noted

1. **Code-on-`main` does NOT equal shipped-in-production.** Pixel + CAPI commits sat on main for ~2 weeks while the live bundle had zero `fbq` references because Vite tree-shakes branches gated on undefined `import.meta.env.VITE_*` vars. Always verify against a built bundle (or hit prod) before assuming a feature is live.
2. **Build-gate divergence:** `vercel.json` ran `npx vite build` only, while `package.json` `build` ran `tsc -b && vite build`. TS errors landed on main without breaking production. Either align them or run `npm run build` locally before merging.
3. **Scope creep watch:** mid-session widened from "ship 3 instrumentation items" to "audit page parity vs. Manus." Pushed back, kept original scope, deferred parity to a follow-up. Verification: instrumentation actually shipped end-to-end (commit + gates green); parity didn't get half-built and abandoned.
4. **Sandbox push works** for at least the Buyers Guide repo. Stale assumption from prior kickoff prompts corrected. Future kickoff prompts should say "sandbox push may work — try once" rather than "sandbox cannot push."

---

## Previous session: 2026-05-11 — Brain reconciliation + transaction-pdfs cleanup + project list pass

### What got done

Reconciled CONTEXT.md against project files (which were ahead), closed six items:

1. **Sherlock 403 on Transaction Manager** — was already marked closed in `projects/transaction-manager.md` on 2026-05-09. Fix was likely security/auth related; no specifics logged. If recurs, start with API key scope.
2. **Mac Mini GitHub auth / Listing Report Portal** — already 🟢 in project file (2026-05-06 favicon push verified).
3. **Exposed GitHub PAT rotation** — closed. Fine-grained brain-app PAT (scoped to `hgpg-context`, contents:write only) superseded the broad old token.
4. **CMA Engine MLS Grid auto-pull** — already marked mature in `projects/cma-engine.md` on 2026-05-09.
5. **NC office routing in ReZEN builder** — **verified live**. Code at `app/api/rezen/create-transaction/route.ts:160` uses `txn.state?.toUpperCase() === "NC" ? REZEN_OFFICE_ID_NC : REZEN_OFFICE_ID`. Both env vars confirmed on Vercel Production.
6. **transaction-pdfs bucket flip → private** — fully closed. Investigation revealed bucket was flipped via MCP on 2026-05-09 (`schema_migrations` version `20260509173839`), code on main since PR #7 (`e45e3c8`), but the `.sql` migration file was never committed alongside the code. Re-applied idempotent migration via MCP (`20260511210146`), staged file, Brian committed and pushed from Mac (`2eb9794`). Bucket, DB ledger, repo file, and code all aligned.

### Brain hygiene fixes

- Rebuilt `projects/brain-app.md` (had been clobbered with handoff-style content earlier — restored as proper project spec with full Write API documentation)
- Updated CONTEXT.md "active right now" to include FUB AI Agent, Sellers Guide, Buyers Guide, Brain App
- **Buyers Guide false-close averted:** during recap, Brian believed Pixel + CAPI + NeverBounce were done. Live-site grep against `buyersguide.homegrownpropertygroup.com` proved otherwise. Items remain open. Saved drift in the wrong direction. (Now actioned on 2026-05-12.)

### Open thread
- `~/Downloads/fix-admin-page.sh` may have unpushed UI changes for Listing Report Portal (SocialPostsManager dark navy headings, MagicLinkCard regenerate placement, admin reorder, zero-value stat hiding). Check next time portal is touched.

### Brain hygiene notes

Three lessons worth keeping:

1. **CONTEXT.md drifts** when project files get updated independently. Worth running a reconciliation pass every couple weeks.
2. **Migrations applied via MCP need their .sql files committed too.** The DB ledger (`schema_migrations`) and the repo (`supabase/migrations/`) are two separate sources of truth.
3. **Don't trust assertions about "this is done" without verifying against the live system.** Cost of verification is near-zero; cost of closing real work as done is real.

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
- Resend custom SMTP wired into `HGPG Core` Supabase
  - Sender: `noreply@homegrownpropertygroup.com`, name: HGPG
  - Rate limit 2/hr → 30/hr
  - Affects ALL apps using this Supabase: TM, CMA, TC Concierge, brain-app
- Supabase project renames for hygiene
- Supabase `HGPG Core` redirect URLs added for `brain.homegrownpropertygroup.com/**` and `http://localhost:3000/**`
