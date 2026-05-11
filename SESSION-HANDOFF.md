<!-- Last Updated: 2026-05-11 -->

# SESSION-HANDOFF

## Where we are

**Sessions 7 + 8 (2026-05-11) shipped.** Multi-field paragraph stitching architecture live and verified. All 3 seller templates with paragraph-too-long issues fixed via DB UPDATE — no code change needed. End-to-end pipeline working. agent_enabled=false in safe state, ready to flip true any time.

Status: 🟢 Operational. One non-blocking quality issue parked for session 9:
- Buyer template `v1.warm.viewed_listing_recent.email` produces voice-clone drafts. 5 of 5 buyer drafts in session 7 were near-identical, only varying by first name and a 1-3 word time phrase.

## Session 7 + 8 summary

- Architecture: paragraphs text[] column + CHECK constraint, FUB Para2/3/4 fields (160/161/162), Template 1156 with newline separators, draftGenerator/fubPusher/edit-route/queue-UI all refactored
- Real outbound verified: Jesse Hernandez 11323 received the test email via full production pipeline
- Bug fix: draftGenerator was inserting paragraphs array even on validation failure, hitting DB CHECK. Patched to insert paragraphs=null when blockReason set
- Template fixes: templates 9, 2, 11 all had paragraph 3 over the 250 cap. Trimmed filler words, all 3 now pass

## TM repo state

- Branch: main
- HEAD: c8f0d51 (fub agent: insert with paragraphs=null when blockReason set)
- Deploy: closings.homegrownpropertygroup.com READY
- Working tree clean
- Stale Claude Code branches still on origin (cosmetic)

## Brain repo state

- Branch: main
- HEAD: bd600f3 (session 8 - template paragraph length sweep)

## Open queue (8 drafts in flight)

agent_enabled=false so nothing is going out without manual approve.

| Draft | Lead | Status | Template | Notes |
|---|---|---|---|---|
| 14 | Seanna Mackey | pending_review | buyer:viewed_listing_recent | clean, ready |
| 16 | Anthony Scott | pending_review | buyer:viewed_listing_recent | clean, ready — same template as 14/18/20 |
| 17 | Jesse Hernandez | approved+sent | seller:listing_thinking | Phase 7 verification send, real outbound |
| 18 | Sam Russell | pending_review | buyer:viewed_listing_recent | clean, ready |
| 20 | Gerard Marmo | pending_review | buyer:viewed_listing_recent | clean. Brian in active manual convo - probably reject |
| 23 | Jesse Hernandez | pending_review | seller:listing_thinking | dupe with 17 - reject |
| 24 | John Miller | pending_review | seller:seller_report_engaged | NEW post-template-fix, clean |
| 25 | Jay Miller | pending_review | seller:seller_report_engaged | NEW post-template-fix, clean |
| 26 | Rachel Delmore | pending_review | seller:listing_thinking | NEW, clean |

Session 7 had drafts 19/21/22 stuck on paragraph_too_long. Those got deleted in session 8 and regenerated as 24/25/26 against the fixed templates.

## Pick up here (session 9 priorities)

1. **Buyer template diversity.** Write 3-4 phrasing variants of `viewed_listing_recent` so selectTemplate can rotate them. Either: (a) 3-4 new scenario rows with random pick, or (b) more LLM-fillable slots in existing template. Lean (a) — cleaner, easier to audit.
2. **Decide agent_enabled flip strategy.** Templates clean, code solid, pipeline verified. Realistically ready for week 1 ramp at daily_send_cap=10 with manual approve.
3. **Optional sweep.** Re-run the template paragraph-length sweep query (in projects/fub-ai-agent.md session 8 entry) before any new template ships.

## Critical context (carry forward)

- agent_enabled blocks BOTH cron AND manual approve via outboundGate.checkOutboundGate. Flip true for sends, flip back to false when done if not yet in production ramp.
- FUB Automation 2.0 delivers email as plaintext. NEVER use `<br>` in templates. Real newlines work in plaintext and HTML modes.
- FUB UI Merge Fields dropdown produces `%snake_case%` tokens. Don't type tokens, use the dropdown.
- Test rig: FUB person 27764 (real Brian in Sphere stage, pond 9 = excluded). Agent Test - Brian (31924) deleted.
- gerardmarmo@yahoo.com (FUB 23552): Brian in active manual conversation. Reject any agent drafts targeting him.
- Template paragraph sweep query lives in projects/fub-ai-agent.md "Session 8" entry. Run before shipping new templates.
