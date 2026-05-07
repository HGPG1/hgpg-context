# FUB AI Agent

Last updated: 2026-05-06 (session 1 complete)

## What this is

An AI-augmented lead nurture layer for HGPG. Scores Brian's eligible FUB leads using a hybrid rules + LLM intent engine, drafts personalized messages, and uses FUB Automations 2.0 (with Sendblue for iMessage) to do the actual sending. The agent does the *thinking* (who, what, when). FUB does the *sending*.

**Status:** Session 1 shipped. Schema, sync, scoring, and read-only dashboard live in TM. 1,023 of 5,363 eligible leads scored. No outbound messaging yet — kill switch `agent_enabled = false`.

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

## Pilot scope

**Brian-only v1.** Other agents come in v2. Buyer-only v1. Sellers in v2.

Lead universe pulled from FUB:
- `assignedUserId = 1` (Brian-assigned, ~5,340 with no pond)
- Pond 10 — Sandy Leads (1,134)
- Pond 13 — New Buyer Leads (4)
- Pond 14 — NYC Relocation (0)
- Pond 16 — IDX Broker Legacy Re-engagement (1,434)

## Eligibility rules (calibrated 2026-05-06 from real FUB data)

**Hard exclusions (`fub_agent_lead_exclusions`):**
- Pond 9 — Brian Excluded Contacts (2,834)
- Pond 11 — Cash Investors (119)
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

**Eligibility formula:**
```
eligible = (assignedUserId = 1 OR pondId IN [10, 13, 14, 16])
       AND pondId NOT IN [9, 11, 5, 15]
       AND no exclusion tag
       AND source != 'SPHERE'
       AND stage not in exclusion stages
       AND lead_type IN ('buyer', 'unknown')
       AND id NOT IN lead_optouts
       AND has phone OR email
```

## Current data state (post-calibration)

| Metric | Count |
|---|---|
| Total leads synced | 10,865 |
| **Buyer prospects (eligible)** | **5,363** |
| Sellers (v2 holding) | 1,631 |
| Hard exclusions | 3,972 |

Note: `C - Cold 6+ Months` stage (6 leads) is NOT excluded — those leads are valid re-engagement targets, the whole point of the project.

## Scoring engine

**Combined score 0-100:**
- `rules_score` (0-60): stage + tags + activity recency + pond + source weights
- `llm_score` (0-40): Haiku reads last 10-20 notes/emails/texts, returns structured intent + signals + friction + confidence
- `combined = rules + (llm_score × confidence if confidence ≥ floor)`

**Tiering (calibrated 2026-05-06 to stale-pool reality):**
- `hot` ≥ 40 (locked at this value — see "decisions" below)
- `warm` ≥ 30 (was 60, lowered for stale pool)
- `cold` < 30
- `llm_confidence_floor` = 0.4 (was 0.6 — too strict for sparse-history leads)

**Performance:**
- ~1.4-1.5s per lead with `concurrency: 5`
- 200 leads scored per ~5min batch
- ~$0.0028 per lead in Haiku tokens

## v1 Files (committed to TM main)

