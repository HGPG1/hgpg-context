# Session Handoff

**Last updated:** 2026-05-12 (end of Session 2)

## Buyers Guide Session 2 — DONE pending deploy

Calculator + Quiz + Exit Intent + Bonus Unlock are ported, type-checked, and built. Eight new files, six modified. Awaiting deploy + 7-step end-to-end smoke.

### What shipped (locally, not yet deployed)

- **Calculator** (`src/pages/Calculator.tsx`) rewritten with move-up equity module, URL prefill, three preset scenarios, server-side completion tracking (+25/+40), PDF download (+20), click-while-locked auto-trigger PDF after unlock. Chart in HGPG palette (navy solid / steel solid / navy dashed).
- **Quiz** (`src/pages/Quiz.tsx`) extended with server submit → `/api/quiz/submit` → bg_quiz_results + FUB buyer-type tag + +50 lead score.
- **Exit Intent** (`src/components/ExitIntentPopup.tsx`, `src/hooks/useExitIntent.ts`) — global mount in App.tsx. Suppressed if user already unlocked OR has seen it this session.
- **Bonus Unlock** wired in Layout.tsx BonusModal — share/copy fires Pixel SubmitApplication + `/api/bonus/unlock` (writes bg_activities, sets bonus_unlocked=true, +30 score, mirrors CAPI).
- **Meta CAPI** mirror for Search + Contact via new `/api/capi-event` proxy. Lead + SubmitApplication mirror inline in their lead-capture routes. All paired events use shared event_id for browser+server dedup.

### Verification status

- ✅ `npx tsc --noEmit` clean
- ✅ `npm run build` clean
- ✅ Local dev: /, /calculator, /quiz, /bonuses all 200
- ⏳ Post-deploy 7-step smoke checklist — Brian to run

### Post-deploy 7-step smoke (from Session 2 prompt)

1. Open incognito, complete the full funnel: home → quiz → calculator → bonuses → exit-intent on tab-close
2. Confirm Supabase `bg_contacts` has new row with correct lead_score
3. Confirm Supabase `bg_quiz_results` has new row
4. Confirm Supabase `bg_activities` logs the bonus unlock
5. Confirm FUB person has all custom fields populated and the correct behavioral tag (`buy_better` / `rent_better` / `move_up_ready`)
6. Confirm both PDFs (Rent-vs-Buy from `/api/calculator/generate-pdf` and Buyers Guide from `/api/exit-intent/submit`) generate and download cleanly via signed URLs
7. Confirm Meta Test Events shows Lead + Search + Contact + SubmitApplication with browser+server dedup (single row each)

### Open items / non-blocking

- 🟡 `META_CAPI_TEST_EVENT_CODE` env var — remove once QA confirms dedup
- 🟡 Chart-color sanity check — three Area series use navy solid, steel solid, navy dashed. If they read muddy in production, swap equity to a marker line per the brand-color constraint (no green/sky/orange fallback).

## Next session

**Session 3 — Agent surfaces + Advisor Mode (~2 hrs queued).** Will port:
- `/:agent` vanity pages (uses preserved `bg_agents` IDs)
- `/admin` route
- `/agent-dashboard` route
- Advisor mode (the in-page scripts + objection handlers seen in Manus quiz/calc components)

Manus snapshot at `~/manus-snapshot/`. Files to read next session:
- `client/src/pages/AgentProfile.tsx`
- `client/src/pages/AgentDashboard.tsx`
- `client/src/pages/Admin.tsx`
- `client/src/components/AdvisorNote.tsx`, `AdvisorBanner.tsx`
- `ADVISOR_MODE_GUIDE.md`

## Other active workstreams

See `CONTEXT.md` "What is active right now" for Transaction Manager, TC Concierge, CMA Engine, Listing Report Portal, Brain App, FUB AI Agent, Sellers Guide.
