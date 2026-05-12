<!-- Last Updated: 2026-05-12 -->

# Buyers Guide

**Status:** 🟢 Instrumentation shipped (Meta Pixel + CAPI + NeverBounce) on `claude/buyers-guide-instrumentation-nGCLI`. Pending: PR merged 2026-05-12, env vars + redeploy + post-deploy verification still pending.

**🚨 Major finding 2026-05-12:** the Vercel rebuild is a **significant downgrade** from the original Manus app. Eight high-value features exist on Manus but NOT on Vercel — Manus migration is a much bigger workstream than originally scoped. Full gap analysis below in "Manus migration scope."

## URLs

- Production (Vercel rebuild): https://buyersguide.homegrownpropertygroup.com
- Original (Manus): https://homegrownbg-hyqbjnnt.manus.space
- Repo (Vercel rebuild): `HGPG1/charlotte-buyers-guide` (private)
- Repo (Manus original — confirmed via agent extraction 2026-05-12): `HGPG1/homegrown-buyer-guide` (private)
- Stack (Vercel): React 19 + Vite 8 + Tailwind v4 + react-router-dom 7
- Stack (Manus): React 19 + Vite 7 + Tailwind v4 + **wouter** router + **tRPC v11** + **Drizzle ORM + MySQL** + Express + AWS S3 + jose (JWT) + Builder.io vite-plugin-jsx-loc + Manus's own `vite-plugin-manus-runtime`
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

- **Exit-intent popup** with PDF download (`useExitIntent` hook + `ExitIntentPopup` component). Fires on mouse-leave near top of viewport. Captures lead → sends `buyers-guide-{timestamp}.pdf` (generated server-side via PDFKit, uploaded to S3, signed URL returned). Lead score +20.
- **Calculator completion → FUB custom fields + tags.** When buyer finishes rent-vs-buy calc, Manus pushes `customCalculatorCompleted=Yes`, `customCalculatorRecommendation`, `customCalculatorNetDifference`, `customCalculatorHomePrice`, `customCalculatorMonthlyRent` to FUB person record, plus a tag of `buy_better` / `rent_better` / `move_up_ready` based on calculation. If equity module used, also pushes `customEquityAvailable`, `customCurrentHomeValue`. Lead score: basic +X, with equity +X.
- **Calculator PDF download** — `rent-vs-buy-analysis-{timestamp}.pdf` (server-side via PDFKit, uploaded to S3). Lead score +X.
- **Quiz completion → FUB tags + custom fields.** Quiz results push `Buyer Type`, `Lifestyle`, `Commute`, `Budget`, `Property Type` to FUB. Lead score +50.
- **Bonus unlock tracking** — `BonusBanner`, `LockedContent`, `trackBonusUnlock`. Activities table logs each bonus unlock event with platform (facebook/twitter/linkedin/copy).

## Meta Pixel + CAPI (Vercel rebuild)

- **Pixel ID:** `1449157226505129` (HGPG - Buyers Guide). Distinct from Sellers Guide (`861295553661596`) per playbook (one pixel per site).
- **Browser:** `src/components/MetaPixel.tsx` loads `fbevents.js` and fires `PageView` on mount + each react-router location change. Reads `VITE_META_PIXEL_ID` at build time — if unset, component renders null and the fbevents loader never injects (Vite tree-shakes the dead path, which is why the pre-2026-05-12 bundle had zero `fbq` references).
- **Server CAPI:** `api/_lib/meta.ts` + invoked from `api/fub-lead.ts`. v21.0, SHA-256 hashed PII per spec, AbortController 5s timeout, fbp/fbc cookie passthrough, shared `event_id` for browser+server dedup. `isCapiConfigured()` returns false if `META_PIXEL_ID` or `META_CAPI_ACCESS_TOKEN` is unset → silently no-ops.
- **Events firing:**
  - `Lead` — UnlockModal submit, ContactDialog submit, GuideContext.unlockContent (browser + CAPI dedup)
  - `ViewContent` — Neighborhoods county tab change
  - `Search` — Quiz reaches results page
  - `Contact` — Calculator scenario click
  - `SubmitApplication` — bonus share / copy
  - `PageView` — base loader + each route change

## NeverBounce (Vercel rebuild)