```
lib/agent/
  types.ts              shared row shapes + pond/user constants
  log.ts                fub_agent_log writer
  fubClient.ts          paginated FUB API (Basic auth, retry/429), notes/emails/texts
  anthropicClient.ts    Anthropic Messages API wrapper, JSON-parse helper
  sync.ts               exclusion seeding, eligibility eval, per-source sync
  scoring.ts            rules + LLM intent, concurrent batched scoring with DB-level filtering

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
- `7720324` — filter recently-scored at DB level (was getting stuck on top-by-activity slice)

## Decisions locked

- **Sendblue is the messaging layer**, accessed via FUB Automations 2.0 — we never call Sendblue directly
- **FUB Automations 2.0 handles trigger/delay/branching natively** — we feed it the right people with the right tags and message bodies pre-staged
- **Hot leads get fresh LLM drafts; cold/warm leads get template-fill** with 2-3 LLM-filled slots
- **Hybrid approval gate** — autonomous below threshold, approval above (gate stays closed during build until calibration is done)
- **Compliance: own opt-outs in `fub_agent_lead_optouts` not FUB tags** (FUB tags unreliable)
- **Surface lives in TM** (not subdomain) — re-evaluate for v2 if multi-agent expansion warrants extraction to `agent.homegrownpropertygroup.com`
- **One voice file (`brian-voice.md`) for v1**; swap to neutral HGPG team voice in v2
- **Hot threshold stays at 40 even though no leads have crossed it.** Real hot leads come from new inflows (calls, fresh inquiries, contact form submits), not stale data. Don't paper over reality with threshold tuning. Revisit only after weeks of operational data.

## Session 1 Results

### Final scoring snapshot (1,023 of 5,363 leads, 19% sample)

| Metric | Value |
|---|---|
| Total scored | 1,023 |
| Hot (≥40) | 0 |
| Warm (30-39) | 41 (4%) |
| Cold (<30) | 982 (96%) |
| Avg combined score | 22.0 |
| Avg rules score | 21.5 |
| Avg LLM raw score | 7.5 |
| Avg LLM confidence | 0.32 |
| LLM credited (conf ≥ 0.4) | 84 (8%) |
| Score range | 12-39 |
| P50 / P90 / P95 / P99 | 20 / 25 / 28 / 32 |
| Estimated Haiku spend | ~$5 |

### Top 10 warm leads found

| Score | Name | Source | LLM-detected friction |
|---|---|---|---|
| 39 | John Miller | Direct Website | (none surfaced) |
| 37 | Gerard Marmo | Qazzoo | (none surfaced) |
| 37 | Seanna Mackey | Qazzoo | $5-20k cash vs $150k+ budget — financing gap or seller concession need |
| 35 | Anthony Scott | my +plus leads | Investor/buyer dependency — flipper, not end-user |
| 35 | Phuong Pham | Ylopo | No direct comm history; unclear sell intent |
| 35 | Jay Miller | HomeLight | No response to outreach since April |
| 35 | Tamela Karnazes | Ylopo | Price band mismatch — tagged $995k, browsing $658k |
| 34 | Jesse Hernandez | Ylopo | Timeline unclear — May 2026 activity, Sept 2024 notes |
| 34 | Sam Russell | Ylopo | (none surfaced) |
| 34 | Jason Smith | Lead Import | Contingent — under contract, needs to close before next |

### Earlier batch standout (caught before sphere fix): Cynthia Patterson (35)
> "Returned to view listings after 269 days of inactivity on 2025-12-29"
> "Viewed 10513 Orchid Hill Lane 3 times in late January 2026"
> Real reactivation pattern — exactly what re-engagement should catch

## Key learnings (session 1)

- **Vercel function timeout is 300s** — full multi-source sync of 13k leads exceeds it. Solution: per-source sync route with optional `source` param.
- **FUB API is sequential bottleneck** for scoring — each lead needs notes + emails + texts. `Promise.all` parallelization + `concurrency: 5` on the lead loop got per-lead time from ~6s to ~1.4s.
- **Original thresholds (hot=85, warm=60) were way too high for stale-pool reality.** Recalibrated to 40/30 once we saw real distribution. Hot threshold STAYS at 40 — see decisions.
- **LLM confidence floor of 0.6 was too strict** for sparse-history leads. Dropped to 0.4 — still rejects hallucinated reads but credits reasonable inferences.
- **Tag inventory matters.** First seller filter was a guess; real FUB tag list revealed `Expired`, `PreForeclousre` (typo), `Cash Offers`, `Brian's Contacts`, etc. Calibration cut eligible from 7,891 to 5,363 — 2,528 leads removed that would have wasted scoring spend or been embarrassing to message.
- **Sphere leakage was a real risk.** Source `SPHERE` and tags `Sphere` / `Brian's Contacts` slipped through the first eligibility filter. Top-scored lead in first batch was an active sphere client (Neal Kelley) — fixed before any messages went out.
- **Stage-based exclusions matter too.** `Past Client`, `Active Client`, `Sphere` stage all leaked through tag-only filtering. Added stage-based check.
- **Re-classification via SQL is fast** when calibration changes — updated 1,631 sellers + added 162 sphere/stage exclusions in seconds without re-syncing FUB.
- **The score is a ranking, not a classification.** With max=39 and threshold=40, no "hot" leads exist in the stale pool — but the warm queue (41 leads) is sorted by genuine signal. Top of list is more interesting than bottom. That's the actionable output.
- **DB-level filtering matters for batch processing.** Original code pulled `limit*3` candidates and filtered in memory — got stuck on top-by-activity slice after 2 batches. Switched to `.not('fub_person_id', 'in', recently_scored_ids)` and the loop chewed through the pool correctly.
- **The LLM intent reads are genuinely useful.** Beyond just scoring, the `specific_friction` field is a brief on each lead — "down payment gap," "investor not end-user," "price band mismatch," "contingent on current closing." That's actionable context for the eventual outreach.
- **No-history short-circuit saves money.** Leads with zero notes/emails/texts get a synthetic low-confidence response without firing Haiku.

