# Buyers Guide - Full Document Content

<!-- Last Updated: 2026-05-19 -->

## Buyers Guide

**Status:** 🟢 Sessions 1 + 2 + 3 of Manus migration complete. Backend infrastructure live 2026-05-12; buyer-facing funnel (Calculator + Quiz + Exit Intent + Bonus Unlock) shipped 2026-05-12. Session 3 (Agent surfaces + Advisor Mode + Admin/Agent Dashboard) shipped to production 2026-05-19 09:45 ET. Session 4 (optional) still queued.

**🟢 Manus migration scope shrunk dramatically 2026-05-12.** Probed the live Manus tRPC endpoints directly. The production database has effectively **no data** (1 fake test row in `contacts`, 0 quiz results, 0 exit-intent submissions, 5 agent config rows). The Manus app shipped feature-rich but never got real usage. The Vercel rebuild is the real production system. The 8 "missing features" are forward-looking ports, NOT regressions from a thriving app.

### URLs

- Production (Vercel rebuild): https://buyersguide.homegrownpropertygroup.com
- Original (Manus, archived workspace): https://homegrownbg-hyqbjnnt.manus.space
- Repo (Vercel rebuild): `HGPG1/charlotte-buyers-guide` (private)
- Repo (Manus original): `HGPG1/homegrown-buyer-guide` (private, ARCHIVED)
- Repo (Manus full source snapshot): `HGPG1/charlotte-buyers-guide-manus-export` (private). 220 files, commit `e107b0e`.
- Local working copy of snapshot: `~/manus-snapshot/`
- Stack (Vercel): React 19 + Vite 8 + Tailwind v4 + react-router-dom 7. `@supabase/supabase-js` + `pdfkit` + `recharts` added through Session 2.
- Stack (Manus): React 19 + Vite 7 + Tailwind v4 + wouter + tRPC v11 + Drizzle ORM + MySQL + Express + Manus Forge storage proxy
- Vercel team: `team_FietQPKCmnyioG2n0FdteQCV`
- Vercel project: `prj_AQ0xkwT0IgQFssJeBbcPOmTFdmOJ` (`charlotte-buyers-guide`)

### Agent surfaces (Session 3)

**Agent slug URLs** — all four agents have a personal entry path:

- https://buyersguide.homegrownpropertygroup.com/brian
- https://buyersguide.homegrownpropertygroup.com/brenda
- https://buyersguide.homegrownpropertygroup.com/ashley
- https://buyersguide.homegrownpropertygroup.com/taylor

