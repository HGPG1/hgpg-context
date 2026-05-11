<!-- Last Updated: 2026-05-11 -->

# Session Handoff

## Last session: 2026-05-06 — Brain App MVP + Phase 1.5 shipped 🟢

### What got built

**Brain App MVP**
- New Vercel project: `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- New repo: `HGPG1/brain-app` (private)
- Live at: `https://brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Single-user lock: `BRIAN_EMAIL=brian@homegrownpropertygroup.com` allow-list
- GitHub auth: fine-grained PAT scoped to `HGPG1/hgpg-context`, contents:write only
- Round-trip verified: edit file in browser → commit lands on `main` with author `brian@homegrownpropertygroup.com`

**Phase 1.5 UX polish (same day)**
- Dashboard redesigned with Pinned cards (4 hardcoded in `lib/config.ts`), Quick actions, Recently edited (top 5) — replaces duplicate file list
- Edit page sticky header with back arrow, breadcrumb, save status (30s polling: "Saved Xm ago" / "Unsaved changes" / "Saving...")
- Mobile hamburger drawer (300px, slides in from left, dimmed overlay, auto-closes on file tap, 200ms ease-out)
- Sidebar sorted by last-modified date desc within each section (Root, projects/, archive/)

### Infra changes that affect other apps

- Resend custom SMTP wired into `HGPG Core` Supabase (project `ioypqogunwsoucgsnmla`)
  - Sender: `noreply@homegrownpropertygroup.com`, name: HGPG
  - API key stored in Resend as "Supabase HGPG Core"
  - Rate limit went from 2/hr (Supabase default) to 30/hr (Resend default), can be raised
  - This affects ALL apps using this Supabase: TM, CMA, TC Concierge, brain-app
- Supabase project renames for hygiene:
  - `ioypqogunwsoucgsnmla` → "HGPG Core"
  - `ngdrliyjtqcwhhfrbxao` → "HGPG FUB Integration"
  - `wdheejgmrqzqxvgjvfee` → "HGPG Listing Reports + MLS"
  - `fkxgdqfnowskflgbuxhm` → "HGPG Signature + Relocation"
- Supabase `HGPG Core` redirect URLs added:
  - `https://brain.homegrownpropertygroup.com/**`
  - `http://localhost:3000/**`
  - (Existing tools.hgpg entries left intact)

### Bugs found and fixed mid-session

- Magic link redirected to `tools.homegrownpropertygroup.com` (Supabase Site URL fallback) — fixed by adding `/auth/callback` route handler that was missing from initial scaffold + pointing `emailRedirectTo` at it
- Supabase free SMTP rate limit (2/hr) hit during testing — fixed permanently by switching to Resend custom SMTP

### Project status updates

- `projects/brain-app.md` — status now 🟢 SHIPPED with full Phase 2 backlog documented
- `projects/hgpg-team-tools2.md` — Site URL in HGPG Core Supabase still points here for the broken app's eventual fix
- `projects/transaction-manager.md` — no code changes today, but TM benefits from Resend SMTP upgrade

### Deferred / Phase 2 for brain-app

- **Multi-token auth for `/api/external/write`** (added 2026-05-11). Today the endpoint compares against a single `BRAIN_WRITE_TOKEN` env var, so the local dev token = the Claude session token = the Vercel prod token. Add support for a comma-separated list of valid tokens (e.g., `BRAIN_WRITE_TOKEN` for master + `BRAIN_WRITE_TOKEN_CLAUDE` for chat sessions) so individual tokens can be rotated independently without disturbing local dev or breaking active automations. Minimal change in the validation function: `validTokens.some(t => timingSafeEqual(...))`. Lets us share session-scoped tokens in chat without exposing the master.
- iPhone smoke test deeper pass (CodeMirror + iOS soft keyboard scroll behavior in real-world editing)
- Cooper Hewitt self-hosted (currently falling back to system sans, not on Google Fonts)
- File rename and delete
- Diff view before save
- Cross-file search
- Multi-user allow-list (current `BRIAN_EMAIL` check is single-string but `lib/auth.ts` is structured for easy expansion)
- Audit log (Supabase table tracking who edited what, when)
- Draft autosave (Supabase table, recovers unsaved work on browser close)
- Drawer Esc-to-close + focus trap (a11y polish)
- Sticky sidebar on long dashboards (currently scrolls past)
- Save-status precision tighter than 30s polling

### Pickup notes for next session

- Brain-app is live and working — use it for any future updates to `hgpg-context`
- Resend API key is in 1Password ("Supabase HGPG Core SMTP") — verify before next rotate
- Brain-app local dev: `cd ~/brain-app && npm run dev` on Mac mini (work machine)
- Brain-app on iMac: needs fresh `gh repo clone HGPG1/brain-app` + `npm install` + `cp env.example .env.local`
- The `package-lock.json` may differ between iMac and Mac mini — push from whichever machine you most recently ran `npm install` on
- Brain-app Phase 2 backlog lives in `projects/brain-app.md`
- Pinned files on dashboard configured in `lib/config.ts` — edit that array to change pins
- Stray bundle in `~/Downloads/fub-agent-tm-files/` is unrelated FUB lead-scoring agent work for `hgpg-transaction-manager`, NOT brain-app — leave alone, separate project for another session
- Cleanup tip: `rm ~/package-lock.json` to clear the harmless "multiple lockfiles" Next.js warning (leftover from earlier npm misuse in home dir)
---

## Phase 1.6: Write API (shipped 2026-05-08)

Programmatic write endpoint for the brain repo so Claude sessions can update files directly without copy-paste through the UI.

### Endpoint

`POST https://brain.homegrownpropertygroup.com/api/external/write`

Request:

    Authorization: Bearer <BRAIN_WRITE_TOKEN>
    Content-Type: application/json
    {
      "path": "SESSION-HANDOFF.md",
      "content": "<full file contents>",
      "message": "commit message",
      "branch": "main"
    }

Response: `{ ok, path, branch, commitSha, created, bytes }`

GET request returns a non-auth health-check showing `configured: true|false` based on env var presence — useful for smoke tests after deploy.

### Auth

Bearer token in `BRAIN_WRITE_TOKEN` Vercel env var. Constant-time compare (timing-attack safe). Token is stored in Claude memory + 1Password ("HGPG Brain Write Token").

The endpoint reuses the existing `GITHUB_PAT` env var that brain-app already uses for UI-based commits — no new GitHub credentials needed. Lookup chain is `GITHUB_PAT` → `GITHUB_TOKEN` → `BRAIN_GITHUB_PAT` for resilience.

### Safety guardrails

- POST only, JSON body only
- Path validator rejects: leading `/`, `..` segments, backslashes, null bytes, control chars, paths >500 chars
- Path blocks: `.git/`, `.github/workflows/`, `.vercel/`, `node_modules/`, plus any `.env*` file or `package-lock.json` at any depth
- Content cap: 1 MB per write
- All commits authored as `brian@homegrownpropertygroup.com` (matches existing history, won't trip Vercel build hooks)

### Bug found and fixed

First deploy had endpoint reading wrong env var name (`GITHUB_TOKEN` only). Brain-app's existing PAT is stored as `GITHUB_PAT`. Fix in commit `127cc0c` added `GITHUB_PAT` as the first lookup. After fix, `configured: true` and end-to-end write loop verified.

### What this unlocks

- Claude sessions can now auto-commit SESSION-HANDOFF.md at session end
- Multi-file batch updates possible (project specs + CONTEXT.md cross-references in one session)
- Future automations (cron-style brain refreshes, scheduled audits) can write without human in the loop