## Next session priorities

**Session 2 (~3 hr) — Templates + draft generation + approval queue UI:**
1. Pull approval-queue UI patterns from existing TM milestone tracker
2. Build template library seed (6-10 v1 templates: warm × buyer × stage)
3. Draft generator (`lib/agent/draftGenerator.ts`) — picks template, fills slots OR drafts fresh for hot tier (when hot leads exist)
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

- **4,340 leads still unscored** — will be scored by daily cron once `agent_enabled = true`, or by ad-hoc batches before then
- Templates list is empty — session 2
- Approval queue UI doesn't exist yet — session 2
- Cron is configured but no-op (kill switch off) — session 3
- Sandy Leads pond name lookup not displayed in dashboard (shows `pond_10` instead of "Sandy Leads") — cosmetic, fix in session 2
- ~127 wasted scores from pre-calibration runs (~$2.50 in Haiku) — acceptable lesson cost
- Dashboard top 50 currently includes excluded leads from before sphere/stage cleanup — fix queued in `dashboard-eligibility-fix.md`

## Configuration reference

`fub_agent_config` keys (current values as of 2026-05-06):

| Key | Value | Notes |
|---|---|---|
| `agent_enabled` | `false` | Master kill switch — cron is no-op while false |
| `auto_below_threshold` | `false` | Approval-only mode during build |
| `hot_threshold` | `40` | Score ≥ this requires human approval — locked, see decisions |
| `warm_threshold` | `30` | Score ≥ this is eligible to send |
| `llm_confidence_floor` | `0.4` | LLM intent score not credited below this |
| `daily_send_cap` | `50` | Max messages queued in 24h window |
| `voice_file` | `"brian-voice.md"` | Voice profile for LLM drafting |
| `scoring_version` | `"v1"` | Current scoring engine version |

## Session 2 shipped 2026-05-06

Built via Claude Code in HGPG1/hgpg-transaction-manager main. Files:

- lib/agent/voices/brian-voice.md
- lib/agent/draftGenerator.ts
- lib/agent/outboundGate.ts
- app/agent/queue/page.tsx
- app/api/agent/drafts/route.ts (list)
- app/api/agent/drafts/generate/route.ts (manual trigger)
- app/api/agent/drafts/[id]/{approve,reject,edit}/route.ts
- migrations/2026-05-06-fub-agent-templates-seed.sql

Migration applied to ioypqogunwsoucgsnmla. 8 v1.warm.* templates seeded (6 email, 2 imessage, all buyer-typed).

Smoke test:
- /generate dryRun returned 5 candidates
- /generate real run created drafts 1-5, all routed to v1.warm.viewed_listing_recent.email
- Slot fills clean and on-voice (Haiku output reads like Brian, no em dashes, fallback wording correct)
- All 5 drafts rejected during review — see classifier gap below

## Session 3 priority — buyer classifier doesn't exist

Distribution check across fub_agent_leads:
- unknown: 9,234 (85%)
- seller: 1,631 (15%)
- buyer: 0

Session 1 only detects sellers. Buyers stay 'unknown' forever. This means:
- 85% of eligible leads have lead_type='unknown' which the draft generator currently treats as buyer-compatible
- John Miller (fub_person_id 23316) was the proof — score row signals listed seller_classification + avm_report_access + seller report engagement, but lead_type was 'unknown' so he got routed into a buyer template
- Failing closed on 'unknown' is not viable — would skip 85% of pool including all actual buyers

Three-part fix for session 3:

1. Build buyer classifier mirroring seller logic. Batch-classify 9,234 unknowns. Expected: ~70% become buyer, ~25% stay unknown, ~5% flip to seller.
2. Patch draftGenerator.ts template selector to read score-row signals as authoritative override. If signals contain seller_classification or similar, never route to buyer template even when lead_type='unknown'. Catches the John Miller case directly.
3. Seed seller templates: 6 v1.warm.*.seller variants (seller_report_recent, avm_recent, long_dormant_seller, listing_thinking, no_response_recent_seller, generic_seller_reengagement).
4. Backfill audit. Sample 50 reclassified leads, eyeball buyer/seller split, verify before re-enabling /generate.

Templates and pipeline are otherwise solid. Voice file is good. Slot fills are clean. Gate works. Approve/reject/edit handlers all functional. Queue UI renders correctly.

Gate stayed closed throughout (agent_enabled=false). No outbound sent. No FUB writes. Production-safe to leave as-is.
