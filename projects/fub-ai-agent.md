# FUB AI Agent

Last updated: 2026-05-06 (session 1 in progress)

## What this is

An AI-augmented lead nurture layer for HGPG. Scores Brian's eligible FUB leads using a hybrid rules + LLM intent engine, drafts personalized messages, and uses FUB Automations 2.0 (with Sendblue for iMessage) to do the actual sending. The agent does the *thinking* (who, what, when). FUB does the *sending*.

**Status:** Session 1 in progress. Schema, sync, and scoring are live in TM. Read-only dashboard at `closings.homegrownpropertygroup.com/agent` (Brian-only). No outbound messaging yet — kill switch `agent_enabled = false` in `fub_agent_config`.

## Architecture (v1)

- **Embedded in TM repo** (`HGPG1/hgpg-transaction-manager`) under `lib/agent/` and `app/agent/`
- **Supabase:** `ioypqogunwsoucgsnmla` — 8 `fub_agent_*` tables, RLS-locked to service role
- **Vercel:** TM project (`prj_oLWVcE4J1UKzJtmggoQCOW35LUhy`) — adds cron + agent routes
- **LLM:** Anthropic Haiku 4.5 via separate `AGENT_ANTHROPIC_API_KEY`
- **Auth:** Reuses TM team_members session; `/agent/*` gated to `brian@homegrownpropertygroup.com`
- **Cron:** `/api/agent/cron` daily at 03:00 ET — no-op while `agent_enabled = false`
- **Manual triggers:** `/api/agent/sync` and `/api/agent/score` accept `Bearer $CRON_SECRET`
  - `sync` accepts `{ source: 'brian_assigned' | 'pond_10' | 'pond_13' | 'pond_14' | 'pond_16', skipSeed?: bool }` to fit 300s Vercel timeout
  - `score` accepts `{ limit, concurrency, rescoreOlderThanHours, orderBy }` and uses `Promise.allSettled` for concurrent batch processing

## Tables

| Table | Purpose |
|---|---|
| `fub_agent_leads` | Denormalized FUB person snapshots with eligibility flag |
| `fub_agent_lead_scores` | Score history (one row per scoring run per lead) |
| `fub_agent_message_templates` | Library with `{{slot}}` markers (session 2) |
| `fub_agent_message_drafts` | Pending/approved/sent drafts (session 2) |
| `fub_agent_lead_optouts` | Source of truth for opt-outs (session 3) |
| `fub_agent_lead_exclusions` | Hard exclusions (sphere, opt-outs, cash investors) |
| `fub_agent_config` | KV thresholds + master kill switch |
| `fub_agent_log` | Full audit trail |

## Pilot scope (locked)

**Brian-only v1.** Other agents come in v2. Buyer-only v1. Sellers in v2.

Lead universe pulled from FUB:
- `assignedUserId = 1` (Brian-assigned, ~5,340 with no pond)
- Pond 10 — Sandy Leads (1,134)
- Pond 13 — New Buyer Leads (4)
- Pond 14 — NYC Relocation (0)
- Pond 16 — IDX Broker Legacy Re-engagement (1,434)

## Eligibility rules (calibrated 2026-05-06 from real FUB tag inventory)

**Hard exclusions (`fub_agent_lead_exclusions`):**
- Pond 9 — Brian Excluded Contacts (2,834)
- Pond 11 — Cash Investors (119)
- Source `SPHERE` (any case)
- Tags: `Sphere`, `Brian's Contacts`
- Tags: `Do Not Contact`, `DO_NOT_CALL`, `Unsubscribed`, `Bounced`, `WRONG_NUMBER`
- Tags: `Cash Offers`, `Cash Buyer Pool`

**Marked seller, skipped from buyer-v1 (`lead_type = 'seller'`):**
- Stage matches `expired|withdrawn|fsbo|listing|seller`
- Tags: `Seller`, `Seller Handraiser email`, `seller_close`, `Listing Lead`, `Listing Appointment`, `Listing Appt`
- Tags: `Expired`, `Withdrawn`, `FSBO`
- Tags: `PreForeclousre` (typo from FUB), `PreForeclosure`
- Tags: `sell_before_buy=Yes`, `I_NEED_TO_SELL_BEFORE_I_CAN_BUY`

