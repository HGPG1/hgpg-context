# Buyers Guide - Full Document Content

<!-- Last Updated: 2026-05-12 -->

## Buyers Guide

**Status:** 🟢 Sessions 1 + 2 of Manus migration complete. Backend infrastructure live since the morning of 2026-05-12; buyer-facing funnel (Calculator + Quiz + Exit Intent + Bonus Unlock) shipped to `main` the same day. Sessions 3-4 queued.

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

### Lead-capture endpoints (Session 2)

| Route | Writes | Score | FUB | CAPI |
|---|---|---|---|---|
| POST `/api/fub-lead` | (pre-S2; no Supabase) | — | event | Lead (inline) |
| POST `/api/calculator/track-completion` | — | +25 basic / +40 equity | search-then-PUT custom fields + behavioral tag | — |
| POST `/api/calculator/generate-pdf` | upload → bg-pdfs/calculator/ | +20 | — | — |
| POST `/api/quiz/submit` | bg_quiz_results | +50 | sendQuizCompletion + buyer-type tag | — |
| POST `/api/exit-intent/submit` | bg_contacts upsert + bg-pdfs/exit-intent/ | +20 | sendLeadCapture | Lead (mirror) |
| POST `/api/bonus/unlock` | bg_activities + bg_contacts.bonus_unlocked=true | +30 | — | SubmitApplication (mirror) |
| POST `/api/capi-event` | — | — | — | Generic proxy (Search, Contact) |

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
- 🟢 **Session 3** — Agent surfaces + Advisor Mode (~2 hrs, queued)
- 🟢 **Session 4 (optional)** — Interactive map, Market Pulse, bonus PDFs (~1.5 hrs, queued)
- 🟡 **Manus app takedown** — after Session 4

## Recent build history

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