Visiting any of these:
1. The slug is parsed by `agentDetection.ts` (hardcoded valid list: `[brian, brenda, ashley, taylor]`)
2. The slug is persisted to `localStorage` as `homegrown_assigned_agent_slug` so it survives route changes within the session
3. The slug is read by `AssignedAgentProvider` context and propagated into every lead-capture payload as `assignedAgentSlug`
4. Server-side, each lead-capture handler resolves the slug → `bg_agents.id` and writes it to `bg_contacts.assigned_agent_id` (and assigns the FUB person to that agent's `fub_user_id`)

**Reserved routes** (NOT treated as agent slugs):
`quiz, calculator, rent-vs-buy, neighborhoods, strategy, checklist, bonuses, thank-you, advisor, admin, agent-dashboard, api`

**Agent profile page (`/:agent`)** — rendered by `AgentProfile.tsx` when the URL slug passes `isAllowedAgentSlug`. Unknown slugs fall through to `<NotFound />`. Flow:
1. On mount, calls `GET /api/agent/by-slug?slug=<slug>` to fetch the `bg_agents` row (id, slug, name, email, phone, fub_user_id)
2. Renders a **personalized hero section** (steel-blue `#F0F0F0` background):
   - Circular initials avatar (navy `#2A384C` bg, white text, computed from name)
   - Eyebrow text: "Your Charlotte buying co-pilot"
   - `<h1>` with the agent's full name
   - Subtitle: "Home Grown Property Group · Charlotte, Fort Mill, Indian Land, Waxhaw"
   - Two CTAs:
     - `tel:` button (navy) — formats phone for display, raw digits in href
     - `mailto:` button (white outline) — "Email <firstname>"
3. Below the hero, a **4-card feature grid**:
   - Buyer Best-Fit Quiz → `/quiz` (Calculator icon)
   - Rent vs Buy Calculator → `/calculator` (DollarSign icon)
   - Neighborhood Guides → `/neighborhoods` (Map icon)
   - Offer Strategy Playbooks → `/strategy` (BookOpen icon)
4. SEO: `<title>` is `${name} | 2026 Charlotte Buyer's Guide`, custom description mentions working with that specific agent

### Advisor Mode (Session 3)

**Entry**: visit `/advisor/<anything>` (e.g. `/advisor/calculator`) or programmatic toggle via `setAdvisorMode(true)`.

**Persistence**: `localStorage` flag `homegrown_advisor_mode`. Persists across sessions on that device until manually exited.

**UI surface**: When active, a persistent banner reads:
- "Advisor Mode · Talking points visible to you only"
- Exit button labelled "Exit advisor mode" — clears the flag, strips `/advisor` from the URL, navigates to clean path

**`AdvisorNote` components** — embedded throughout the buyer-facing pages, only render when `isAdvisorMode === true`. Used for internal coaching notes / talking points that the agent sees but the buyer never does.

**Intent**: Brian (or any HGPG agent) can pre-load advisor mode on their phone/iPad during a buyer consultation, click around the guide with the buyer watching, and see internal-only prompts about what to highlight, what objections to anticipate, and which sections deserve emphasis.

### Admin + Agent Dashboard (Session 3)

Two internal routes gated by a query-string access key (`ADMIN_ACCESS_KEY` env var on the server).

**`/admin`** — internal admin landing
- Auth: `GET /api/admin/verify?key=<ADMIN_ACCESS_KEY>` returns 200 ok / 401 unauthorized
- 3 UI states: verifying, error ("Verification failed. Try again or contact Brian."), verified
- When verified: `<h1>Admin</h1>` title, "Internal landing for HGPG team members." description, two link cards:
  - "Agent Dashboard" → `/agent-dashboard?key=<same key>` (passes key through), sub-copy: "Per-agent lead counts, quiz completions, bonus unlocks, hot/warm splits."
  - "Supabase Editor" → direct link to https://supabase.com/dashboard/project/ioypqogunwsoucgsnmla/editor, sub-copy: "Raw bg_contacts / bg_quiz_results / bg_activities for ad-hoc queries."
- UX hint when no key: "Append ?key=... to the URL with the current admin access key."
- `noIndex: true` (not crawled by Google)

**`/agent-dashboard`** — internal stats dashboard
- Same `/api/admin/verify` gate
- Once verified, fetches `GET /api/agent/performance-stats?key=<ADMIN_ACCESS_KEY>` and renders:
  - **Four stat cards across the top:** Total contacts, Quiz completions, Activities logged, Unassigned
  - **Per-agent table** with columns: Agent (name + /slug), Contacts, Hot (lead_score ≥80), Warm (40-79), Quiz, Bonus, Exit-intent, FUB user ID
  - One row per agent in `bg_agents` (active)
- Refresh button (re-fires `/api/agent/performance-stats`)
- `noIndex: true`

Both routes are lazy-loaded chunks (`Admin-yDBH8GRR.js`, `AgentDashboard-DO4RiWXi.js`) so they don't bloat the main bundle for buyer-facing visitors.

**Access key location:** `ADMIN_ACCESS_KEY` in Vercel project env vars for `charlotte-buyers-guide`. Required scope: Production runtime.

### Plumbing details

**`assignedAgentSlug` is sent in payloads to:**
- `POST /api/fub-lead` (unlock modal, contact dialog)
- `POST /api/exit-intent/submit`
- `POST /api/bonus/unlock`

**Server-side resolution flow:**
1. Handler receives `assignedAgentSlug` (string, optional)
2. Looks up `bg_agents WHERE slug = ?` → gets `id` + `fub_user_id`
3. Writes `bg_contacts.assigned_agent_id` (FK to `bg_agents.id`)
4. Calls FUB `assignLeadToAgent(personId, fub_user_id)` to route the FUB person to that agent
5. Returns `{success: true, contact: {...}, assignedAgent: {id, slug}}` so the client can confirm

**Default behaviour when no slug present:**
- Contact still created
- `assigned_agent_id` stays NULL (or defaults to the `is_default=true` row — agent ID 1, the owner placeholder)
- FUB person stays unassigned at the user level (still flows through normal FUB lead routing rules)

**Session 3 added 3 new server endpoints (`/api/agent/by-slug`, `/api/agent/default`, `/api/admin/verify`) plus a stats endpoint (`/api/agent/performance-stats`)**, and extended the 3 existing lead-capture routes (`fub-lead`, `exit-intent/submit`, `bonus/unlock`) to accept and resolve `assignedAgentSlug`. The agent slug allowlist (`AGENT_SLUGS`) is statically declared in both client (`src/lib/agentDetection.ts`) and server (`api/_lib/agents.ts`); agent records themselves live in Supabase `bg_agents` and are fetched at runtime.

### Verified live activity (2026-05-19)

Real Session 3 traffic confirmed in `bg_contacts`:

| id | source | lead_score | assigned_agent_id | created_at |
|---|---|---|---|---|
| 11 | buyers-guide-unlock | 50 | 30001 (Brian) | 2026-05-19 12:36 UTC |
| 9 | buyers-guide-quiz | 50 | 30003 (Ashley) | 2026-05-19 12:33 UTC |
| 7 | buyers-guide-unlock | 50 | 30003 (Ashley) | 2026-05-19 12:21 UTC |
| 5 | buyers-guide-unlock | 50 | 30003 (Ashley) | 2026-05-19 10:10 UTC |

All four have a named `assigned_agent_id` populated. Agent slug → agent ID resolution confirmed working end-to-end.

### Supabase (HGPG Core `ioypqogunwsoucgsnmla`) - tables added Session 1

**Tables** (all RLS enabled, no policies, service-role-only):

- `bg_contacts` - id, name, email (UNIQUE), phone, message, bonus_unlocked bool, lead_score int, assigned_agent_id FK, source, timestamps
- `bg_activities` - id, type, contact_email, contact_name, platform, cta_type, agent_id FK, metadata jsonb, created_at
- `bg_quiz_results` - id, contact_email, contact_name, buyer_type, lifestyle_priority, commute_tolerance, budget_range, property_type, answers_json jsonb, synced_to_fub bool, created_at
- `bg_agents` - id, slug, name, email, phone, fub_user_id, is_active, is_default, timestamps

**Storage bucket:** `bg-pdfs` (private, 10 MB cap, `application/pdf` only).

**Seeded agents** (IDs preserved from Manus):

| id | slug | name | email | fub_user_id | is_default |
|---|---|---|---|---|---|
| 1 | owner | Brian McCarron | contact@homegrown.com | NULL | true |
| 30001 | brian | Brian McCarron | brian@homegrownpropertygroup.com | 1 | false |
| 30002 | brenda | Brenda Bain | brenda@homegrownpropertygroup.com | 14 | false |
| 30003 | ashley | Ashley Johnson | Ashley@HomeGrownPropertyGroup.com | 18 | false |
| 30004 | taylor | Taylor Sheehan-Watson | Taylor@HomeGrownPropertyGroup.com | 21 | false |

### Server helpers (`api/_lib/`)

- `supabase.ts` - cached service-role client. Soft-fails when env unset.
- `signBgPdfUrl.ts` - 7-day signed URL helper + `uploadBgPdf` + `BG_PDFS_BUCKET` constant.
- `leadScoring.ts` - `SCORE_VALUES` constants + `applyLeadScore(email, scoreType)` (UPDATE WHERE email; no-op if contact missing).
- **`contacts.ts` (Session 2)** - `upsertBgContact({email, name, phone, source, message, assignedAgentId})` does the INSERT-with-existence-check that pairs with the UNIQUE(email) constraint. Plus `setBonusUnlocked(email)`.
- `fub.ts` - `sendFUBEvent`, `sendLeadCapture`, `sendQuizCompletion`, `assignLeadToAgent`, `testFUBConnection`. **Session 2 added** `updateFubPersonByEmail(email, {customFields, tags})` — search-then-PUT for the Calculator track-completion flow.
- `pdfContent.ts` / `generatePDF.ts` / `generateCalculatorPDF.ts` - PDF copy + generators (HGPG palette).
- `marketData.ts` - FRED fetcher with static fallback.
- `meta.ts` - Conversions API helper (hash + send).

### Health endpoints (`/api/health/*`)

- `GET /api/health/db`, `/api/health/fub`, `/api/health/fred`, `/api/health/storage`

### Lead-capture endpoints (Session 2, extended in Session 3)

| Route | Writes | Score | FUB | CAPI | Session 3 |
|---|---|---|---|---|---|
| POST `/api/fub-lead` | (pre-S2; no Supabase) | — | event | Lead (inline) | accepts `assignedAgentSlug` |
| POST `/api/calculator/track-completion` | — | +25 basic / +40 equity | search-then-PUT custom fields + behavioral tag | — | — |
| POST `/api/calculator/generate-pdf` | upload → bg-pdfs/calculator/ | +20 | — | — | — |
| POST `/api/quiz/submit` | bg_quiz_results | +50 | sendQuizCompletion + buyer-type tag | — | — |
| POST `/api/exit-intent/submit` | bg_contacts upsert + bg-pdfs/exit-intent/ | +20 | sendLeadCapture | Lead (mirror) | accepts `assignedAgentSlug` |
| POST `/api/bonus/unlock` | bg_activities + bg_contacts.bonus_unlocked=true | +30 | — | SubmitApplication (mirror) | accepts `assignedAgentSlug` |
| POST `/api/capi-event` | — | — | — | Generic proxy (Search, Contact) | — |
| POST `/api/validate-email` | — | — | — | — | (S2; unchanged) |

**Behavioral tags pushed to FUB from calculator track-completion** (Manus rules verbatim):
- `move_up_ready` — whenever the equity module was on (overrides everything else)
- `buy_better` — `netDifference > 0`
- `rent_better` — `netDifference < -50000`
- `calculator_completed` — otherwise

**FUB custom fields pushed:**
- `customCalculatorCompleted`, `customCalculatorRecommendation`, `customCalculatorNetDifference`, `customCalculatorHomePrice`, `customCalculatorMonthlyRent`
- `customEquityAvailable`, `customCurrentHomeValue` — only when equity module is on

### Lead scoring (`api/_lib/leadScoring.ts`)

    QUIZ_COMPLETION: 50
    CALCULATOR_BASIC: 25
    CALCULATOR_WITH_EQUITY: 40
    CALCULATOR_PDF: 20
    BONUS_UNLOCK: 30
    EXIT_INTENT_PDF: 20
    PAGE_VIEW: 5
    RETURN_VISIT: 15
    TIME_ON_SITE_5MIN: 10

Categories: Hot ≥80, Warm 40-79, Cold <40. `upsertBgContact` (Session 2) ensures contacts exist before scoring fires.

### Buyer funnel (Session 2)

- **Calculator** (`src/pages/Calculator.tsx`) — Rent-vs-Buy with equity module, URL prefill (`?price=&rent=&preset=move-up`), 3 preset scenarios. Fires `Contact` event + CAPI mirror on first scenario click. Fires server `track-completion` once per session once email AND interaction are present. PDF download supports "click-while-locked → unlock-modal → auto-fire-PDF". Chart strokes: navy solid (rent), steel solid (buy), navy dashed (equity).
- **Quiz** (`src/pages/Quiz.tsx`) — 5-question buyer best-fit. Fires `Search` event + CAPI mirror on results page. Submits to `/api/quiz/submit` once per session once email captured.
- **Exit Intent** (`src/components/ExitIntentPopup.tsx` + `src/hooks/useExitIntent.ts`) — mouse-leave near top of viewport after 5s dwell. Suppressed if user already unlocked OR has seen it this session.
- **Bonus Unlock** (Layout.tsx BonusModal) — three share buttons + copy link. Each fires `SubmitApplication` Pixel + `/api/bonus/unlock` (which mirrors CAPI server-side).

### Meta Pixel events (cumulative through Session 2)

| Event | Where | Server CAPI mirror |
|---|---|---|
| `PageView` | every route change | — |
| `Lead` | UnlockModal, ContactDialog | `/api/fub-lead` |
| `Lead` | ExitIntentPopup | `/api/exit-intent/submit` |
| `Contact` | Calculator first-scenario-click | `/api/capi-event` |
| `Search` | Quiz results page | `/api/capi-event` |
| `SubmitApplication` | BonusModal share/copy | `/api/bonus/unlock` |

All paired events use a shared `event_id` (UUID from `newClientEventId()`) so Meta dedups browser+server signals.

### Vercel env vars in Production

| Var | Scope | Status |
|---|---|---|
| `VITE_META_PIXEL_ID` | Build | ✅ live |
| `META_PIXEL_ID` | Runtime | ✅ live |
| `META_CAPI_ACCESS_TOKEN` | Runtime | ✅ live |
| `META_CAPI_TEST_EVENT_CODE` | Runtime | optional |
| `NEVERBOUNCE_API_KEY` | Runtime | ✅ live |
| `FUB_API_KEY` | Runtime | ✅ live |
| `SUPABASE_URL` | Runtime | ✅ live |
| `SUPABASE_SERVICE_ROLE_KEY` | Runtime | ✅ live |
| `FRED_API_KEY` | Runtime | ✅ live |

## Pending

- 🟡 **Session 2 post-deploy verification** — 7-step smoke (full funnel, Supabase rows, FUB person+tags, both PDFs, Meta Test Events dedup)
- 🟡 **`META_CAPI_TEST_EVENT_CODE` cleanup** — remove once QA confirms dedup
- 🟡 **Session 3 mobile smoke test** — verify AdvisorMode banner displays correctly on iPad and iPhone (Safari + landscape rotation); verify agent slug + advisor mode stack cleanly (e.g. `/advisor/brian/calculator` or `/brian/advisor/calculator`); verify "Exit advisor mode" button correctly drops to clean path.
- 🟡 **AdvisorNote content review pending** — full inventory pulled from `AdvisorNote-C4olMM97.js` and documented in `projects/buyers-guide-advisor-notes.md` (40 talking points across 12 sections spanning /quiz, /neighborhoods, /strategy, /checklist). Reviewed for accuracy before any agent uses in a live consult — flag any line that needs rework in that file with a `🔴 REVIEW:` prefix. Source-of-truth file in the repo is `src/data/advisor-content.ts` (per Session 3 commit body) — editing copy = edit that one file, no JSX touch needed.
- 🟢 **Session 4 (optional)** — Interactive map, Market Pulse, bonus PDFs (~1.5 hrs, queued)
- 🟡 **Manus app takedown** — after Session 4

## Recent build history

- **2026-05-19 09:23 ET** — Brain-app docs cleanup, retraction of false /admin copy-bug claim. Commits to `hgpg-context`: `8ed76b5c`, `180f4997`.
- **2026-05-19 10:23 ET** — Session 3 cleanup pass in `charlotte-buyers-guide`:
  - `9c3168c` — chore: strip orphan ADVISOR_MODE_GUIDE.md reference from `api/_lib/agents.ts`
  - `2e72260` — chore: bump Last Updated header on `api/bonus/unlock.ts` 2026-05-12 → 2026-05-13
  - `2b405da` — chore: bump Last Updated header on `api/exit-intent/submit.ts`
  - `11a0a4a` — chore: bump Last Updated header on `api/fub-lead.ts`
- **2026-05-19 08:29 ET** — `58687cc` Fix FUB quiz tag keys to match canonical Quiz values. Pre-fix audit revealed 4 of 5 buyer-type selections (first_time, relocator, move_up, downsizer) and 1 of 4 lifestyle selections (water) were producing no FUB tag because `sendQuizCompletion`'s buyerTypeMap and lifestyleMap had stale keys predating current `Quiz.tsx` option values. Dead `luxury: "luxury_buyer"` entry dropped (luxury is a budget tier, not a buyer type). After fix: tags follow `<value>_buyer` / `<value>_lifestyle` pattern; bg_quiz_results.buyer_type stays unchanged. ⚠️ Implication for historical leads: any Quiz lead captured between Session 2 (2026-05-12) and 2026-05-19 has no buyer-type or lifestyle tag in FUB.
- **2026-05-19 08:20 ET** — `24543bc` Add `CLAUDE.md` to repo root with Brian's secret-handling preference (macOS Keychain, not 1Password), Vercel framework override quirk, RLS-by-design posture, and the lead-capture flow.
- **2026-05-19 06:20 ET** — `2af5340` Post-Session-3 polish: AgentProfile renders phone as `(xxx) xxx-xxxx` with raw digits in `tel:` href; "owner" dropped from public slug allowlist (so `/owner` 404s) but backend allowlist + bg_agents default-fallback row stay intact; `sendQuizCompletion` now sends real `person.tags` instead of a dead `customBuyerTags` custom field that nothing read.
- **2026-05-13 17:36 ET** — `54b952d` **Session 3: agent surfaces + advisor mode** (the canonical Session 3 commit). Adds:
  - `/:agent` route (allowlist: owner, brian, brenda, ashley, taylor) renders `AgentProfile` with personalized hero + guide feature grid; unknown slugs fall through to NotFound
  - `/advisor` and `/advisor/{quiz,calculator,neighborhoods,strategy,checklist}` auto-enable advisor mode; AdvisorBanner mounts in Layout
  - AdvisorNote callouts on Quiz (5), Strategy (5), Neighborhoods (1), Checklist (1); copy lives in `src/data/advisor-content.ts` so brand voice can be tuned without touching JSX
  - `agentDetection.ts` reads URL slug and threads it through GuideContext; every lead-capture POST includes `assignedAgentSlug`
  - `/admin` and `/agent-dashboard` gated by `ADMIN_ACCESS_KEY` via `/api/admin/verify`; `noIndex` meta added
  - Backend: `api/_lib/agents.ts` (lookupAgentBySlug, lookupDefaultAgent), `api/agent/by-slug`, `/api/agent/default`, `/api/agent/performance-stats`, `/api/admin/verify`
  - `fub-lead`, `quiz/submit`, `exit-intent/submit`, `bonus/unlock`, `calculator/track-completion`, `calculator/generate-pdf` all resolve slug → `bg_agents.id` (`assigned_agent_id`) and `bg_agents.fub_user_id` (FUB `assignLeadToAgent` on event success)
- **2026-05-12** — Session 2 complete: Calculator + Quiz + Exit Intent + Bonus Unlock ported. Eight new files, six modified. Build + tsc clean.
- **2026-05-12** — Session 1 (backend) complete. Commits `6548d53`, `a134f9a`, `04e4a21`.
- **2026-05-12** — Manus migration scope shrunk after live tRPC probe revealed empty production DB.
- **2026-05-12** — `HGPG1/charlotte-buyers-guide-manus-export` repo created with full Manus source snapshot.
- **2026-05-12** — feat(buyers): NeverBounce email validation + build-gate fixes. Commit `5c233ee`.

## Lessons noted (cumulative)

1. Code-on-`main` does NOT equal shipped-in-production.
2. Build-gate divergence: `vercel.json` vs `package.json` can drift.
3. Sandbox push works for at least the Buyers Guide repo.
4. Migration-without-source = downgrade — but only when the source was actually used.
5. AI agents inside SaaS platforms are an unofficial export channel.
6. AWS SDK deps != AWS usage.
7. Verify before rotation chaos.
8. When a SaaS AI agent loops, check whether the workspace is archived.
9. When AI agent extraction stalls, hit the deployed app directly.
10. Usage signal trumps feature inventory.
11. Vercel excludes folders starting with `_` from API discovery.
12. Vercel + Vite SPA fallback caches index.html per URL.
13. Don't poll for "endpoint live" using HTTP 200 alone.
14. Manus contact-insert had no UNIQUE(email) — fixed-on-port with UNIQUE + upsert helper.
15. Preserve historical IDs across migrations.
16. **Session 2** — Manus's "fire on unlock transition" calculator trigger doesn't map to apps where the calc isn't behind an unlock gate. Replacement gate: fire once when (a) email is captured AND (b) user has actually interacted. Avoids passive-page-load false positives.
17. **Session 2** — Auto-trigger PDF after unlock cleanly handled with a `useRef<boolean>` flag + `useEffect` on `contact?.email` transition. No callback threading through GuideContext.
18. **Session 2** — Generic `/api/capi-event` proxy lets us mirror server-side CAPI for browser-fired events (Search, Contact) without a bespoke endpoint per event. Lead/SubmitApplication mirror inline because they already POST to a lead-capture route.
19. **Session 3** — Agent slug allowlist is baked into both client (`src/lib/agentDetection.ts`) and server (`api/_lib/agents.ts`) as a static `AGENT_SLUGS` tuple. Agent DATA (name, phone, email, fub_user_id) lives in `bg_agents` and is fetched at runtime via `GET /api/agent/by-slug?slug=...` (used by `AgentProfile`) and `GET /api/agent/default` (fallback when no slug present). Roster changes that require a redeploy: add a slug to the allowlist. Roster changes that don't: edit `bg_agents` row (name, phone, fub_user_id, is_active, is_default).
20. **Session 3** — Reserved-routes table (`quiz, calculator, ...`) prevents legitimate feature paths from being misclassified as agent slugs. Centralized in `agentDetection.ts` so any new top-level route has to be added in one place.
21. **Session 3** — `localStorage` for `homegrown_assigned_agent_slug` survives the buyer closing the tab and coming back later. If the buyer follows a different agent's link the second time, the new slug overwrites. Last-touch attribution at the device level, which matches FUB's last-touch routing model.
22. **Session 3 post-mortem (2026-05-19)** — `sendQuizCompletion`'s buyerTypeMap and lifestyleMap silently drifted from `Quiz.tsx` option values across Sessions 1-3. 4 of 5 buyer-type tags and 1 of 4 lifestyle tags produced no FUB tag for ~7 days. Failure mode was silent because the map's miss simply returned undefined and the tag wasn't sent — no error, no log. Lesson: any code that maps an enum from one file to another file's set of expected values needs either (a) a build-time exhaustiveness check (TypeScript `satisfies` or a `Record<EnumKey, T>` type), or (b) a runtime warning when a value lands in the default branch. Otherwise this kind of drift is invisible.
23. **Session 3 post-mortem (2026-05-19)** — Public agent slug allowlist diverged from backend agent allowlist intentionally: `/owner` 404s on the public site (no AgentProfile rendered) but `bg_agents` still has the owner row with `is_default = true`, and the server-side `AGENT_SLUGS` still includes `owner` so default-fallback flows keep working. Two-tier allowlist is fine as long as the divergence is documented (now in `src/lib/agentDetection.ts` vs `api/_lib/agents.ts`).
