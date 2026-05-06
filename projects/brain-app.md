<!-- Last Updated: 2026-05-06 -->

# Brain App

**Status:** 🟢 SHIPPED (MVP, 2026-05-06)
**Live at:** https://brain.homegrownpropertygroup.com
**Repo:** HGPG1/brain-app (private)
**Vercel project:** brain-app on team `team_FietQPKCmnyioG2n0FdteQCV`

## Purpose

Web-based editor for the `HGPG1/hgpg-context` brain repo. Lets Brian edit context files (CONTEXT.md, SESSION-HANDOFF.md, project specs) from any device with auth, instead of being tied to a Mac with `gh` CLI configured. Commits land directly on `main` with proper author attribution.

## Stack

- Next.js 16.2.4 (App Router, TypeScript)
- Tailwind v4 (brand tokens in `app/globals.css` `@theme`)
- CodeMirror 6 (markdown lang, custom HGPG light theme)
- Supabase Auth — magic link only, shared with HGPG Core project (`ioypqogunwsoucgsnmla`)
- @octokit/rest for GitHub API
- marked + DOMPurify for live markdown preview
- Resend custom SMTP for magic link delivery (`noreply@homegrownpropertygroup.com`)

## Auth model

- Single-user MVP: `BRIAN_EMAIL` env var allow-lists `brian@homegrownpropertygroup.com`
- Magic link login via Supabase, callback handler at `/auth/callback`
- All write API routes check session email against `BRIAN_EMAIL` — 401 if mismatch
- GitHub PAT is server-side only, never exposed to client
- Read flow: dashboard fetches file tree via `/api/files`, requires server-side auth check

## Infrastructure

**GitHub PAT:** fine-grained, scoped to `HGPG1/hgpg-context` only, Contents read+write + Metadata read-only, 1-year expiry. Stored in Vercel as `GITHUB_PAT`.

**Env vars (8 total):**
- `GITHUB_PAT`
- `GITHUB_OWNER=HGPG1`
- `GITHUB_REPO=hgpg-context`
- `GITHUB_BRANCH=main`
- `NEXT_PUBLIC_SUPABASE_URL=https://ioypqogunwsoucgsnmla.supabase.co`
- `NEXT_PUBLIC_SUPABASE_ANON_KEY`
- `SUPABASE_SERVICE_ROLE_KEY`
- `BRIAN_EMAIL=brian@homegrownpropertygroup.com`

**Supabase redirect URLs added to HGPG Core:**
- `https://brain.homegrownpropertygroup.com/**`
- `http://localhost:3000/**`

**DNS:** GoDaddy CNAME `brain` → `cname.vercel-dns.com`

## API surface

- `GET /api/files` — repo tree (top-level, projects/, archive/)
- `GET /api/files/[...path]` — single file content + sha
- `PUT /api/files/[...path]` — update file (auth required)
- `POST /api/files` — create new file (auth required)
- `GET /api/raw/[...path]` — public mirror of raw.githubusercontent.com with edge cache (s-maxage=60, swr=300)

## Pages

- `/login` — magic-link form
- `/` — dashboard with file tree sidebar + recent files
- `/edit/[...path]` — split editor/preview, Cmd+S save, dirty-state guard, commit message input
- `/new` — folder picker + filename + initial content
- `/auth/callback` — code-for-session exchange handler

## Phase 2 backlog

- iPhone smoke test + iOS Safari keyboard handling (CodeMirror scroll-jump on soft keyboard appearance)
- Cooper Hewitt self-hosted (currently falling back to system sans because it's not on Google Fonts)
- File rename and delete
- Diff view before save
- Cross-file search
- Multi-user allow-list (current `BRIAN_EMAIL` check is single-string but auth helper is structured for easy expansion to array)
- Audit log (Supabase table tracking who edited what, when)
- Draft autosave (Supabase table, recovers unsaved work on browser close)

## Notes for future sessions

- This app dogfoods itself: future edits to `hgpg-context` should happen via brain-app, not local `gh` workflow, to validate the tool stays working
- Tailwind v4 means brand colors live in `app/globals.css` `@theme` blocks — `tailwind.config.ts` is a mirror for editor IntelliSense only, doesn't affect builds
- Build deploys auto on every push to `main`, ~90 sec
- Local dev: `npm run dev` on port 3000, requires `.env.local` with all 8 env vars
- Repo lives at `~/brain-app` on Mac mini, `~/Developer/brain-app` on iMac (or fresh clone needed)

## Original spec history

Original brain-app spec called out: single-user lock, mobile editing as primary use case, fail-closed auth, no client-side PAT exposure, edge-cached raw mirror for downstream consumers (project instructions, AI agents). All MVP requirements met. Phase 2 features deliberately scoped out of MVP to ship quickly.