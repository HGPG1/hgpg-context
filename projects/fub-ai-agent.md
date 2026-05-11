<!-- Last Updated: 2026-05-11 -->

# FUB AI Agent

- **Status:** 🔴 Sessions 1-6 shipped. Smoke test attempted 2026-05-11, CP4 failed on FUB platform constraints. `agent_enabled = false`. Architecture redesign required before any further outbound. Session 7 needs: multi-field paragraph stitching + revert of the HTML body conversion shipped this morning.

## Session 7 entry — 2026-05-11 (smoke test attempted, CP4 failed)

### What happened

Ran the 4-checkpoint smoke test gate end-to-end. Three of four passed cleanly. CP4 (first real outbound to Jay Miller, draft id 9) revealed TWO foundational FUB platform constraints that the current architecture doesn't handle:

1. **FUB custom fields silently truncate at ~255 characters on API write.** Jay's body was 423 chars; FUB stored only 255 of them, cut off mid-word ("doesn't know wha"). The POST returned 200, no error, our pusher logged `fub_push_success`, and Viktor's Automation 2.0 fired and delivered the truncated email to Jay's inbox.
2. **FUB API HTML-escapes text custom field values on write.** Our `<br><br>` got stored as `&lt;br&gt;&lt;br&gt;` in field 157, then rendered as literal `<br><br>` text in the delivered email. This is opposite behavior from when HTML is typed into a custom field via FUB's UI HTML editor — UI writes preserve HTML, API writes escape it.

Jay received a confusing email at 10:56 AM ET (14:56 UTC). Brian sent a personal recovery email from his own account within 10 minutes. Damage contained.

### Checkpoint results

- ✅ **CP1 PASS** (10:35 ET) — Viktor's Automation 2.0 trigger fires on `agent_draft_email_ready` tag. Test person "Agent Test - Brian" (FUB ID 31924) created with email BrianMcCarron@icloud.com.
- ✅ **CP2 PASS** (~10:50 ET, with caveats) — Initial CP2 failed with literal `[-customAgentDraftEmailSubject-]` text in delivered email. Root cause: the `[-camelCase-]` merge syntax from session 6 was stored byte-for-byte through `POST/GET /v1/templates` but FUB never renders that syntax. Fix: opened Template 1156 in FUB UI, re-inserted merge fields via the "Merge Fields" dropdown which substituted `%custom_agent_draft_email_subject%` and `%custom_agent_draft_email_body%`. Subsequent test with HTML body typed into custom field rendered with paragraph breaks.
- ✅ **CP3 PASS** (10:42 ET) — Smoke test SQL `scripts/session-5-smoke-test.sql` Section A → B → C executed via Supabase MCP. Test draft id 11 inserted, Brian's record temporarily flipped, then fully restored. Final state byte-identical to pre-test baseline across all 7 verification columns. Audit log empty because direct-SQL test bypasses agent code path.
- ❌ **CP4 FAIL** (10:56 ET) — Jay Miller draft id 9 pushed successfully (sub-2-second approve → push → FUB chain confirmed in audit log: `gate_check` → `draft_approved` → `fub_push_success`). Delivered email was unreadable due to 256-char truncation + HTML escape.

### What shipped today

**`lib/agent/emailFormat.ts`** (commit `6a1fa72`) — `formatEmailBodyForFub()` helper that HTML-escapes `&`, `<`, `>` then converts `\n\n` to `<br><br>` and `\n` to `<br>`. Applied in `lib/agent/fubPusher.ts` for `channel === 'email'` drafts only. **This must be reverted in Session 7** — the conversion is counterproductive because FUB escapes the result on write regardless.

