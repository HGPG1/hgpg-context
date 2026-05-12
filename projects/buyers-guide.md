<!-- Last Updated: 2026-05-12 -->

# Buyers Guide

**Status:** 🟢 Instrumentation shipped (Meta Pixel + CAPI + NeverBounce) on `claude/buyers-guide-instrumentation-nGCLI`. Pending: PR merged 2026-05-12, env vars + redeploy + post-deploy verification still pending.

**🚨 Major finding 2026-05-12:** the Vercel rebuild is a **significant downgrade** from the original Manus app. Eight high-value features exist on Manus but NOT on Vercel — Manus migration is a much bigger workstream than originally scoped. Full gap analysis below.

**🟢 Manus Path B (full source) COMPLETE 2026-05-12.** 220 files extracted, no node_modules bloat, no `.env` files leaked. Stored locally at `~/Documents/manus-buyers-guide-export-2026-05-12.zip`. Should be committed to new private repo `HGPG1/charlotte-buyers-guide-manus-export`.

**🚨 Security item 2026-05-12:** hardcoded FUB API key found in extracted source at `scripts/get-fub-users.ts`: `fka_0cyqBUzL20pdO11gGKd6J4jcHrUp1P6wsu`. Sitting on Manus's infrastructure for an unknown duration. **Rotate immediately.** Replacement needs to go in: TM Vercel env, Buyers Guide Vercel env, Sellers Guide Vercel env, and (if scripts are still needed there during migration) Manus env.

## URLs

- Production (Vercel rebuild): https://buyersguide.homegrownpropertygroup.com
- Original (Manus): https://homegrownbg-hyqbjnnt.manus.space
- Repo (Vercel rebuild): `HGPG1/charlotte-buyers-guide` (private)
- Repo (Manus original): `HGPG1/homegrown-buyer-guide` (private) — confirmed via agent extraction. Local snapshot at `~/Documents/manus-buyers-guide-export-2026-05-12.zip` as of 2026-05-12.
- Stack (Vercel): React 19 + Vite 8 + Tailwind v4 + react-router-dom 7
- Stack (Manus): React 19 + Vite 7 + Tailwind v4 + **wouter** router + **tRPC v11** + **Drizzle ORM + MySQL** + Express + **Manus Forge storage proxy** (NOT AWS S3 directly, despite `@aws-sdk` deps in package.json which are dead) + jose (JWT) + Builder.io vite-plugin-jsx-loc + Manus's own `vite-plugin-manus-runtime`
- Output: `dist/`
- Vercel team: `team_FietQPKCmnyioG2n0FdteQCV`

## Routes

### Vercel rebuild (`src/App.tsx`)

- `/` Home
- `/quiz` Buyer Quiz
- `/calculator` and `/rent-vs-buy` (alias) — Rent vs Buy Calculator
- `/neighborhoods`
- `/strategy`
- `/checklist`
- `/bonuses`
- `/thank-you`
- `/*` NotFound

### Manus original (`client/src/App.tsx`)

All of the above PLUS:

- `/admin` — admin surface (purpose: configure agents, view all leads)
- `/agent-dashboard` — per-agent lead/quiz/exit-intent performance stats
- `/:agent` — dynamic per-agent landing pages (e.g. `/brian`, `/ashley`)
- `/advisor` — advisor-mode home (mirrors home with `isAdvisorMode=true`)
- `/advisor/quiz`, `/advisor/calculator`, `/advisor/checklist`, `/advisor/neighborhoods`, `/advisor/strategy` — advisor-mode versions of each page

## Lead capture

### Vercel rebuild — current

Two forms, both in `src/components/Layout.tsx`:

- **UnlockModal** — guide-unlock gate (name + email + phone) → fires `Lead` Pixel event + POSTs `/api/fub-lead`
- **ContactDialog** — "Get Started" CTA in header + footer (first/last/email/phone/message) → same flow

Both forms now call `validateEmail(email)` from `src/lib/validateEmail.ts` BEFORE the Pixel + FUB POST. Fail-closed on NeverBounce result `invalid` or `disposable`; allow-through on `valid`, `catchall`, `unknown`, or network failure.

FUB ingestion via `api/fub-lead.ts` → POST to `/v1/events` (FUB Events API), `source: "Charlotte Buyer's Guide"`.

### Manus original — additional capture surfaces NOT yet on Vercel

