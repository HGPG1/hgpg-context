<!-- Last Updated: 2026-05-14 -->

# Brain App

- **Status:** 🟢 Live, MVP + Phase 1.5 + Write API + GitHub App migration shipped
- **URL:** https://brain.homegrownpropertygroup.com
- **Repo:** HGPG1/brain-app (private)
- **Vercel project:** `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- **Supabase:** `ioypqogunwsoucgsnmla` (HGPG Core — auth only, no app tables)
- **Stack:** Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- **Local dev (Mac mini):** `cd ~/brain-app && npm run dev`
- **Local dev (iMac):** `cd ~/Developer/brain-app && npm run dev` (already cloned and configured)

## Purpose

Web-based editor for the brain repo (`HGPG1/hgpg-context`). Any device, magic-link auth, single-user. Replaces the need to clone the brain repo and hand-edit markdown to keep session continuity working. Also exposes programmatic write APIs so Claude sessions can commit directly without copy-paste through the UI — to the brain via `/api/external/write` or to any HGPG1 repo via `/api/external/commit`.

## Auth

- Supabase Auth magic link, sent via Resend custom SMTP (30/hr rate limit)
- Single-user allow-list via `BRIAN_EMAIL=brian@homegrownpropertygroup.com` env var
- `lib/auth.ts` is structured for easy multi-user expansion if needed
- Redirect URLs configured in Supabase HGPG Core: `https://brain.homegrownpropertygroup.com/**` and `http://localhost:3000/**`

## GitHub integration (GitHub App, since 2026-05-14)

