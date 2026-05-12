<!-- Last Updated: 2026-05-12 -->

# Session Handoff

## Last session: 2026-05-12 - Buyers Guide Session 1 (backend infrastructure) shipped 🟢

Session 1 of the 4-session Manus → Vercel migration is complete. Backend plumbing is live in production; zero buyer-facing UI changes this session.

### What landed

**Supabase schema in HGPG Core (`ioypqogunwsoucgsnmla`):**
- `bg_contacts` (UNIQUE email + lead_score + assigned_agent_id FK + bonus_unlocked + source)
- `bg_activities` (type + contact_email + platform + cta_type + agent_id + metadata jsonb)
- `bg_quiz_results` (full buyer-quiz answer storage + synced_to_fub flag)
- `bg_agents` (5 production agents seeded with preserved Manus IDs 1, 30001-30004)
- RLS enabled on all 4 tables, zero policies (service-role-only)
- `updated_at` triggers on `bg_agents` + `bg_contacts`
- FKs `ON DELETE SET NULL` so deleting an agent never orphans/cascades contacts

**Decision: switched from Manus's bare-insert pattern to UNIQUE(email) + upsert.** Manus had no `.unique()` on contacts.email and used `db.update WHERE email = ?` for score updates, which would double-count any repeat lead. Latent bug never hit them because live DB had 1 row. Fresh schema, no data to preserve, fixed it.

**Storage:** Private `bg-pdfs` bucket (10 MB cap, application/pdf only). Renamed from `bg_pdfs` (underscore) to `bg-pdfs` (hyphen) to match Transaction Manager's `transaction-pdfs` convention.

