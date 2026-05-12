<!-- Last Updated: 2026-05-12 -->

# Buyers Guide - Manus to Vercel Full Migration Plan

**Sister doc:** `projects/buyers-guide.md` for current Vercel rebuild state, instrumentation status, and history. This doc is the structured 4-session port plan.

## Status

- ✅ Source snapshot extracted: `HGPG1/charlotte-buyers-guide-manus-export` (commit `e107b0e`)
- ✅ Production DB confirmed effectively empty (1 fake test lead, 0 quiz completions, 0 exit intents, 5 agent rows)
- ✅ 5 agents seeded in Manus DB (preserved in this doc, see Session 1 prompt for the seed data)
- ✅ No 301 from Manus needed (Brian decision: no dependence on Manus going forward; take Manus app down separately)
- ✅ **Session 1 (backend infrastructure) DONE 2026-05-12.** Migration plus helpers plus health endpoints live in production.
- ⏳ Session 2 queued (Calculator + Quiz + Capture)
- ⏳ Sessions 3 + 4 queued

## Scope summary

Port everything Brian built on Manus, judged on its merit (no false "regression recovery" framing, the Manus app never got real usage). Total scope ~6-10 hours across 4 Claude Code sessions, 4-6 prompts.

## Pre-session decisions (locked in)

1. **Database:** new tables in HGPG Core Supabase (`ioypqogunwsoucgsnmla`), prefixed `bg_` to namespace from other projects (`bg_contacts`, `bg_activities`, `bg_quiz_results`, `bg_agents`).
2. **FRED API key:** done. Configured on Vercel; live data confirmed via `/api/health/fred`.
3. **Google Maps API key:** deferred to Session 4.
4. **Bonus content gating:** keep current UnlockModal pattern (name + email + phone).
5. **PDF storage:** Supabase Storage bucket `bg-pdfs` (private, signed URLs). Note: bucket name uses HYPHEN per HGPG Supabase convention (e.g. `transaction-pdfs`), not underscore as initially drafted.
6. **Email uniqueness:** `bg_contacts.email` is UNIQUE. Helpers will upsert via `INSERT ... ON CONFLICT (email) DO UPDATE` in Session 2. Manus pattern was bare-insert without UNIQUE, a latent bug fixed-on-port.

## Existing agents data (seeded in `bg_agents` Session 1)

Probed live from Manus tRPC `agent.getBySlug` on 2026-05-12. Live in Supabase as of 2026-05-12 with IDs preserved.

| id | slug | name | email | phone | fubUserId | isDefault |
|---|---|---|---|---|---|---|
| 1 | owner | Brian McCarron | contact@homegrown.com | null | null | true |
| 30001 | brian | Brian McCarron | brian@homegrownpropertygroup.com | 7046779191 | 1 | false |
| 30002 | brenda | Brenda Bain | brenda@homegrownpropertygroup.com | null | 14 | false |
| 30003 | ashley | Ashley Johnson | Ashley@HomeGrownPropertyGroup.com | null | 18 | false |
| 30004 | taylor | Taylor Sheehan-Watson | Taylor@HomeGrownPropertyGroup.com | null | 21 | false |

---

## Session 1 - Backend infrastructure (~2.5 hrs) DONE 2026-05-12

Goal: stand up all the plumbing the buyer-facing features will depend on. No buyer-visible changes this session.

### What shipped (verified live)

**Supabase migrations (HGPG Core, applied via MCP, committed to repo):**

- `supabase/migrations/20260512_bg_create_tables.sql`, 4 tables, RLS, FKs, triggers
- `supabase/migrations/20260512_bg_create_storage_bucket.sql`, private `bg-pdfs` bucket
- `supabase/migrations/20260512_bg_seed_agents.sql`, idempotent DELETE-then-INSERT of 5 agents with preserved IDs

**Helpers (all in `api/_lib/`):**

- `supabase.ts`, cached service-role client
- `signBgPdfUrl.ts`, 7-day signed URLs plus `uploadBgPdf` upload helper
- `leadScoring.ts`, `SCORE_VALUES` (verbatim from Manus) plus `applyLeadScore(email, scoreType)` plain UPDATE
- `fub.ts`, `sendFUBEvent`, `sendLeadCapture`, `sendQuizCompletion`, `assignLeadToAgent`, `testFUBConnection`
- `pdfContent.ts`, verbatim
- `generatePDF.ts` plus `generateCalculatorPDF.ts`, HGPG palette substituted for Manus red/orange/sky-blue
- `marketData.ts`, FRED fetcher with fallback