- **App name:** HGPG Brain Commit
- **App ID:** 3712986
- **Installation ID:** 132322328 (HGPG1 org, installed on all repositories)
- **Permissions:** Contents read/write, Metadata read-only
- **Private key:** stored as Vercel env var `GITHUB_APP_PRIVATE_KEY` on brain-app project (multi-line PEM). Local backup at `~/.hgpg-secrets/github-app-private-key.pem` on iMac (chmod 600). Recommend also saving to 1Password.
- **Auth helper:** `lib/octokit-app.ts` — exports `getAppOctokit()` and `getInstallationToken()`. Installation tokens auto-rotate every hour, no manual rotation needed.
- All commits authored as `brian@homegrownpropertygroup.com` (matches existing history, won't trip Vercel build hooks)

### Env vars in use (Production)

- `BRAIN_WRITE_TOKEN` — bearer token for `/api/external/write` and `/api/external/commit`
- `GITHUB_APP_ID` — App ID (sensitive)
- `GITHUB_APP_INSTALLATION_ID` — Installation ID (sensitive)
- `GITHUB_APP_PRIVATE_KEY` — full PEM contents (sensitive, multi-line)
- `ALLOWED_REPOS` — optional; not currently set, meaning all HGPG1 repos the App can access are commit targets

### Env vars NO LONGER USED (dead weight, safe to remove from Vercel)

- `GITHUB_PAT`
- `GITHUB_TOKEN`
- `BRAIN_GITHUB_PAT`

## Write API: /api/external/write (Phase 1.6, shipped 2026-05-08, App-backed since 2026-05-14)

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

GET returns a non-auth health check showing `authMode: "github-app"` and `configured: true|false` based on env var presence. Useful for smoke tests after deploy.

### Scope

This endpoint writes ONLY to `HGPG1/hgpg-context` (the brain). For writing to other repos, use `/api/external/commit`.

## Commit API: /api/external/commit (NEW, shipped 2026-05-14)

### Endpoint

`POST https://brain.homegrownpropertygroup.com/api/external/commit`

Request:

    Authorization: Bearer <BRAIN_WRITE_TOKEN>
    Content-Type: application/json
    {
      "repo": "hgpg-transaction-manager",
      "path": "app/api/foo/route.ts",
      "content": "<full file contents>",
      "message": "Add foo endpoint",
      "branch": "main"
    }

Response: `{ ok, repo, path, branch, commitSha, created, bytes }`

GET returns a non-auth health check showing `authMode: "github-app"`, `allowedRepos` (current value: `"all"`), and `configured: true|false`.

### What this unlocks

- Claude sessions can push code to ANY HGPG1 repo from any device, including mobile
- No "I'll handle that from my Mac" handoff needed for routine code changes
- Brian still pushes from his Mac when editing locally — standard git, unchanged

### Repo restriction (optional)

Set `ALLOWED_REPOS` env var to a comma-separated list to restrict commit targets at runtime. Currently unset → all HGPG1 repos the App can access are valid targets.

## Auth model (both endpoints)

- Bearer token in `BRAIN_WRITE_TOKEN` Vercel env var. Constant-time compare (timing-attack safe). Token stored in Claude memory + 1Password ("HGPG Brain Write Token").
- Same token for both `/api/external/write` and `/api/external/commit` — rotation is one operation.

### Safety guardrails (both endpoints)

- POST only, JSON body only
- Path validator rejects: leading `/`, `..` segments, backslashes, null bytes, control chars, paths >500 chars
- Path blocks: `.git/`, `.github/workflows/`, `.vercel/`, `node_modules/`, plus any `.env*` file or `package-lock.json` at any depth
- Content cap: 1 MB per write
- `/api/external/commit` also validates the `repo` parameter against an allowlist character set and `ALLOWED_REPOS` if set

### Rotation playbooks

- **Bearer token leaks**: `vercel env rm BRAIN_WRITE_TOKEN production && vercel env add BRAIN_WRITE_TOKEN production` then redeploy. Update Claude memory with new value. ~60 seconds.
- **GitHub App private key leaks**: go to https://github.com/organizations/HGPG1/settings/installations/132322328, generate new private key, delete old one, update `GITHUB_APP_PRIVATE_KEY` env var on Vercel. App ID and Installation ID don't change.

## UI (MVP + Phase 1.5)

- Dashboard: Pinned cards (4 hardcoded in `lib/config.ts`), Quick actions, Recently edited (top 5)
- Edit page sticky header with back arrow, breadcrumb, save status (30s polling: "Saved Xm ago" / "Unsaved changes" / "Saving...")
- Mobile hamburger drawer (300px, slides in from left, dimmed overlay, auto-closes on file tap, 200ms ease-out)
- Sidebar sorted by last-modified date desc within each section (Root, projects/, archive/)
- Cooper Hewitt currently falling back to system sans (not on Google Fonts; self-host pending)

## Infrastructure dependencies

- **Resend SMTP** wired into HGPG Core Supabase for magic links (sender `noreply@homegrownpropertygroup.com`, name HGPG). API key in 1Password ("Supabase HGPG Core SMTP"). This SMTP upgrade affects ALL apps using HGPG Core: TM, CMA, TC Concierge, brain-app.

## Phase 2 backlog

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

## Backlog items now obsolete (closed by 2026-05-14 migration)

- ~~Multi-token auth for `/api/external/write`~~ — no longer needed. Single `BRAIN_WRITE_TOKEN` covers both endpoints, and GitHub App auth means the bearer token is the only rotation surface.

## Bugs encountered + fixed (history)

- Magic link redirected to `tools.homegrownpropertygroup.com` (Supabase Site URL fallback) — fixed by adding `/auth/callback` route handler + pointing `emailRedirectTo` at it
- Supabase free SMTP rate limit (2/hr) hit during testing — switched to Resend custom SMTP
- First Write API deploy read wrong env var (`GITHUB_TOKEN` only). Brain-app's existing PAT was `GITHUB_PAT`. Fix in commit `127cc0c` added `GITHUB_PAT` as first lookup. (Obsoleted 2026-05-14 by GitHub App migration — all PAT env var lookups removed.)
- PAT leak 2026-05-13 — a fine-grained PAT was exposed in a chat. All PATs revoked. Brain-app went down for writes (502 "GitHub read failed"). Fixed 2026-05-14 by migrating from PAT to GitHub App (commit `bc8a0bf`). New `/api/external/commit` endpoint added in the same migration.

## Pickup notes

- The `package-lock.json` may differ between iMac and Mac mini — push from whichever machine you most recently ran `npm install` on
- Pinned files on dashboard configured in `lib/config.ts` — edit that array to change pins
- Stray bundle in `~/Downloads/fub-agent-tm-files/` is unrelated FUB lead-scoring agent work for `hgpg-transaction-manager`, NOT brain-app
- Cleanup tip: `rm ~/package-lock.json` to clear the harmless "multiple lockfiles" Next.js warning (leftover from earlier npm misuse in home dir)
- Diagnostic files at `_diagnostics/` in both `hgpg-context` and `brain-app` repos can be deleted whenever — they were one-shot verification artifacts from the 2026-05-14 migration
