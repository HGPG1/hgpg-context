<!-- Last Updated: 2026-05-12 -->

# Buyers Guide

**Status:** 🟢 Instrumentation shipped (Meta Pixel + CAPI + NeverBounce) on `claude/buyers-guide-instrumentation-nGCLI`. PR merged. Pending: env vars + redeploy + post-deploy verification.

**🟢 Manus migration scope shrunk dramatically 2026-05-12.** Probed the live Manus tRPC endpoints directly. The production database has effectively **no data** (1 fake test row in `contacts`, 0 quiz results, 0 exit-intent submissions, 5 agent config rows). The Manus app shipped feature-rich but never got real usage. The Vercel rebuild is the real production system. The 8 "missing features" are forward-looking ports, NOT regressions from a thriving app. Re-evaluate each on its own merit, not on "we used to have this."

## URLs

- Production (Vercel rebuild): https://buyersguide.homegrownpropertygroup.com
- Original (Manus, archived workspace): https://homegrownbg-hyqbjnnt.manus.space (still live, still serving but ~empty DB)
- Repo (Vercel rebuild): `HGPG1/charlotte-buyers-guide` (private)
- Repo (Manus original): `HGPG1/homegrown-buyer-guide` (private, ARCHIVED — caused Manus AI agent to loop on schema/migration files instead of reading live data)
- Repo (Manus full source snapshot): `HGPG1/charlotte-buyers-guide-manus-export` (private, created 2026-05-12). 220 files, first commit `e107b0e`.
- Local working copy: `~/Documents/charlotte-buyers-guide-manus-export/`
- Stack (Vercel): React 19 + Vite 8 + Tailwind v4 + react-router-dom 7
- Stack (Manus): React 19 + Vite 7 + Tailwind v4 + wouter + tRPC v11 + Drizzle ORM + MySQL + Express + Manus Forge storage proxy
- Vercel team: `team_FietQPKCmnyioG2n0FdteQCV`

## Routes

### Vercel rebuild (`src/App.tsx`)

- `/` Home, `/quiz`, `/calculator` (and `/rent-vs-buy` alias), `/neighborhoods`, `/strategy`, `/checklist`, `/bonuses`, `/thank-you`, `/*` NotFound

### Manus original (`client/src/App.tsx`) — all of the above PLUS

- `/admin` — admin surface (configure agents, view all leads). **Never used in production** per DB probe.
- `/agent-dashboard` — per-agent lead/quiz/exit-intent performance stats. **Empty data 2026-05-12.**
- `/:agent` — dynamic per-agent landing pages (`/brian`, `/ashley`, etc.). **Zero leads attributed via DB probe.**
- `/advisor`, `/advisor/quiz`, `/advisor/calculator`, `/advisor/checklist`, `/advisor/neighborhoods`, `/advisor/strategy` — advisor-mode versions.

## Lead capture (current Vercel rebuild)

Two forms in `src/components/Layout.tsx`: **UnlockModal** (guide-unlock gate) and **ContactDialog** ("Get Started" CTA). Both call `validateEmail()` from `src/lib/validateEmail.ts` BEFORE Pixel + FUB POST. Fail-closed on `invalid`/`disposable`; allow-through on `valid`/`catchall`/`unknown`/network failure. FUB ingestion via `api/fub-lead.ts` → POST to `/v1/events` with `source: "Charlotte Buyer's Guide"`.

## Lead scoring (extracted from `server/leadScoring.ts`)

    QUIZ_COMPLETION: 50
    CALCULATOR_BASIC: 25
    CALCULATOR_WITH_EQUITY: 40
    CALCULATOR_PDF: 20
    BONUS_UNLOCK: 30
    EXIT_INTENT_PDF: 20
    PAGE_VIEW: 5
    RETURN_VISIT: 15
    TIME_ON_SITE_5MIN: 10

Categories: Hot ≥80, Warm 40-79, Cold <40.

## Vercel env vars required

