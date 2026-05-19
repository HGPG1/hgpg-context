<!-- Last Updated: 2026-05-19 -->

# Operations

## Brain hygiene

### Last-Updated header convention

Every brain file starts with an HTML comment showing the last meaningful update date:

```
<!-- Last Updated: YYYY-MM-DD -->
```

Bump this whenever the file's content materially changes. The Brain App at `brain.homegrownpropertygroup.com` should surface this date so stale files are easy to spot.

### Auto-update triggers (Claude session pattern)

When Brian says any of these phrases mid-session, Claude proactively prepares a brain update without being asked:

- "start a new thread" / "new thread" / "fresh thread"
- "wrapping for the day" / "wrapping up" / "calling it"
- "bueno" (Brian's tell that something just worked)

What Claude does on trigger:

1. Generate updated `SESSION-HANDOFF.md` reflecting current session's work
2. Generate updated project file(s) for anything materially changed
3. Drop them in `/mnt/user-data/outputs/` as downloadable files
4. Provide a single copy-paste block of `mv` + `git add` + `git commit` + `git push` commands for Brian to run from his Mac

The actual push happens on Brian's Mac (gh CLI authenticated via Keychain). Claude prepares the prep work; Brian runs one block.

**Alternative path (Phase 2 brain-app build):** add a `/api/session-update` endpoint to brain-app that accepts a session-summary payload, allowing Claude to POST directly. Pushes become fully automatic.

## Session handoff protocol

- When a conversation exceeds **~30 exchanges** OR involves heavy code/build work, proactively tell Brian it's time to write a `SESSION-HANDOFF.md` and start a new chat. Don't wait to be asked.
- Handoff captures: what was built, files changed, current state, blockers, exact next steps.
- New sessions read both `CONTEXT.md` (brain index) and `SESSION-HANDOFF.md` (scratchpad) to resume seamlessly.
- Handoff lives in iCloud at `~/Library/Mobile Documents/com~apple~CloudDocs/HGPG-Cowork/SESSION-HANDOFF.md` and in this repo (committed).

## Browser automation

- **Always use Claude in Chrome** (Chrome DevTools MCP) for any browser task: navigating, filling forms, clicking, reading, screenshots.
- Never default to `web_fetch` or manual instructions when Chrome MCP tools are available.

## Claude interface preference

- **Heavy build sessions:** claude.ai in Chrome (with the Claude Auto-Continue extension).
- **Claude Desktop:** only when Chrome MCP or FUB MCP tools are specifically required (those only run in Desktop).

## Page build workflow

- Claude builds single HTML files (all CSS/JS embedded, CDN links for fonts/libraries).
- Brian downloads and pushes to GitHub.

## Formatting rules

- **No em dashes** in any output. Use hyphens, commas, parentheses, or period breaks.

## iMessage relay to Brian

When Brian asks Claude to text/iMessage him, use bash + curl to the internal endpoint. The full URL and secret are stored in Claude memory (not in this public repo).

## Pending tasks (do NOT proactively remind)

- Migrate `.net` Google Workspace Drive (~9.23 GB, 466 docs) to `.com` via Takeout.
- Audit `.net` Gmail for accounts using that address as login/recovery (MLS, SiteGround, GitHub, Vercel, Canva). Update to `.com`.
- Cancel `.net` Workspace after audit.
- Squarespace `.net` registration stays (defensive, $15/yr auto-renew Oct 26).
- No email forwarding (too much junk).
- Search Console `.net` property to be removed.


## Brain repo commits (default workflow)

When Brian asks to update, add to, or change anything in the brain repo (`hgpg-context`), commit it directly via the Brain App's external endpoint. Do not draft files for him to paste, do not send him to the Brain App UI, do not ask permission - just commit.

### Endpoints (all at `https://brain.homegrownpropertygroup.com`, all use the same Bearer token)

- **Write to brain** — `POST /api/external/write` `{path, content, message}` — commits to `HGPG1/hgpg-context` only.
- **Write to any HGPG1 repo** — `POST /api/external/commit` `{repo, path, content, message}` — commits to any HGPG1 repo the GitHub App has access to (currently all of them).
- **Read any HGPG1 file or dir** — `GET /api/external/read?repo=<repo>&path=<path>&ref=<ref>` (or POST with JSON body). Returns file content (utf8-decoded, ≤1 MB) or directory listing. `ref` defaults to `main`.
- **Read brain files** — `GET /api/files/<path>` returns `{content: "..."}`. Brain-only shortcut for the read endpoint.
- **Git log for any HGPG1 repo** — `GET /api/external/log?repo=<repo>&since=<iso>&until=<iso>&path=<optional path>&per_page=<n>&page=<n>` (or POST with JSON body). Returns commits with sha, short_sha, message, full_message, author, date. `per_page` caps at 100.
- **All five share** the same `Authorization: Bearer <BRAIN_WRITE_TOKEN>` and the same GitHub App auth backing.
- **Cannot delete files** via any endpoint — write/commit creates or overwrites only. Real deletes need `gh` from the Mac.

### Workflow for any brain or repo edit

1. Fetch the current file via `/api/external/read` (or `/api/files/<path>` for brain) so the splice is against the live file, not a stale copy.
2. Modify the content.
3. POST to `/api/external/write` (brain) or `/api/external/commit` (any HGPG1 repo) with the full new content.
4. Confirm the returned `commitSha` to Brian.

### Reading commit history

When Brian wants commit SHAs, a commit body, or to know "what changed between X and Y," call `/api/external/log` instead of asking him to run `gh` locally. Filter by `path` to find history for a single file. The endpoint is cheap; use it liberally.

Do NOT default to: drafting files for paste, telling Brian to use the Brain App UI, asking for permission, or suggesting `gh` commands. The direct API is the default and only path unless the operation requires a delete.


## BRAIN_WRITE_TOKEN

`5153df26d2d7829e26d8fffce05067459541b7d374952bb6142092ab0788793e`

This token covers `/api/external/{write,commit,read,log}` plus `/api/files/<path>` reads. Write scope: HGPG1/hgpg-context for `/api/external/write`, any HGPG1 repo for `/api/external/commit`. Read scope: any HGPG1 repo via `/api/external/read` and `/api/external/log`. Low-risk to keep in chat history. Don't bother rotating after every session, don't bother asking Brian for it again. Use it directly from this file.

