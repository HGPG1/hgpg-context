<!-- Last Updated: 2026-05-12 -->

# Buyers Guide

**Status:** 🟢 Instrumentation live (Pixel + CAPI + NeverBounce). **Session 1 of Manus migration complete** - backend schema, helpers, and health endpoints in production. Sessions 2-4 queued.

**🟢 Manus migration scope shrunk dramatically 2026-05-12.** Probed the live Manus tRPC endpoints directly. The production database has effectively **no data** (1 fake test row in `contacts`, 0 quiz results, 0 exit-intent submissions, 5 agent config rows). The Manus app shipped feature-rich but never got real usage. The Vercel rebuild is the real production system. The 8 "missing features" are forward-looking ports, NOT regressions from a thriving app. Re-evaluate each on its own merit, not on "we used to have this."

## URLs

- Production (Vercel rebuild): https://buyersguide.homegrownpropertygroup.com
- Original (Manus, archived workspace): https://homegrownbg-hyqbjnnt.manus.space (still live, still serving but ~empty DB)
- Repo (Vercel rebuild): `HGPG1/charlotte-buyers-guide` (private)
- Repo (Manus original): `HGPG1/homegrown-buyer-guide` (private, ARCHIVED - caused Manus AI agent to loop on schema/migration files instead of reading live data)
- Repo (Manus full source snapshot): `HGPG1/charlotte-buyers-guide-manus-export` (private, created 2026-05-12). 220 files, first commit `e107b0e`.
- Local working copy of snapshot: `~/manus-snapshot/`
- Stack (Vercel): React 19 + Vite 8 + Tailwind v4 + react-router-dom 7. `@supabase/supabase-js` + `pdfkit` added Session 1.
- Stack (Manus): React 19 + Vite 7 + Tailwind v4 + wouter + tRPC v11 + Drizzle ORM + MySQL + Express + Manus Forge storage proxy
- Vercel team: `team_FietQPKCmnyioG2n0FdteQCV`
- Vercel project: `prj_AQ0xkwT0IgQFssJeBbcPOmTFdmOJ` (`charlotte-buyers-guide`)

## Supabase (HGPG Core `ioypqogunwsoucgsnmla`) - added Session 1

Tables (all RLS enabled, no policies, service-role-only):

- `bg_contacts` - id, name, email (UNIQUE), phone, message, bonus_unlocked bool, lead_score int, assigned_agent_id FK, source, timestamps
- `bg_activities` - id, type, contact_email, contact_name, platform, cta_type, agent_id FK, metadata jsonb, created_at
- `bg_quiz_results` - id, contact_email, contact_name, buyer_type, lifestyle_priority, commute_tolerance, budget_range, property_type, answers_json jsonb, synced_to_fub bool, created_at
- `bg_agents` - id, slug, name, email, phone, fub_user_id, is_active, is_default, timestamps

**Storage bucket:** `bg-pdfs` (private, 10 MB cap, `application/pdf` only). Signed-URL access via `api/_lib/signBgPdfUrl.ts`.

**Migration files:**
- `supabase/migrations/20260512_bg_create_tables.sql`
- `supabase/migrations/20260512_bg_create_storage_bucket.sql`
- `supabase/migrations/20260512_bg_seed_agents.sql`

**Seeded agents (live `bg_agents` rows, IDs preserved from Manus):**

| id | slug | name | email | fub_user_id | is_default |
|---|---|---|---|---|---|
| 1 | owner | Brian McCarron | contact@homegrown.com | NULL | true |
| 30001 | brian | Brian McCarron | brian@homegrownpropertygroup.com | 1 | false |
| 30002 | brenda | Brenda Bain | brenda@homegrownpropertygroup.com | 14 | false |
| 30003 | ashley | Ashley Johnson | Ashley@HomeGrownPropertyGroup.com | 18 | false |
| 30004 | taylor | Taylor Sheehan-Watson | Taylor@HomeGrownPropertyGroup.com | 21 | false |

## Server helpers (`api/_lib/`)

- `supabase.ts` - cached service-role client. Soft-fails (returns null) when env unset.
- `signBgPdfUrl.ts` - 7-day signed URL helper, accepts bucket path or legacy public URL. Also exports `uploadBgPdf` and `BG_PDFS_BUCKET` constant.
- `leadScoring.ts` - `SCORE_VALUES` constants + `applyLeadScore(email, scoreType)` (plain UPDATE WHERE email; no-op if contact missing).
- `fub.ts` - `sendFUBEvent`, `sendLeadCapture`, `sendQuizCompletion`, `assignLeadToAgent`, `testFUBConnection`.
- `pdfContent.ts` - Buyers Guide PDF copy (text-only, ported verbatim).
- `generatePDF.ts` - Buyers Guide PDF generator (HGPG palette).
- `generateCalculatorPDF.ts` - Rent-vs-Buy analysis PDF (HGPG palette).
- `marketData.ts` - FRED fetcher (`MORTGAGE30US`, `MEDLISPRI16740`, `MEDDAYONMAR16740`, `ACTLISCOU16740`) with static fallback.

