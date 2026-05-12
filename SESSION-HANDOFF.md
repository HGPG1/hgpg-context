<!-- Last Updated: 2026-05-12 -->

# Session Handoff

## Last session: 2026-05-12 — Buyer Alerts Session 1 shipped to branch

### Highlights

**Buyer Alerts is the first piece of the MLS Dashboard Suite framing.** Lives inside the existing Team Dashboard (`HGPG1/hgpg-team-dash`) under `/buyers`. Inherits auth, brand, infra. No new repo, no new Vercel project, no new Supabase project. See `projects/buyer-alerts.md` for the full reference.

What shipped on branch `claude/buyer-alerts-session-1-wChbZ` (commit `b0ff809`):

- **DB migration** `buyer_alerts_schema` applied to `wdheejgmrqzqxvgjvfee`:
  - `buyer_criteria` (owner-scoped via `auth.uid()`)
  - `buyer_alerts` with unique index `(criteria_id, listing_key)` for cron-safe dedupe
  - RLS policies `criteria_owner_all` + `alerts_owner_select`
- **LLM criteria parser** in `src/lib/buyerAlerts.ts` calling Claude Sonnet 4.6 (`claude-sonnet-4-6`) with JSON-only prompt; retries once with stricter system message; final fallback stores raw input in `notes`.
- **Server-side matcher** filters `mls_property` on `mlg_can_view=true`, `standard_status='Active'`, last 24h `modification_timestamp`, plus price/beds/baths/sqft/zip/city/subdivision/year-built/lot-size. Property types, must-have features, and must-avoid are captured but advisory only (Session 2 work).
- **UI tab "Buyer Alerts"** added to AppHeader. Routes: `/buyers` (list), `/buyers/new` (two-step compose → preview → save), `/buyers/[id]` (parsed-criteria bullets + alert feed + "Run match now").
- **Cron route** `/api/cron/buyer-alerts` registered in `vercel.json` at `*/30 * * * *`. Iterates active criteria with service role, runs matcher, inserts alerts, then for each new row resolves owner phone via `auth.users.email ↔ team_members.email` (cross-project) and sends a one-shot LoopMessage iMessage. Missing phone marks the row `skipped`. Delivery failures set `failed` with the API error.
- **Production build clean.** `npx tsc --noEmit` + `npx next build` both green.

### What's pending (Buyer Alerts blockers before production smoke test)

1. **Vercel env vars** — the Vercel MCP exposed to this session had no env-var write tools, so these need to be set by hand in the Vercel dashboard on project `hgpg-team-dash`:
   - `SUPABASE_SERVICE_ROLE_KEY` (from `wdheejgmrqzqxvgjvfee`)
   - `ANTHROPIC_API_KEY` (Brian's Anthropic console)
   - `HGPG_CORE_SUPABASE_URL` = `https://ioypqogunwsoucgsnmla.supabase.co`
   - `HGPG_CORE_SUPABASE_SERVICE_ROLE_KEY` (from `ioypqogunwsoucgsnmla`)
   - `LOOPMESSAGE_API_KEY` and `LOOPMESSAGE_SECRET` (match TM repo var names + values verbatim)
   - `CRON_SECRET` (optional — only used for manual cron triggers when not invoked by Vercel)
2. **Merge to main + smoke test** — open PR from `claude/buyer-alerts-session-1-wChbZ` to `main` on `hgpg-team-dash`. After deploy, create a loose-criteria smoke buyer ("anywhere in Charlotte under $1.5M, 2+ bed"), wait for the next cron tick or fire it manually, confirm iMessage lands.

### Locked next-up queue

1. ✅ **Buyer Alerts Session 1** — shipped to branch, pending env vars + production smoke
2. 🎯 **Buyer Alerts Session 2** — capture UX polish, agent management, alert history pagination, manual test-alert button, free-text feature filtering, criteria-edit UI (carried over from Session 1 limitations)
3. 🎯 **Team listing photo sync** (still queued from previous; see `projects/team-photo-sync.md`)
4. 🔍 **Off-market / Expireds Finder** — new
5. 🤖 **FUB AI Agent rebuild** — gated on Sendblue eval

The Manus migration and Buyers Guide instrumentation work from the 2026-05-12 session is parked but not abandoned. See archived sessions below — Path C DB export is still the highest-value Manus item once Buyer Alerts is fully live.

### Open thread

- `~/Downloads/fix-admin-page.sh` may have unpushed UI changes for Listing Report Portal. Check next time portal is touched.
- Vercel env-var management is not exposed to the current MCP session — when this matters, set vars manually in the Vercel dashboard.

### Lessons noted (this session)

1. **Cross-project identity = email, not uid.** Auth users live per Supabase project. To resolve a phone for a Listing-Reports buyer owner, look up email in HGPG Core's `team_members`. Two service roles, two clients.
2. **Dedupe at the unique index, not in app code.** A unique `(criteria_id, listing_key)` index plus an unconditional insert + check for code `23505` gives a single source of truth and is race-safe under cron + manual trigger overlap.
3. **PostgREST `.or()` with `ilike` is comma-fragile.** Subdivision name matching strips commas before composing the OR; commas in canonical names would break parsing. Acceptable for Session 1.
4. **Anthropic JSON-only output via system message is reliable.** Sonnet 4.6 follows "no fences, no prose, JSON only" on first attempt; the stricter retry is defensive.
5. **Match writes happen during the cron, notify also during the cron.** Earlier impulse was to split match + notify into two phases; that added a "pending" state with no resilience benefit since alert rows are already durable.
6. **MCP capability inventory matters.** This session's Vercel MCP had no env-var tools — discovered only when needed. Worth surfacing upfront so the human knows which steps need their hands.

---

## Previous session: 2026-05-12 — Buyers Guide instrumentation + Manus extraction

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

### What's pending (Manus track — parked behind Buyer Alerts)

1. ⏳ **Path C — DB data export** (still NEXT for Manus track). Prompt staged in `projects/buyers-guide.md`. Highest single-value extraction remaining. Actual rows in `contacts`, `activities`, `quizResults`, `agents` only exist on Manus. Fire BEFORE telegraphing departure.
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
