<!-- Last Updated: 2026-05-07 -->

# FUB AI Agent

- **Status:** 🟡 Sessions 1-4 shipped, gate still closed (no outbound messaging yet)

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

Current state:
- `agent_enabled` still **false**. No outbound messages have ever been sent.
- `daily_send_cap` = 10
- 159 confidently classified leads (count from session 3.5)
- 3 FUB custom fields configured (admin visibility pending UI step)
- 2 FUB tags scheduled to appear on first apply

Open items for session 5:
- Confirm Brian's own FUB person ID
- Wire FUB Automations 2.0 triggers for `agent_draft_email_ready` and `agent_draft_imessage_ready` tags
- Brian-as-client smoke test: flip `agent_enabled` true on a single draft, verify FUB roundtrip, flip back to false
- Build daily-cap flush cron at midnight ET that re-pushes approved-but-unpushed drafts
- Wire the inbound classifier to `fub_agent_lead_optouts` (when class=opt_out) and to `fub_agent_lead_cooldowns` (when class=not_now or hostile)
- Set FUB UI visibility on the three custom fields to admins-only



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
| `fub_agent_message_templates` | Library with `{{slot}}` markers (session 2) |
| `fub_agent_message_drafts` | Pending/approved/sent/discarded drafts (session 2) |
| `fub_agent_lead_optouts` | Source of truth for opt-outs (session 3) |
| `fub_agent_lead_exclusions` | Hard exclusions (sphere, opt-outs, cash investors) |
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

## Session 3 (in flight)

- Sendblue connection sanity (live FUB integration test)
- FUB Automations 2.0 setup for each (lead_type × stage × channel) combo
- `lib/agent/fubPusher.ts` - write message body to FUB custom field, apply triggering tag
- Inbound parser at `app/api/agent/sendblue-inbound/route.ts` - regex + LLM opt-out detection
- Open the autonomous gate: `agent_enabled = true`, `auto_below_threshold = true`

## Session 4 (planned, not started)

- **Reject reason taxonomy:** if you reject with reason `'voice_off'`, that signals the template needs work; if `'wrong_lead'`, that lead gets excluded from agent forever
- **Cooldown:** lead can't be re-drafted within N days of a rejection
- **Hard exclusion on N rejections:** auto-add to `fub_agent_lead_exclusions` after 2-3 rejects

## Configuration reference

`fub_agent_config` keys (current values as of 2026-05-07):

| Key | Value | Notes |
|---|---|---|
| `agent_enabled` | `false` | Master kill switch - cron is no-op while false |
| `auto_below_threshold` | `false` | Approval-only mode during build |
| `hot_threshold` | `40` | Score ≥ this requires human approval - locked |
| `warm_threshold` | `30` | Score ≥ this is eligible to send |
| `llm_confidence_floor` | `0.4` | LLM intent score not credited below this |
| `daily_send_cap` | `50` | Max messages queued in 24h window |
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
