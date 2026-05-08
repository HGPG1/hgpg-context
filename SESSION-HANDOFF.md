<!-- Last Updated: 2026-05-08 -->

# Session Handoff

## Last session: 2026-05-07 → 2026-05-08 — Home Grown Selling Score v2 + Brain Write API 🟢

### What shipped

**1. Home Grown Selling Score v2** — replaced the 46-item wizard at `sellersguide.homegrownpropertygroup.com/home-selling-score/` with a tighter 5-category, 4-item-per-category structure (20 items total). Lead capture flipped to the END of the flow.

- New page: `public/public/public/home-selling-score/index.html` (full rewrite)
- New API endpoint: `api/assessment/submit.js` (single-call write to Supabase + fire-and-forget FUB forward)
- Supabase migration applied: `seller_assessments_v2_2026_05_07` on project `fkxgdqfnowskflgbuxhm`
- Renamed from "Home Selling Score" to "Home Grown Selling Score" throughout (title, brand header, results header, FUB source, 3x Meta Pixel content_name fields)
- Math curved: raw -20..+80 maps to display 0..80 (smooth scale, no flat ceiling)

**Score model:**
- Internal (saved as `total_score`): 4/2/1/-1 points per item, range -20 to +80, NEVER shown to clients
- Client-facing (saved as `score_percentage`): normalized 0-80 via `Math.round((raw + 20) * 0.8)`, no value over 80 reachable
- Tier (saved as `tier_name`):
  - 85+ → Market Ready (UNREACHABLE BY DESIGN — there's always a reason for a consult)
  - 70-84 → Strong Foundation (top tier client can hit, requires raw 68+)
  - 55-69 → Solid Bones
  - 0-54 → Pre-Market Project

**Categories:** Curb Appeal, Kitchen, Bathrooms, Living Spaces, Exterior and Systems.

**Pixel safety verified.** All anchor markers preserved (META_BYPASS, AssessmentStarted, ScoreCompleted, metaBypass, getUtmsForFub) so `patch-home-selling-score.py` will detect "already patched" and exit clean if re-run. Server-side `/api/fub-lead.js` and `/api/meta/capi.js` untouched.

**Final commits (charlotte-sellers-guide-vercel main):**
- `7b8f27c` (rebased to TBD) — gitignore harden (.vercel + .env*.local)
- `dad98fb` — Home Grown Selling Score: rename + curve math to cap at 80

**2. Brain Write API** — programmatic write endpoint for the brain repo at `https://brain.homegrownpropertygroup.com/api/external/write`.

- New file: `app/api/external/write/route.ts` (in brain-app repo)
- POST with `Authorization: Bearer <token>` and JSON `{path, content, message?, branch?}` commits to HGPG1/hgpg-context
- Bearer token stored in Vercel env as `BRAIN_WRITE_TOKEN`, also saved to Claude memory
- Auto-commits authored as `brian@homegrownpropertygroup.com`
- Path validator blocks `.git/`, `.github/workflows/`, `.vercel/`, `node_modules/`, `.env*`, `package-lock.json`
- Constant-time bearer compare, 1MB content cap, POST only
- Final fix: route checks `process.env.GITHUB_PAT` first (the env var brain-app actually uses). Original version checked GITHUB_TOKEN/BRAIN_GITHUB_PAT and didn't match. Fixed in commit `127cc0c`.

**This very file was written via the new API as the live end-to-end test.** No copy-paste from Claude to GitHub UI required.

### Pickup notes for next session

- Both deploys are LIVE. Smoke-test the score flow end-to-end (fill it out with a real email, verify FUB lead lands with `source: home-selling-score`).
- Sample SQL to verify v2 rows landing:
  
      SELECT * FROM seller_assessments_v2_summary LIMIT 5;

- Old `/api/assessment/create.js` and `/api/assessment/submit-ratings.js` left in place for backward compat; can be removed in a future cleanup once we confirm zero traffic for a week.
- Brain write API token is in Claude memory; future sessions can write directly without prompting Brian.

### Open follow-ups

- Verify a real lead lands and the FUB push works end-to-end (need a test submission)
- Consider whether to also rename "Home Selling Score" anywhere else in marketing/Sellers Guide nav (probably yes)

---

## Previous session: 2026-05-06 — Brain App MVP shipped 🟢

### What got built
- New Vercel project: `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- New repo: `HGPG1/brain-app` (private)
- Live at: `https://brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Single-user lock: `BRIAN_EMAIL=brian@homegrownpropertygroup.com` allow-list
- GitHub auth: fine-grained PAT scoped to `HGPG1/hgpg-context`, contents:write only (env var: `GITHUB_PAT`)
- Round-trip verified: edit file in browser → commit lands on `main` with author `brian@homegrownpropertygroup.com`

### Infra changes that affect other apps
- Resend custom SMTP wired into `HGPG Core` Supabase (project `ioypqogunwsoucgsnmla`)
  - Sender: `noreply@homegrownpropertygroup.com`, name: HGPG
  - API key stored under "Supabase HGPG Core" in Resend
  - Rate limit went from 2/hr (Supabase default) to 30/hr (Resend default), can be raised
  - This affects ALL apps using this Supabase: TM, CMA, TC Concierge, brain-app

### Pickup notes for next session
- Brain-app is live and working — can now also be updated via `/api/external/write` (added 2026-05-08)
- Resend API key is in 1Password ("Supabase HGPG Core SMTP")
- Brain-app local dev: `cd ~/Developer/brain-app && npm run dev`