## Health endpoints (`/api/health/*`)

Live in production 2026-05-12. **Path uses `health` not `_health`** because Vercel excludes leading-underscore folders from function discovery.

- `GET /api/health/db` - counts `bg_agents` rows. Returns `{ok, agentCount, expected, healthy}`.
- `GET /api/health/fub` - calls `testFUBConnection`. Returns `{ok, keyConfigured}`.
- `GET /api/health/fred` - fetches current 30-yr mortgage rate. Returns `{ok, mortgageRate, asOf}`.
- `GET /api/health/storage` - uploads dummy PDF to `bg-pdfs`, signs URL, deletes dummy. Returns `{ok, bucket, signedUrlPreview}`.

## Lead capture (current Vercel rebuild - pre-Session 2)

Two forms in `src/components/Layout.tsx`: **UnlockModal** (guide-unlock gate) and **ContactDialog** ("Get Started" CTA). Both call `validateEmail()` from `src/lib/validateEmail.ts` BEFORE Pixel + FUB POST. Fail-closed on `invalid`/`disposable`; allow-through on `valid`/`catchall`/`unknown`/network failure. FUB ingestion via `api/fub-lead.ts` → POST to `/v1/events` with `source: "Charlotte Buyer's Guide"`.

**Session 2 will add:** the upsert-contact helper that pairs with the new UNIQUE(email) constraint, and writes to `bg_contacts` alongside the FUB POST.

## Lead scoring (`api/_lib/leadScoring.ts` - ported from Manus)

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

`applyLeadScore(email, scoreType)` is a plain UPDATE WHERE email = ?. It does NOT upsert because the contact is created at lead-capture time. No-ops with a warn log if the email has no contact row.

## Vercel env vars in Production

| Var | Value / source | Scope | Status |
|---|---|---|---|
| `VITE_META_PIXEL_ID` | `1449157226505129` | Build | ✅ live |
| `META_PIXEL_ID` | `1449157226505129` | Runtime | ✅ live |
| `META_CAPI_ACCESS_TOKEN` | Meta Events Manager → Settings → CAPI | Runtime | ✅ live |
| `META_CAPI_TEST_EVENT_CODE` | TEST code (remove before scaling) | Runtime | optional |
| `NEVERBOUNCE_API_KEY` | reuse Sellers Guide value | Runtime | ✅ live |
| `FUB_API_KEY` | confirmed via /api/health/fub | Runtime | ✅ live |
| `SUPABASE_URL` | `https://ioypqogunwsoucgsnmla.supabase.co` | Runtime | ✅ live |
| `SUPABASE_SERVICE_ROLE_KEY` | HGPG Core service role | Runtime | ✅ live |
| `FRED_API_KEY` | confirmed via /api/health/fred | Runtime | ✅ live |

---

## Manus migration - REVISED PLAN (2026-05-12)

### Session progress

- ✅ **Round 1 (probe):** done
- ✅ **Round 2 (map):** done
- ✅ **Path B (full source export):** done. 220 files committed to `HGPG1/charlotte-buyers-guide-manus-export`.
- ✅ **Path C (data export):** done via direct tRPC probe of live Manus site.
- ✅ **Session 1 (backend infrastructure):** done 2026-05-12. Schema, helpers, health endpoints all green in production.
- ⏳ **Session 2 (Calculator + Quiz + capture):** next.
- ⏳ **Session 3 (Agent surfaces + Advisor Mode):** queued.
- ⏳ **Session 4 (Polish - maps, market pulse, bonus PDFs, optional AI chat):** queued.

### Live Manus DB state (probed via tRPC 2026-05-12)

- `contacts`: **1 row** - `Test User / test@example.com / Exit-intent PDF download` from 2026-01-20. Fake test data, not a real lead.
- `quizResults`: **0 rows**
- `activities`: not directly queryable, but `agent.getPerformanceStats` returned `totalExitIntent: 0`. Likely 0.
- `agents`: **5 rows** (the only real production data; preserved verbatim into new `bg_agents` table with IDs intact).

### Why the Manus AI agent looped on Path C

The Manus workspace repo (`HGPG1/homegrown-buyer-guide`) is GitHub-archived. The Manus agent appears to be sandboxed to that workspace's files. When it tried to find production data, all it could see were repo files: schema (DDL only), migrations (DDL only), and `.manus/db/*.json` (cached DDL queries). It substituted those for production data because the real DB query path was unreachable from its tooling. Lesson noted below.

