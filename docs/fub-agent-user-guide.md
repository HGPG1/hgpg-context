/* Last Updated: 2026-05-12 */

# FUB AI Agent — User Guide

The FUB AI Agent watches your Follow Up Boss pool, scores leads on intent, drafts outreach for the warm/hot ones, and pushes approved messages back into FUB to send via Automations 2.0. It does NOT send anything without you approving it.

This guide covers the three screens, what each panel means, and the common day-to-day workflows.

## The three screens

All three live under `/agent` and share a nav strip at the top. Click the active tab name to switch.

### `/agent` — Overview
The score distribution histogram and top-50 leads table. Use this when you want to inspect the pool: who's hot right now, what the score curve looks like, what the master switch and thresholds are set to.

This is the "what does the agent currently think about my pool" screen.

### `/agent/queue` — Approval queue
The drafts the agent has generated, waiting for you to approve or reject. This is the screen Don lives in.

Each draft shows: lead name, score, template scenario, the full message body (paragraph-by-paragraph for email), and Approve / Reject / Edit buttons. Rejection has 5 reason categories: bad_content, wrong_channel, wrong_timing, off_brand, other.

### `/agent/ops` — Operational dashboard
The live monitoring view. 7 panels covering pool health, daily cap usage, 7-day volume, queue depth, template performance, top reject reasons, and a recent activity feed.

Auto-refreshes every 30 seconds when the tab is visible. Use this for the daily glance: is the agent working? Is the queue building up? Are any templates burning through reject reasons?

## Day-to-day workflows

### Morning glance (≈30 seconds)
1. Open `/agent/ops`
2. Check the Daily Cap card: any pushes today? Anywhere near the cap?
3. Check the Queue Depth card: how many drafts are waiting?
4. Skim Recent Activity: any errors? Any new scores or pushes overnight?
5. If queue is high → open `/agent/queue` and approve / reject through them

### Approving a draft (Don's main workflow)
1. Open `/agent/queue`
2. Click a draft row to expand
3. Read the message body. Watch for: factual errors, tone misfires, anything off-brand
4. **If good** → click Approve. The draft moves to "approved" and waits for the next daily flush cron to push it to FUB
5. **If close but not right** → click Edit, fix it inline, then Approve
6. **If wrong** → click Reject and pick a reason category. The lead goes on cooldown so the agent doesn't immediately re-draft them
7. Repeat through the queue. The daily cap (currently 10) limits how many can actually push to FUB per day even if you approve more

### Reading the Volume card
The "Volume (last 7 days)" card shows 5 numbers vs the prior 7 days:
- **Generated** — drafts the agent created
- **Approved** — drafts you approved
- **Sent** — drafts that actually pushed to FUB (approved + within daily cap)
- **Rejected** — drafts you rejected
- **Blocked** — drafts that failed validation (paragraph too long, etc.) and got held for manual edit

If Sent is much lower than Approved, the daily cap is the bottleneck and you might want to raise it. If Rejected is high, something's off with the templates — check Top Reject Reasons.

### Reading Template Performance
Shows the last 30 days per template scenario:
- **Total** — drafts generated using this template
- **Appr / Rej / Block** — what happened to them
- **Approve %** — of decided drafts (approved + rejected), what % got approved

Aim for >70% approve %. Below 50% means the template is producing drafts you mostly reject — worth a rewrite or pulling it from rotation.

### Reading Pool Health
- **Hot / Warm / Cold** — leads with a current score in each tier (warm threshold = 60, hot threshold = 85)
- **Unscored** — eligible leads with no score yet
- **Draftable (hot + warm)** — the pool the agent can actually generate drafts from
- **Scored % of eligible** — how much of your FUB pool the agent has actually evaluated

The unscored count drops over time as the daily cron scores ~200 leads/day. You can also manually backfill (see "Growing the pool" below).

## Toggling the master switch

There's an `agent_enabled` config flag that blocks BOTH the daily cron AND manual approve. It's the kill switch.

- **From the queue page:** the header has a Turn on / Turn off button
- **From Supabase:** `UPDATE fub_agent_config SET value = 'false'::jsonb WHERE key = 'agent_enabled'`

When it's off, the agent generates nothing and pushes nothing. Existing drafts in the queue stay there but can't be approved. Use this if anything looks weird and you want to pause everything fast.

## Adjusting the daily send cap

The agent will only push N drafts to FUB per day even if more are approved. This is the safety throttle.

- **From the queue page:** click the daily cap number in the header to edit it inline
- **From Supabase:** `UPDATE fub_agent_config SET value = '20'::jsonb WHERE key = 'daily_send_cap'`

Current default: 10. Raise as you build confidence.

## Growing the pool (backfill more leads)

If `/agent/ops` shows lots of unscored leads, you can run a manual backfill to score more:

```
curl -sS -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $CRON_SECRET" \
  -X POST https://closings.homegrownpropertygroup.com/api/agent/score \
  -d '{"limit":200,"onlyUnscored":true}'
```

This scores 200 leads that have never been scored before. The `onlyUnscored: true` flag prevents re-scoring leads we already have.

Cost: roughly $1 per 200 leads (Haiku tokens). Tail-end pool typically converts at 2-4% warm.

## When to call for help

- A real prospect got a weird email → flip `agent_enabled = false` immediately, then DM Brian
- The queue keeps growing and nothing's pushing → check Daily Cap in /agent/ops. If pushed_today equals daily_cap, that's expected. If not, something's broken
- Lead names look wrong or empty → most likely missing FK on a related table. DM Brian, this is a query-layer issue
- Reject rate climbing on a specific template → DM Brian, time to rewrite or pull that template

## Glossary

- **Tier** — hot / warm / cold based on combined score (rules + LLM intent)
- **Template scenario** — like `v1.warm.viewed_listing_recent.email`. Templates are picked by tier + lead type + channel. Variants (.v2, .v3) are picked at random for diversity
- **Cooldown** — after rejection or hostile reply, leads can't be re-drafted for a set period
- **Daily cap** — max drafts pushed to FUB per ET day. Approved-but-unpushed drafts queue for the next day's flush cron
- **Flush cron** — runs at 12:01 AM ET. Picks up approved drafts oldest-first until cap is hit