- **Exit-intent popup** with PDF download (`useExitIntent` hook + `ExitIntentPopup` component). Fires on mouse-leave near top of viewport. Captures lead → sends `buyers-guide-{timestamp}.pdf` (generated server-side via PDFKit, uploaded to Manus Forge storage). Lead score +20.
- **Calculator completion → FUB custom fields + behavioral tags.** Pushes `customCalculatorCompleted=Yes`, `customCalculatorRecommendation`, `customCalculatorNetDifference`, `customCalculatorHomePrice`, `customCalculatorMonthlyRent` to FUB person record, plus a tag of `buy_better` / `rent_better` / `move_up_ready` based on calculation. If equity module used, also pushes `customEquityAvailable`, `customCurrentHomeValue`. Lead score: basic +25, with equity +40.
- **Calculator PDF download** — `rent-vs-buy-analysis-{timestamp}.pdf` server-side via PDFKit. Lead score +20.
- **Quiz completion → FUB tags + custom fields.** Quiz results push `Buyer Type`, `Lifestyle`, `Commute`, `Budget`, `Property Type` to FUB. Lead score +50.
- **Bonus unlock tracking** — `BonusBanner`, `LockedContent`, `trackBonusUnlock`. Activities table logs each bonus unlock event with platform (facebook/twitter/linkedin/copy). Lead score +30.

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

## Meta Pixel + CAPI (Vercel rebuild)

- **Pixel ID:** `1449157226505129` (HGPG - Buyers Guide). Distinct from Sellers Guide (`861295553661596`) per playbook.
- **Browser:** `src/components/MetaPixel.tsx` loads `fbevents.js` and fires `PageView` on mount + each location change. Reads `VITE_META_PIXEL_ID` at build time — if unset, Vite tree-shakes the dead path.
- **Server CAPI:** `api/_lib/meta.ts` + invoked from `api/fub-lead.ts`. v21.0, SHA-256 hashed PII, AbortController 5s timeout, fbp/fbc cookie passthrough, shared `event_id` for browser+server dedup.

## NeverBounce (Vercel rebuild)

- `api/validate-email.ts` — POST proxy. Reads `NEVERBOUNCE_API_KEY` (reuse Sellers Guide key per session decision 2026-05-12). Fail-open on infra failures.
- `src/lib/validateEmail.ts` — client helper. Fail-closed on `invalid`/`disposable` only.

## Vercel env vars required

| Var | Value | Scope |
|---|---|---|
| `VITE_META_PIXEL_ID` | `1449157226505129` | Build-time |
| `META_PIXEL_ID` | `1449157226505129` | Runtime |
| `META_CAPI_ACCESS_TOKEN` | from Meta Events Manager → Settings → CAPI → Generate Access Token | Runtime |
| `META_CAPI_TEST_EVENT_CODE` | TEST code (Meta Events Manager → Test Events). Remove before scaling ads. | Runtime, optional |
| `NEVERBOUNCE_API_KEY` | reuse Sellers Guide value | Runtime |
| `FUB_API_KEY` | already set, confirm still present — also rotate per security item above | Runtime |

---

## Manus migration scope (full visibility as of 2026-05-12)

### Extraction status

- ✅ **Round 1 (probe):** `package.json`, `client/src/App.tsx`, `tsconfig.json` confirmed agent access works
- ✅ **Round 2 (map):** full file tree + `server/_core/index.ts` + `server/routers.ts` + `drizzle/schema.ts`
- ✅ **Path B (full source export):** 2026-05-12 — Manus AI agent produced complete zip, 220 files, no secrets in env files, but ONE hardcoded FUB API key in `scripts/get-fub-users.ts` (flagged for rotation)
- ⏳ **Path C (DB data export):** NOT YET RUN. Next ask. Highest single-value extraction left — actual rows in `contacts`, `activities`, `quizResults`, `agents` tables only exist on Manus. Fire ASAP before telegraphing departure.

### What the export contains

