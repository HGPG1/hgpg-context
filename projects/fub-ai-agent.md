<!-- Last Updated: 2026-05-09 -->

# FUB AI Agent

- **Status:** 🟡 Sessions 1-5 shipped + session 6. **Viktor's Automation 2.0 setup is COMPLETE as of 2026-05-09 PM.** Template 1156 live, both FUB custom fields confirmed exact-name match, queue has 4 warm pending drafts. The 4-checkpoint smoke test gate sequence is the only remaining step before flipping `agent_enabled = true`.

## Session 6 wrap-up — 2026-05-09 (PM status reconciliation)

Brian confirmed Viktor's Automation 2.0 is fully wired and complete. The build is now sitting at the smoke-test gate.

**Smoke test gate (unchanged from session 6 stopping point):**

1. Manual test through the Automation — Brian sets custom fields 157/158 by hand on a test person, saves, watches it fire. Confirms tag-trigger fires + merge tags resolve.
2. Confirm email arrives plaintext, paragraph breaks intact, merge tags fully resolved (not literal `[-...-]` text in inbox). HTML-mode collapse on `\n\n` is the most likely failure here.
3. Run `scripts/session-5-smoke-test.sql` Section A→B→C — synthetic Brian-only iMessage draft against fub_person_id 27764, exercises agent code path end-to-end.
4. Push **one** real warm draft (Gerard, draft id 7, score 37) and monitor. First real outbound.

If all 4 pass → flip `agent_enabled = true`, ramp `daily_send_cap` 10 → 25 over week 1, watch reject rate.

`agent_enabled` still false. Cron healthy and gated. 4 warm drafts ready (Gerard, Seanna, Jay, Anthony Scott). 4,340 unscored eligible leads queued for a scoring sweep after smoke test passes.

**Recommendation for next session:** desktop, not mobile — multi-window watching (FUB, Automation log, TM agent queue, Supabase) makes the test cleaner.

## Session 6 (cont.) — 2026-05-09 (PM stopping point)

What shipped:
- **Pre-flight verification, both green.** GET `/v1/customFields` confirmed exact-name match for `customAgentDraftEmailBody` (id 157) and `customAgentDraftEmailSubject` (id 158); `customAgentDraftIMessageBody` (id 159) also present for the future iMessage path. Capitalization, prefix, casing all align with `lib/agent/fubAgentConstants.ts`. Supabase queue check confirmed 4 drafts in `pending_review`; top 3 by combined_score: Gerard (draft 7, score 37, scenario `v1.warm.viewed_listing_recent.email`), Seanna (draft 8, 37, same scenario), Jay (draft 9, 35, `v1.warm.seller_report_engaged.seller`). Anthony Scott is the 4th. No empty-queue blocker.
- **Repo-root `CLAUDE.md` created** with a session-end checklist that codifies the brain-write workflow: at session end, push code to repo + push brain files via `POST https://brain.homegrownpropertygroup.com/api/external/write`, verify 2xx, do not use `_brain-pending/` as a default staging area. Includes a minimal env-driven helper snippet that reads the token from `BRAIN_WRITE_TOKEN`.
- **`BRAIN_WRITE_TOKEN` moved to env var.** Real value appended to local `.env.local` (gitignored). Created `.env.local.example` with the variable name only (no value), shipped via repo. Added `!.env*.example` to `.gitignore` since the existing `.env*` rule was over-broad and would have ignored the example. Added `_brain-pending/` to `.gitignore` so the fallback staging dir never accidentally lands in git.
- **`fub_cleanup/` decommissioned.** The April 2026 one-shot Python tool that cleaned up Texting Betty SMS action plans had a real FUB API key (suffix `wz1BQa`) sitting in plaintext in `fub_cleanup/.env` for ~2 weeks in an iCloud-synced directory. Verified against FUB Admin: Brian had already revoked it during an earlier FUB key cleanup, so the key was inert by the time it was caught. Local `fub_cleanup/.env` replaced with a `REVOKED before 2026-05-09` marker; `fub_cleanup/README.md` got a status banner. Whole `fub_cleanup/` directory now gitignored — audit CSVs from the 2026-04-24 run stay on disk as receipts but never reach the repo. Production FUB key (suffix `5j9xeR`, in `.env.local` and `.env.vercel.local`) is a separate value, untouched.
- **Brain push pattern proven end-to-end** in this session — earlier today's AM micro-task pushed `projects/fub-ai-agent.md` and `SESSION-HANDOFF.md` directly to brain via the write API (200 on both, commits `b90d85a` and `ffcefd9`); CLAUDE.md now codifies this so future sessions don't drift back to `_brain-pending/`-as-default.

