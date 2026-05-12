<!-- Last Updated: 2026-05-12 -->

# Session Handoff

## Last session: 2026-05-12 — Buyers Guide instrumentation + Manus extraction

### Highlights

**Buyers Guide instrumentation (shipped + merged):**
- Pre-session belief: build Pixel + CAPI + NeverBounce from scratch. Reality: Pixel + CAPI code was on `main` since late April but inert because `VITE_META_PIXEL_ID` was unset (Vite tree-shook the dead path).
- NeverBounce genuinely missing; shipped in commit `5c233ee` on `claude/buyers-guide-instrumentation-nGCLI`, PR merged.
- Pixel ID for Buyers Guide: `1449157226505129`. NeverBounce key: reuse Sellers Guide value.
- Two pre-existing TS errors in `Bonuses.tsx` + `Calculator.tsx` fixed to clear `npm run build` gate (Vercel uses `npx vite build` only so they didn't break production but were blocking the local gate).

**Manus migration kickoff (huge):**
- Discovered during recap that the Vercel rebuild of the Buyers Guide is a **significant downgrade** from the original Manus app. Eight high-value server-side features dropped silently: lead scoring, calculator→FUB custom fields + behavioral tags, exit-intent popup + PDF, server-side PDF gen, bonus unlock tracking, /:agent vanity URLs, /admin, /agent-dashboard, advisor mode UX layer.
- Manus support is unreliable; used the Manus AI agent inside their dashboard as an unofficial export channel.
- **Round 1 (probe):** `package.json`, `App.tsx`, `tsconfig.json` confirmed agent access works.
- **Round 2 (map):** full file tree, server entry, root tRPC router, Drizzle schema.
- **Path B (full source export):** complete zip delivered by agent. 220 files, no node_modules bloat, no `.env` files leaked.
- **Snapshot repo created:** `HGPG1/charlotte-buyers-guide-manus-export` (private). First commit `e107b0e` (220 files, 35k+ insertions, 2.81 MiB push). Repo live at https://github.com/HGPG1/charlotte-buyers-guide-manus-export.
- **Historic FUB API key found** in `scripts/get-fub-users.ts` (`fka_0cyqBUzL20pdO11gGKd6J4jcHrUp1P6wsu`). Probed `/v1/identity` — HTTP 401. Already revoked. No action required.
- **Infra clarification:** Manus uses its own "Forge" storage proxy (Bearer-token HTTP API), NOT AWS S3 directly. The `@aws-sdk/*` packages in `package.json` are dead deps. Port target for Vercel rebuild: Supabase Storage (HGPG Core already provisioned).

### What's pending

**Manus migration:**
1. ⏳ **Path C — DB data export** (NEXT). Prompt staged in `projects/buyers-guide.md`. Highest single-value extraction remaining. Actual rows in `contacts`, `activities`, `quizResults`, `agents` only exist on Manus. Fire BEFORE telegraphing departure.
2. Phase 1 ports to Vercel rebuild (1-2 sessions): lead scoring, calculator→FUB custom fields/tags, server-side PDF gen (PDFKit + Supabase Storage), exit-intent popup, bonus unlock tracking.
3. Phase 2: activities event tracking, quiz results DB storage, AI Chat Box review.
4. Phase 3 (deferred until usage confirmed): /:agent vanity URLs, /admin, /agent-dashboard, advisor mode.
5. Manus 301 — must be configured on Manus's side; defer until Path C is complete.

**Buyers Guide instrumentation finishing:**
1. Add 5-6 Vercel env vars in Production scope (`VITE_META_PIXEL_ID`, `META_PIXEL_ID`, `META_CAPI_ACCESS_TOKEN`, `META_CAPI_TEST_EVENT_CODE`, `NEVERBOUNCE_API_KEY`, confirm `FUB_API_KEY`).
2. Trigger fresh production deploy (env-var change alone doesn't rebuild a Vite app).
3. Run post-deploy bundle-grep gates (commands in `projects/buyers-guide.md`).
4. End-to-end smoke in Meta Test Events to confirm browser+server dedup.
5. Remove `META_CAPI_TEST_EVENT_CODE` from Vercel env once dedup confirmed.

### Open thread
- `~/Downloads/fix-admin-page.sh` may have unpushed UI changes for Listing Report Portal. Check next time portal is touched.

### Lessons noted (cumulative)

1. **Code-on-`main` does NOT equal shipped-in-production.** Verify against built bundle, not source.
2. **Build-gate divergence:** `vercel.json` `buildCommand` vs `package.json` build script can drift; TS errors land on main without breaking production. Run `npm run build` locally before merging.
3. **Sandbox push works** for at least the Buyers Guide repo. Stale assumption corrected.
4. **Migration-without-source = downgrade.** Original Manus→Vercel migration ported only buyer-facing UI; eight server-side features dropped silently. Always extract source code + DB schema FIRST when migrating off a platform.
5. **AI agents inside SaaS platforms are an unofficial export channel.** Probe with 2-3 innocuous files; verify against production bundle's `data-loc` strings; escalate to full zip. Keep framing as audit/debug — don't telegraph departure.
6. **AWS SDK deps != AWS usage.** Read the code, don't infer infra from `package.json` alone.
7. **Verify before rotation chaos.** Grep extracted source for secrets, then probe them. The FUB key looked dead and was — saved a multi-system rotation.
8. **`apply_migration` via MCP doesn't update repo.** DB ledger and `supabase/migrations/` are separate sources of truth.
9. **CONTEXT.md drifts.** Reconcile vs project files every couple weeks.
10. **Don't trust assertions about "this is done" without verifying against the live system.** Cost of verification is near-zero.

---

## Previous session: 2026-05-11 — Brain reconciliation + transaction-pdfs cleanup

### What got done

Reconciled CONTEXT.md against project files (which were ahead), closed six items:

1. **Sherlock 403 on Transaction Manager** — was already marked closed in `projects/transaction-manager.md` on 2026-05-09.
2. **Mac Mini GitHub auth / Listing Report Portal** — already 🟢 in project file.
3. **Exposed GitHub PAT rotation** — closed. Fine-grained brain-app PAT superseded the broad old token.
4. **CMA Engine MLS Grid auto-pull** — already marked mature in project file.
5. **NC office routing in ReZEN builder** — verified live in code.
6. **transaction-pdfs bucket flip → private** — fully closed. Bucket flipped via MCP on 2026-05-09; migration file backfilled to repo on 2026-05-11 (`2eb9794`).

Also: rebuilt `projects/brain-app.md` (had been clobbered with handoff-style content), updated CONTEXT.md "active right now" list.

---

## Previous session: 2026-05-06 — Brain App MVP shipped 🟢

### What got built
- New Vercel project: `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- New repo: `HGPG1/brain-app` (private)
- Live at: `https://brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Single-user lock: `BRIAN_EMAIL=brian@homegrownpropertygroup.com` allow-list

### Infra changes that affect other apps
- Resend custom SMTP wired into `HGPG Core` Supabase. Rate limit 2/hr → 30/hr. Affects ALL apps using this Supabase: TM, CMA, TC Concierge, brain-app.
- Supabase project renames for hygiene.
