# Brain App — Project Spec

> Web-based editor for the `hgpg-context` (HGPG brain) repo, accessible from any device. Solves the "did I push from the right Mac?" friction and future-proofs for AI-driven updates.

## Status

🟡 **Not started** — spec drafted May 5, 2026. Queue for a focused 60-90 min Claude Code session when ready.

## Why

Right now updating the brain requires:
- Be at a Mac with gh CLI configured
- Be in the `~/Documents/hgpg-context` folder
- Run the `brain` shell function (which copies from Downloads, commits, pushes)

That's better than where we started, but still:
- Mac-only (no iPhone/iPad updates on the go)
- File-based (have to drop into Downloads first)
- Manual commits

A web app at `brain.homegrownpropertygroup.com` makes the brain editable from anywhere, including future scenarios where Claude or another AI agent updates it via API.

## Subdomain

`brain.homegrownpropertygroup.com`

## Stack

Next.js App Router + Supabase Auth + Vercel. Standard HGPG pattern.

**Starter pattern:** Clone `hgpg-listing-report` admin page — same Next.js + Supabase + magic link auth shape, just swap the data source from Supabase tables to GitHub API.

## Features (MVP)

### Auth
- Supabase magic link auth gated to `brian@homegrownpropertygroup.com` only
- Use existing primary Supabase project (`ioypqogunwsoucgsnmla`)
- Google OAuth as secondary login option

### File listing
- Fetch repo contents via GitHub API
- Display tree: top-level files + `projects/` folder + `archive/` folder
- Show last modified timestamp on each file

### File editor
- Click a file → opens in editor
- Monaco editor (VS Code's editor) for syntax highlighting
- Live markdown preview pane (split view, toggle on/off)
- Save button writes via GitHub API
- Auto-commit message: "Update {filename} via brain app" (with optional override)

### File creation
- "New file" button at top of file list
- Pick destination folder (root, projects/, archive/)
- Filename + initial content
- Saves and creates commit

### Public read endpoint
- `GET /api/raw/[...path]` → mirrors `raw.githubusercontent.com/HGPG1/hgpg-context/main/{path}` with caching
- Future: replace `raw.githubusercontent.com` URLs in user prefs / project instructions with this endpoint for faster fetches

## Auth + write flow

- Server-side: stores a fine-grained GitHub PAT in Vercel env var `GITHUB_PAT`
- PAT scoped only to `HGPG1/hgpg-context` repo, contents:write only
- Server function uses Octokit to read/write files
- PAT NEVER exposed to client
- Brian's session checked on every write request — no auth, no write

## Brand

- Colors: `#2A384C` navy, `#A0B2C2` steel blue, `#D1D9DF` light steel, `#F0F0F0` off-white
- Fonts: Cooper Hewitt (body), Sansita Regular (display)
- No green, no Cormorant Garamond
- Match the look of `reports.homegrownpropertygroup.com` admin page

## Env vars needed

| Variable | Purpose |
|---|---|
| `GITHUB_PAT` | Fine-grained PAT for hgpg-context contents:write |
| `NEXT_PUBLIC_SUPABASE_URL` | Auth |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Auth |
| `SUPABASE_SERVICE_ROLE_KEY` | Server-side auth checks |
| `BRIAN_EMAIL` | Hardcoded allow-list for write access |

## Phase 2 (after MVP works)

- Mobile-friendly editor (Monaco doesn't work great on mobile — consider markdown-it preview only, or a simpler textarea)
- File rename + delete
- Diff view between commits (read-only history)
- Commit message override
- Optional: AI assist button that uses Anthropic API to clean up / restructure / suggest edits to the current file

## Phase 3 (eventually)

- Webhook endpoint so external AI agents (Cowork, custom integrations) can write to the brain via authenticated POST
- Audit log of all changes (who, when, what)
- Notion-style block editor (much later)

## Build prompt (paste into Claude Code session)

```
Build a Next.js App Router web app for editing GitHub repo files. Stack: Next.js 14+, Supabase Auth, Vercel deploy.

Repo: HGPG1/hgpg-context (private). Edit/read via GitHub API using a server-side PAT (env var GITHUB_PAT).

Auth: Supabase magic link, gated to a single email (env var BRIAN_EMAIL). Google OAuth optional.

Pages:
- /login: Magic link form
- /: File tree (top-level files + projects/ + archive/)
- /edit/[...path]: Monaco editor with markdown live preview, save button

API routes:
- GET /api/files — list repo contents (tree)
- GET /api/files/[...path] — read file content
- PUT /api/files/[...path] — write file content (auth required, writes to main branch with auto-commit)
- POST /api/files — create new file
- GET /api/raw/[...path] — public read mirror with caching

Use Octokit (@octokit/rest) for GitHub API.

Style: Tailwind, brand colors #2A384C navy / #A0B2C2 steel blue / #D1D9DF light steel / #F0F0F0 off-white. Fonts: Cooper Hewitt body, Sansita Regular display. Match the look of an admin dashboard with a sidebar file tree and main editor pane.

Single repo: HGPG1/brain-app. Subdomain: brain.homegrownpropertygroup.com on Vercel team_FietQPKCmnyioG2n0FdteQCV.

After scaffolding, output a SETUP.md with the env vars I need to add in Vercel and a checklist for first-time deploy.
```

## Pre-build checklist

Before running the Claude Code session:
- [ ] Generate fine-grained GitHub PAT scoped to `HGPG1/hgpg-context`, contents:write
- [ ] Decide auth Supabase project (primary `ioypqogunwsoucgsnmla` recommended)
- [ ] Reserve `brain.homegrownpropertygroup.com` in GoDaddy DNS (CNAME to `cname.vercel-dns.com`)
- [ ] Create empty repo `HGPG1/brain-app` on GitHub

## Open questions to resolve before build

1. **Editor library** — Monaco is heavy (~3MB). For an admin tool used by one person, fine. For mobile, consider CodeMirror or a textarea fallback.
2. **Real-time sync vs. save button** — start with explicit save (safer, fewer surprises). Auto-save can come later.
3. **Multi-user later?** — currently single-user (just Brian). If team members ever need to edit, add a `users` allow-list in env vars or move to Supabase users table.
4. **Mobile priority** — true mobile-first or desktop-first? Probably desktop-first since most editing happens on Mac.
