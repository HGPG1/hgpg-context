# Transaction Manager

- **URL:** closings.homegrownpropertygroup.com
- **Repo:** HGPG1/hgpg-transaction-manager
- **Supabase:** ioypqogunwsoucgsnmla (shared with TC Concierge)
- **PIN:** 2026
- **Stack:** Next.js + Supabase + Tailwind
- **Local path:** `~/Projects/hgpg-transaction-manager`
- **Claude Code launch:** `cd ~/Projects && claude --dangerously-skip-permissions`

## Architecture

- Architecture doc has full ListedKit data model + ReZEN SC buyer & NC seller checklists.
- **TC Concierge = intake; Transaction Manager = lifecycle.**

## Open items (from April sessions)

- Verify closing date fix
- **Resolve Sherlock 403** (likely API key scope issue)
- **NC office routing:** Swap to office ID `924dac0e-91f2-471d-80c4-f06d80fb6d94` when `txn.state === 'NC'`
- Flip `transaction-pdfs` bucket to **signed URLs**
- Lamington duplicate cleanup

## ReZEN APIs

- **Arrakis:** `arrakis.therealbrokerage.com/api/v1` - uses `X-API-KEY` header
- **Sherlock:** Returning 403 (open issue)
