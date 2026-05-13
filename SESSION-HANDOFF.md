<!-- Last Updated: 2026-05-13 -->

# Session Handoff

## Last session: 2026-05-13 — Buyers Guide Session 2 SHIPPED ✅

Session 2 of the Manus migration is in production. Branch session-2-funnel merged to main. All four health endpoints green on prod.

### What shipped to production

New routes:
- /api/quiz/submit
- /api/calculator/track-completion
- /api/calculator/generate-pdf
- /api/exit-intent/submit
- /api/bonus/unlock
- /api/capi-event (Meta CAPI proxy for browser+server dedup)

New helpers (api/_lib/):
- contacts.ts — upsertBgContact (INSERT ... ON CONFLICT DO UPDATE)
- fub.ts updated — added updateFubPersonByEmail
- signBgPdfUrl.ts (Session 1 holdover)

Frontend:
- Quiz with locked overlay + UnlockModal capture path + autoAdvanceTimer race fix
- Calculator with equity module, URL param prefill, behavioral tag push (buy_better / rent_better / move_up_ready), auto-PDF-fire after unlock
- Exit-intent popup with topThresholdPx + minimum time + sessionStorage fire-once guard
- Meta Pixel + CAPI mirroring (Lead, Search, Contact, SubmitApplication)

### Backend verification at prod-ready point (smoke-s2-r4-20260513 email)
- bg_contacts: row exists, lead_score=125 (50 quiz + 25 calc basic + 20 pdf + 30 bonus), bonus_unlocked=true
- bg_quiz_results: row exists, lifestyle_priority='urban' (R4 canary), synced_to_fub=true
- bg_activities: bonus_unlock row with parent contact, no orphans
- All four health endpoints (db/fub/fred/storage) return 200 on prod

### Patch round history (closed out)
- R1 (35d5da8): initial Session 2 build
- 04e4a21: health endpoint rename /api/_health → /api/health for Vercel discovery
- R2 (2878215): Quiz capture gate + Calculator PDF gate + exit-intent guard
- R3 (87b316d): upsertBgContact added everywhere — fixed silent no-op chain
- R4 (16d1e79): Quiz autoAdvanceTimer race-condition fix
- Final merge to main: session-2-funnel → main

### Next: Session 3 — Agent surfaces + Advisor Mode
Goal: per-agent URLs (/brian, /ashley, /taylor, /brenda), advisor mode UX layer, agent dashboard, agent admin. Depends on Sessions 1 + 2 — both complete.

Full Session 3 kickoff prompt preserved in projects/buyers-guide-manus-migration.md. Pickup at any time.

Pre-session reminder: Session 3 introduces a /:agent slug route. Confirm with Brian whether HGPG agents are currently sharing any /brian-style URLs in the wild (Manus) — this affects the route allowlist design.

### Standing pickup notes
- Brain Write API token: macOS Keychain under HGPG_BRAIN_TOKEN (auto-exports via ~/.zshrc)
- Vercel preview bypass token: macOS Keychain under VERCEL_BYPASS_TOKEN (auto-exports via ~/.zshrc)
- Preview-scope env vars all set: SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, FUB_API_KEY, FRED_API_KEY
- Buyers Guide prod: buyersguide.homegrownpropertygroup.com on Vercel project charlotte-buyers-guide
- Supabase: HGPG Core (ioypqogunwsoucgsnmla), bg_* tables

### Deferred items (not blocking)
- Playwright headless smoke harness — would have saved hours of manual testing across Session 2 patch rounds. Worth ~15min to build before Session 3 or Session 4
- Pre-existing accessibility nits (form input id/name + label-for) on Quiz/Unlock forms
- Brain handoff hygiene: per-project handoff files (SESSION-HANDOFF-buyers-guide.md, etc.) to prevent overwrite collisions between projects worked in same week
