# Buyers Guide - Manus to Vercel Full Migration Plan

**Document Last Updated:** 2026-05-12

## Executive Summary

Structured 4-session plan to port the complete Buyers Guide application from Manus to Vercel. The migration encompasses backend infrastructure, buyer-facing calculators and quizzes, agent surfaces, and polishing features. Sessions 1 + 2 complete as of 2026-05-12.

## Current Status

**Session 1 (Backend Infrastructure) — DONE.** Production deployment includes:
- Supabase schema with `bg_contacts`, `bg_activities`, `bg_quiz_results`, `bg_agents` tables
- Server helpers for PDF generation, lead scoring, FUB integration
- Health endpoints confirming FRED API, FUB connection, and storage functionality
- 5 agent records seeded with preserved IDs

**Session 2 (Calculator + Quiz + Exit Intent + Bonus Unlock) — DONE 2026-05-12.**
Eight new files, six modified files. Build + tsc clean. Awaiting deploy + 7-step e2e smoke.

### Files added in Session 2

```
api/_lib/contacts.ts                    bg_contacts upsert + bonus-unlock flag
api/calculator/track-completion.ts      +25/+40 score + FUB custom fields + behavioral tag
api/calculator/generate-pdf.ts          PDFKit render → bg-pdfs/calculator/ → signed URL → +20
api/quiz/submit.ts                      bg_quiz_results + FUB quiz event + +50
api/exit-intent/submit.ts               upsert + FUB lead capture + PDF + signed URL + +20 + CAPI Lead
api/bonus/unlock.ts                     bg_activities + bonus_unlocked=true + +30 + CAPI SubmitApplication
api/capi-event.ts                       generic CAPI proxy for browser-fired events
src/components/ExitIntentPopup.tsx      modal mounted in App.tsx
src/hooks/useExitIntent.ts              mouse-leave detection hook
```

### Files modified in Session 2

```
api/_lib/fub.ts                         +updateFubPersonByEmail (search-then-PUT)
src/App.tsx                             mount ExitIntentPopup globally
src/components/Layout.tsx               BonusModal fires server-side unlock + shared event_id
src/lib/pixel.ts                        +trackPixelWithCapi (paired Pixel + CAPI helper)
src/pages/Calculator.tsx                rewritten — equity module, URL prefill, server completion tracking, PDF flow, CAPI Contact
src/pages/Quiz.tsx                      +server submit, +CAPI Search
```

**Sessions 3–4 queued** for Agent Surfaces/Advisor Mode and optional Polish features respectively.

## Key Pre-Session Decisions

- **Database:** New namespaced tables in HGPG Core Supabase using `bg_` prefix
- **Email uniqueness:** Implemented as `UNIQUE` constraint with upsert logic (improvement over Manus)
- **PDF storage:** Supabase Storage bucket `bg-pdfs` (hyphen per HGPG convention)
- **No Manus redirect:** "take Manus app down separately" (Brian decision)
- **Lead scoring:** Verbatim port of Manus `SCORE_VALUES` logic
- **Brand:** HGPG palette only (#2A384C navy, #A0B2C2 steel blue, #D1D9DF light steel, #F0F0F0 off-white). Three-series chart strokes use navy solid / steel solid / navy dashed.

## Session 2 implementation decisions

- **Calculator completion trigger:** Manus fired on `!isLocked` transition + email-in-localStorage. Vercel calc isn't behind a lock gate, so that trigger doesn't map. Replaced with: fire once when (a) `contact?.email` set in GuideContext AND (b) user has clicked a scenario OR moved any slider. Avoids passive-page-load false positives.
- **Risk-level → FUB tag:** Verbatim Manus. `move_up_ready` fires whenever equity module is on, regardless of holding-period risk. Risk-level string is display-only.
- **PDF unlock UX:** Click-while-locked sets `pdfAfterUnlockRef.current = true` + opens UnlockModal. A `useEffect` on `contact?.email` transition auto-fires PDF generation once email becomes set. One click = one PDF.
- **netDifference formula:** Manus: `finalEquity - (totalBuyPaid - totalRentPaid)`. Vercel: `homeEquity - totalBuyCosts + totalRentPaid`. Mathematically identical, kept Vercel form.
- **CAPI strategy:** Lead and SubmitApplication mirror inline in their lead-capture routes. Search and Contact go through `/api/capi-event` because they don't already have a route. All four use shared event_id with the browser Pixel.

## Seeded Agents Data

Five agents with preserved IDs from Manus (as of 2026-05-12):
- Owner/default agent (ID 1): Brian McCarron
- Four regional agents (IDs 30001–30004): Brian, Brenda, Ashley, Taylor

## Session Breakdown

**Session 1** (~2.5 hrs, DONE 2026-05-12): Database tables, storage bucket, health endpoints, core helpers
**Session 2** (~2.5 hrs, DONE 2026-05-12): Calculator, Quiz, exit-intent popup, bonus unlock tracking
**Session 3** (~2 hrs): Agent profile pages, advisor mode context, agent dashboard, admin routes
**Session 4** (~1.5 hrs, optional): Interactive map, market pulse, bonus PDFs, AI chatbox decision point

## Session 2 verification gates

- ✅ `npx tsc --noEmit` clean
- ✅ `npm run build` clean (Calculator bundle 365.81 KB / 106.90 KB gz, Quiz 9.99 KB / 3.47 KB gz)
- ✅ Local dev server returns 200 on /, /calculator, /quiz, /bonuses
- ⏳ Post-deploy: full incognito funnel → Supabase bg_contacts/bg_quiz_results/bg_activities rows present → FUB person populated with custom fields + tag → both PDFs download via signed URL → Meta Test Events shows Lead/Search/Contact/SubmitApplication with browser+server dedup

## Post-Migration

The Manus application will be deactivated without 301 redirects. All functionality replicates on the Vercel rebuild with the same agent slug pattern (`/brian`, `/ashley`, etc.), just on a new host.