**Eligibility formula:**
```
eligible = (assignedUserId = 1 OR pondId IN [10, 13, 14, 16])
       AND pondId NOT IN [9, 11, 5, 15]
       AND no exclusion tag
       AND source != 'SPHERE'
       AND lead_type IN ('buyer', 'unknown')
       AND id NOT IN lead_optouts
       AND has phone OR email
```

## Current data state (post-calibration)

| Metric | Count |
|---|---|
| Total leads synced | 10,865 |
| **Buyer prospects (eligible)** | **5,423** |
| Sellers (v2 holding) | 1,631 |
| Hard exclusions | 3,912 |

## Scoring engine

**Combined score 0-100:**
- `rules_score` (0-60): stage + tags + activity recency + pond + source weights
- `llm_score` (0-40): Haiku reads last 10-20 notes/emails/texts, returns structured intent + signals + friction + confidence
- `combined = rules + (llm_score × confidence if confidence ≥ floor)`

**Tiering (calibrated 2026-05-06 to stale-pool reality):**
- `hot` ≥ 40 (was 85 — too high for stale pool)
- `warm` ≥ 30 (was 60)
- `cold` < 30
- `llm_confidence_floor` = 0.4 (was 0.6 — too strict for sparse-history leads)

**Performance:**
- ~1.4-1.5s per lead with `concurrency: 5`
- 200 leads scored in ~5 minutes
- ~$0.003 per lead in Haiku tokens (~$15-18 to score full eligible pool)

## v1 Files (committed to TM main)

```
lib/agent/
  types.ts              shared row shapes + pond/user constants
  log.ts                fub_agent_log writer
  fubClient.ts          paginated FUB API (Basic auth, retry/429), notes/emails/texts
  anthropicClient.ts    Anthropic Messages API wrapper, JSON-parse helper
  sync.ts               exclusion seeding, eligibility eval, per-source sync
  scoring.ts            rules + LLM intent, concurrent batched scoring

app/api/agent/
  cron/route.ts         GET — Vercel cron entry (sync + score), respects kill switch
  sync/route.ts         POST — manual sync (Bearer CRON_SECRET, optional source param)
  score/route.ts        POST — manual scoring (Bearer CRON_SECRET)

app/agent/
  page.tsx              Brian-only read-only dashboard (distribution + top 50)
  lead/[id]/page.tsx    Brian-only lead detail (profile + score breakdown + history)

vercel.json             cron entry at 0 7 * * * (03:00 ET)
app/layout.tsx          Agent nav link (Brian-only)
components/MobileNav.tsx Agent nav link in mobile (Brian-only)
```

## Key commits (chronological)

- `4f5afa1` — initial v1 scaffold
- `c2ca4e6` — Agent nav link (desktop)
- `34fa633` — calibrated seller/exclusion tags from FUB inventory
- `8068fcc` — per-source sync to fit Vercel 300s timeout
- `ad947b3` — scoring perf: bulk query already-scored IDs, concurrent batches
- `112497a` — manual routes use CRON_SECRET (middleware compatibility)

## Decisions locked

- **Sendblue is the messaging layer**, accessed via FUB Automations 2.0 — we never call Sendblue directly
- **FUB Automations 2.0 handles trigger/delay/branching natively** — we feed it the right people with the right tags and message bodies pre-staged
- **Hot leads get fresh LLM drafts; cold/warm leads get template-fill** with 2-3 LLM-filled slots
- **Hybrid approval gate** — autonomous below threshold, approval above (gate stays closed during build until calibration is done)
- **Compliance: own opt-outs in `fub_agent_lead_optouts` not FUB tags** (FUB tags unreliable)
- **Surface lives in TM** (not subdomain) — re-evaluate for v2 if multi-agent expansion warrants extraction to `agent.homegrownpropertygroup.com`
- **One voice file (`brian-voice.md`) for v1**; swap to neutral HGPG team voice in v2

## Key learnings (session 1)