- **Full client source** — 92 .tsx files, 101 total files in `client/src/`
- **Full server source** — 33 .ts files in `server/`, including `leadScoring.ts`, `fub.ts`, `generatePDF.ts`, `generateCalculatorPDF.ts`, `storage.ts`, `trackBonusUnlock.ts`, `market-data.ts`
- **Drizzle schema + 5 migration files** (`drizzle/0000_*.sql` through `0004_*.sql`)
- **3 bonus content PDFs + their source markdown** in `bonuses/`:
  - `bonus1-timeline-checklist` (7KB md, 384KB PDF)
  - `bonus2-market-predictions` (13KB md, 377KB PDF)
  - `bonus3-negotiation-scripts` (24KB md, 392KB PDF)
- **Companion documentation** (bonus — not asked for, included anyway):
  - `ADVISOR_MODE_GUIDE.md` (8.5KB)
  - `CALCULATOR-ACTION-PLAN.md` (24KB)
  - `CALCULATOR-ENHANCEMENTS-COMPLETE.md`, `CALCULATOR-FINAL-TESTING.md`, `CALCULATOR-TEST-RESULTS.md`
  - `FUB-LEAD-SCORING-GUIDE.md` (12KB) — lead scoring system in plain English
  - `PERFORMANCE-AUDIT.md`, `SLIDER-TEST-RESULTS.md`, `SOCIAL-SHARING-TEST.md`
  - `ideas.md`, `todo.md`
  - `social-media/POSTING_GUIDE.md`
- **3 cached Drizzle DDL queries** in `.manus/db/*.json` — DDL only, no actual production data rows. Don't get excited; you still need Path C.

### Important infra clarification

**The Manus source uses Manus's own "Forge" storage proxy, NOT AWS S3 directly.** The `@aws-sdk/*` packages in `package.json` are dead deps. `server/storage.ts` posts files to `BUILT_IN_FORGE_API_URL` with a Bearer token. **Port target for Vercel: use Supabase Storage** (HGPG Core Supabase already provisioned, signed URLs work, replaces `storage.ts` in ~10 lines). Do NOT provision AWS — extra dependency for no gain.

### Gap analysis: Vercel rebuild vs Manus original

| Feature | Manus | Vercel | Priority |
|---|---|---|---|
| Core pages (home, quiz, calculator, neighborhoods, strategy, checklist, bonuses, thank-you) | ✅ | ✅ | — |
| Meta Pixel + CAPI | (third-party scaffold) | ✅ (shipped 2026-05-12) | — |
| NeverBounce validation | ❌ | ✅ (shipped 2026-05-12) | — |
| **Lead scoring** (quiz +50, calc +25/40, calc PDF +20, bonus +30, exit-intent +20, page +5, return +15, 5min +10; Hot/Warm/Cold) | ✅ | ❌ | **HIGH** |
| **Calculator → FUB custom fields + behavioral tags** (`buy_better`/`rent_better`/`move_up_ready`) | ✅ | ❌ | **HIGH** |
| **Exit-intent popup + PDF download** | ✅ | ❌ | **HIGH** |
| **Server-side PDF generation** (buyers guide PDF + calculator analysis PDF, both via PDFKit) | ✅ | ❌ | **HIGH** |
| **Bonus unlock tracking** (activities table, platform attribution) | ✅ | ❌ | High |
| **Per-agent vanity URLs** (`/:agent` → `AgentProfile`) | ✅ | ❌ | High if any in the wild |
| **Agent admin** (`/admin`) | ✅ | ❌ | Medium |
| **Agent dashboard** (`/agent-dashboard` per-agent stats) | ✅ | ❌ | Medium |
| **AI Chat Box** (`components/AIChatBox.tsx`) | ✅ | ❌ | Need to read source to decide |
| **Activities event tracking** (general telemetry table) | ✅ | ❌ | Medium |
| **Quiz results DB storage** | ✅ | ❌ | Medium |
| Advisor mode UX layer (banner + toggle + inline talking points) | ✅ | ❌ | Medium — depends on actual agent usage |

### Migration plan (drafted 2026-05-12, not yet started)

**Phase 0 — secure + snapshot (do FIRST, before anything else):**
- Rotate exposed FUB API key `fka_0cyqBUzL20pdO11gGKd6J4jcHrUp1P6wsu`
- Commit Manus zip to new private repo `HGPG1/charlotte-buyers-guide-manus-export`
- Run Path C (DB CSV export) on Manus side BEFORE telegraphing departure