What was confirmed during the session:
- The `[-fieldname-]` template syntax FUB uses is stored byte-identical on round-trip — this was verified explicitly via GET `/v1/templates/1156`. No HTML escaping, no smart-quote rewriting. So any future templates we create via API don't need defensive encoding.
- The same Basic + `X-System` headers used by `lib/agent/fubClient.ts` have scope to GET `/v1/customFields` and POST `/v1/templates`. No higher-scoped key needed for these admin paths.
- Brain-app write API auto-stamps the author from the bearer token (currently `brian@homegrownpropertygroup.com`); per-file size limit is 1 MB and both files (~22 KB and ~9 KB) fit comfortably.

Did not flip / out of scope:
- `agent_enabled` still **false**. No outbound. Cron healthy and gated.
- Smoke-test SQL not run (gated on Viktor finishing the Automation 2.0 wire-up).
- iMessage shell still not built.
- Hot tier / iMessage seller variants / cooldown re-touch templates: deferred until after smoke test.
- Scoring sweep on the 4,340 unscored eligible leads: queued for after smoke test passes (keeps the queue fed past day 4 of the 10/day cap).

Pickup hints for next session:
- **Don't conflate "Viktor's done" with "ready to flip the switch."** There are 4 checkpoints between those two states, each with a kill switch:
  1. Brian sends a manual test through the Automation (sets custom fields 157/158 by hand on a test person, saves, watches it fire). Confirms the Automation triggers on the tag and the merge tags resolve.
  2. Confirm the email arrives, plaintext, with paragraph breaks intact and the merge tags fully resolved (not literal `[-...-]` text in the inbox).
  3. Run `scripts/session-5-smoke-test.sql` Section A→B→C — synthetic Brian-only iMessage draft against fub_person_id 27764, exercises the agent code path end-to-end (draft generator → approve → push → cooldown → cleanup).
  4. Push **one** real warm draft (Gerard, draft id 7, score 37) and monitor. This is the first real outbound.
- If all 4 gates pass, *then* flip `agent_enabled = true` and ramp `daily_send_cap` 10 → 25 over week 1, watch reject rate.
- Spec for Viktor: select template **id 1156** in Automation 2.0 → "Send Email" step picker, set type to plaintext, add a Step 2 "Remove tag `agent_draft_email_ready`" so the Automation doesn't re-fire on the same person.
- Once smoke test passes, run a scoring sweep on the 4,340 unscored eligible leads to keep the queue fed past day 4.
- Template library expansion (hot tier, iMessage seller variants, cooldown re-touch templates) deferred until smoke test passes.

## Session 6 (cont.) — 2026-05-09 (AM micro-task)

What shipped:
- **FUB email template created via API.** `POST /v1/templates` returned **id 1156**, name `Agent Draft Outreach (System)`, subject `[-customAgentDraftEmailSubject-]`, body `[-customAgentDraftEmailBody-]`. Created at `2026-05-09T09:51:30Z` (UTC) / `2026-05-09T05:51:30-04:00` (ET, EDT). **Round-trip verified** via `GET /v1/templates/1156` — both merge tags came back byte-for-byte identical. No HTML escaping, no smart-quote rewriting, no merge-tag mangling.
- **scripts/fub-email-shell.md** updated with a "Template created" footer block (id, timestamps, verification status, next-step note for Viktor).

What this unblocks:
- Viktor (or Brian) can now open Automation 2.0 → "Send Email" step → template picker → select template id **1156**. Subject and body auto-populate from the template; no copy-paste required.

What was confirmed during the micro-task:
- The standard repo FUB API auth pattern (Basic + `X-System: HGPG-FUB-Agent` / `X-System-Key: fub-agent-v1`) has scope to create templates. Same code path used by `lib/agent/fubClient.ts`.
- FUB's template engine stores `[-fieldname-]` merge tags raw — no escaping on either POST or GET. The earlier worry about merge-tag mangling on round-trip was unfounded.

Did not flip:
- `agent_enabled` still false. No outbound. Per the micro-task scope.
- Smoke-test SQL not run.
- iMessage shell still not built.

## Session 6 — 2026-05-08

