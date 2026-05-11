<!-- Last Updated: 2026-05-11 -->

# Session Handoff

## Last session: 2026-05-11 — FUB AI Agent smoke test attempted, CP4 failed on architecture 🔴

### What happened

Ran the 4-checkpoint smoke test gate. CP1, CP2, CP3 passed. CP4 (first real outbound, Jay Miller draft id 9) revealed two foundational FUB constraints that break the current architecture:

1. **FUB custom fields silently truncate at ~255 chars on API write.** Jay's body was 423 chars; FUB stored 255 of them, cut off mid-word ("doesn't know wha"). No 400 error, no warning — POST returned 200, our pusher logged `fub_push_success`, the email went out with a truncated body.

2. **FUB API HTML-escapes text custom field values on write.** Our `<br><br>` got stored as `&lt;br&gt;&lt;br&gt;` in field 157, then rendered as literal `<br><br>` text in the delivered email. This is the OPPOSITE behavior from when HTML is typed into a custom field via the FUB UI's HTML editor (which we proved in CP2). API writes get escaped; UI writes don't.

Jay received a confusing email at 10:56 AM ET. Brian sent a personal recovery email from his own account immediately after. Damage mitigated.

`agent_enabled` flipped back to false at 15:02 UTC. No further outbound possible until Session 7 ships the redesign.

### Today's checkpoint results

- ✅ **CP1 PASS** — Viktor's Automation 2.0 trigger fires on `agent_draft_email_ready` tag
- ✅ **CP2 PASS** (with caveats — see learnings) — Merge fields render when inserted via FUB UI Merge Fields dropdown, NOT via the literal `[-camelCase-]` syntax that was assumed to work based on round-trip storage verification
- ✅ **CP3 PASS** — Smoke test SQL Section A → B → C ran cleanly via Supabase MCP. Brian's record (fub_person_id 27764) fully restored byte-identical post-test
- ❌ **CP4 FAIL** — Architecture problem, not bug. Jay's draft pushed successfully (sub-2-second approve→push→FUB chain), but delivered email was unreadable

### What shipped today

- **`lib/agent/emailFormat.ts`** — `formatEmailBodyForFub()` HTML escape + `\n\n` → `<br><br>` converter (commit `6a1fa72`). **MUST BE REVERTED in Session 7** — the `<br>` tags get re-escaped by FUB on write, making the conversion worse than useless.
- **`scripts/backfill-html-email-bodies.ts`** — one-shot backfill that converted 4 pending drafts (ids 7, 8, 9, 10) from plaintext `\n\n` to `<br><br>` HTML breaks. **Also must be reverted** when Session 7 rolls back the email format helper. The 3 still-pending drafts (Gerard #7, Seanna #8, Anthony #10) currently have `<br>` in their bodies — those bodies need re-conversion back to plaintext, OR the drafts need to be discarded and regenerated.
- **Template 1156 fixed in FUB UI** — merge field placeholders re-inserted via the FUB Merge Fields dropdown. Subject and body now correctly reference `%custom_agent_draft_email_subject%` and `%custom_agent_draft_email_body%`. The original `[-camelCase-]` syntax from session 6 was discovered to be cosmetic — stored fine, never rendered.
- **PR #7 verified in prod** — bucket privatization migration confirmed applied: `storage.buckets.transaction-pdfs.public = false`. Signed-URL flow for transaction-pdfs is live.

### Pickup notes for Session 7

The agent build is paused at "approved-but-architecturally-broken." Don't touch it without a redesign first. Session 7 needs to:

1. **Revert `lib/agent/emailFormat.ts` HTML conversion.** The `<br>` approach was wrong. FUB escapes on write regardless of channel.
2. **Re-backfill drafts 7/8/10** from `<br><br>` back to plaintext `\n\n`, OR discard them and regenerate. Pick before coding.
3. **Spec the multi-field paragraph stitching redesign** (Option A from today's analysis):
   - Create 3-4 additional FUB custom fields: `customAgentDraftEmailPara2`, `Para3`, `Para4` (text type)
   - Update Template 1156 in FUB UI to stitch paragraphs with literal `<br><br>` BETWEEN merge tokens (literal `<br>` in template body is HTML, not escaped; merge values stay plaintext per-paragraph)
   - Refactor `lib/agent/draftGenerator.ts` to emit body as an array of paragraphs, each under 256 chars (target 200 chars per paragraph to leave headroom)
   - Refactor `lib/agent/fubPusher.ts` to write each paragraph to its respective custom field
   - Add `fub_agent_message_drafts.body_paragraphs jsonb` column (or rename `body` and migrate)
   - Add defensive length check in pusher: refuse to push if any paragraph > 250 chars, log `block_reason='paragraph_too_long'`, surface in queue UI
4. **Verify the redesign on Agent Test - Brian (FUB ID 31924) FIRST** before any real lead. New CP1-CP4 sequence with multi-field draft.
5. **Then resume the 4-checkpoint smoke test** against Gerard or Seanna once architecture proven.

### Jay status

- Draft id 9 status = 'approved', fub_pushed_at = 14:56:17 UTC
- FUB record shows `agent_email_sent` tag applied (Automation 2.0 step 2 fired correctly)
- `agent_draft_email_ready` tag was likely removed by the Automation (tag-removal step)
- Brian sent recovery email from personal account at ~15:05 UTC (manual, outside agent path)
- Leave draft 9 in 'approved' status as audit trail. Do NOT mutate it.

### Other state

- `agent_enabled = false` (confirmed via MCP at session end)
- `pushed_today_total = 1` (Jay's truncated email)
- 3 pending drafts still in queue, all with HTML breaks (need re-backfill or discard)
- 4 unscored eligible leads pool = 4,340 (unchanged)
- TM main = commit `6a1fa72` (HTML body fix that needs reverting)
- Brain main = needs updating with this handoff

### Brain updates pending in this push

- `projects/fub-ai-agent.md` — full Session 7 entry with today's smoke test failure analysis
- `SESSION-HANDOFF.md` — this file
- `CONTEXT.md` — date bump to 2026-05-11, FUB AI Agent line updated

### Don't forget

- Update FUB custom fields 157/158/159 visibility to admin-only in FUB UI (carryover from session 4 — still not done)
- The `--experimental-strip-types` Node 25 backfill script pattern that Claude Code used is a reusable pattern for future one-shot data migrations
- TM repo has no `npm run typecheck` script — just `lint`. Future Claude Code prompts should say "run tsc directly"
