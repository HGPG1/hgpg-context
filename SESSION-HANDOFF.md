# SESSION HANDOFF — Next TM Session

_Last updated: 2026-04-24 after Chunk 5 ship day_

## Next session objective

Knock out three items in one Claude Code session on the home iMac:

1. **Nag cron** — detect transactions `under_contract` for 24h+ with missing `loop_group_id` or `loop_client_group_id`, ping Brian+Don via iMessage
2. **Notification copy tuning** — split every milestone template into `clientCopy` vs `proCopy`; client group hears celebrations + status, pro group hears logistics + handoffs
3. **Email-to-transaction promote UI** — button on staged email list that pre-fills `/transactions/new` from the selected `transaction_emails` row

Estimated total: 2-3 hours. Work in the order above (smallest → largest surface area).

## Prereqs — run these BEFORE starting Claude Code

```bash
# 1) Sync local main
cd ~/Developer/hgpg-transaction-manager
git checkout main
git pull

# 2) Refresh env vars in case anything drifted
vercel env pull .env.local

# 3) Verify build is clean before touching anything
npm run build
```

If `npm run build` fails, stop and diagnose before letting Code loose — don't start a build-loop session on a broken base.

## Test data (already live, use these)

- **Test Harbor** — `cf299082-52a3-42ce-8b8f-41bad5993be9` — under_contract, both groups attached
- **Magnolia** — `15d83bd2-3c22-44d2-a427-18368713beb3` — fell_through (still useful for edge case tests)
- **Apps Script staging** — `transaction_emails` table has real rows from the MIME walker rescues; good fodder for promote UI testing

## Claude Code mode recommendation

- **#1 nag cron** + **#2 copy tuning**: `claude --dangerously-skip-permissions` is fine — bounded scope, pure file edits
- **#3 promote UI**: stay in approval mode — touches intake form + new API route + staged email view, more places to go sideways

## Work order detail

### #1 — Nag cron (~20 min)

- New file: `app/api/cron/group-setup-nag/route.ts` (or wherever crons live in this repo)
- Query: transactions where `status = 'under_contract'` AND `created_at < now() - interval '24 hours'` AND (`loop_group_id IS NULL` OR `loop_client_group_id IS NULL`)
- For each match, send a single iMessage to Brian + Don saying "[Address] — client/pro group still not set up, milestones can't route yet"
- One send per transaction per day — add a `group_setup_nag_sent_at` column or check `audit_log` to avoid spam
- Register in `vercel.json` crons, 9am ET daily (13:00 UTC in EDT, 14:00 UTC in EST — ET = EDT right now through November)
- Verify with a manual POST to the route, then wait a cycle

### #2 — Notification copy tuning (~45 min)

- Target file: wherever milestone template bodies live (likely `lib/notifications/templates.ts` or similar — Code should grep for the current earnest_money copy to find it)
- Every milestone gets two bodies:
  - `clientCopy`: celebration tone, plain language, tracker link, no internal logistics
  - `proCopy`: logistics, next action, deadline, owner
- Reference examples that shipped today:
  - Client earnest_money: "Your earnest money is safely in escrow — a real commitment locked in on [address]. Track: [tracker_url]"
  - That was the RIGHT client tone. Use it as the model. Most existing copy is too transactional.
- Test: mark a milestone complete on Test Harbor, verify both groups receive the correct flavor
- Pay attention to the leading-whitespace quirk in LoopMessage group names — webhook handles it but double check

### #3 — Email-to-transaction promote UI (~60-90 min)

- New route: `app/api/transaction-emails/[id]/promote/route.ts` — reads row, returns prefill payload
- Staged email list view: add "Promote to transaction" button
- Intake form (`/transactions/new`): accept prefill via query string or form state, auto-fill address/parties/dates
- Don't auto-create the transaction — open the form with fields filled, let user verify before submit
- Mark the `transaction_emails` row as promoted (add column or use status field) when the new transaction is saved, so same email can't be re-promoted

## Not in scope this session

- 24h cron beyond group setup (no other nag categories yet)
- Retiring `client_notified_at` column (waits until ~2026-05-08, 2-week window from Chunk 5 ship)
- middleware.ts → proxy.ts rename (mechanical, do in a gap)
- Any Google Calendar / MLS Grid / IDX Broker work — external blockers

## Session close checklist

When all three ship clean:

- [ ] Three commits on main, each with Co-Authored-By Claude Code
- [ ] Vercel deploy READY on latest commit
- [ ] Test Harbor verified for #2 (trigger a milestone, confirm dual-group copy)
- [ ] Cron dry-run verified via manual POST
- [ ] Promote flow walked through on a real staged email
- [ ] Memory updated via web Claude (tell me "done with tier 1" and I'll update line 10/11)
- [ ] This SESSION-HANDOFF.md rewritten for the next session

## Reference

- Repo: `HGPG1/hgpg-transaction-manager` branch `main`
- Production: `closings.homegrownpropertygroup.com`
- Supabase: `ioypqogunwsoucgsnmla`
- Vercel project: `prj_oLWVcE4J1UKzJtmggoQCOW35LUhy`
- Today's ship commits: `cd5aece`, `2d40e6c`, `2351b63`, `ccdaa63`, `123fd9b`
- Twilio is DEAD — iMessage only

---

_Previous session summary (2026-04-24 ship day): Chunk 5 dual iMessage groups shipped + E2E verified on Test Harbor. GroupSetupHelper UI shipped on 123fd9b after two sandbox build failures, resurrected via Claude Code on home iMac. Apps Script MIME walker deployed to both closings@ and brian@. All tier-1 ship goals verified live via Chrome MCP test with client-group-null-then-restore flow._
