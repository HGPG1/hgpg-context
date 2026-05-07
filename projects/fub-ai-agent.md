<!-- Last Updated: 2026-05-07 -->

# FUB AI Agent

- **Status:** 🟡 Sessions 1-3 in flight, gate still closed (no outbound messaging yet)

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
