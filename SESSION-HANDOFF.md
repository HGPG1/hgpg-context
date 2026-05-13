<!-- Last Updated: 2026-05-13 -->

# Session Handoff

## Active: 2026-05-13 — Buyers Guide Session 2 still in progress 🟡

Note: this handoff was overwritten yesterday by a ReZEN review-request session that landed between Buyers Guide working windows. ReZEN work is shipped and documented elsewhere. This handoff is back to Buyers Guide pickup.

### Where we are
- Branch: session-2-funnel (HEAD = 16d1e79, R4 patch shipped)
- Production: unchanged from end-of-Session-1 (main HEAD = 04e4a21)
- Latest preview to smoke: https://charlotte-buyers-guide-c9ydyjmsh.vercel.app

### Patch round history (all on session-2-funnel)
- R1 (35d5da8 → 04e4a21): initial Session 2 build, then health endpoint rename
- R2 (2878215): Quiz capture gate + Calculator PDF gate + exit-intent guard
- R3 (87b316d): upsertBgContact added everywhere — fixed silent no-op chain
- R4 (16d1e79): Quiz autoAdvanceTimer race-condition fix

### Pickup task (one cold-state smoke against c9ydyjmsh)
1. Fresh incognito + bypass cookie URL on c9ydyjmsh
2. DevTools Console + Network tab open BEFORE clicking
3. Test email format: smoke-s2-r4-YYYYMMDD@homegrownpropertygroup.com
4. Full funnel at NORMAL CADENCE: Quiz (slow-and-steady on all 5 questions) → results page → Unlock my profile → Calculator → Download PDF → Bonuses → unlock one bonus
5. Skip exit-intent (P1, working from prior runs)
6. Report any red console errors or 4xx/5xx in Network tab
7. Claude verifies backend via Supabase MCP (project ioypqogunwsoucgsnmla): bg_contacts, bg_quiz_results, bg_activities rows for test email + score totals

### Expected lead score totals
- Quiz: +50
- Calculator basic: +25 (+40 if equity used)
- Calculator PDF: +20
- Bonus unlock: +30
- Total: 125 (basic) or 140 (equity)

bg_quiz_results row must have lifestyle_priority populated (was the canary for R4 race-condition fix).

### Merge path once smoke is green
git checkout main
git pull origin main
git merge session-2-funnel
git push origin main
Verify prod health endpoints, then push corrective brain docs (Sessions 2 actually shipped).

### Infrastructure state
- Vercel project: charlotte-buyers-guide on team home-grown-property-groups-projects
- Preview env vars (all 4 set): SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, FUB_API_KEY, FRED_API_KEY
- Bypass token: macOS Keychain under VERCEL_BYPASS_TOKEN (auto-exports via ~/.zshrc)
- Brain Write API token: macOS Keychain under HGPG_BRAIN_TOKEN (auto-exports via ~/.zshrc)
- Health endpoints renamed in 04e4a21: /api/health/* not /api/_health/* (no underscore)

### Deferred to next pickup or later
- Playwright headless smoke harness — would have saved hours of manual smoke testing across patch rounds
- Pre-existing accessibility nits (form input id/name + label-for)
- Brain handoff hygiene — Session 2 notes got clobbered by ReZEN once already. Consider per-project handoff files (SESSION-HANDOFF-buyers-guide.md, SESSION-HANDOFF-rezen.md) or append-only log pattern

### Things to remember
- Check preview URL hash matches latest deployment before testing (vercel ls)
- DevTools Console open BEFORE smoke
- Clear localStorage + sessionStorage between cold runs
- Backend verification via Supabase MCP > eyeballing the UI
- Use slow-and-steady cadence on Quiz to verify R4 race fix held