**Server helpers in `api/_lib/`:**
- `supabase.ts` - cached service-role client
- `signBgPdfUrl.ts` - 7-day signed URLs, accepts path OR legacy public URL, never throws. Mirrors TM's signTransactionPdfUrl pattern. Also exports `uploadBgPdf`.
- `leadScoring.ts` - `SCORE_VALUES` ported verbatim + `applyLeadScore(email, scoreType)` (plain UPDATE WHERE email; no-ops if contact doesn't exist)
- `fub.ts` - `sendFUBEvent`, `sendLeadCapture`, `sendQuizCompletion`, `assignLeadToAgent`, `testFUBConnection`
- `pdfContent.ts` - Buyers Guide PDF copy, ported verbatim
- `generatePDF.ts` + `generateCalculatorPDF.ts` - ported with HGPG palette substituted for Manus's red/orange/sky-blue
- `marketData.ts` - FRED fetcher with static fallback

**Smoke-test endpoints at `/api/health/*`** (NOT `_health` - Vercel excludes leading-underscore folders from function discovery; learned the hard way):
- `/api/health/db` → `{"ok":true,"agentCount":5,"expected":5,"healthy":true}`
- `/api/health/fub` → `{"ok":true,"keyConfigured":true}`
- `/api/health/fred` → `{"ok":true,"mortgageRate":"6.37%","asOf":"2026-05-07"}`
- `/api/health/storage` → `{"ok":true,"bucket":"bg-pdfs","signedUrlPreview":"..."}`

All verified live on production 2026-05-12.

### Commits (all on `main`)

- `6548d53` feat(buyers): Session 1 migration. bg_* tables + storage bucket + agent seed
- `a134f9a` feat(buyers): Session 1 backend helpers + health endpoints
- `04e4a21` fix(buyers): rename api/_health to api/health for Vercel API discovery

### Lessons noted (key ones from today)

1. **Vercel excludes folders starting with `_` from API discovery.** Same rule that hides `api/_lib/`. So `api/_health/*` was silently absent from the deployed function bundle and hitting the SPA index.html fallback. Confirmed via `lambdaRuntimeStats: nodejs:2` before rename, `nodejs:6` after.
2. **Vercel + Vite SPA fallback caches the index.html response per URL.** When the function deploy finally landed, the CDN was still serving the cached SPA for previously-404'd `/api/health/*` URLs. A cache-buster query string broke through; the URLs then served correctly cold.
3. **Don't poll for "endpoint live" using HTTP 200 alone** when the SPA fallback also returns 200. Check the response body shape or use a cache-buster.
4. **Manus contact-insert had no UNIQUE(email).** Repeat visits would have multiplied lead_score N times per `UPDATE WHERE email = ?`. Latent bug, never triggered (1 fake row). Fixed-on-port with UNIQUE constraint + upsert helper planned for Session 2.
5. **Preserve historical IDs across migrations when downstream docs reference them by ID** (Session 3 prompt names "id 30003 = Ashley"). Cheap to preserve, expensive to retro-update if you don't.

---

## Next session queued: Buyers Guide Session 2 (Calculator + Quiz + capture)

~2.5 hrs. Buyer-facing surfaces. Calculator port is the heavy lift (1,351 lines). End of session: full funnel end-to-end with leads flowing into bg_contacts + FUB with proper scoring and behavioral tags.

**Session 2 kickoff prompt:** verbatim in `projects/buyers-guide-manus-migration.md` under "Session 2 - Calculator + Quiz + Lead Capture."

### Pre-Session 2 prereqs (Brian)

1. Open a fresh Claude Code thread in `~/Documents/charlotte-buyers-guide` and paste the Session 2 prompt.
2. Manus snapshot stays at `~/manus-snapshot`.
3. All Session 1 env vars already provisioned. No new env vars needed for Session 2 unless we surface them in flight.

---

## Previous session: 2026-05-12 (earlier same day) - Buyers Guide instrumentation + Manus full extraction

(Preserved below for continuity.)

### Highlights

**Buyers Guide instrumentation (shipped + merged):**
- Pixel + CAPI code was on `main` since late April but inert (`VITE_META_PIXEL_ID` unset → Vite tree-shook). Made live this session.
- NeverBounce shipped from scratch in commit `5c233ee` on `claude/buyers-guide-instrumentation-nGCLI`, PR merged.
- Pixel ID for Buyers Guide: `1449157226505129`. NeverBounce key: reuse Sellers Guide value.
- Env vars + redeploy + verification pending Brian's hands.

**Manus full extraction (Path B + Path C complete):**
- Manus AI agent (inside their dashboard) cooperatively exported full source as a zip after 2 probe rounds.
- 220 files extracted, committed to new private repo `HGPG1/charlotte-buyers-guide-manus-export` (commit `e107b0e`).
- One historic FUB key found in `scripts/get-fub-users.ts` (`fka_0cyqBU…`) - probed FUB API, returned 401, already revoked.
- Forge storage clarification: Manus uses its own proxy, NOT AWS S3 directly (despite `@aws-sdk` deps).
- **Path C resolved via direct tRPC probe of live Manus site, NOT via AI agent** (which was looping on archived repo files - the GitHub-archived workspace appears to sandbox the agent away from live DB).
- **Production DB state: effectively empty.** 1 fake test lead, 0 quiz completions, 0 exit intents, 5 agent config rows.
- Manus migration scope COLLAPSED. 4 phases → 1 forward-looking phase. Phase 3 features (admin, dashboard, advisor mode, /:agent) re-judged: forward-looking ports, not regressions.
- Brian decided: no 301 redirect (Manus burned bridges around build time). Take Manus down after Vercel migration completes.

**Full migration plan staged for new thread:** see Next session above.

---

## Previous session: 2026-05-11 - Brain reconciliation + transaction-pdfs cleanup

Reconciled CONTEXT.md against project files (which were ahead), closed six items: Sherlock 403, Mac Mini GitHub auth, exposed GitHub PAT rotation, CMA Engine MLS Grid auto-pull, NC office routing in ReZEN (verified live in code), transaction-pdfs bucket flip → private (bucket flipped via MCP on 2026-05-09; migration file backfilled to repo on 2026-05-11 as `2eb9794`).

Also: rebuilt `projects/brain-app.md` (had been clobbered), updated CONTEXT.md "active right now" list.