- `api/validate-email.ts` — POST proxy. Reads `NEVERBOUNCE_API_KEY` (reuse Sellers Guide key per session decision 2026-05-12). Calls v4 single/check, returns `{ result, flags, suggested_correction }`. All infra failures resolve to `result: "unknown"` with a `server_error` tag — keeps lead capture alive during NeverBounce outages.
- `src/lib/validateEmail.ts` — client helper used by both forms. Fail-closed on `invalid`/`disposable` (returns `ok: false` + a user-facing message). Fail-open on everything else.

## Vercel env vars required

Must be set in **Production** scope on the Buyers Guide project before the new deploy goes out:

| Var | Value | Scope |
|---|---|---|
| `VITE_META_PIXEL_ID` | `1449157226505129` | Build-time |
| `META_PIXEL_ID` | `1449157226505129` | Runtime |
| `META_CAPI_ACCESS_TOKEN` | from Meta Events Manager → Settings → CAPI → Generate Access Token | Runtime |
| `META_CAPI_TEST_EVENT_CODE` | TEST code (Meta Events Manager → Test Events). Remove before scaling ads. | Runtime, optional |
| `NEVERBOUNCE_API_KEY` | reuse Sellers Guide value | Runtime |
| `FUB_API_KEY` | already set, confirm still present | Runtime |

## Post-deploy verification gates