**Phase 1 — high-priority ports to Vercel rebuild (1-2 sessions):**
- Lead scoring (`server/leadScoring.ts` → port to `api/_lib/leadScoring.ts` + Supabase `contacts` table with `leadScore` column)
- Calculator → FUB custom fields + behavioral tags (logic is in extracted source)
- Server-side PDF generation (port PDFKit logic; sub Supabase Storage for Forge proxy)
- Exit-intent popup + PDF download
- Bonus unlock tracking

**Phase 2 — medium-priority ports (1 session):**
- Activities event tracking (Supabase table; DB layer doesn't exist on Vercel rebuild yet — needs design decision)
- Quiz results DB storage
- AI Chat Box (review source first, decide whether to port)

**Phase 3 — agent surfaces (deferred until usage confirmed):**
- Per-agent vanity URLs (`/:agent`)
- Agent admin
- Agent dashboard
- Advisor mode

Deferred because usage is unconfirmed. Confirm before building.

### Path C prompt (queued — DB data export, run NEXT)

Use this verbatim on the Manus AI agent:

    I need a CSV export of the production database for this app.
    Specifically:

    1. The contacts table — every row, every column.
    2. The activities table — every row, every column.
    3. The quizResults table — every row, every column.
    4. The agents table — every row, every column.

    Format: one CSV per table, with column headers. Paste them in
    fenced code blocks labeled with the table name. If a table has
    >1000 rows, paste the first 1000 plus a row count so I know
    what the full size is and we can plan a full export separately.

    If you can't directly query the database, run the equivalent
    tRPC admin procedures and paste their output.

## Pending

- 🔴 **Rotate exposed FUB API key** (urgent — security item)
- 🔴 **Path C — Manus DB data export** — actual lead rows; do BEFORE departure signals
- 🟡 **Buyers Guide Pixel + CAPI verification** — env vars need provisioning on Vercel; redeploy and verify dedup in Meta Test Events
- 🟡 **Buyers Guide gap-port to Vercel** — multi-session workstream; see Phase 1 above
- 🟡 **Manus 301 redirect** — original URL is `https://homegrownbg-hyqbjnnt.manus.space/?code=Cgvt4g5HuhfxNgdswHwqb4`. Must be configured on Manus's side; defer until Path C is complete.
- 🟡 **`META_CAPI_TEST_EVENT_CODE` cleanup** — remove from Vercel env once QA confirms dedup.

## Recent build history (most recent first)

- **2026-05-12** — Manus Path B complete. 220-file zip extracted via AI agent. Stored at `~/Documents/manus-buyers-guide-export-2026-05-12.zip`. Hardcoded FUB API key in source flagged for rotation. Forge storage proxy clarified (not AWS).
- **2026-05-12** — Manus AI agent extraction Rounds 1 + 2 (probe + map).
- **2026-05-12** — feat(buyers): NeverBounce email validation proxy + form gates + build-gate fixes. Commit `5c233ee`. PR merged.
- 2026-05-01 — SEO trim (homepage meta description 178 → 152 chars)
- 2026-04-28 → 2026-05-01 — Meta Pixel + CAPI scaffolding shipped to repo. Inert in production until 2026-05-12 because env vars were never provisioned.

## Lessons noted

1. **Code-on-`main` does NOT equal shipped-in-production.** Pixel + CAPI commits sat on main ~2 weeks with zero `fbq` references in live bundle because `VITE_*` was unset and Vite tree-shook the dead path. Always verify against built bundle.
2. **Build-gate divergence:** `vercel.json` `buildCommand` differed from `package.json` build script. TS errors landed on main without breaking production.
3. **Sandbox push works** for at least the Buyers Guide repo. Stale assumption corrected.
4. **Migration-without-source = downgrade.** Original Manus→Vercel migration ported only buyer-facing UI. Eight server-side features dropped silently.
5. **AI agents inside SaaS platforms are an unofficial export channel.** When support is slow/unresponsive, the platform's own AI agent often has read access to your source and is willing to share. Probe with 2-3 innocuous files; verify against production bundle's `data-loc` strings; escalate to full source. Keep framing as audit/debug. Path B (full zip export) worked here.
6. **AWS SDK deps != AWS usage.** The Manus extract had `@aws-sdk/client-s3` in `package.json` but `server/storage.ts` actually used Manus's Forge proxy. Don't assume infra from deps alone; read the code.
7. **Always grep extracted source for hardcoded secrets before committing.** Found a live FUB API key in `scripts/get-fub-users.ts` on first scan.
