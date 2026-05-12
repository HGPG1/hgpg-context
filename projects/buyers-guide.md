<!-- Last Updated: 2026-05-12 -->

# Buyers Guide

**Status:** 🟢 Instrumentation shipped (Meta Pixel + CAPI + NeverBounce) on `claude/buyers-guide-instrumentation-nGCLI`. Pending: PR merged 2026-05-12, env vars + redeploy + post-deploy verification still pending.

**🚨 Major finding 2026-05-12:** the Vercel rebuild is a **significant downgrade** from the original Manus app. Eight high-value features exist on Manus but NOT on Vercel — Manus migration is a much bigger workstream than originally scoped. Full gap analysis below.

**🟢 Manus Path B (full source) COMPLETE 2026-05-12.** 220 files extracted, no node_modules bloat, no `.env` files leaked. Stored locally at `~/Documents/manus-buyers-guide-export-2026-05-12.zip` (and re-presented from sandbox to `~/Downloads/`). Should be committed to new private repo `HGPG1/charlotte-buyers-guide-manus-export`.

**✅ FUB API key exposure (closed 2026-05-12):** hardcoded key `fka_0cyqBUzL20pdO11gGKd6J4jcHrUp1P6wsu` found in extracted source at `scripts/get-fub-users.ts`. Probed against FUB API `/v1/identity` endpoint — returned HTTP 401 "Invalid API Key." Key already revoked (date unknown; likely an earlier rotation). No further action required. Lesson 7 below still applies: always grep extracted source for secrets, even if they turn out to be dead.

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

- `/` Home, `/quiz`, `/calculator` (and `/rent-vs-buy` alias), `/neighborhoods`, `/strategy`, `/checklist`, `/bonuses`, `/thank-you`, `/*` NotFound

### Manus original (`client/src/App.tsx`) — all of the above PLUS

- `/admin` — admin surface (configure agents, view all leads)
- `/agent-dashboard` — per-agent lead/quiz/exit-intent performance stats
- `/:agent` — dynamic per-agent landing pages (e.g. `/brian`, `/ashley`)
- `/advisor`, `/advisor/quiz`, `/advisor/calculator`, `/advisor/checklist`, `/advisor/neighborhoods`, `/advisor/strategy` — advisor-mode versions of each page

## Lead capture

### Vercel rebuild — current

Two forms in `src/components/Layout.tsx`: **UnlockModal** (guide-unlock gate) and **ContactDialog** ("Get Started" CTA). Both call `validateEmail()` from `src/lib/validateEmail.ts` BEFORE Pixel + FUB POST. Fail-closed on `invalid`/`disposable`; allow-through on `valid`/`catchall`/`unknown`/network failure. FUB ingestion via `api/fub-lead.ts` → POST to `/v1/events` with `source: "Charlotte Buyer's Guide"`.

### Manus original — additional capture surfaces NOT yet on Vercel

- **Exit-intent popup** with PDF download (`useExitIntent` hook + `ExitIntentPopup` component). Captures lead → sends `buyers-guide-{timestamp}.pdf` (PDFKit + Manus Forge storage). Lead score +20.
- **Calculator completion → FUB custom fields + behavioral tags.** Pushes `customCalculatorCompleted=Yes`, `customCalculatorRecommendation`, `customCalculatorNetDifference`, `customCalculatorHomePrice`, `customCalculatorMonthlyRent` to FUB; tag of `buy_better` / `rent_better` / `move_up_ready` based on calculation. If equity module used, also pushes `customEquityAvailable`, `customCurrentHomeValue`. Lead score: basic +25, with equity +40.
- **Calculator PDF download** — `rent-vs-buy-analysis-{timestamp}.pdf` server-side via PDFKit. Lead score +20.
- **Quiz completion → FUB tags + custom fields.** Pushes `Buyer Type`, `Lifestyle`, `Commute`, `Budget`, `Property Type`. Lead score +50.
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

## Vercel env vars required