**Health endpoints (`api/health/*`, NOT `_health` because Vercel excludes leading-underscore folders):**

- `/api/health/db`, 5 agents confirmed
- `/api/health/fub`, testFUBConnection ok
- `/api/health/fred`, live mortgage rate (6.37% as of 2026-05-07)
- `/api/health/storage`, upload plus sign plus delete cycle works on bg-pdfs

**Commits (all on main):**

- `6548d53` Session 1 migration. bg_* tables + storage bucket + agent seed
- `a134f9a` Session 1 backend helpers + health endpoints
- `04e4a21` rename api/_health to api/health for Vercel API discovery

### Session 1 issues encountered (folded into Lessons noted)

1. **`api/_health/` was silently excluded from function discovery.** Vercel treats leading-underscore folders the same way it treats `api/_lib/`, helpers not endpoints. Renamed to `api/health/`.
2. **CDN cache served SPA index.html for `/api/health/*` even after rename redeployed.** The cached negative response stayed warm. Bypassed with a cache-buster query string for verification; the URLs then served correctly cold.
3. **Manus did NOT upsert contacts.** All inserts were bare `db.insert(contacts).values(...)`. Score updates ran `UPDATE WHERE email = ?` against potentially-duplicated rows. Latent bug never bit them (1 real row). Decision: add UNIQUE(email) to fix-on-port. Upsert helper will land in Session 2.

