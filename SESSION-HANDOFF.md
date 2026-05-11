<!-- Last Updated: 2026-05-11 -->

# SESSION-HANDOFF

## Where we are

**Session 7 (2026-05-11) shipped.** FUB AI Agent multi-field paragraph stitching architecture is live and verified end-to-end. Real outbound email delivered to Jesse Hernandez via the full production pipeline. agent_enabled=false in safe state.

Status: 🟡 Operational with caveats. Two non-blocking issues parked for session 8:
1. Seller template `v1.warm.seller_report_engaged.seller` has a 252-char hardcoded paragraph (2 over the 250 cap). Blocks 100% of drafts that hit this template. 3 drafts in queue right now (John, Jay, Tamela) all blocked on this.
2. Buyer template `v1.warm.viewed_listing_recent.email` produces voice-clone drafts. 4 of 4 buyer drafts in session 7 were near-identical, only varying by first name and a 1-3 word time phrase.

## What shipped today

- Schema migration: paragraphs text[] column + CHECK constraint via fub_agent_paragraphs_valid() IMMUTABLE function (NOT VALID so historical rows exempt). body nullable.
- Commit 76ba8a2: reverted the Session 6 HTML body conversion (commit 6a1fa72) cleanly.
- 3 new FUB custom fields: id 160 Para2, 161 Para3, 162 Para4.
- Commit 504f116: 9-file code refactor for multi-field paragraph architecture. lib/agent/paragraphs.ts is new. draftGenerator splits body, validates, inserts with block_reason on failure. fubPusher writes 4 fields with empty-string padding. Edit route re-validates. Queue UI shows per-paragraph char counts + amber warning on blocked drafts.
- Commit c8f0d51: bugfix - draftGenerator was inserting the paragraphs array even when validation failed, hitting the DB CHECK and erroring instead of surfacing as a queue draft. Fix: paragraphs=null when blockReason set.
- FUB Template 1156 updated in UI: 4 merge tokens with actual newlines (Enter twice) between them. NOT `<br><br>` - those render as literal text because Automation 2.0 delivers as plaintext.
- Verification: draft 17 (Jesse Hernandez) generated → manual approve → FUB push success → email delivered. 532ms approve-to-push. Audit chain clean.

## Open queue at session end

9 drafts in fub_agent_message_drafts. agent_enabled=false so nothing is going out without manual approve.

| Draft | Lead | Status | Template | Notes |
|---|---|---|---|---|
| 14 | Seanna Mackey | pending_review | buyer:viewed_listing_recent | clean, ready |
| 16 | Anthony Scott | pending_review | buyer:viewed_listing_recent | clean, ready - same template as #14/18/20, near-identical content |
| 17 | Jesse Hernandez | approved + sent | seller:listing_thinking | the Phase 7 verification send, real outbound |
| 18 | Sam Russell | pending_review | buyer:viewed_listing_recent | clean, ready |
| 19 | John Miller | pending_review | seller:seller_report_engaged | BLOCKED paragraph_too_long, surfaced in queue |
| 20 | Gerard Marmo | pending_review | buyer:viewed_listing_recent | clean, ready. Note Brian is in active conversation with Gerard outside the agent - probably reject |
| 21 | Jay Miller | pending_review | seller:seller_report_engaged | BLOCKED paragraph_too_long |
| 22 | Tamela Karnazes | pending_review | seller:seller_report_engaged | BLOCKED paragraph_too_long |
| 23 | Jesse Hernandez | pending_review | seller:listing_thinking | dupe with #17 which already sent - probably reject |

## Pick up here

**Session 8 priorities in order:**

1. Fix seller template `v1.warm.seller_report_engaged.seller` paragraph 3. The hardcoded prose is 252 chars. Edit in fub_agent_message_templates (via Supabase MCP), split that sentence into two paragraphs OR trim ~10 chars of filler. Sweep other seller templates too, look for any > 250-char paragraphs at the template-body level.
2. Decide on buyer template diversity strategy. Options in projects/fub-ai-agent.md Session 8 backlog. Lean (b) - 3-4 phrasing variants of viewed_listing_recent with random pick.
3. Once templates pass validation, regenerate the 3 blocked drafts (John, Jay, Tamela) — they should land clean.
4. Decide on agent_enabled flip strategy. Probably keep manual-approve-only for week 1 at daily_send_cap=10.

## Critical context

- agent_enabled is the master kill switch. It blocks BOTH cron AND manual approve (the outboundGate checks it on every send). To do verification sends, flip to true, send, flip back. Briefly flipped on during session 7 for the Jesse verification, then back off.
- FUB Automation 2.0 delivers email as plaintext (or mail clients pick plaintext). NEVER put `<br>` in templates. Use real newlines.
- FUB UI Merge Fields dropdown produces `%snake_case%` tokens, not the camelCase API field names. Don't type tokens, use the dropdown.
- Test rig is FUB person 27764 (real Brian in Sphere stage, pond 9 = excluded). Agent Test - Brian (31924) was deleted.
- gerardmarmo@yahoo.com (FUB 23552) - Brian is in active manual conversation with Gerard. Reject any agent drafts targeting him.

## TM repo state

- Branch: main
- HEAD: c8f0d51 (fub agent: insert with paragraphs=null when blockReason set)
- Deploy: closings.homegrownpropertygroup.com READY on c8f0d51
- Working tree clean
- Two stale Claude Code branches still on origin (claude/sendblue-imessage-poc-6JOf1, claude/transaction-pdfs-private-AqXkA) - prior experiments, not blocking

## Brain repo state

- Branch: main
- HEAD before session 7 close: c48a6ab
- This session's brain commits include the session 7 wrap-up in projects/fub-ai-agent.md and this SESSION-HANDOFF.md update.