What shipped:
- **FUB email shell artifact** at `scripts/fub-email-shell.md`. Specifies the exact merge-tag content for the FUB Automation 2.0 "Send Email" step (subject `[-customAgentDraftEmailSubject-]`, body `[-customAgentDraftEmailBody-]`, nothing else, since the agent's body already ends with `Brian` and FUB auto-appends the user-level signature). Includes a setup checklist for the Automation 2.0 (trigger, tag-removal step for idempotency, optional custom-field clear), user-level FUB requirements (CAN-SPAM-compliant signature, connected email, per-user daily send limit ≥ 10), a manual dry-test flow that runs before the smoke test SQL, and open questions for Viktor.
- **scripts/README.md** updated to reference the new artifact and call it the prerequisite for `session-5-smoke-test.sql`.
- **Critical gotcha documented loud:** the email step **must be plaintext, not HTML**. The agent writes plaintext with `\n` separators; HTML mode collapses them into one run-on paragraph. Most likely failure mode in the smoke test, so it's flagged at the top of the artifact.

What was confirmed during the session:
- **Smoke test never ran.** Database state proves it: 4 drafts in `pending_review` since 2026-05-07 02:31, no `fub_pushed_at` on any draft, no `draft_pushed` events, no `inbound_*` events, no `cron_flush_complete` events. Cron is healthy (firing nightly, correctly no-op'ing while `agent_enabled=false`; `cron_skipped` events at 07:00 UTC on 2026-05-07 + 2026-05-08).
- **Viktor's FUB Automations 2.0 are built but missing the email shell.** Brian confirmed Viktor finished the trigger/tag wiring; gap was the Send Email action's subject/body content.

Did not ship:
- iMessage shell artifact (out of scope; LoopMessage didn't apply this session)
- Code changes to `lib/agent/` (the gap is FUB-side configuration, not code)
- Brain-app interested-inbound surfacing in queue UI (still session 6+ TODO)
- Week 1 metrics dashboard (still session 6+ TODO)

Current state:
- `agent_enabled` still **false**. No outbound messages have ever been sent.
- `daily_send_cap` = 10
- 4 pending_review drafts from 2026-05-07 (Anthony Scott, Jay Miller, Seanna Mackey, Gerard Marmo). Keep or discard depending on whether Brian wants the smoke test to use a fresh draft or one of these.
- 1 blocked draft (id=6, John Miller). Should be cleaned up; was Brian's earlier session 4 manual approve attempt that the gate correctly refused.
- 0 optouts, 0 cooldowns
- FUB email shell artifact ready to paste; iMessage path still untouched

Open items for next session:
- Brian or Viktor opens FUB Automation 2.0 → Send Email step → selects template **1156** (`Agent Draft Outreach (System)`), sets type to plaintext, enables the automation
- Brian runs the manual dry-test flow at the bottom of `scripts/fub-email-shell.md` (subject "Test from agent", body with `\n\n` paragraph break) to verify plaintext newlines render correctly
- If dry test passes, run `scripts/session-5-smoke-test.sql` Section A→B→C with Brian as the test recipient (fub_person_id 27764)
- If smoke test passes, flip `agent_enabled = true`, watch the daily-flush cron handle real outbound, ramp `daily_send_cap` from 10 to 25 over week 1
- Build iMessage shell (`scripts/fub-imessage-shell.md`) when LoopMessage work resumes
- Build interested-inbound surfacing in queue UI
- Build week-1 metrics dashboard at `/agent/metrics`
- Set FUB UI visibility on custom fields 157/158/159 to admins-only (carryover from session 4)

## Session 5 — 2026-05-07

What shipped:
- **Timezone helper** at `lib/agent/timezone.ts`. Single source of truth for ET start-of-day math (DST-aware via `Intl.DateTimeFormat` with `America/New_York`). Approve route, queue header endpoint, and the new daily-flush cron all import from it; the previous duplicated copies in approve/header are gone.
- **Daily-cap flush cron** at `app/api/cron/agent-daily-flush/route.ts`. Reads `agent_enabled` (skips and logs `cron_skipped` when false), reads `daily_send_cap`, pulls approved-but-unpushed drafts oldest first (capped at 100/run), checks today's pushed count before each `pushDraftToFUB` call, defers when cap is hit. Logs `cron_flush_complete` summary. Defensive try/catch — always returns 200 so Vercel does not retry.
- **vercel.json schedule** `"1 5 * * *"` UTC = 12:01 AM EST or 1:01 AM EDT. The DST-drift choice is documented in the cron route's docstring (vercel.json is strict JSON so the comment lives there). Cron only needs to fire after the ET day rolls over; an hour of drift is fine.
- **Inbound classifier wired to side effects** in `app/api/agent/inbound/route.ts`:
  - `opt_out` → insert `fub_agent_lead_optouts` (using existing schema: `optout_source`, `detected_channel`, `optout_message`, `optout_confidence`) AND flip `fub_agent_leads.is_eligible = false` with `eligibility_reason = 'inbound_optout'`. Logs `inbound_optout`.
  - `hostile` → 90-day cooldown (`reason='hostile_reply'`). Logs `inbound_hostile_cooldown`.
  - `not_now` → 60-day cooldown (`reason='not_now_reply'`). Logs `inbound_not_now_cooldown`.
  - `interested` → log only (`inbound_interested`). Future surfacing in queue UI is a session 6 item.
  - `irrelevant` → log only (`inbound_irrelevant`).
  - Each branch try/catch'd. Route always returns 200 (FUB retries on non-2xx).
- **Cooldown reason enum extended** via `supabase/migrations/20260507_fub_agent_session_5.sql`. Added `hostile_reply` and `not_now_reply` to the `fub_agent_lead_cooldowns_reason_check` constraint. Applied via Supabase MCP.
- **draftGenerator optout skip**: belt+suspenders check before the existing cooldown skip. The inbound classifier flips `is_eligible=false` already, but defending against any path where an optout row exists without the eligibility flag flipped is cheap and prevents accidental sends.
- **Smoke test SQL artifact** at `scripts/session-5-smoke-test.sql` + walkthrough at `scripts/README.md`. Three-section flow: pre-flight verification, manual draft insert (temporarily removes Brian's exclusion row, sets is_eligible=true, inserts iMessage draft for fub_person_id=27764), post-test cleanup (restores exclusion, deletes the test draft, inspects audit log). Brian runs manually after Viktor's Automations 2.0 are live.

Schema-realism deviations from the prompt:
- The prompt's `fub_agent_optouts` table already exists as `fub_agent_lead_optouts` (session 1, with the `_lead_` infix). Used the existing table; no new migration. Existing columns (`optout_source`, `optout_message`, `optout_confidence`, `detected_channel`, `notes`) are sufficient.
- The prompt asked for cooldown ordering by `reviewed_at` in the cron — that column does not exist on `fub_agent_message_drafts`. Used `approved_at` (which does exist).

Current state at end of session 5:
- `agent_enabled` still **false**. No outbound messages have ever been sent.
- `daily_send_cap` = 10
- Cooldown reason enum: `voice_off | wrong_lead_type | wrong_signal_read | low_quality | other | hostile_reply | not_now_reply`
- `fub_agent_lead_cooldowns` empty
- `fub_agent_lead_optouts` empty (until inbound classifier sees real traffic)
- Cron `agent-daily-flush` registered in vercel.json (no-op until `agent_enabled = true`)
- Smoke test artifact lives in `scripts/`; not yet executed
- Brian's FUB person id = **27764**, in `fub_agent_lead_exclusions` (pond:brian_excluded_contacts), phone 8039023700

## Session 4 — 2026-05-07

What shipped:
- **FUB pusher** (`lib/agent/fubPusher.ts`) — replaces the session-2 staged_fub_action stub. Writes draft body (and subject for email) into channel-specific custom fields, merges-on the ready tag, stamps `fub_pushed_at` for idempotency, never throws.
- **Reject taxonomy with 5-class enum** (`voice_off`, `wrong_lead_type`, `wrong_signal_read`, `low_quality`, `other`) plus optional free-form notes. Per-reason side effects: voice_off logs only; wrong_lead_type resets `fub_agent_leads.lead_type='unknown'` and clears `lead_type_classified_at`; wrong_signal_read inserts 30-day cooldown; low_quality + other insert 14-day cooldowns. Each side effect wrapped in try/catch so a secondary failure does not undo the rejection.
- **Cooldown table** `fub_agent_lead_cooldowns` (id uuid, fub_person_id bigint, cooldown_until timestamptz, reason check-constrained, created_by_draft_id bigint FK to drafts on delete set null). RLS enabled, no policies — service-role-only same as drafts table. `lib/agent/draftGenerator.ts` queries this and skips leads with active cooldown.
- **Daily cap default lowered to 10** (was 50). Approve route checks today's pushed count (ET start-of-day) before calling pushDraftToFUB; if at cap, draft stays approved-but-unpushed. Midnight-ET flush cron is a session-5 TODO.
- **Queue header** with master kill switch (Agent ON/OFF), today's pushed count, inline daily-cap editor. Reads from `/api/agent/config/header`. Toggle endpoint `/api/agent/config/toggle` flips `agent_enabled`. Cap endpoint `/api/agent/config/cap` validates 1..1000.
- **Inbound classifier stub** at `app/api/agent/inbound/route.ts`. Shared-secret auth via `FUB_AGENT_INBOUND_SECRET` env var (returns 503 if unset, never auto-generates). Haiku 5-class enum: opt_out, interested, not_now, hostile, irrelevant. Logs only — no optout/cooldown writes (session-5 work). Always returns 200 to avoid FUB retry loops.
- **Three FUB custom fields** created via `POST /v1/customFields`:
  - id 157 — Agent Draft Email Body — API name `customAgentDraftEmailBody`
  - id 158 — Agent Draft Email Subject — API name `customAgentDraftEmailSubject`
  - id 159 — Agent Draft iMessage Body — API name `customAgentDraftIMessageBody`
  - All `text` type (FUB does not differentiate short/long via API). Visibility (admins-only) is a FUB UI setting that must be toggled manually in FUB.
- **Two FUB tags** scheduled to be created on first apply (FUB does not expose `/v1/tags` to our API key): `agent_draft_email_ready`, `agent_draft_imessage_ready`.
- **Single source of truth** for FUB-side identifiers in `lib/agent/fubAgentConstants.ts`. Pusher and inbound stub both import from here.
- **Schema additions** in `supabase/migrations/20260507_fub_agent_session_4.sql` (applied via Supabase MCP): cooldowns table + index, drafts.fub_pushed_at + reject_reason + reject_notes, drafts reject_reason check constraint, drafts.fub_pushed_at partial index, daily_send_cap value updated to 10.

### Learnings captured in session 4

1. **FUB tag list endpoint not accessible to standard API key.** The `/v1/peopleTags` endpoint (or equivalent listing endpoint) returns 403 or 404 with the API key in env. Workaround used in session 4: tags are implicit-on-first-apply. The pusher's POST to apply a tag to a person creates the tag if it does not exist. This is fine for known tag names hardcoded in `fubAgentConstants.ts`. Becomes a problem if we ever need to enumerate tags programmatically — we would need a higher-scoped API key or a scraping fallback.

2. **Schema realism beats spec when adding new tables that reference existing ones.** Session 4 spec said `fub_person_id` should be text and FK should be uuid. Existing schema has `fub_person_id` as bigint and `drafts.id` as uuid. Claude Code correctly matched the existing types rather than introducing a type mismatch that would have required casts everywhere. Principle for future sessions: "match what's there" beats "follow the spec literally" when the two conflict on type or naming conventions.



## What this is

An AI-augmented lead nurture layer for HGPG. Scores Brian's eligible FUB leads using a hybrid rules + LLM intent engine, drafts personalized messages, and uses FUB Automations 2.0 (with Sendblue for iMessage) to do the actual sending. The agent does the *thinking* (who, what, when). FUB does the *sending*.

## Architecture (v1)

- **Embedded in TM repo** (`HGPG1/hgpg-transaction-manager`) under `lib/agent/` and `app/agent/`
- **Supabase:** `ioypqogunwsoucgsnmla` (HGPG Core) - 8 `fub_agent_*` tables, RLS-locked to service role
- **Vercel:** TM project (`prj_oLWVcE4J1UKzJtmggoQCOW35LUhy`) - adds cron + agent routes
- **LLM:** Anthropic Haiku 4.5 via separate `AGENT_ANTHROPIC_API_KEY`
- **Auth:** Reuses TM team_members session; `/agent/*` gated to `brian@homegrownpropertygroup.com`
- **Cron:** `/api/agent/cron` daily at 03:00 ET - no-op while `agent_enabled = false`
- **Manual triggers:** `/api/agent/sync` and `/api/agent/score` accept `Bearer $CRON_SECRET`
  - `sync` accepts `{ source: 'brian_assigned' | 'pond_10' | 'pond_13' | 'pond_14' | 'pond_16', skipSeed?: bool }` to fit 300s Vercel timeout
  - `score` accepts `{ limit, concurrency, rescoreOlderThanHours, orderBy }` and uses `Promise.allSettled` for concurrent batch processing

## Tables

| Table | Purpose |
|---|---|
| `fub_agent_leads` | Denormalized FUB person snapshots with eligibility flag |
| `fub_agent_lead_scores` | Score history (one row per scoring run per lead) |
| `fub_agent_message_templates` | Library with `{{slot}}` markers (14 templates seeded as of session 3) |
| `fub_agent_message_drafts` | Pending/approved/sent/discarded drafts |
| `fub_agent_lead_optouts` | Source of truth for opt-outs (wired in session 5) |
| `fub_agent_lead_exclusions` | Hard exclusions (sphere, opt-outs, cash investors) |
| `fub_agent_lead_cooldowns` | Per-lead temporary cooldown windows (session 4) |
| `fub_agent_config` | KV thresholds + master kill switch |
| `fub_agent_log` | Full audit trail |

## Pilot scope

**Brian-only v1.** Other agents come in v2. Buyer-only v1. Sellers in v2.

Lead universe pulled from FUB:
- `assignedUserId = 1` (Brian-assigned, ~5,340 with no pond)
- Pond 10 - Sandy Leads (1,134)
- Pond 13 - New Buyer Leads (4)
- Pond 14 - NYC Relocation (0)
- Pond 16 - IDX Broker Legacy Re-engagement (1,434)

## Eligibility rules (calibrated 2026-05-06 from real FUB data)

**Hard exclusions (`fub_agent_lead_exclusions`):**
- Pond 9 - Brian Excluded Contacts (2,834)
- Pond 11 - Cash Investors (119)
- Source `SPHERE` (any case)
- Tags: `Sphere`, `Brian's Contacts`
- Tags: `Do Not Contact`, `DO_NOT_CALL`, `Unsubscribed`, `Bounced`, `WRONG_NUMBER`
- Tags: `Cash Offers`, `Cash Buyer Pool`
- Stages: `Past Client`, `Active Client`, `Sphere`, `Closed`, `Do Not Contact`

**Marked seller, skipped from buyer-v1 (`lead_type = 'seller'`):**
- Stage matches `expired|withdrawn|fsbo|listing|seller`
- Tags: `Seller`, `Seller Handraiser email`, `seller_close`, `Listing Lead`, `Listing Appointment`, `Listing Appt`
- Tags: `Expired`, `Withdrawn`, `FSBO`
- Tags: `PreForeclousre` (typo from FUB), `PreForeclosure`
- Tags: `sell_before_buy=Yes`, `I_NEED_TO_SELL_BEFORE_I_CAN_BUY`

## Scoring engine

**Combined score 0-100:**
- `rules_score` (0-60): stage + tags + activity recency + pond + source weights
- `llm_score` (0-40): Haiku reads last 10-20 notes/emails/texts, returns structured intent + signals + friction + confidence
- `combined = rules + (llm_score × confidence if confidence ≥ floor)`

**Tiering (calibrated 2026-05-06 to stale-pool reality):**
- `hot` ≥ 40 (locked - real hot leads come from new inflows, not stale data)
- `warm` ≥ 30
- `cold` < 30
- `llm_confidence_floor` = 0.4

## Decisions locked

- **Sendblue is the messaging layer**, accessed via FUB Automations 2.0 - we never call Sendblue directly
- **FUB Automations 2.0 handles trigger/delay/branching natively** - we feed it the right people with the right tags and message bodies pre-staged
- **Hot leads get fresh LLM drafts; cold/warm leads get template-fill** with 2-3 LLM-filled slots
- **Hybrid approval gate** - autonomous below threshold, approval above (gate stays closed during build until calibration is done)
- **Compliance: own opt-outs in `fub_agent_lead_optouts` not FUB tags** (FUB tags unreliable)
- **Surface lives in TM** (not subdomain) - re-evaluate for v2 if multi-agent expansion warrants extraction to `agent.homegrownpropertygroup.com`
- **One voice file (`brian-voice.md`) for v1**; swap to neutral HGPG team voice in v2
- **Hot threshold stays at 40 even though no leads have crossed it.** Don't paper over reality with threshold tuning.
- **FUB Automation 2.0 email step must be plaintext, not HTML** (decision made 2026-05-08). The agent writes plaintext bodies with `\n` separators; HTML mode collapses them. See `scripts/fub-email-shell.md`.

## Session 1 (complete)

Schema, sync, scoring, and read-only dashboard live in TM.

### Final scoring snapshot (1,023 of 5,363 leads)

| Metric | Value |
|---|---|
| Total scored | 1,023 |
| Hot (≥40) | 0 |
| Warm (30-39) | 41 (4%) |
| Cold (<30) | 982 (96%) |
| Avg combined score | 22.0 |
| LLM credited (conf ≥ 0.4) | 84 (8%) |

## Session 2 (complete)

- Pulled approval-queue UI patterns from TM milestone tracker
- Built template library seed (6-10 v1 templates: warm × buyer × stage)
- Draft generator (`lib/agent/draftGenerator.ts`) - picks template, fills slots OR drafts fresh for hot tier
- Voice file: copied `brian-voice.md`
- Approval queue UI at `app/agent/queue/page.tsx`
- Outbound gate (`lib/agent/outboundGate.ts`) - checks `lead_optouts` + `lead_exclusions`
- Manual test: approve a draft -> verified FUB tag + custom field write (no actual send yet)

### Session 2 - rejection handling

When a draft is rejected:
- Marked `status='discarded'` in `fub_agent_message_drafts`
- `draft_rejected` event lands in `fub_agent_log` capturing `rejected_by` and `rejection_reason` (defaults `'manual_reject'`)
- Draft preserved in DB (body, subject, slot values, template scenario, LLM confidence, original timestamp)
- Disappears from queue UI (filtered on `status='pending_review'`)
- Lead stays eligible - rejection doesn't unclassify or exclude
- Lead becomes eligible for new draft on next `/drafts/generate` run

### Session 2 - classifier rebuild (mid-session)

- Buyer/seller classifier rebuilt to fix mis-classification of stale leads
- Verified working with seller templates end-to-end

## Session 3 (complete)

- Sendblue connection sanity (live FUB integration test)
- FUB Automations 2.0 setup for each (lead_type × stage × channel) combo
- `lib/agent/fubPusher.ts` stub - write message body to FUB custom field, apply triggering tag (replaced in session 4)
- Inbound parser at `app/api/agent/inbound/route.ts` - regex + LLM opt-out detection (full wiring in session 5)
- Open the autonomous gate: `agent_enabled = true`, `auto_below_threshold = true`. DEFERRED until smoke test passes.

## Configuration reference

`fub_agent_config` keys (current values as of 2026-05-08):

| Key | Value | Notes |
|---|---|---|
| `agent_enabled` | `false` | Master kill switch - cron is no-op while false |
| `auto_below_threshold` | `false` | Approval-only mode during build |
| `hot_threshold` | `40` | Score ≥ this requires human approval - locked |
| `warm_threshold` | `30` | Score ≥ this is eligible to send |
| `llm_confidence_floor` | `0.4` | LLM intent score not credited below this |
| `daily_send_cap` | `10` | Max messages queued in 24h window |
| `voice_file` | `"brian-voice.md"` | Voice profile for LLM drafting |
| `scoring_version` | `"v1"` | Current scoring engine version |

## Key learnings

- Vercel function timeout is 300s - per-source sync route with optional `source` param solves it
- FUB API is sequential bottleneck - `Promise.all` parallelization + `concurrency: 5` got per-lead time from ~6s to ~1.4s
- Tag inventory matters - real FUB tag list revealed `Expired`, `PreForeclousre` (typo), `Cash Offers`, `Brian's Contacts`, etc. Calibration cut eligible from 7,891 to 5,363
- Sphere leakage was a real risk - top-scored lead in first batch was an active sphere client (Neal Kelley) - fixed before any messages went out
- The score is a ranking, not a classification - with max=39 and threshold=40, no "hot" leads exist in stale pool
- DB-level filtering matters for batch processing - `.not('fub_person_id', 'in', recently_scored_ids)` beats in-memory filtering
- The LLM intent reads are genuinely useful beyond just scoring - `specific_friction` field is a brief on each lead
- No-history short-circuit saves money - leads with zero notes/emails/texts get synthetic low-confidence response without firing Haiku
- Plaintext bodies vs HTML email rendering is the most likely smoke-test failure mode (session 6 finding) - if FUB Automation 2.0 sends as HTML, the agent's `\n\n` paragraph breaks collapse into a run-on. Always send plaintext.
