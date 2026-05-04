# HGPG Context

Private context repo for Brian McCarron / Home Grown Property Group. This is the source of truth for project state, brand standards, infrastructure, and operating procedures.

## How Claude should use this

When a new session starts and Brian references project work, fetch the relevant file(s) from this repo using the raw GitHub URL pattern:

```
https://raw.githubusercontent.com/HGPG1/hgpg-context/main/<file>
```

Examples:
- `https://raw.githubusercontent.com/HGPG1/hgpg-context/main/CONTEXT.md`
- `https://raw.githubusercontent.com/HGPG1/hgpg-context/main/projects/cma-engine.md`

Always start with `CONTEXT.md` for high-level orientation, then drill into the topical or project file relevant to the task.

## Structure

- **CONTEXT.md** - High-level brain dump and pointer index
- **SESSION-HANDOFF.md** - Scratchpad, overwritten each heavy build session
- **brand.md** - Colors, fonts, voice, banned phrases
- **team.md** - Roster, FUB IDs, contractors
- **infrastructure.md** - GitHub org, Vercel, Supabase, DNS, MCP servers
- **compliance.md** - SC HB 4754, referral fee statutes, Lando Law
- **marketing.md** - Meta Ads, lead re-engagement, South Charlotte Report
- **operations.md** - Session handoff protocol, formatting rules, iMessage relay
- **projects/** - One file per active project

## Update cadence

- Update project files when state changes (commits, blockers resolved, decisions made)
- `SESSION-HANDOFF.md` rewrites each heavy build session
- `CONTEXT.md` updates monthly or when something material changes

## Sync to iCloud (optional)

The iCloud workspace at `~/Library/Mobile Documents/com~apple~CloudDocs/HGPG-Cowork/` can mirror this repo as a backup. Repo is canonical.
