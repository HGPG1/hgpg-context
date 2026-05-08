<!-- Last Updated: 2026-05-08 -->

# TC Concierge

- **Status:** 🟢 Live, Don running real deals
- **URL:** tc.homegrownpropertygroup.com
- **Stack:** Google Apps Script (frontend), Vercel-hosted LLM proxy
- **Supabase:** `ioypqogunwsoucgsnmla` (HGPG Core) — `transaction_emails` table

## Purpose

Automated email ingestion + classification system that watches Don's transaction inboxes (`closings@` and `brian@`), extracts deal data from inbound emails (PDFs, forwarded threads, attorney correspondence), and surfaces it as staged transactions in TM at `/transactions/staged`.

Don's primary workflow is: emails come in → tc-concierge classifies + extracts → Don confirms or edits in TM → transaction goes live with parties + addresses + dates pre-populated.

## Architecture

### Apps Script (in both `closings@` and `brian@` Gmail)

Same `Code.gs` deployed to both inboxes. Hourly trigger, business-hours gated (Mon-Fri 8am-6pm ET) to prevent after-hours API calls.

- Lookback: `newer_than:7d`
- Per-run cap: `MAX_PER_RUN=50`
- Retry cap: 3 per email
- LLM via `LLM_PROXY_URL` Vercel proxy (model: `claude-sonnet-4-6`)

### MIME walker

Custom MIME parser to rescue forwarded PDFs that Gmail's standard attachment API misses. Critical for forwarded attorney emails where the PDF is nested two layers deep.

### Skip-memo pattern

Non-transactional senders (newsletters, GitHub, MLSGrid, Stripe) get skip-memo'd in `transaction_email_match_skipped` table so they don't burn Supabase egress on every cron tick. This pattern dropped round-trips from 120+ to 5-10 per cron run.

`MatcherSnapshot` pattern: fetch active transactions once per tick, not per email.

### NO-PDF emails

Even emails without PDFs get a full LLM call by design — the body often carries address/parties/attorneys that surface useful matches at `/transactions/staged`.

## Current state

- ~2,300+ rows in `transaction_emails`, all `status='ok'`
- Don is daily-using it on real deals
- API key issue from earlier: resolved

## Key constraints

- **Business hours gate** is intentional — avoids late-night Apps Script triggers running into Anthropic rate limits during off-hours when nobody's checking output
- **Hourly cadence** is the sweet spot — more frequent = wastes API calls on the same emails, less frequent = Don waits too long for staged transactions to appear

## Decisions made

- **tc-concierge absorption into TM: DEFERRED.** Higher lift than expected. Revisit only if tc-concierge breaks. Current Apps Script + Vercel proxy architecture is stable.

## Pickup notes

- If tc-concierge stops surfacing emails: first check Apps Script execution log in the affected Gmail account. Most outages are quota/auth, not logic
- LLM proxy lives at separate Vercel project — needs `x-proxy-secret` header
- The MIME walker is brittle and was tuned to specific attorney email formats — if a new attorney starts forwarding PDFs differently, expect a few rounds of tuning
- Don's feedback for tc-concierge comes through TM's built-in feedback widget (same channel as TM feedback)