**`scripts/backfill-html-email-bodies.ts`** (commit `6a1fa72`) — one-shot Node 25 `--experimental-strip-types` runner that backfilled drafts 7, 8, 9, 10 from plaintext `\n\n` to `<br><br>`. **Also must be reverted** — Session 7 needs the inverse migration to put `\n\n` back, OR discard the 3 still-pending drafts (Gerard #7, Seanna #8, Anthony #10) and regenerate them post-architecture-fix.

**Template 1156 fixed in FUB UI** — merge fields re-inserted via Merge Fields dropdown. Now correctly uses `%snake_case%` syntax. Subject and body render correctly when underlying custom field values are valid.

### Diagnostic confirmation via FUB MCP (read after CP4 failure)

Pulled Jay's record via `getPerson(24825)` to see what FUB actually stored:

- `customAgentDraftEmailBody`: 255 chars, ends mid-word ("doesn't know wha"). The `<br><br>` we sent is stored as `&lt;br&gt;&lt;br&gt;`. Both problems confirmed.
- `customAgentDraftEmailSubject`: "More than a number on the report?" — stored correctly (under 256, no HTML).
- Tags: `agent_email_sent` (Automation 2.0 step 2 fired), plus the lead's existing tags. `agent_draft_email_ready` was already removed by the Automation's tag-removal step.

### Learnings captured in session 7

5. **FUB merge syntax for templates is `%snake_case%` at render time, not `[-camelCase-]` at storage time.** The session 6 round-trip-verification was insufficient because storing the literal characters through `POST/GET /v1/templates` doesn't prove rendering. FUB's UI Merge Fields dropdown is the source of truth for what merge syntax actually renders. Future template work via API must include a render-test (actual Automation invocation against a real test person + inbox check) before being considered shipped.

6. **FUB text custom fields are 256-char hard limited and silently truncate on API write.** No 400, no warning, no truncation flag in the response. Pusher needs defensive `LENGTH(value) <= 256` check before write, log `block_reason='value_too_long'` event, surface in queue UI for manual edit. The 256 limit applies to ALL text custom field writes, not just our 157/158/159.

7. **FUB API HTML-escapes text custom field values on write but FUB UI does not.** Two write paths, two storage behaviors. Conclusion: HTML cannot be sent through API-written custom fields and survive into rendered emails. Multi-field paragraph stitching (Option A) is the path — store each paragraph in its own custom field as plaintext, use literal `<br><br>` in the template BODY between merge tokens. The literal `<br>` in the template body is HTML (not a merge value), so it doesn't get escaped.

8. **TM has no `npm run typecheck` script** — only `lint`. Direct `./node_modules/.bin/tsc --noEmit` works. Future Claude Code prompts should say "run tsc" not "run npm run typecheck."

## Session 7 redesign spec (for the next build session)

### What needs to change

**Database:**
- Add `fub_agent_message_drafts.body_paragraphs jsonb` column, OR migrate the existing `body` column to store an array of paragraph strings
- Backfill or discard the 3 still-pending drafts (Gerard #7, Seanna #8, Anthony #10) — they're currently in HTML format and will need re-conversion or regeneration

**FUB-side:**
- Create 3-4 additional text custom fields via `POST /v1/customFields`:
  - `customAgentDraftEmailPara2` (id TBD)
  - `customAgentDraftEmailPara3` (id TBD)
  - `customAgentDraftEmailPara4` (id TBD)
- Update Template 1156 body via FUB UI:
  - Merge field 1 (existing `customAgentDraftEmailBody` — keep this as paragraph 1)
  - Literal `<br><br>` between
  - Merge field 2 (`customAgentDraftEmailPara2`)
  - Literal `<br><br>` between
  - Etc.
  - The literal `<br><br>` in the template body IS HTML and renders correctly because it's not a merge value
- Repeat for iMessage template if/when iMessage path is built (LoopMessage is plaintext-native so this may not be needed)

**Agent code:**
- Revert `lib/agent/emailFormat.ts` and its application in `fubPusher.ts` (commit `6a1fa72`)
- Refactor `lib/agent/draftGenerator.ts` to emit body as an array of paragraphs:
  - Each paragraph under 250 chars (target — leaves 6 chars of FUB headroom for safety)
  - Templates need adjustment so generated content naturally breaks into paragraphs
  - Voice file (`brian-voice.md`) constraint: paragraphs are SHORT in Brian's voice anyway, this should be a small lift
- Refactor `lib/agent/fubPusher.ts` to write each paragraph to its respective custom field
- Add defensive length check: refuse to push if any paragraph > 250 chars, log `block_reason='paragraph_too_long'`, surface in queue UI for manual edit
- For drafts with fewer paragraphs than there are fields, write empty string to unused fields (FUB will render nothing, no orphan whitespace)

### Verification sequence for Session 7

1. Run all redesign on Agent Test - Brian (FUB ID 31924) FIRST with multi-paragraph synthetic content
2. Verify all paragraphs render with correct breaks, no truncation, no escaped HTML
3. Then re-run real CP1-CP4 against Gerard or Seanna (or fresh draft)
4. Then flip `agent_enabled = true` and ramp daily cap as originally planned

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

FUB email shell artifact at `scripts/fub-email-shell.md`. Critical gotcha (now superseded): the email step "must be plaintext, not HTML." This guidance was based on incorrect assumption about FUB's template editor having a plaintext mode — it does not. Email templates are HTML-only.

## Session 5 — 2026-05-07

Timezone helper, daily-cap flush cron, inbound classifier wired to side effects, cooldown reason enum extended, draftGenerator optout skip, smoke test SQL artifact.

## Session 4 — 2026-05-07

FUB pusher, reject taxonomy with 5-class enum, cooldown table, daily cap default 10, queue header with kill switch, inbound classifier stub, three FUB custom fields created (157/158/159), two FUB tags scheduled.

### Learnings captured in session 4

1. FUB tag list endpoint not accessible to standard API key.
2. Schema realism beats spec when adding new tables that reference existing ones.

## What this is

An AI-augmented lead nurture layer for HGPG. Scores Brian's eligible FUB leads using a hybrid rules + LLM intent engine, drafts personalized messages, and uses FUB Automations 2.0 (with Sendblue for iMessage) to do the actual sending. The agent does the *thinking* (who, what, when). FUB does the *sending*.

## Architecture (v1 — currently broken, see session 7)

- Embedded in TM repo (`HGPG1/hgpg-transaction-manager`) under `lib/agent/` and `app/agent/`
- Supabase: `ioypqogunwsoucgsnmla` (HGPG Core)
- Vercel: TM project
- LLM: Anthropic Haiku 4.5 via `AGENT_ANTHROPIC_API_KEY`
- Auth: gated to `brian@homegrownpropertygroup.com`
- Cron: `/api/agent/cron` + `/api/cron/agent-daily-flush`

## Tables (8 fub_agent_* tables, RLS-locked to service role)

| Table | Purpose |
|---|---|
| `fub_agent_leads` | Denormalized FUB person snapshots |
| `fub_agent_lead_scores` | Score history |
| `fub_agent_message_templates` | Template library |
| `fub_agent_message_drafts` | Pending/approved/sent/discarded/blocked drafts |
| `fub_agent_lead_optouts` | Source of truth for opt-outs |
| `fub_agent_lead_exclusions` | Hard exclusions |
| `fub_agent_lead_cooldowns` | Per-lead temporary cooldown windows |
| `fub_agent_config` | KV thresholds + master kill switch |
| `fub_agent_log` | Full audit trail |

## Configuration as of session 7 end

| Key | Value |
|---|---|
| `agent_enabled` | `false` (flipped back at 15:02 UTC) |
| `auto_below_threshold` | `false` |
| `hot_threshold` | `40` |
| `warm_threshold` | `30` |
| `llm_confidence_floor` | `0.4` |
| `daily_send_cap` | `10` |
| `voice_file` | `brian-voice.md` |
| `scoring_version` | `v1` |

## Pilot scope

Brian-only v1. Buyer-only v1 (sellers in v2, though Jay was a seller draft test).

## Open items going into session 7

- Revert HTML body conversion + backfill
- Multi-field paragraph stitching architecture
- Hot tier templates, iMessage seller variants, cooldown re-touch templates (still deferred)
- Scoring sweep on 4,340 unscored eligible leads (deferred until architecture proven)
- FUB UI visibility on custom fields 157/158/159 to admins-only (still carryover from session 4)