After Brian pushes and Vercel redeploys with all env vars set:

    BUNDLE=$(curl -s https://buyersguide.homegrownpropertygroup.com/ | grep -oE 'src="[^"]*assets/index[^"]*"' | head -1 | sed 's/src="//;s/"$//')
    curl -s "https://buyersguide.homegrownpropertygroup.com$BUNDLE" > /tmp/bg.js
    grep -o fbq /tmp/bg.js | wc -l                  # > 0 (locally: 9)
    grep -c connect.facebook.net /tmp/bg.js          # 1
    grep -c 1449157226505129 /tmp/bg.js              # 1
    grep -c validate-email /tmp/bg.js                # >= 1

Note: don't grep for `neverbounce` in the client bundle — it lives only in the server-side proxy.

End-to-end: submit a test lead → confirm in Meta Events Manager Test Events that browser + server Lead events deduplicate (single row with both sources). Then confirm FUB person created with correct source.

## Vercel build config gotcha

`vercel.json` runs `npx vite build` directly — does NOT run `tsc -b`. TS errors don't block production deploy but do block the local `npm run build` gate (`tsc -b && vite build`). Two pre-existing TS errors in `Bonuses.tsx` (missing `@/` path alias) and `Calculator.tsx` (Recharts Tooltip formatter type widening) were fixed in commit `5c233ee`.

---

## Manus migration scope (added 2026-05-12)

### Context

Manus support is unreliable, so escalation isn't an option. Used the Manus AI agent inside their dashboard to extract source code directly. Round 1 (package.json, App.tsx, tsconfig.json) and Round 2 (file tree + server entry + root tRPC router + Drizzle schema) confirmed the agent has full read access and is cooperative. Plan: extract everything before announcing any departure from the platform.

### Extraction progress

- ✅ **Round 1 (probe):** `package.json`, `client/src/App.tsx`, `tsconfig.json`. Confirmed real repo `HGPG1/homegrown-buyer-guide`. Live source matches `data-loc` strings in production bundle.
- ✅ **Round 2 (map):** full file tree, `server/_core/index.ts`, `server/routers.ts`, `drizzle/schema.ts`. Confirmed 5 DB tables, ~140 files total.
- ⏳ **Round 3 (pending):** full source dump or Path B archive. Next prompt prepared (see below).
- ⏳ **Round 4 (pending):** DB data export — leads CSV, activities CSV, quiz results CSV, agents CSV. Highest-value extraction; do BEFORE departure becomes obvious to platform.

### Manus-original DB schema

From `drizzle/schema.ts`:

- `users` — Manus OAuth identities (will NOT carry over; tied to their platform). `id`, `openId`, `name`, `email`, `loginMethod`, `role`, `createdAt`, `updatedAt`, `lastSignedIn`.
- `agents` — HGPG agent profiles. `id`, `slug`, `name`, `email`, `phone`, `fubUserId`, `isActive`, `isDefault`, timestamps.
- `contacts` — captured leads. `id`, `name`, `email`, `phone`, `message`, `bonusUnlocked`, `leadScore`, `assignedAgentId`, timestamps.
- `activities` — event log. `type` enum (`bonus_unlock`, `cta_click`, `email_signup`), `contactEmail`, `contactName`, `platform`, `ctaType`, `agentId`, `metadata` (JSON string), `createdAt`.
- `quizResults` — full quiz responses. `contactEmail`, `contactName`, `buyerType`, `lifestylePriority`, `commuteTolerance`, `budgetRange`, `propertyType`, `answersJson`, `syncedToFub`, `createdAt`.

No explicit Drizzle relations defined — connections are logical (`assignedAgentId` → `agents.id`, etc.).

### Gap analysis: Vercel rebuild vs Manus original

| Feature | Manus | Vercel | Priority |
|---|---|---|---|
| Core pages (home, quiz, calculator, neighborhoods, strategy, checklist, bonuses, thank-you) | ✅ | ✅ | — |
| Meta Pixel + CAPI | (third-party scaffold) | ✅ (shipped 2026-05-12) | — |
| NeverBounce validation | ❌ | ✅ (shipped 2026-05-12) | — |
| **Lead scoring** (quiz +50, calc +X, equity +X, exit-intent +20, calc PDF +X) | ✅ | ❌ | **HIGH** — real biz logic |
| **Calculator → FUB custom fields + behavioral tags** (`buy_better`/`rent_better`/`move_up_ready`) | ✅ | ❌ | **HIGH** — real regression |
| **Exit-intent popup + PDF download** | ✅ | ❌ | **HIGH** — proven conversion mechanic |
| **Server-side PDF generation** (buyers guide PDF + calculator analysis PDF, both via PDFKit + S3) | ✅ | ❌ | **HIGH** — used by exit-intent and calculator |
| **Bonus unlock tracking** (`activities.type='bonus_unlock'`, platform attribution) | ✅ | ❌ | High |
| **Per-agent vanity URLs** (`/:agent` → `AgentProfile` page) | ✅ | ❌ | High if any in the wild |
| **Agent admin** (`/admin`) | ✅ | ❌ | Medium |
| **Agent dashboard** (`/agent-dashboard` — per-agent lead/quiz/exit-intent stats from Drizzle) | ✅ | ❌ | Medium |
| **AI Chat Box** (`components/AIChatBox.tsx`) | ✅ | ❌ | Unknown — need to extract file before deciding |
| **Activities event tracking** (general telemetry table) | ✅ | ❌ | Medium |
| **Quiz results DB storage** (saved to `quizResults` table with `syncedToFub` flag) | ✅ | ❌ | Medium |
| Advisor mode UX layer (banner + toggle + inline talking points in pages) | ✅ | ❌ | Medium — depends on actual agent usage |

**Bottom line:** original Vercel migration ported the buyer-facing pages but dropped 8 high-value backend systems. This is a meaningful downgrade, not a clean migration.

### Migration plan (drafted 2026-05-12, NOT YET STARTED)

Wait for full Manus export before committing to scope. Once export is in hand:

**Phase 1 — defensive snapshot:**
- Drop entire Manus export into new private repo `HGPG1/charlotte-buyers-guide-manus-export`
- Commit raw, before any edits
- Pull DB CSV exports (contacts, activities, quizResults, agents) — these are gold; do BEFORE telegraphing departure to Manus

**Phase 2 — high-priority ports to Vercel rebuild (1-2 sessions):**
- Lead scoring (`server/leadScoring.ts` → port to `api/_lib/leadScoring.ts`)
- Calculator → FUB custom fields + behavioral tags
- Server-side PDF generation (PDFKit + S3; will require AWS access keys provisioned in Vercel env)
- Exit-intent popup + PDF download
- Bonus unlock tracking

**Phase 3 — medium-priority ports (1 session):**
- Activities event tracking (likely needs Supabase table or similar — DB layer doesn't exist on Vercel rebuild yet)
- Quiz results DB storage
- AI Chat Box (after reviewing what it does)

**Phase 4 — agent surfaces (deferred until usage confirmed):**
- Per-agent vanity URLs (`/:agent`)
- Agent admin
- Agent dashboard
- Advisor mode

Deferred because: usage is unconfirmed. Before building, check whether HGPG agents actually use `/advisor` mode in client meetings, and whether `/brian`/`/ashley`/etc. vanity URLs are live on email signatures or social bios. If neither is true, skip Phase 4 entirely.

### Path B prompt (queued for next Manus AI agent message)

Use this verbatim:

    Great map. Now please bundle the entire repository (excluding
    node_modules and dist) into a single archive I can download.
    A zip or tar.gz works. If the platform has a "download project"
    or "export project" function in your tools, use it and give me
    the download link or attachment.

    If you can't bundle, just stream every file in the tree to me
    sequentially across multiple replies. Start with the server/
    directory (every .ts file), then client/src/, then drizzle/,
    then config files at the root. Paste each file in a fenced
    code block labeled with its full path. Don't summarize, don't
    truncate, don't skip files claiming they're "boilerplate."

### Path C prompt (DB data export — queued for after Path B succeeds)

This is the highest-value single ask. Send AFTER source code is in hand:

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

- 🔴 **Phase 1 Manus extraction** — Path B (full source) and Path C (DB CSV export) still to run
- 🟡 **Manus 301 redirect** — original URL is `https://homegrownbg-hyqbjnnt.manus.space/?code=Cgvt4g5HuhfxNgdswHwqb4`. CAN NOT be redirected from our Vercel project (the host doesn't resolve to us). Must be configured on Manus's side. Defer until extraction is complete — don't telegraph departure yet.
- 🟡 **Page-parity audit vs. Manus** — partially complete via bundle inspection + AI agent extraction. Full parity not the goal; selective porting per gap analysis is the plan.
- 🟡 **`META_CAPI_TEST_EVENT_CODE` cleanup** — remove from Vercel env once QA confirms dedup. Leaving it on routes all CAPI events to Test Events instead of production, blocking ad-spend scaling.
- 🟢 FUB Automation 2.0 — not in scope.

## Recent build history (most recent first)

- **2026-05-12** — Manus AI agent extraction Rounds 1 + 2 complete. Repo confirmed as `HGPG1/homegrown-buyer-guide`. Full gap analysis documented; Vercel rebuild identified as meaningful downgrade from original.
- **2026-05-12** — feat(buyers): NeverBounce email validation proxy + form gates (fail-closed on invalid/disposable, fail-open on infra) + build-gate fixes for two pre-existing TS errors. Commit `5c233ee` on `claude/buyers-guide-instrumentation-nGCLI`. PR merged 2026-05-12.
- 2026-05-01 — SEO trim (homepage meta description 178 → 152 chars)
- 2026-05-01 — Lowercase email in schema + visible references
- 2026-05-01 — SEO/AEO: schema enrichment + footer link fix + author byline
- 2026-04-28 → 2026-05-01 — Meta Pixel + CAPI scaffolding shipped to repo (events: PageView, ViewContent, Search, Contact, Lead, SubmitApplication; CAPI mirror with `event_id` dedup). Inert in production until 2026-05-12 because env vars were never provisioned.

## Lessons noted

1. **Code-on-`main` does NOT equal shipped-in-production.** The Pixel + CAPI commits had been on main since late April but the live bundle had zero `fbq` references because `VITE_META_PIXEL_ID` was unset — Vite tree-shook the dead path. Always verify against a built bundle, not just the source.
2. **Build-gate divergence:** `vercel.json` `buildCommand` (`npx vite build`) differed from `package.json` build script (`tsc -b && vite build`). Means TS errors can land on main without breaking production. Either align them, or accept the divergence and run `npm run build` locally before merging.
3. **Sandbox push works.** Stale assumption corrected this session — `git push` from the sandbox succeeded against `HGPG1/charlotte-buyers-guide`. Future kickoff prompts shouldn't assume sandbox can't push.
4. **Migration-from-platform without source = downgrade.** The original Manus→Vercel migration ported only what was visible in the buyer-facing UI. Eight server-side features (lead scoring, FUB custom fields, PDF gen, exit intent, bonus tracking, activities log, quiz DB storage, AI chat) were dropped silently. When migrating off a platform in the future, always extract source code and DB schema FIRST, then port — never rebuild from observed UI behavior alone.
5. **AI agents inside SaaS platforms are an unofficial export channel.** When support is slow/unresponsive, the platform's own AI agent (inside the dashboard) often has read access to your source and is willing to share verbatim file contents. Probe with 2-3 innocuous files first to confirm access is real; verify against the production bundle's `data-loc` strings or equivalent traceable artifacts; then escalate to a full source dump. Keep framing as audit/debug — don't telegraph departure.