| Var | Value | Scope |
|---|---|---|
| `VITE_META_PIXEL_ID` | `1449157226505129` | Build-time |
| `META_PIXEL_ID` | `1449157226505129` | Runtime |
| `META_CAPI_ACCESS_TOKEN` | from Meta Events Manager → Settings → CAPI | Runtime |
| `META_CAPI_TEST_EVENT_CODE` | TEST code (remove before scaling) | Runtime, optional |
| `NEVERBOUNCE_API_KEY` | reuse Sellers Guide value | Runtime |
| `FUB_API_KEY` | already set, confirm still present | Runtime |

---

## Manus migration scope (as of 2026-05-12)

### Extraction status

- ✅ **Round 1 (probe):** `package.json`, `App.tsx`, `tsconfig.json`
- ✅ **Round 2 (map):** full file tree + server entry + root tRPC router + Drizzle schema
- ✅ **Path B (full source export):** 2026-05-12 — Manus AI agent produced complete zip, 220 files, no live secrets (one historic FUB key in `scripts/get-fub-users.ts` already revoked, confirmed via 401)
- ⏳ **Path C (DB data export):** NOT YET RUN. Highest single-value extraction left. Fire ASAP before telegraphing departure.

### What the export contains

- **Full client source** — 92 .tsx files, 101 total files in `client/src/`
- **Full server source** — 33 .ts files in `server/`, including `leadScoring.ts`, `fub.ts`, `generatePDF.ts`, `generateCalculatorPDF.ts`, `storage.ts`, `trackBonusUnlock.ts`, `market-data.ts`
- **Drizzle schema + 5 migration files** (`drizzle/0000_*.sql` through `0004_*.sql`)
- **3 bonus content PDFs + their source markdown** in `bonuses/`: timeline-checklist, market-predictions, negotiation-scripts (~25KB md + ~380KB PDF each)
- **Companion documentation** (bonus — not asked for):
  - `ADVISOR_MODE_GUIDE.md`, `CALCULATOR-ACTION-PLAN.md`, `CALCULATOR-ENHANCEMENTS-COMPLETE.md`, `CALCULATOR-FINAL-TESTING.md`, `CALCULATOR-TEST-RESULTS.md`, `FUB-LEAD-SCORING-GUIDE.md`, `PERFORMANCE-AUDIT.md`, `SLIDER-TEST-RESULTS.md`, `SOCIAL-SHARING-TEST.md`, `ideas.md`, `todo.md`, `social-media/POSTING_GUIDE.md`
- **3 cached Drizzle DDL queries** in `.manus/db/*.json` — DDL only, no production data rows

### Important infra clarification

**The Manus source uses Manus's own "Forge" storage proxy, NOT AWS S3 directly.** The `@aws-sdk/*` packages in `package.json` are dead deps. `server/storage.ts` posts files to `BUILT_IN_FORGE_API_URL` with a Bearer token. **Port target for Vercel: use Supabase Storage** (HGPG Core Supabase already provisioned, signed URLs work, replaces `storage.ts` in ~10 lines).

### Gap analysis: Vercel rebuild vs Manus original

| Feature | Manus | Vercel | Priority |
|---|---|---|---|
| Core pages | ✅ | ✅ | — |
| Meta Pixel + CAPI | (third-party scaffold) | ✅ shipped 2026-05-12 | — |
| NeverBounce validation | ❌ | ✅ shipped 2026-05-12 | — |
| **Lead scoring** | ✅ | ❌ | **HIGH** |
| **Calculator → FUB custom fields + behavioral tags** | ✅ | ❌ | **HIGH** |
| **Exit-intent popup + PDF download** | ✅ | ❌ | **HIGH** |
| **Server-side PDF generation** (PDFKit) | ✅ | ❌ | **HIGH** |
| **Bonus unlock tracking** | ✅ | ❌ | High |
| **Per-agent vanity URLs** (`/:agent`) | ✅ | ❌ | High if any in wild |
| **Agent admin** (`/admin`) | ✅ | ❌ | Medium |
| **Agent dashboard** (`/agent-dashboard`) | ✅ | ❌ | Medium |
| **AI Chat Box** | ✅ | ❌ | Need source review |
| **Activities event tracking** | ✅ | ❌ | Medium |
| **Quiz results DB storage** | ✅ | ❌ | Medium |
| Advisor mode UX layer | ✅ | ❌ | Medium — confirm agent usage |

