<!-- Last Updated: 2026-05-11 -->

# FUB AI Agent

- **Status:** 🟢 Operational. Session 8 (2026-05-11) shipped seller template fixes for all 3 templates with too-long paragraphs. Production pipeline verified end-to-end in session 7. agent_enabled=false; can flip true any time. Buyer template voice-clone issue still parked (session 9).

## Session 8 — 2026-05-11 (template paragraph length sweep)

### What shipped

Three seller-track email templates had paragraph 3 over the 250-char cap, baked into the template body before any slot fill. Found via a 1-query sweep across all active templates, fixed via three UPDATE statements through Supabase MCP. No code change required — pure data.

- **Template 9** `v1.warm.seller_report_engaged.seller` — para 3 was 252 chars. Dropped "actually" and "specific" as filler words. New para 3 = 235 chars.
- **Template 2** `v1.warm.long_dormant_buyer.email` — para 3 was 252 chars. Dropped "straight up" and "actually". New para 3 = 231 chars.
- **Template 11** `v1.warm.long_dormant_seller.seller` — para 3 was 286 chars (worst offender). Tightened "still thinking about a sale" → "still thinking about selling", "did life take you a different direction" → "did life take you elsewhere", dropped "actually" and "probably", removed "in the area". New para 3 = 222 chars.

After fixes, regenerated drafts for John Miller (23316), Jay Miller (24825), Rachel Delmore (24498) — all 3 inserted clean with no block_reason. Paragraph 3 lengths now sit at 235/235/175 chars respectively.

### Sweep query for future reference

This query catches any active template paragraph over the cap. Worth re-running before any new templates ship:

```sql
SELECT 
  t.id, t.scenario, t.channel, t.lead_type,
  para.ordinality as para_num,
  length(para.paragraph) as para_len,
  left(para.paragraph, 80) as preview,
  CASE 
    WHEN length(para.paragraph) > 250 THEN 'OVER CAP'
    WHEN length(para.paragraph) > 230 THEN 'tight - risky after slot fill'
    ELSE 'ok'
  END as status
FROM fub_agent_message_templates t
CROSS JOIN LATERAL unnest(string_to_array(t.body, E'\n\n')) WITH ORDINALITY AS para(paragraph, ordinality)
WHERE t.active = true
ORDER BY length(para.paragraph) DESC;
```

After session 8 fixes, the worst paragraph across all 14 active templates is 235 chars — comfortable 15-char headroom under the 250 cap. Worth flagging template 13 paragraph 2 at 223 chars and template 10 paragraph 2 at 222 chars as "tight" — those have less than ~30 chars of headroom for slot fills like `{{last_outreach_phrase}}`. If those start blocking, the same trim approach applies.

### Pattern recognition for future template work

All three offending paragraphs followed the same structure: two complete thoughts joined into one paragraph by a sentence boundary. The trim path (drop 2-4 filler words) preserves voice better than splitting into 2 paragraphs, because splitting would push the template over the 4-paragraph max. **Rule of thumb for template authors:** if a paragraph has both an offer and a clarification, those usually want to be two adjacent sentences with light filler-word density rather than a single dense paragraph.

## Session 8 backlog (parked for session 9)

### Issue 2: Buyer template `v1.warm.viewed_listing_recent.email` produces voice-clone drafts

[Carried over from session 7 backlog, no change in scope. 5 buyer drafts in queue right now (14, 16, 18, 20) are all near-identical content. Lean toward path (b): 3-4 phrasing variants of viewed_listing_recent with random pick in selectTemplate.]

### Issue 3: hideIfEmpty inconsistency on Para fields

[Carried over.]

### Decide agent_enabled flip strategy

[Carried over. Templates are now clean, code is solid, real outbound verified. Probably ready to flip true with daily_send_cap=10 for week 1 once buyer template diversity lands.]

## Session 7 wrap-up — 2026-05-11

### What shipped

**Architecture redesign complete.** Email body now split across 4 FUB custom fields (Body=para1 + Para2 + Para3 + Para4, IDs 157/160/161/162) with newline-based paragraph stitching in Template 1156. The platform constraints discovered in Session 6 (256-char silent truncation, API-side HTML escaping) are no longer load-bearing.

