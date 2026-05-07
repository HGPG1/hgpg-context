<!-- Last Updated: 2026-05-07 -->

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