### Migration plan (drafted 2026-05-12)

**Phase 0 — secure + snapshot:**
- ~~Rotate exposed FUB API key~~ ✅ Already revoked (verified 2026-05-12)
- Commit Manus zip to new private repo `HGPG1/charlotte-buyers-guide-manus-export`
- Run Path C (DB CSV export) on Manus side BEFORE telegraphing departure

**Phase 1 — high-priority ports (1-2 sessions):**
- Lead scoring (`server/leadScoring.ts` → `api/_lib/leadScoring.ts` + Supabase `contacts` table with `leadScore` column)
- Calculator → FUB custom fields + behavioral tags
- Server-side PDF generation (port PDFKit logic; sub Supabase Storage for Forge proxy)
- Exit-intent popup + PDF download
- Bonus unlock tracking

**Phase 2 — medium-priority ports (1 session):**
- Activities event tracking (Supabase table; DB layer design decision)
- Quiz results DB storage
- AI Chat Box (review source first)

**Phase 3 — agent surfaces (deferred until usage confirmed):**
- Per-agent vanity URLs, agent admin, agent dashboard, advisor mode

### Path C prompt (queued — DB data export, run NEXT)

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

- 🔴 **Path C — Manus DB data export** — actual lead rows; do BEFORE departure signals
- 🟡 **Snapshot commit** to `HGPG1/charlotte-buyers-guide-manus-export` (zip is in `~/Downloads/` and sandbox outputs; needs proper git home)
- 🟡 **Buyers Guide Pixel + CAPI verification** — env vars need provisioning on Vercel; redeploy and verify dedup in Meta Test Events
- 🟡 **Buyers Guide gap-port to Vercel** — multi-session workstream; see Phase 1 above
- 🟡 **Manus 301 redirect** — must be configured on Manus's side; defer until Path C is complete
- 🟡 **`META_CAPI_TEST_EVENT_CODE` cleanup** — remove from Vercel env once QA confirms dedup

## Recent build history (most recent first)

- **2026-05-12** — Manus Path B complete. 220-file zip extracted. Stored locally. Historic FUB API key in scripts confirmed already revoked (401 from FUB API).
- **2026-05-12** — Manus AI agent extraction Rounds 1 + 2 (probe + map).
- **2026-05-12** — feat(buyers): NeverBounce email validation proxy + form gates + build-gate fixes. Commit `5c233ee`. PR merged.
- 2026-05-01 — SEO trim (homepage meta description 178 → 152 chars)
- 2026-04-28 → 2026-05-01 — Meta Pixel + CAPI scaffolding shipped to repo. Inert in production until 2026-05-12 because env vars were never provisioned.

## Lessons noted

1. **Code-on-`main` does NOT equal shipped-in-production.** Pixel + CAPI commits sat on main ~2 weeks with zero `fbq` references in live bundle. Always verify against built bundle.
2. **Build-gate divergence:** `vercel.json` `buildCommand` differed from `package.json` build script. TS errors landed on main without breaking production.
3. **Sandbox push works** for at least the Buyers Guide repo. Stale assumption corrected.
4. **Migration-without-source = downgrade.** Original Manus→Vercel migration ported only buyer-facing UI. Eight server-side features dropped silently.
5. **AI agents inside SaaS platforms are an unofficial export channel.** Probe with 2-3 innocuous files; verify against production bundle's `data-loc` strings; escalate to full source. Path B (full zip export) worked here.
6. **AWS SDK deps != AWS usage.** Manus extract had `@aws-sdk/client-s3` in `package.json` but `server/storage.ts` used Manus's Forge proxy. Don't assume infra from deps alone.
7. **Always grep extracted source for hardcoded secrets, then verify their status before assuming worst case.** Found `fka_*` key in extracted scripts; probed FUB `/v1/identity`; got 401; closed item without rotation chaos. Cheap to verify, easy to over-react.