**Phases completed in order:**
1. Audit + schema migration: paragraphs text[] column on fub_agent_message_drafts with CHECK constraint via fub_agent_paragraphs_valid() IMMUTABLE function, NOT VALID so historical rows are exempt. body column now NULLABLE.
2. Reverted commit 6a1fa72 cleanly (revert commit 76ba8a2). lib/agent/emailFormat.ts and scripts/backfill-html-email-bodies.ts deleted.
3. Hard-deleted 3 pending drafts (Gerard #7, Seanna #8, Anthony #10) that had <br><br> contamination. Jay #9 migrated in place as audit row, status='approved' preserved.
4. Created 3 new FUB custom fields via POST /v1/customFields: id 160 (Para2), 161 (Para3), 162 (Para4). Note: these were created with hideIfEmpty=false vs. field 157's hideIfEmpty=true. Cosmetic only; can normalize later.
5. Code refactor (commit 504f116): 9 files. lib/agent/fubAgentConstants.ts adds Para2-4 + FUB_CF_AGENT_DRAFT_EMAIL_BODY_FIELDS tuple + per-paragraph cap. lib/agent/paragraphs.ts NEW (splitBodyToParagraphs, validateParagraphs, padParagraphsToFubFields). draftGenerator.ts splits rendered body, validates against 250-char cap, inserts with block_reason on failure. fubPusher.ts writes 4 fields with empty-string padding so stale paragraphs from prior longer drafts get cleared on FUB person record. Edit route re-splits + re-validates, clears block_reason on success. Queue UI renders paragraphs individually with per-paragraph char counts, amber warning banner + disabled Approve when block_reason=paragraph_too_long. types.ts updated for nullable body + paragraphs field. log.ts added draft_created_with_block + fub_push_blocked event types.
6. Template 1156 update done via FUB UI: 4 merge tokens (%custom_agent_draft_email_body% + Para2 + Para3 + Para4) separated by actual newlines typed via Enter. Critical correction from spec: <br><br> separators do NOT work because FUB's Automation 2.0 Send Email step delivers as plaintext (or mail client picks plaintext alternative), so <br> arrives as literal text. Newlines work in both plaintext and HTML modes.
7. Pre-flight test against FUB person 27764 (Brian-in-Sphere) confirmed all 4 paragraphs render with breaks, no truncation, no escaped HTML.
8. Production verification: generated draft 17 for Jesse Hernandez (FUB 11323) via cron path, manually approved via queue UI, email landed in Jesse's inbox via Automation 2.0. Audit chain clean: lead_classified → draft_created → gate_check → draft_approved → fub_push_success in 532ms.

**Constraint-violation bug found and patched same-session (commit c8f0d51).** During Phase 7 verification, regenerated drafts for John Miller 23316 and Jay Miller 24825 failed at DB insert with "violates check constraint paragraphs_length_check". Root cause: draftGenerator.ts always inserted the paragraphs array even when validateParagraphs returned not-ok. The CHECK rejected the row and the lead disappeared from anywhere visible. Fix: insert paragraphs=null when blockReason is set. block_reason still populated for queue UI surfacing.

### Learnings captured in session 7

1. **FUB Automation 2.0 Send Email step delivers as plaintext (or the plaintext-alternative wins), NOT HTML.** The session 7 spec assumed `<br><br>` between template merge tokens would render as line breaks. In practice the recipient sees literal `<br><br>` text. Real newlines (typed via Enter in the FUB template editor) work in both plaintext and HTML modes. This is a permanent platform truth: never put HTML in FUB email templates for the agent.
2. **FUB UI Merge Fields dropdown produces `%snake_case%` syntax, not the camelCase API field names.** Field 157 API name is `customAgentDraftEmailBody`, but the merge token in templates is `%custom_agent_draft_email_body%`. Don't try to type merge tokens — use the dropdown.
3. **Postgres CHECK constraints cannot contain subqueries directly, but CAN call IMMUTABLE functions that contain subqueries.** First migration attempt with `SELECT max(length(p)) FROM unnest(paragraphs)` in the CHECK clause failed with `cannot use subquery in check constraint`. Wrapping in `CREATE FUNCTION ... LANGUAGE sql IMMUTABLE` then using the function in the CHECK works. Saved as `fub_agent_paragraphs_valid(text[])`.
4. **NOT VALID is the right tool for adding constraints when historical rows might violate them.** `ADD CONSTRAINT ... CHECK (...) NOT VALID` enforces all future INSERT/UPDATE without scanning existing rows. Used for paragraphs_length_check because Jay's audit draft (status=approved, body=423 chars from CP4) would have been rejected by a regular CHECK.
5. **The kill switch (agent_enabled=false) blocks BOTH cron AND manual approve.** Earlier session work treated agent_enabled as cron-only, but the outboundGate checks it on every send. To do verification sends with agent_enabled=false, flip it true momentarily, do the send, flip back to false. Better design than what I initially assumed; correct safety behavior.
6. **The generator's structured paragraph output already existed.** Templates have `\n\n` paragraph separators baked into their body strings. splitBodyToParagraphs(rendered.body) just splits on those. No prompt rewriting needed; the generator was always producing paragraph-structured content, we just weren't storing it as an array.

## Session 8 backlog (parked for next session)

### Issue 1: Seller template `v1.warm.seller_report_engaged.seller` paragraph 3 is 252 chars (over the 250 cap, before slot-fill)

3 of 3 seller drafts attempted with this template have blocked on paragraph_too_long: John Miller 23316 (draft 19), Jay Miller 24825 (draft 21), Tamela Karnazes 13307 (draft 22). Same paragraph, same overflow, every time.

The offending paragraph is hardcoded template prose:

```
The number on the report is a solid starting point but it doesn't know what your home actually shows like or what's trading on your specific block. If you're getting closer to listing, or just want a real PMV before you decide, happy to walk through it.
```

That's 252 chars literal. Slots in this template (`{{first_name}}`, `{{recency_phrase}}`) don't even land in this paragraph — it's the template body itself that's too long.

**Fix:** edit the template body in the DB (fub_agent_message_templates row where scenario = 'v1.warm.seller_report_engaged.seller'). Either split into two paragraphs on the period after "specific block" OR trim ~10 chars of filler. Same fix likely applies across other seller templates that haven't been exercised yet; sweep all seller templates and validate each paragraph against 250 chars at template-load time.

### Issue 2: Buyer template `v1.warm.viewed_listing_recent.email` is producing voice-clone drafts

5 buyer drafts generated this session (Gerard, Seanna, Anthony, Sam, plus Gerard regen): same template, same paragraphs, near-identical content. Only variables are first_name and a 1-3 word time phrase ("recently" / "a few days ago" / "last week"). Reads like a mail-merge if anyone receives multiple agent emails over a few weeks.

**Root cause:** the template has 1-2 LLM-fillable slots and a large amount of hardcoded text. The LLM's slot fill is essentially Mad Libs.

**Fix options for session 8:**
- (a) Expand slot count in buyer templates so more of the body is LLM-generated (more variation, but harder to keep voice consistent)
- (b) Create 3-4 buyer template variants in the `v1.warm.viewed_listing_recent` family so selectTemplate's priority cascade can rotate them randomly per draft
- (c) Just accept the cloning since daily_send_cap=10 limits exposure across recipients — unlikely two recipients compare emails

Recommend (b) — write 3 alternate phrasings of the same scenario, store as separate scenarios, and add a random pick step in selectTemplate when multiple scenarios match.

### Issue 3: hideIfEmpty inconsistency on Para fields

Field 157 (Body) has hideIfEmpty=true. Fields 160/161/162 (Para2/3/4) have hideIfEmpty=false. Means empty Para2-4 will show as empty fields in the FUB person profile UI, where Body hides itself when empty. Cosmetic; not blocking. Normalize via PATCH /v1/customFields/{id} if it bothers you.

### Test rig housekeeping

- Agent Test - Brian (FUB ID 31924) was deleted during session 7. Test rig now uses FUB ID 27764 (real Brian person, in Sphere stage, pond 9 which is POND_BRIAN_EXCLUDED so never picked up by agent). Keep this as the test rig going forward.
- Person 27764 has a stale tag `agent_draft_email_` (trailing underscore, no "ready" suffix). Probably from earlier session work. Safe to leave or clean up.

### Queue state at session 7 end (9 drafts in flight)

- Drafts 14, 16, 18, 20: buyer template `viewed_listing_recent`, clean, ready to approve. Same template; pick at most one or two if approving. (Or wait for session 8 template diversity fix.)
- Drafts 19, 21, 22: seller template `seller_report_engaged`, blocked on paragraph_too_long. Surfaced in queue with amber warning + disabled Approve. Need session 8 template fix OR manual edit.
- Draft 23: seller template `listing_thinking`, clean. Jesse Hernandez, same lead that received the verification send. Skip (probably reject) since he just got the other email.
- Draft 17: Jesse's approved+sent verification draft. Real outbound. Audit row.

## Session 7 entry — 2026-05-11 (smoke test attempted, CP4 failed)

[Original session 7 kickoff entry preserved below — describes the CP4 failure that motivated this whole session]

### What happened

Ran the 4-checkpoint smoke test gate end-to-end. Three of four passed cleanly. CP4 (first real outbound to Jay Miller, draft id 9) revealed TWO foundational FUB platform constraints that the current architecture doesn't handle:

1. **FUB custom fields silently truncate at ~255 characters on API write.** Jay's body was 423 chars; FUB stored only 255 of them, cut off mid-word ("doesn't know wha"). The POST returned 200, no error, our pusher logged `fub_push_success`, and Viktor's Automation 2.0 fired and delivered the truncated email to Jay's inbox.
2. **FUB API HTML-escapes text custom field values on write.** Our `<br><br>` got stored as `&lt;br&gt;&lt;br&gt;` in field 157, then rendered as literal `<br><br>` text in the delivered email. This is opposite behavior from when HTML is typed into a custom field via FUB's UI HTML editor — UI writes preserve HTML, API writes escape it.

Jay received a confusing email at 10:56 AM ET (14:56 UTC). Brian sent a personal recovery email from his own account within 10 minutes. Damage contained.

### Diagnostic confirmation via FUB MCP (read after CP4 failure)

Pulled Jay's record via `getPerson(24825)` to see what FUB actually stored:

- `customAgentDraftEmailBody`: 255 chars, ends mid-word ("doesn't know wha"). The `<br><br>` we sent is stored as `&lt;br&gt;&lt;br&gt;`. Both problems confirmed.
- `customAgentDraftEmailSubject`: "More than a number on the report?" — stored correctly (under 256, no HTML).
- Tags: `agent_email_sent` (Automation 2.0 step 2 fired), plus the lead's existing tags. `agent_draft_email_ready` was already removed by the Automation's tag-removal step.

## Session 6 wrap-up — 2026-05-09 (PM status reconciliation)

Brian confirmed Viktor's Automation 2.0 is fully wired and complete. The build is sitting at the smoke-test gate.

[Prior session 6 content preserved below this point - see session entries 1-6 history]

## Session 6 (cont.) — 2026-05-09 (PM stopping point)

What shipped:
- Pre-flight verification: GET `/v1/customFields` confirmed exact-name match for `customAgentDraftEmailBody` (id 157), `customAgentDraftEmailSubject` (id 158), `customAgentDraftIMessageBody` (id 159).
- Repo-root `CLAUDE.md` created with session-end checklist.
- `BRAIN_WRITE_TOKEN` moved to env var. `.env.local.example` shipped.
- `fub_cleanup/` decommissioned. Stale API key confirmed revoked.
- Brain push pattern proven end-to-end.

## Session 6 (cont.) — 2026-05-09 (AM micro-task)

FUB email template created via API. `POST /v1/templates` returned id 1156. Round-trip verified via GET. **NOTE: round-trip verification was insufficient — see session 7 learnings.**

## Session 6 — 2026-05-08

FUB email shell artifact at `scripts/fub-email-shell.md`. Critical gotcha (now superseded): the email step "must be plaintext, not HTML." This guidance was based on incorrect assumption about FUB's template editor having a plaintext mode — it does not. Email templates are HTML-only. (Session 7 update: the email is DELIVERED as plaintext regardless of how the template is constructed in the editor. The "plaintext" instinct was right; the implementation path was wrong.)

## Session 5 — 2026-05-07

Timezone helper, daily-cap flush cron, inbound classifier wired to side effects, cooldown reason enum extended, draftGenerator optout skip, smoke test SQL artifact.

## Session 4 — 2026-05-07

FUB pusher, reject taxonomy with 5-class enum, cooldown table, daily cap default 10, queue header with kill switch, inbound classifier stub, three FUB custom fields created (157/158/159), two FUB tags scheduled.

### Learnings captured in session 4

1. FUB tag list endpoint not accessible to standard API key.
2. Schema realism beats spec when adding new tables that reference existing ones.

## What this is

An AI-augmented lead nurture layer for HGPG. Scores Brian's eligible FUB leads using a hybrid rules + LLM intent engine, drafts personalized messages, and uses FUB Automations 2.0 (with Sendblue for iMessage) to do the actual sending. The agent does the *thinking* (who, what, when). FUB does the *sending*.

## Architecture (v2 - session 7 multi-field paragraph stitching)

- Embedded in TM repo (`HGPG1/hgpg-transaction-manager`) under `lib/agent/` and `app/agent/`
- Supabase: `ioypqogunwsoucgsnmla` (HGPG Core)
- Vercel: TM project
- LLM: Anthropic Haiku 4.5 via `AGENT_ANTHROPIC_API_KEY`
- Auth: gated to `brian@homegrownpropertygroup.com`
- Cron: `/api/agent/cron` + `/api/cron/agent-daily-flush`
- Email body split across 4 FUB custom fields (160/161/162 added 2026-05-11) to dodge FUB's 256-char silent truncation. FUB Template 1156 stitches the 4 paragraphs with newlines (NOT `<br>` tags - those render as literal text in delivered email).

## Tables (8 fub_agent_* tables, RLS-locked to service role)

| Table | Purpose |
|---|---|
| `fub_agent_leads` | Denormalized FUB person snapshots |
| `fub_agent_lead_scores` | Score history |
| `fub_agent_message_templates` | Template library |
| `fub_agent_message_drafts` | Pending/approved/sent/discarded/blocked drafts. paragraphs text[] added 2026-05-11; body NULLABLE. |
| `fub_agent_lead_optouts` | Source of truth for opt-outs |
| `fub_agent_lead_exclusions` | Hard exclusions |
| `fub_agent_lead_cooldowns` | Per-lead temporary cooldown windows |
| `fub_agent_config` | KV thresholds + master kill switch |
| `fub_agent_log` | Full audit trail |

## Configuration as of session 7 end

| Key | Value |
|---|---|
| `agent_enabled` | `false` (flipped on briefly during session 7 Phase 7 verification, flipped back) |
| `auto_below_threshold` | `false` |
| `hot_threshold` | `40` |
| `warm_threshold` | `30` |
| `llm_confidence_floor` | `0.4` |
| `daily_send_cap` | `10` |
| `voice_file` | `brian-voice.md` |
| `scoring_version` | `v1` |

## FUB custom fields

| ID | API Name | Label | Purpose |
|---|---|---|---|
| 157 | customAgentDraftEmailBody | Agent Draft Email Body | Paragraph 1 (was full body pre-session-7) |
| 158 | customAgentDraftEmailSubject | Agent Draft Email Subject | Email subject |
| 159 | customAgentDraftIMessageBody | Agent Draft IMessage Body | iMessage body (single field, plaintext via LoopMessage) |
| 160 | customAgentDraftEmailPara2 | Agent Draft Email Para2 | Paragraph 2 (session 7) |
| 161 | customAgentDraftEmailPara3 | Agent Draft Email Para3 | Paragraph 3 (session 7) |
| 162 | customAgentDraftEmailPara4 | Agent Draft Email Para4 | Paragraph 4 (session 7) |

## Pilot scope

Brian-only v1. Buyer-only v1 (sellers in v2, though Jay was a seller draft test). Seller templates exist but session 7 surfaced that seller template `seller_report_engaged` has a too-long paragraph; awaiting session 8 fix.

## Open items going into session 8

- Fix seller template `v1.warm.seller_report_engaged.seller` paragraph 3 (252 chars hardcoded, blocks 100% of drafts using this template)
- Sweep all seller templates and validate each paragraph ≤ 250 chars at template-load time (consider a one-shot script that runs once over the template table)
- Add buyer template variation to break the voice-clone problem
- Decide whether to flip `agent_enabled=true` for sustained operation OR keep it manual-approve-only for week 1 ramp
- Scoring sweep on 4,340 unscored eligible leads (still deferred from session 6)
- Hot tier templates, iMessage seller variants, cooldown re-touch templates (still deferred)
- FUB UI visibility on custom fields 157-162 to admins-only (still carryover from session 4)
- Normalize hideIfEmpty across fields 157 vs 160/161/162 (cosmetic)
