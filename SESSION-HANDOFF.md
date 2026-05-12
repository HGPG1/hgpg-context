<!-- Last Updated: 2026-05-12 -->

# Session Handoff

## Next session queued: Manus → Vercel full migration, Session 1 (backend infrastructure)

Full 4-session migration plan is staged at `projects/buyers-guide-manus-migration.md`. Brian opening a new chat thread for execution (this thread is getting long).

**Total scope:** ~6-10 hours / 4 sessions / 4-6 prompts.

**Session 1 (next):** ~2.5 hrs. Backend infrastructure only. Supabase tables (bg_*), seed agents, Storage bucket, server helpers in `api/_lib/`, FRED + FUB + Storage smoke tests. No buyer-facing UI changes.

**Session 1 kickoff prompt:** verbatim in `projects/buyers-guide-manus-migration.md` under "Session 1 — Backend infrastructure." Copy-paste ready.

**Sessions 2-4 prompts** also staged verbatim in the same doc.

### Pre-session prereqs Brian needs to handle BEFORE Session 1

1. **FRED API key** — sign up at https://fred.stlouisfed.org/docs/api/api_key.html (free, 10 min). Save somewhere accessible.
2. **Confirm Manus production decision** — Brian's call from 2026-05-12: NO 301 redirect. After all 4 sessions done, Manus app gets unpublished/deactivated on their side. Source is permanently safe in `HGPG1/charlotte-buyers-guide-manus-export`.
3. **Buyers Guide Pixel + CAPI verification still pending separately** — env vars need provisioning on Vercel from the 2026-05-12 instrumentation session. Worth knocking out before Session 1 starts since the verification is unrelated and Brian's already at the Vercel dashboard.

### Decisions locked in (recorded in migration plan)

- DB: HGPG Core Supabase (`ioypqogunwsoucgsnmla`), new tables `bg_*` namespace, service-role-only RLS
- PDF storage: Supabase Storage bucket `bg_pdfs` (private, signed URLs), NOT AWS S3
- 5 agents pre-known and documented in migration plan for Session 1 seeding
- Google Maps deferred to Session 4
- Bonus content gating stays as current UnlockModal pattern

---

## Last session: 2026-05-12 — Buyers Guide instrumentation + Manus full extraction

### Highlights

**Buyers Guide instrumentation (shipped + merged):**
- Pixel + CAPI code was on `main` since late April but inert (`VITE_META_PIXEL_ID` unset → Vite tree-shook). Made live this session.
- NeverBounce shipped from scratch in commit `5c233ee` on `claude/buyers-guide-instrumentation-nGCLI`, PR merged.
- Pixel ID for Buyers Guide: `1449157226505129`. NeverBounce key: reuse Sellers Guide value.
- Env vars + redeploy + verification pending Brian's hands.

**Manus full extraction (Path B + Path C complete):**
- Manus AI agent (inside their dashboard) cooperatively exported full source as a zip after 2 probe rounds.
- 220 files extracted, committed to new private repo `HGPG1/charlotte-buyers-guide-manus-export` (commit `e107b0e`).
- One historic FUB key found in `scripts/get-fub-users.ts` (`fka_0cyqBU…`) — probed FUB API, returned 401, already revoked.
- Forge storage clarification: Manus uses its own proxy, NOT AWS S3 directly (despite `@aws-sdk` deps).
- **Path C resolved via direct tRPC probe of live Manus site, NOT via AI agent** (which was looping on archived repo files — the GitHub-archived workspace appears to sandbox the agent away from live DB).
- **Production DB state: effectively empty.** 1 fake test lead, 0 quiz completions, 0 exit intents, 5 agent config rows.
- Manus migration scope COLLAPSED. 4 phases → 1 forward-looking phase. Phase 3 features (admin, dashboard, advisor mode, /:agent) re-judged: forward-looking ports, not regressions.
- Brian decided: no 301 redirect (Manus burned bridges around build time). Take Manus down after Vercel migration completes.

**Full migration plan staged for new thread:** see Next session above.

### Lessons noted (cumulative, key ones from today)

1. **Code-on-`main` does NOT equal shipped-in-production.** Verify against built bundle.
2. **Build-gate divergence:** `vercel.json` vs `package.json` can drift; run `npm run build` locally before merging.
3. **Sandbox push works** for at least the Buyers Guide repo.
4. **Migration-without-source = downgrade — but only when the source app was actually being used.** Check usage signal before treating missing features as regressions.
5. **AI agents inside SaaS platforms are an unofficial export channel.** Probe with innocuous files; verify against production bundle's `data-loc` strings; escalate to full zip. Worked here.
6. **AWS SDK deps != AWS usage.** Read the code, don't infer infra from `package.json`.
7. **Verify before rotation chaos.** Grep extracted source for secrets, then probe them. Cheap.
8. **When a SaaS AI agent loops, check whether the workspace is archived/disconnected.** GitHub-archived workspace appears to sandbox Manus AI agents to dead files. Agent substitutes the closest thing in its workspace and hopes you don't notice.
9. **When AI agent extraction stalls, hit the deployed app directly.** Live tRPC endpoints are usually faster and more reliable than coaxing an agent to query its own DB. `curl /api/trpc/contact.list` returned the full table in one request after multiple agent rounds failed.
10. **Usage signal trumps feature inventory.** Eight "missing features" in a gap analysis is meaningless if the source app had zero real users.
11. **`apply_migration` via MCP doesn't update repo.** DB ledger and `supabase/migrations/` are separate sources of truth. Commit the .sql alongside.
12. **CONTEXT.md drifts.** Reconcile vs project files every couple weeks.
13. **Don't trust assertions about "this is done" without verifying against the live system.**

---

## Previous session: 2026-05-11 — Brain reconciliation + transaction-pdfs cleanup

Reconciled CONTEXT.md against project files (which were ahead), closed six items: Sherlock 403, Mac Mini GitHub auth, exposed GitHub PAT rotation, CMA Engine MLS Grid auto-pull, NC office routing in ReZEN (verified live in code), transaction-pdfs bucket flip → private (bucket flipped via MCP on 2026-05-09; migration file backfilled to repo on 2026-05-11 as `2eb9794`).

Also: rebuilt `projects/brain-app.md` (had been clobbered), updated CONTEXT.md "active right now" list.

---

## Previous session: 2026-05-06 — Brain App MVP shipped 🟢

- New Vercel project: `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- New repo: `HGPG1/brain-app` (private)
- Live at: `https://brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Single-user lock: `BRIAN_EMAIL=brian@homegrownpropertygroup.com` allow-list
- Resend custom SMTP wired into HGPG Core Supabase (2/hr → 30/hr)