| Var | Value | Scope |
|---|---|---|
| `VITE_META_PIXEL_ID` | `1449157226505129` | Build-time |
| `META_PIXEL_ID` | `1449157226505129` | Runtime |
| `META_CAPI_ACCESS_TOKEN` | from Meta Events Manager → Settings → CAPI | Runtime |
| `META_CAPI_TEST_EVENT_CODE` | TEST code (remove before scaling) | Runtime, optional |
| `NEVERBOUNCE_API_KEY` | reuse Sellers Guide value | Runtime |
| `FUB_API_KEY` | confirm still present | Runtime |

---

## Manus migration — REVISED PLAN (2026-05-12)

### Extraction status (final)

- ✅ **Round 1 (probe):** done
- ✅ **Round 2 (map):** done
- ✅ **Path B (full source export):** done. 220 files committed to `HGPG1/charlotte-buyers-guide-manus-export`.
- ✅ **Path C (data export):** done via direct tRPC probe of live Manus site, NOT via the AI agent (which kept looping on archived repo files). Results below.

### Live Manus DB state (probed via tRPC 2026-05-12)

- `contacts`: **1 row** — `Test User / test@example.com / Exit-intent PDF download` from 2026-01-20. Fake test data, not a real lead.
- `quizResults`: **0 rows**
- `activities`: not directly queryable, but `agent.getPerformanceStats` returned `totalExitIntent: 0`. Likely 0.
- `agents`: **5 rows** (the only real production data):

| id | slug | name | email | phone | fubUserId | isDefault |
|---|---|---|---|---|---|---|
| 1 | `owner` | Brian McCarron | contact@homegrown.com | — | — | ✅ |
| 30001 | `brian` | Brian McCarron | brian@homegrownpropertygroup.com | 7046779191 | 1 | — |
| 30002 | `brenda` | Brenda Bain | brenda@homegrownpropertygroup.com | — | 14 | — |
| 30003 | `ashley` | Ashley Johnson | Ashley@HomeGrownPropertyGroup.com | — | 18 | — |
| 30004 | `taylor` | Taylor Sheehan-Watson | Taylor@HomeGrownPropertyGroup.com | — | 21 | — |

**Useful artifact: FUB user IDs (1, 14, 18, 21) provide the agent → FUB mapping if per-agent routing is ever rebuilt on the Vercel side.**

### Why the Manus AI agent looped on Path C

The Manus workspace repo (`HGPG1/homegrown-buyer-guide`) is GitHub-archived. The Manus agent appears to be sandboxed to that workspace's files. When it tried to find production data, all it could see were repo files: schema (DDL only), migrations (DDL only), and `.manus/db/*.json` (cached DDL queries). It substituted those for production data because the real DB query path was unreachable from its tooling. Lesson noted below.

### Migration plan (revised — much smaller scope)

**Phase 0 — secure + snapshot:**
- ✅ FUB key in scripts confirmed revoked
- ✅ Source snapshot committed to private repo
- ✅ Data export: confirmed no real data exists; agents table preserved in this brain doc

**Phase 1 — high-value forward-looking ports (1-2 sessions):**

Judged on standalone merit, not as regression-recovery:

- **Lead scoring** — useful regardless of historic data; will improve future lead prioritization. Logic is in `server/leadScoring.ts` of the snapshot. Port to `api/_lib/leadScoring.ts` + Supabase `contacts` table with `leadScore` column.
- **Calculator → FUB custom fields + behavioral tags** (`buy_better`/`rent_better`/`move_up_ready`) — drives FUB Automations 2.0 and ads optimization. Logic is in `server/routers.ts` calculator router.
- **Server-side PDF generation** (PDFKit) + **exit-intent popup** — proven conversion mechanic per Sellers Guide playbook. Sub Supabase Storage for the Forge proxy (~10 lines vs `server/storage.ts`).
- **Bonus content** — the 3 PDFs (timeline checklist, market predictions, negotiation scripts) are already in the snapshot. Decide whether to surface them on the Vercel rebuild.

**Phase 2 — possibly skip entirely (was "medium priority" in old plan):**

