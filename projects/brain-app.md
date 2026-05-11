<!-- Last Updated: 2026-05-11 -->

# Brain App

- **Status:** 🟢 Live, MVP + Phase 1.5 + Write API shipped
- **URL:** https://brain.homegrownpropertygroup.com
- **Repo:** HGPG1/brain-app (private)
- **Vercel project:** `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- **Supabase:** `ioypqogunwsoucgsnmla` (HGPG Core — auth only, no app tables)
- **Stack:** Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- **Local dev (Mac mini):** `cd ~/brain-app && npm run dev`
- **Local dev (iMac):** `gh repo clone HGPG1/brain-app && npm install && cp env.example .env.local`

## Purpose

Web-based editor for the brain repo (`HGPG1/hgpg-context`). Any device, magic-link auth, single-user. Replaces the need to clone the brain repo and hand-edit markdown to keep session continuity working. Also exposes a programmatic write API so Claude sessions can commit directly without copy-paste through the UI.

## Auth

- Supabase Auth magic link, sent via Resend custom SMTP (30/hr rate limit)
- Single-user allow-list via `BRIAN_EMAIL=brian@homegrownpropertygroup.com` env var
- `lib/auth.ts` is structured for easy multi-user expansion if needed
- Redirect URLs configured in Supabase HGPG Core: `https://brain.homegrownpropertygroup.com/**` and `http://localhost:3000/**`

## GitHub integration

- Fine-grained PAT scoped to `HGPG1/hgpg-context`, `contents:write` only
- Stored in Vercel env var (lookup chain: `GITHUB_PAT` → `GITHUB_TOKEN` → `BRAIN_GITHUB_PAT` for resilience)
- All commits authored as `brian@homegrownpropertygroup.com` (matches existing history, won't trip Vercel build hooks)

## Write API (Phase 1.6, shipped 2026-05-08)

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

GET returns a non-auth health-check showing `configured: true|false` based on env var presence — useful for smoke tests after deploy.

### Auth

Bearer token in `BRAIN_WRITE_TOKEN` Vercel env var. Constant-time compare (timing-attack safe). Token stored in Claude memory + 1Password ("HGPG Brain Write Token").

### Safety guardrails

- POST only, JSON body only
- Path validator rejects: leading `/`, `..` segments, backslashes, null bytes, control chars, paths >500 chars
- Path blocks: `.git/`, `.github/workflows/`, `.vercel/`, `node_modules/`, plus any `.env*` file or `package-lock.json` at any depth
- Content cap: 1 MB per write

### What this unlocks

- Claude sessions auto-commit SESSION-HANDOFF.md at session end
- Multi-file batch updates (project specs + CONTEXT.md cross-references in one session)
- Future automations (cron-style brain refreshes, scheduled audits) can write without human in the loop

## UI (MVP + Phase 1.5)

- Dashboard: Pinned cards (4 hardcoded in `lib/config.ts`), Quick actions, Recently edited (top 5)
- Edit page sticky header with back arrow, breadcrumb, save status (30s polling: "Saved Xm ago" / "Unsaved changes" / "Saving...")
- Mobile hamburger drawer (300px, slides in from left, dimmed overlay, auto-closes on file tap, 200ms ease-out)
- Sidebar sorted by last-modified date desc within each section (Root, projects/, archive/)
- Cooper Hewitt currently falling back to system sans (not on Google Fonts; self-host pending)

## Infrastructure dependencies

- **Resend SMTP** wired into HGPG Core Supabase for magic links (sender `noreply@homegrownpropertygroup.com`, name HGPG). API key in 1Password ("Supabase HGPG Core SMTP"). This SMTP upgrade affects ALL apps using HGPG Core: TM, CMA, TC Concierge, brain-app.

## Phase 2 backlog

- **Multi-token auth for `/api/external/write`** (added 2026-05-11). Today endpoint compares against single `BRAIN_WRITE_TOKEN`. Add comma-separated list support (e.g. `BRAIN_WRITE_TOKEN` master + `BRAIN_WRITE_TOKEN_CLAUDE` for chat sessions) so tokens can rotate independently. Minimal change: `validTokens.some(t => timingSafeEqual(...))`. Lets us share session-scoped tokens in chat without exposing master.
- iPhone smoke test deeper pass (CodeMirror + iOS soft keyboard scroll behavior in real-world editing)
- Cooper Hewitt self-hosted
- File rename and delete
- Diff view before save
- Cross-file search
- Multi-user allow-list expansion
- Audit log (Supabase table tracking who edited what, when)
- Draft autosave (Supabase table, recovers unsaved work on browser close)
- Drawer Esc-to-close + focus trap (a11y polish)
- Sticky sidebar on long dashboards
- Save-status precision tighter than 30s polling

## Bugs encountered + fixed (history)

- Magic link redirected to `tools.homegrownpropertygroup.com` (Supabase Site URL fallback) — fixed by adding `/auth/callback` route handler + pointing `emailRedirectTo` at it
- Supabase free SMTP rate limit (2/hr) hit during testing — switched to Resend custom SMTP
- First Write API deploy read wrong env var (`GITHUB_TOKEN` only). Brain-app's existing PAT is `GITHUB_PAT`. Fix in commit `127cc0c` added `GITHUB_PAT` as first lookup.

## Pickup notes

- The `package-lock.json` may differ between iMac and Mac mini — push from whichever machine you most recently ran `npm install` on
- Pinned files on dashboard configured in `lib/config.ts` — edit that array to change pins
- Stray bundle in `~/Downloads/fub-agent-tm-files/` is unrelated FUB lead-scoring agent work for `hgpg-transaction-manager`, NOT brain-app
- Cleanup tip: `rm ~/package-lock.json` to clear the harmless "multiple lockfiles" Next.js warning (leftover from earlier npm misuse in home dir)
