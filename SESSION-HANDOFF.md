<!-- Last Updated: 2026-05-12 -->

# Session Handoff

## Last session: 2026-05-12 — Buyers Guide Session 2 IN PROGRESS 🟡

Session 2 of the Manus migration is mid-flight. All code shipped to branch `session-2-funnel`, NOT merged to main. Production is unchanged from end-of-Session-1.

### What got built (lives on session-2-funnel branch)

New files:
- `api/_lib/contacts.ts` — upsertBgContact() helper (INSERT ... ON CONFLICT (email) DO UPDATE)
- `api/calculator/track-completion.ts` — score push + FUB custom fields + behavioral tag
- `api/calculator/generate-pdf.ts` — PDFKit -> bg-pdfs Storage -> signed URL
- `api/quiz/submit.ts` — bg_quiz_results insert + FUB sync + +50 score
- `api/exit-intent/submit.ts` — popup capture path
- `api/bonus/unlock.ts` — bonus_unlock activity + score push
- `api/capi-event.ts` — Meta CAPI proxy for server-side dedup
- `src/components/ExitIntentPopup.tsx`
- `src/hooks/useExitIntent.ts`

Modified files:
- `api/_lib/fub.ts` — added updateFubPersonByEmail()
- `src/pages/Calculator.tsx` — equity module, URL param prefill, track-completion trigger, PDF auto-fire post-unlock, HGPG palette chart colors
- `src/pages/Quiz.tsx` — locked-overlay gate on results page, UnlockModal capture path
- `src/lib/pixel.ts` — trackPixelWithCapi helper
- `src/App.tsx`, `src/components/Layout.tsx` — wiring
- `.gitignore` — added .vercel + .env*.local

### Three patch rounds during the session
1. R1 (commit 04e4a21): initial Session 2 build, all features implemented
2. R2: Quiz capture gate, Calculator PDF gate, exit-intent guard fixes
3. R3 (commit 87b316d): upsertBgContact added to /api/fub-lead + defensive upserts on every score-touching route — fixed silent no-op chain across the funnel

### Outstanding bug at pause
Quiz state-key mismatch on lifestyle field. DevTools diagnostic logs caught it:
- Quiz state captures four of five answers under expected keys
- lifestyle answer is captured (renders on results page) but under a different key than hasAllAnswers reads
- Result: /api/quiz/submit never fires, no row in bg_quiz_results, no +50 score
- Claude Code is patching at pause — task: unify the state key across (a) the lifestyle question setter, (b) hasAllAnswers check, (c) profile-display block, (d) server route payload schema

### Backend reality check (Supabase MCP, project ioypqogunwsoucgsnmla)
As of 8:30 PM ET 2026-05-12:
- bg_contacts: 2 rows total, both source=exit_intent (Quiz path never wrote contacts due to upsert-missing bug, fixed in R3 but not yet re-verified)
- bg_quiz_results: 0 rows (Quiz submit blocked by lifestyle-key bug)
- bg_activities: 2 bonus_unlock rows
- Orphan activity rows (bonus unlocks without parent contact) confirmed pre-R3, should be fixed but unverified post-patch

### Infra changes this session
- Added Preview-scope env vars: SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, FUB_API_KEY (FRED was already preview-scoped). All four needed for preview deployments to hit live backends.
- Bypass token for Vercel deployment protection stored in macOS Keychain under VERCEL_BYPASS_TOKEN, auto-exports via ~/.zshrc. Used as ?x-vercel-protection-bypass=$TOKEN&x-vercel-set-bypass-cookie=true on preview URLs.

### Brain Write API token also lives in Keychain
Stored under HGPG_BRAIN_TOKEN. Auto-exports via ~/.zshrc. Used for POST https://brain.homegrownpropertygroup.com/api/external/write with Bearer header.

### Preview URLs created this session (newest first)
- charlotte-buyers-guide-1btetyi8k.vercel.app — R3 + upsert chain
- charlotte-buyers-guide-dugl76exy.vercel.app — R2 patches (quiz/calc/exit gates)
- charlotte-buyers-guide-mqtpmesuh.vercel.app — R1 initial build (env-scope rebuild)
- charlotte-buyers-guide-me0t7ltx6.vercel.app — R1 initial (incomplete env scope)

Lifestyle-key patch (R4) is in flight from Claude Code as of pause — new preview URL will follow.

### Pickup notes for next session

1. Get latest preview URL after R4 deploys:
   vercel ls --scope home-grown-property-groups-projects | head -5

2. Cold-state smoke against new preview with fresh test email:
   - Open fresh incognito + bypass cookie URL
   - DevTools Console open BEFORE clicking anything
   - Full funnel: Quiz -> Calculator -> Bonuses
   - Watch for [Quiz submit] logs — confirm hasAllAnswers becomes true and POST fires
   - Note: Claude Code was asked to remove diagnostic logs post-fix, so they may be gone

3. Backend verification via Supabase MCP after smoke (queries to run against ioypqogunwsoucgsnmla):
   - SELECT * FROM bg_contacts WHERE email = '<test-email>';
   - SELECT * FROM bg_quiz_results WHERE contact_email = '<test-email>';
   - SELECT * FROM bg_activities WHERE contact_email = '<test-email>';
   Expected: all three return rows, bg_contacts.lead_score is sum of (50 quiz + 25 or 40 calc basic/equity + 20 calc pdf + 30 bonus) = 125 if Calculator basic, 140 if equity used.

4. FUB person verification:
   Search FUB for test email, confirm custom fields populated + correct behavioral tag (buy_better / rent_better / move_up_ready depending on calc inputs) + buyer-type tag from Quiz.

5. PDF visual eyeball:
   Open one Calculator PDF + one buyers-guide PDF (exit-intent path) from the signed URLs, confirm they render correctly inside, not just that they return 200.

6. Meta Test Events:
   Open Meta Events Manager -> Test Events tab -> confirm Lead + Search + Contact + SubmitApplication each show ONE row per event (browser+server dedup working).

7. If all green: merge session-2-funnel to main.
   git checkout main
   git pull origin main
   git merge session-2-funnel
   git push origin main
   Production preview auto-deploys. Re-run health checks against prod URL to confirm env-var drift didn't break anything.

8. Brain update (one push, not three):
   After prod is green, update SESSION-HANDOFF.md + projects/buyers-guide.md + projects/buyers-guide-manus-migration.md via brain write API. Note: brain docs from earlier in the session already claim "Session 2 shipped" — those need a corrective push.

### Deferred to Session 3 or later
- Playwright headless smoke harness — discussed but not built. Would have saved most of today's manual-smoke time. Worth ~15min in a future session for re-usability across Sessions 3+4.
- Pre-existing accessibility nits in form inputs (missing id/name + label-for associations). Not Session 2 work.

### Things to remember next session
- Always check the preview URL hash matches the latest deployment. Half a session was spent debugging stale builds. The hash is in vercel ls --scope home-grown-property-groups-projects | head -5.
- DevTools Console open BEFORE smoke testing. Diagnostic logs are useless if you don't see them.
- Clear localStorage + sessionStorage between cold-state runs. Stale state masks real bugs.
- Backend verification via Supabase MCP > eyeballing the UI. Caught the silent no-op chain that visual smoke missed.