### Session 1 kickoff prompt (preserved as historical record)

    You are starting Session 1 of a 4-session migration: porting the
    full feature set from the original Manus-built Buyers Guide to
    this Vercel rebuild. This session is BACKEND ONLY, no
    buyer-facing UI changes.

    CONTEXT TO LOAD FIRST

    1. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/CONTEXT.md
    2. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/SESSION-HANDOFF.md
    3. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/projects/buyers-guide.md
    4. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/projects/buyers-guide-manus-migration.md (this doc)

    The Manus source snapshot is already cloned at ~/manus-snapshot
    (symlink to ~/Documents/charlotte-buyers-guide-manus-export,
    commit e107b0e). Read these files from the snapshot as your
    reference for what to port:

    - server/leadScoring.ts (full file, 46 lines, port verbatim logic)
    - server/fub.ts (255 lines, sendLeadCapture, sendQuizCompletion,
      assignLeadToAgent, testFUBConnection helpers)
    - server/generatePDF.ts (82 lines, buyers guide PDF)
    - server/generateCalculatorPDF.ts (388 lines, calculator analysis PDF)
    - server/pdfContent.ts (128 lines, PDF text content)
    - server/storage.ts (102 lines, but DON'T port the Forge proxy;
      replace with Supabase Storage)
    - server/market-data.ts (205 lines, FRED API fetcher)
    - server/trackBonusUnlock.ts (23 lines)
    - server/emailNotification.ts (38 lines)
    - drizzle/schema.ts (Manus DB schema for reference; we are using
      Supabase, not Drizzle/MySQL)

    DELIVERABLES THIS SESSION

    1. SUPABASE SCHEMA in HGPG Core (project ioypqogunwsoucgsnmla).
       Use Supabase MCP apply_migration. Tables:

       - bg_contacts (id, name, email, phone, message, bonus_unlocked,
         lead_score, assigned_agent_id, source, created_at, updated_at)
       - bg_activities (id, type, contact_email, contact_name, platform,
         cta_type, agent_id, metadata jsonb, created_at)
       - bg_quiz_results (id, contact_email, contact_name, buyer_type,
         lifestyle_priority, commute_tolerance, budget_range,
         property_type, answers_json jsonb, synced_to_fub bool,
         created_at)
       - bg_agents (id, slug, name, email, phone, fub_user_id,
         is_active, is_default, created_at, updated_at)

       RLS: service-role only on all four tables (no anon access; all
       reads/writes happen via Vercel server routes with the
       service-role key).

       After creating the schema, ALSO commit the migration .sql file
       to supabase/migrations/ in this repo and push. (Lesson from
       2026-05-11: apply_migration alone doesn't update the repo,
       DB ledger and supabase/migrations/ are separate sources of
       truth.)

    2. SEED bg_agents with the 5 rows documented in
       projects/buyers-guide-manus-migration.md "Existing agents data
       to seed in Session 1." Use a separate migration file for
       idempotent DELETE-then-INSERT seeding.

    3. SUPABASE STORAGE bucket bg_pdfs (private). Add a signing
       helper at api/_lib/signBgPdfUrl.ts (7-day expiry, accepts path
       or legacy public URL, never throws). Mirror the pattern from
       Transaction Manager's lib/storage/signTransactionPdfUrl.ts.

    4. SERVER HELPERS in api/_lib/:

       - leadScoring.ts, port the SCORE_VALUES constant verbatim from
         the snapshot, plus a helper applyLeadScore(email, scoreType)
         that does the Supabase update
       - fub.ts, port sendLeadCapture, sendQuizCompletion,
         assignLeadToAgent. Adapt to Vercel serverless (fetch instead
         of any framework-bound HTTP)
       - pdfContent.ts, port verbatim (it's just text content)
       - generatePDF.ts plus generateCalculatorPDF.ts, port verbatim,
         change storage call to use Supabase Storage signer instead
         of Forge proxy
       - marketData.ts, port FRED fetcher; reads FRED_API_KEY env var

    5. VERCEL ENV VARS, already provisioned in Production:

       - SUPABASE_SERVICE_ROLE_KEY (HGPG Core)
       - SUPABASE_URL (https://ioypqogunwsoucgsnmla.supabase.co)
       - FRED_API_KEY
       - FUB_API_KEY

    6. SMOKE TESTS via /api/_health/* routes:

       - GET /api/_health/db, counts rows in bg_agents (should return 5)
       - GET /api/_health/fub, calls testFUBConnection helper
       - GET /api/_health/fred, fetches current 30-year mortgage rate
       - GET /api/_health/storage, signs a dummy URL

    VERIFICATION GATES

    - npm run build clean, npx tsc --noEmit clean
    - All four /api/_health/* endpoints return 200 with real data
      (not SPA index.html fallback)
    - Supabase MCP execute_sql confirms bg_agents has 5 rows
    - Migration .sql files exist in supabase/migrations/ on main

    CONSTRAINTS

    - Last-Updated comment at top of every new file
    - Commit author brian@homegrownpropertygroup.com
    - HGPG brand colors only (#2A384C navy, #A0B2C2 steel blue,
      #D1D9DF light steel, #F0F0F0 off-white, no green); no em dashes
    - Update projects/buyers-guide.md AND
      projects/buyers-guide-manus-migration.md AND
      SESSION-HANDOFF.md via the brain write API at session end.
      Brain write API: POST https://brain.homegrownpropertygroup.com/api/external/write
      with Bearer token + JSON {path, content, message}

    START BY

    Reading the four brain files above, then proposing the four
    migration .sql files (schema + seed) before applying them. Wait
    for my approval on the schema before running apply_migration.
    Then begin.

    (Notes from execution 2026-05-12:
      - Bucket renamed to bg-pdfs (hyphen) before applying, matches
        HGPG storage convention (e.g. transaction-pdfs).
      - bg_contacts.email is UNIQUE; Manus had no uniqueness which
        was a latent bug.
      - Smoke endpoints are at /api/health/* not /api/_health/*
        because Vercel excludes leading-underscore folders from API
        discovery.
    )

---

## Session 2 - Calculator + Quiz + Lead Capture (~2.5 hrs)

Goal: ship the two highest-value buyer-facing surfaces with full FUB plumbing. Calculator alone is 1,351 lines in the snapshot.

Kickoff prompt:

    You are starting Session 2 of a 4-session migration. Session 1
    backend infrastructure is complete (Supabase tables bg_*, server
    helpers in api/_lib/, FRED plus FUB plus Storage health endpoints
    green at /api/health/db, /api/health/fub, /api/health/fred,
    /api/health/storage).

    This session ports the two highest-value buyer-facing surfaces:
    Calculator and Quiz, plus exit-intent popup and bonus unlock
    tracking. End of session: buyers can complete the full funnel
    end-to-end and leads flow into Supabase plus FUB with proper
    scoring and behavioral tags.

    CONTEXT TO LOAD FIRST

    1. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/CONTEXT.md
    2. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/SESSION-HANDOFF.md
    3. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/projects/buyers-guide.md
    4. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/projects/buyers-guide-manus-migration.md

    Manus snapshot is already cloned at ~/manus-snapshot from Session
    1. Files to read from snapshot for this session:

    - client/src/pages/Calculator.tsx (1,351 lines, the beast)
    - client/src/pages/Quiz.tsx (272 lines)
    - client/src/components/ExitIntentPopup.tsx
    - client/src/hooks/useExitIntent.ts
    - client/src/components/BonusBanner.tsx
    - client/src/components/LockedContent.tsx
    - server/routers.ts (specifically the calculator.trackCompletion,
      calculator.generatePDF, quiz.submit, exitIntent.submit,
      contact.trackBonusUnlock routes, port their logic to Vercel
      /api/* routes)

    DELIVERABLES

    1. CONTACT UPSERT HELPER. New api/_lib/upsertContact.ts.
       INSERT INTO bg_contacts (name, email, phone, message, source,
       assigned_agent_id) VALUES (...)
       ON CONFLICT (email) DO UPDATE
       SET name              = EXCLUDED.name,
           phone             = COALESCE(EXCLUDED.phone, bg_contacts.phone),
           message           = COALESCE(EXCLUDED.message, bg_contacts.message),
           source            = COALESCE(EXCLUDED.source, bg_contacts.source),
           assigned_agent_id = COALESCE(EXCLUDED.assigned_agent_id, bg_contacts.assigned_agent_id),
           updated_at        = now()
       RETURNING *. Used by every capture surface (Calculator PDF,
       Quiz submit, Exit Intent, UnlockModal, ContactDialog).

    2. CALCULATOR PORT (replace existing src/pages/Calculator.tsx).
       Full rent-vs-buy with equity module, scenario switching, all
       1,351 lines of UI. On completion:
       - POST /api/calculator/track-completion, upserts contact,
         updates lead score (+25 basic, +40 with equity), pushes FUB
         custom fields (customCalculatorCompleted,
         customCalculatorRecommendation,
         customCalculatorNetDifference, customCalculatorHomePrice,
         customCalculatorMonthlyRent, plus customEquityAvailable plus
         customCurrentHomeValue if equity used), applies behavioral
         tag (buy_better / rent_better / move_up_ready)
       - POST /api/calculator/generate-pdf, server-side PDF via
         api/_lib/generateCalculatorPDF.ts, upload to Supabase Storage
         bg-pdfs bucket via uploadBgPdf, return signed URL via
         signBgPdfUrl, lead score +20

    3. QUIZ PORT (replace existing src/pages/Quiz.tsx). Full 5-question
       buyer quiz. On completion:
       - POST /api/quiz/submit, inserts row into bg_quiz_results,
         upserts contact, pushes FUB custom fields plus buyer-type
         tag, lead score +50

    4. EXIT-INTENT POPUP (new). Port useExitIntent hook plus
       ExitIntentPopup. Mouse-leave near top of viewport triggers
       modal. On submit:
       - POST /api/exit-intent/submit, upserts contact with
         source="exit_intent", generates buyers-guide PDF via
         api/_lib/generatePDF.ts, uploads to bg-pdfs bucket via
         uploadBgPdf, returns signed URL, FUB lead capture, lead
         score +20

    5. BONUS UNLOCK TRACKING. Port BonusBanner plus LockedContent
       into existing Bonuses page. On unlock:
       - POST /api/bonus/unlock, inserts row into bg_activities
         with type='bonus_unlock' and platform attribution, sets
         bg_contacts.bonus_unlocked=true, lead score +30

    6. META PIXEL EVENTS. Wire up additional Pixel events the existing
       MetaPixel.tsx doesn't fire:
       - Search event when Quiz reaches results page
       - Contact event when Calculator scenario is clicked
       - SubmitApplication event on bonus share/copy
       Mirror server-side CAPI for each via the existing
       api/_lib/meta.ts helper with shared event_id for dedup.

    VERIFICATION GATES

    - npm run build clean, npx tsc --noEmit clean
    - After deploy, end-to-end smoke:
      1. Open incognito, complete the full funnel: home, quiz,
         calculator, bonuses, exit-intent on tab-close
      2. Confirm Supabase bg_contacts has new row with correct lead_score
      3. Confirm Supabase bg_quiz_results has new row
      4. Confirm Supabase bg_activities logs the bonus unlock
      5. Confirm FUB person has all custom fields populated and the
         correct behavioral tag
      6. Confirm both PDFs generate and download cleanly via signed URLs
      7. Confirm Meta Test Events shows Lead plus Search plus Contact
         plus SubmitApplication with browser plus server dedup (single
         row each)

    CONSTRAINTS, same as Session 1 (Last-Updated comment, brian
    commit author, HGPG palette only, no em dashes).

    START BY

    Reading the brain files, then reading the Manus Calculator.tsx
    fully BEFORE writing any port code. Calculator is the biggest
    single file in this migration; a careful read pays off. Propose
    the port plan for Calculator first (component breakdown, state
    management approach), wait for my approval, then implement.

---

## Session 3 - Agent surfaces + Advisor Mode (~2 hrs)

Goal: ship per-agent URLs, advisor mode UX layer, agent dashboard, and agent admin. Depends on Session 1 + 2 being complete.

Kickoff prompt:

    You are starting Session 3 of a 4-session migration. Sessions 1
    (backend) and 2 (Calculator + Quiz + capture) are complete.

    This session ports the agent-facing surfaces and the advisor-mode
    UX layer that lets HGPG agents use the site as a co-pilot during
    client meetings.

    CONTEXT TO LOAD FIRST

    1. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/CONTEXT.md
    2. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/SESSION-HANDOFF.md
    3. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/projects/buyers-guide.md
    4. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/projects/buyers-guide-manus-migration.md
    5. ~/manus-snapshot/ADVISOR_MODE_GUIDE.md (read this fully, it
       documents the design intent for advisor mode)

    Files to read from snapshot for this session:

    - client/src/contexts/AdvisorModeContext.tsx
    - client/src/components/AdvisorBanner.tsx
    - client/src/components/AdvisorNote.tsx
    - client/src/pages/AgentProfile.tsx (the /:agent route)
    - client/src/pages/AgentDashboard.tsx
    - client/src/pages/Admin.tsx
    - client/src/lib/agentDetection.ts (URL-based agent slug detection)
    - server/routers.ts (agent.getBySlug, agent.getDefault,
      agent.getPerformanceStats, port to Vercel /api/agent/* routes)
    - The 3 pages that use advisor mode inline: Quiz.tsx (5 refs),
      Neighborhoods.tsx (1 ref), Strategy.tsx (5 refs)

    DELIVERABLES

    1. AGENT ROUTES on Vercel side. Add to src/App.tsx:
       - /:agent, AgentProfile page (must NOT collide with existing
         static routes; use a slug allowlist or regex check)
       - /advisor, home with isAdvisorMode=true
       - /advisor/quiz, /advisor/calculator, /advisor/neighborhoods,
         /advisor/strategy, /advisor/checklist, same components,
         advisor mode active
       - /admin, Admin page (protected, see point 6 below)
       - /agent-dashboard, AgentDashboard page (protected)

    2. AGENT DETECTION. Port lib/agentDetection.ts, reads slug from
       URL and sets ContactContext.assignedAgentSlug. Used by lead
       capture routes to associate new contacts with the right agent
       and route to the right FUB user.

    3. ADVISOR MODE CONTEXT. Port AdvisorModeContext + AdvisorBanner.
       localStorage key homegrown_advisor_mode persists toggle across
       sessions. AdvisorBanner shows when active.

    4. ADVISOR NOTES on existing pages. Port AdvisorNote component.
       Add inline talking-points content (extracted from the snapshot
       page files) to Quiz, Neighborhoods, Strategy, Checklist,
       gated by useAdvisorMode().

    5. AGENT API ROUTES on Vercel:
       - GET /api/agent/by-slug?slug=X, bg_agents row or null
       - GET /api/agent/default, bg_agents row where is_default=1
       - GET /api/agent/performance-stats, aggregated stats from
         bg_contacts plus bg_quiz_results plus bg_activities, grouped
         by assigned_agent_id

    6. PROTECTION on /admin and /agent-dashboard. Use the simplest
       viable gate: query string ?key=X compared to ADMIN_ACCESS_KEY
       Vercel env var. Not real auth (low-stakes internal pages,
       low traffic). Stash the key in 1Password.

    7. SEED ADVISOR MODE CONTENT. The Manus pages have inline talking
       points; extract them into a structured JSON or TS file under
       src/data/advisor-content.ts so it's editable separately from
       page components.

    VERIFICATION GATES

    - npm run build clean, npx tsc --noEmit clean
    - Hit /brian, /ashley, /taylor, /brenda on prod, all render
      personalized AgentProfile pages
    - Toggle advisor mode, banner appears, talking points show on
      Quiz/Neighborhoods/Strategy/Checklist
    - /admin and /agent-dashboard return 401-equivalent without key,
      200 with correct key
    - Lead captured on /ashley shows up in bg_contacts with
      assigned_agent_id matching Ashley's row (id 30003)
    - FUB person assigned to Ashley's fubUserId (18)

    CONSTRAINTS, same as Sessions 1 and 2.

    START BY

    Reading the brain files, then ADVISOR_MODE_GUIDE.md from the
    snapshot. Confirm with me whether HGPG agents have any external
    pages/email signatures pointing to /brian-style URLs today (this
    determines whether the existing /:agent route needs to preserve
    any specific slug behavior). Then propose the route structure and
    advisor content extraction approach. Wait for my approval before
    implementing.

---

## Session 4 - Polish (~1.5 hrs, optional)

Goal: complete parity nice-to-haves. Can be skipped or deferred indefinitely.

Kickoff prompt:

    You are starting Session 4 of a 4-session migration. Sessions 1-3
    are complete. This session ships the remaining nice-to-haves to
    achieve full parity with what Manus had. None of these are
    business-critical.

    CONTEXT TO LOAD FIRST

    1. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/CONTEXT.md
    2. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/SESSION-HANDOFF.md
    3. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/projects/buyers-guide.md
    4. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/projects/buyers-guide-manus-migration.md

    Files to read from ~/manus-snapshot:

    - client/src/components/InteractiveMap.tsx
    - client/src/components/Map.tsx (Google Maps wrapper)
    - client/src/components/MarketPulse.tsx
    - bonuses/*.pdf (3 PDF files plus their .md sources)
    - client/src/components/AIChatBox.tsx (335 lines, review whether
      worth wiring up)

    DELIVERABLES

    1. INTERACTIVE MAP on Neighborhoods page. Port InteractiveMap.tsx
       and Map.tsx. Requires GOOGLE_MAPS_API_KEY in Vercel env
       (Brian to provision before this session).

    2. MARKET PULSE component on Home page. Port MarketPulse.tsx,
       displays FRED-sourced market stats with the existing
       /api/market-data route from Session 1.

    3. BONUS CONTENT WIRED into Bonuses page. The 3 PDFs from the
       Manus snapshot (bonus1-timeline-checklist.pdf,
       bonus2-market-predictions.pdf, bonus3-negotiation-scripts.pdf)
       should be uploaded to Supabase Storage bg-pdfs bucket once,
       then served via signed URLs on the Bonuses page. Gated behind
       the existing UnlockModal.

    4. AI CHAT BOX, DECISION POINT. The Manus AIChatBox component is
       a fully built UI but was never wired to a real endpoint (only
       used in ComponentShowcase). Three options to discuss with Brian
       BEFORE implementing:

       a. Skip entirely. Component sits in snapshot for future use.
       b. Wire to Anthropic API as a buyer Q+A bot with a curated
          system prompt about Charlotte real estate. ~1 hr extra.
       c. Wire as a calculator-help bot only (narrower scope).
          ~30 min.

       Default to (a) unless Brian says otherwise.

    VERIFICATION GATES

    - npm run build clean, npx tsc --noEmit clean
    - Map renders on Neighborhoods with all neighborhood markers
    - Market Pulse shows current FRED 30-year mortgage rate
    - All 3 bonus PDFs download via signed URLs after UnlockModal
      completion

    CONSTRAINTS, same as Sessions 1-3.

    START BY

    Asking Brian (a) whether to wire up AIChatBox, and (b) confirming
    the Google Maps API key is provisioned in Vercel env. Then
    proceed.

---

## After Session 4: take Manus down

No 301 redirect (Brian decision 2026-05-12). Just unpublish/deactivate the Manus app in their dashboard. Source is permanently safe in `HGPG1/charlotte-buyers-guide-manus-export`.

If any agent has shared a `/brian`-style Manus URL in the wild and someone hits it after takedown: they get a Manus 404. Acceptable because (a) zero attributable leads have come through those URLs in 4+ months and (b) the new Vercel rebuild will have the same `/brian` slug pattern post-Session 3, just on a different host.

## Files updated by this plan

- `projects/buyers-guide.md` (status updates each session)
- `projects/buyers-guide-manus-migration.md` (this doc, session completion logs)
- `SESSION-HANDOFF.md` (rolling session summaries)
- `CONTEXT.md` (only on Session 1 start to flip Buyers Guide status, and on Session 4 completion to mark migration done)