- **Vercel function timeout is 300s** — full multi-source sync of 13k leads exceeds it. Solution: per-source sync route with optional `source` param.
- **FUB API is sequential bottleneck** for scoring — each lead needs notes + emails + texts from FUB. Parallelizing with `Promise.all` saves ~30%, but real perf win came from `concurrency: 5` on the lead-scoring loop itself.
- **Original thresholds were too high for stale-pool reality.** Hot=85/Warm=60 produced zero non-cold leads. Recalibrated to 40/30 once we saw real distribution.
- **LLM confidence floor of 0.6 was too strict** for sparse-history leads. Dropped to 0.4 — still rejects hallucinated reads but credits reasonable inferences from thin data.
- **Tag inventory matters.** First seller filter was a guess; real FUB tag list revealed `Expired`, `PreForeclousre` (typo), `Cash Offers`, `Brian's Contacts`, etc. Calibration cut eligible from 7,891 to 5,423 — 2,468 leads removed that would have wasted scoring spend or been embarrassing to message.
- **Sphere leakage is a real risk.** Source `SPHERE` and tags `Sphere` / `Brian's Contacts` slipped through the first eligibility filter. Top-scored lead in first batch was an active sphere client (Neal Kelley) — fixed before any messages went out.
- **Re-classification via SQL is fast** when calibration changes — we updated 1,631 sellers + added 102 sphere exclusions in seconds without re-syncing FUB.
- **The score is a ranking, not a classification.** Even with the new thresholds, the value is "show me the top 50" not "is this lead hot." Don't over-tune absolute thresholds before seeing the full distribution.

## Current scoring run

Session 1 ramp in progress. Scoring 5,423 eligible leads in batches of 200 (~5 min each).

### Results (fill in after batches complete)

| Metric | Value |
|---|---|
| Total scored | TBD |
| Hot (≥40) | TBD |
| Warm (30-39) | TBD |
| Cold (<30) | TBD |
| Avg combined score | TBD |
| Avg rules_score | TBD |
| Avg llm_score (when credited) | TBD |
| LLM credited rate (conf ≥ 0.4) | TBD |
| Cost in Haiku tokens | TBD |

### Top hot leads observed (fill in after batches)

TBD

## Next session priorities

**Session 2 (~3 hr) — Templates + draft generation + approval queue UI:**
1. Pull approval-queue UI patterns from existing TM milestone tracker
2. Build template library seed (6-10 v1 templates: cold/warm/hot × buyer × stage)
3. Draft generator (`lib/agent/draftGenerator.ts`) — picks template, fills slots OR drafts fresh
4. Voice file: copy `brian-voice.md` from existing AI assistant build
5. Approval queue UI at `app/agent/queue/page.tsx`
6. Outbound gate (`lib/agent/outboundGate.ts`) — checks `lead_optouts` + `lead_exclusions`
7. Manual test: approve a draft → verify FUB tag + custom field write (no actual send yet)

**Session 3 (~3 hr) — FUB Automation wiring + Sendblue + inbound parser + open the gate:**
1. Sendblue connection sanity (live FUB integration test)
2. FUB Automations 2.0 setup for each (lead_type × stage × channel) combo
3. `lib/agent/fubPusher.ts` — write message body to FUB custom field, apply triggering tag
4. Inbound parser at `app/api/agent/sendblue-inbound/route.ts` — regex + LLM opt-out detection
5. Open the autonomous gate: `agent_enabled = true`, `auto_below_threshold = true`

## Known issues / followups

- Templates list is empty — session 2
- Approval queue UI doesn't exist yet — session 2
- Cron is configured but no-op (kill switch off) — session 3
- Sandy Leads pond name lookup not displayed in dashboard (shows `pond_10` instead of "Sandy Leads") — cosmetic, fix in session 2
- ~127 wasted scores from pre-calibration runs (hit Haiku ~$2.50) — acceptable lesson cost

## Configuration reference

`fub_agent_config` keys (current values as of 2026-05-06):

| Key | Value | Notes |
|---|---|---|
| `agent_enabled` | `false` | Master kill switch — cron is no-op while false |
| `auto_below_threshold` | `false` | Approval-only mode during build |
| `hot_threshold` | `40` | Score ≥ this requires human approval |
| `warm_threshold` | `30` | Score ≥ this is eligible to send |
| `llm_confidence_floor` | `0.4` | LLM intent score not credited below this |
| `daily_send_cap` | `50` | Max messages queued in 24h window |
| `voice_file` | `"brian-voice.md"` | Voice profile for LLM drafting |
| `scoring_version` | `"v1"` | Current scoring engine version |