### Manus 301 redirect

Safe to set up. No data to preserve, no users to disrupt. Configure on Manus's side: set destination URL on the app to `https://buyersguide.homegrownpropertygroup.com`. Or take the app down with a forwarding URL. Brian's decision was to skip the 301 entirely and take the Manus app down after Session 4.

## Pending

- 🟡 **Buyers Guide Pixel + CAPI verification** - dedup confirmation in Meta Test Events (separate from migration work)
- 🟡 **`META_CAPI_TEST_EVENT_CODE` cleanup** - remove from Vercel env once QA confirms dedup
- 🟢 **Session 2** - Calculator + Quiz + Exit Intent + Bonus Unlock tracking (~2.5 hrs, queued)
- 🟢 **Session 3** - Agent surfaces + Advisor Mode (~2 hrs, queued)
- 🟢 **Session 4 (optional)** - Interactive map, Market Pulse, bonus PDFs (~1.5 hrs, queued)
- 🟡 **Manus app takedown** - after Session 4 completes

## Recent build history (most recent first)

- **2026-05-12** - Session 1 (backend) complete: bg_* schema, server helpers, health endpoints. Commits `6548d53`, `a134f9a`, `04e4a21`. All four health endpoints green in production.
- **2026-05-12** - Manus migration scope shrunk after live tRPC probe revealed empty production DB. Skipping Phases 2 + 3.
- **2026-05-12** - `HGPG1/charlotte-buyers-guide-manus-export` repo created with full Manus source snapshot (220 files, commit `e107b0e`).
- **2026-05-12** - Manus Path B complete. 220-file zip extracted via AI agent.
- **2026-05-12** - Manus AI agent extraction Rounds 1 + 2.
- **2026-05-12** - feat(buyers): NeverBounce email validation + form gates + build-gate fixes. Commit `5c233ee`. PR merged.
- 2026-04-28 → 2026-05-01 - Meta Pixel + CAPI scaffolding shipped to repo. Inert in production until 2026-05-12 because env vars never provisioned.

## Lessons noted

1. **Code-on-`main` does NOT equal shipped-in-production.** Verify against built bundle.
2. **Build-gate divergence:** `vercel.json` vs `package.json` can drift; run `npm run build` locally before merging.
3. **Sandbox push works** for at least the Buyers Guide repo.
4. **Migration-without-source = downgrade** - but only when the source app was actually being used. Original Manus→Vercel migration ported only buyer-facing UI; eight server-side features dropped. Turns out NONE of those features had real usage, so the downgrade was theoretical. Always check usage signal before treating a missing feature as a regression.
5. **AI agents inside SaaS platforms are an unofficial export channel.** Probe with innocuous files; verify against production bundle; escalate to full zip. Worked here for source code.
6. **AWS SDK deps != AWS usage.** Read the code, don't infer infra from `package.json`.
7. **Verify before rotation chaos.** Grep extracted source for secrets, then probe them. Cheap to verify.
8. **When a SaaS AI agent loops, check whether the workspace is archived/disconnected.** A GitHub-archived workspace can leave the AI agent sandboxed to dead files.
9. **When AI agent extraction stalls, hit the deployed app directly.** Live tRPC endpoints, REST endpoints, or even authenticated admin URLs are usually faster.
10. **Usage signal trumps feature inventory.** Eight "missing features" in a gap analysis is meaningless if the source app had zero real users.
11. **Vercel excludes folders starting with `_` from API discovery.** Same rule that hides `api/_lib/`. So `api/_health/*` was silently absent from the deployed function bundle and hitting the SPA index.html fallback. Confirmed via `lambdaRuntimeStats: nodejs:2` before rename, `nodejs:6` after.
12. **Vercel + Vite SPA fallback caches the index.html response per URL.** When the function deploy finally landed, the CDN was still serving the cached SPA for previously-404'd `/api/health/*` URLs. A cache-buster query string broke through; the URLs then served correctly cold.
13. **Don't poll for "endpoint live" using HTTP 200 alone** when the SPA fallback also returns 200. Check the response body shape or use a cache-buster.
14. **Manus contact-insert had no UNIQUE(email)** and used `UPDATE WHERE email` for score updates, which would multiply lead_score N times on repeat visits. Latent bug never hit them (1 fake row). Fixed-on-port with UNIQUE constraint + upsert helper for Session 2.
15. **Preserve historical IDs across migrations when downstream docs reference them by ID.** Session 3 prompt names "id 30003 = Ashley". Cheap to preserve, expensive to retro-update.