Usage signal from live DB: zero. Re-evaluate before building.

- ~~Activities event tracking~~ — nothing to track at this app's scale
- ~~Quiz results DB storage~~ — 0 quiz completions ever
- AI Chat Box — review source, decide independently

**Phase 3 — SKIP entirely:**

- ~~Per-agent vanity URLs~~ — zero attributed leads, agents probably never shared the links
- ~~Agent admin~~ — no real data, no real use case
- ~~Agent dashboard~~ — same
- ~~Advisor mode~~ — zero real-world advisor sessions

### Manus 301 redirect

Now safe to set up. No data to preserve, no users to disrupt. Configure on Manus's side: set destination URL on the app to `https://buyersguide.homegrownpropertygroup.com`. Or take the app down with a forwarding URL.

## Pending

- 🟡 **Buyers Guide Pixel + CAPI verification** — env vars need provisioning on Vercel; redeploy and verify dedup in Meta Test Events
- 🟡 **`META_CAPI_TEST_EVENT_CODE` cleanup** — remove from Vercel env once QA confirms dedup
- 🟡 **Phase 1 forward-looking ports** — lead scoring, calculator→FUB tags, server-side PDF + exit intent (1-2 sessions when prioritized)
- 🟡 **Manus 301 redirect** — configure on Manus's side; safe to do now

## Recent build history (most recent first)

- **2026-05-12** — Manus migration scope shrunk dramatically after live tRPC probe revealed empty production DB. Skipping Phases 2 + 3. Manus 301 redirect now safe to configure.
- **2026-05-12** — `HGPG1/charlotte-buyers-guide-manus-export` repo created with full Manus source snapshot (220 files, commit `e107b0e`).
- **2026-05-12** — Manus Path B complete. 220-file zip extracted via AI agent.
- **2026-05-12** — Manus AI agent extraction Rounds 1 + 2.
- **2026-05-12** — feat(buyers): NeverBounce email validation + form gates + build-gate fixes. Commit `5c233ee`. PR merged.
- 2026-04-28 → 2026-05-01 — Meta Pixel + CAPI scaffolding shipped to repo. Inert in production until 2026-05-12 because env vars never provisioned.

## Lessons noted

1. **Code-on-`main` does NOT equal shipped-in-production.** Verify against built bundle.
2. **Build-gate divergence:** `vercel.json` vs `package.json` can drift; run `npm run build` locally before merging.
3. **Sandbox push works** for at least the Buyers Guide repo.
4. **Migration-without-source = downgrade** — but only when the source app was actually being used. Original Manus→Vercel migration ported only buyer-facing UI; eight server-side features dropped. Turns out NONE of those features had real usage, so the downgrade was theoretical. Always check usage signal before treating a missing feature as a regression.
5. **AI agents inside SaaS platforms are an unofficial export channel.** Probe with innocuous files; verify against production bundle; escalate to full zip. Worked here for source code.
6. **AWS SDK deps != AWS usage.** Read the code, don't infer infra from `package.json`.
7. **Verify before rotation chaos.** Grep extracted source for secrets, then probe them. Cheap to verify.
8. **When a SaaS AI agent loops, check whether the workspace is archived/disconnected.** A GitHub-archived workspace can leave the AI agent sandboxed to dead files. The agent can't say "I can't reach the DB" — it substitutes the closest thing in its workspace (schema, migrations, DDL cache) and hopes you don't notice. Symptoms: re-reading the same files, returning impossibly clean schema dumps, never producing actual rows.
9. **When AI agent extraction stalls, hit the deployed app directly.** Live tRPC endpoints, REST endpoints, or even authenticated admin URLs are usually faster and more reliable than coaxing an agent to query its own DB. Today: `curl https://homegrownbg-hyqbjnnt.manus.space/api/trpc/contact.list` returned the full table in one request after multiple agent rounds failed.
10. **Usage signal trumps feature inventory.** Eight "missing features" in a gap analysis is meaningless if the source app had zero real users. Always check live data state before scoping a migration as regression-recovery.
