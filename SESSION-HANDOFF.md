<!-- Last Updated: 2026-05-09 -->

# Session Handoff

## Last session: 2026-05-09 (AM micro-task) — FUB email template 1156 created 🟡

### What got shipped
- **FUB email template created via API.** `POST /v1/templates` returned **id 1156**, name `Agent Draft Outreach (System)`, subject `[-customAgentDraftEmailSubject-]`, body `[-customAgentDraftEmailBody-]`. Created at `2026-05-09T09:51:30Z` UTC / `2026-05-09T05:51:30-04:00` ET (EDT).
- **Round-trip verified.** `GET /v1/templates/1156` returned both merge tags identical to the POST. No escaping, no mangling. The `[-fieldname-]` syntax is what FUB's template engine wants.
- `scripts/fub-email-shell.md` updated with a "Template created" footer block (id, timestamps, verification status, next-step instruction for Viktor).

### What did NOT ship
- `agent_enabled` still false. No outbound.
- Smoke-test SQL not run.
- Viktor's Automation 2.0 not yet wired to the new template.
- iMessage shell still not built.
- Brain App write. Drafts staged in `_brain-pending/` for paste-and-commit.

### Pickup notes for next session

**Sequence to unblock outbound (updated):**

1. Brian or Viktor opens FUB Automation 2.0 → "Send Email" step → template picker → selects template **id 1156** (`Agent Draft Outreach (System)`). Subject and body auto-populate from the template — no copy-paste of merge tags required.
2. Set email type to **plaintext**. Same gotcha as 2026-05-08: HTML will collapse `\n\n` paragraph breaks.
3. Add Step 2: Remove tag `agent_draft_email_ready` (idempotency).
4. Enable the automation.
5. Brian runs manual dry-test (subject "Test from agent", body with paragraph break) per the verification section of `scripts/fub-email-shell.md`.
6. If dry test passes → run `scripts/session-5-smoke-test.sql` Section A→B→C with Brian as test recipient (fub_person_id 27764).
7. If smoke test passes → flip `agent_enabled = true`, ramp `daily_send_cap` from 10 to 25 over week 1, watch reject rate.

---

## Prior session: 2026-05-08 (PM) — FUB Agent email shell shipped 🟡

### What got shipped
- New artifact at `scripts/fub-email-shell.md` in `HGPG1/hgpg-transaction-manager`. Specifies the FUB Automation 2.0 "Send Email" step content (subject `[-customAgentDraftEmailSubject-]`, body `[-customAgentDraftEmailBody-]`, type **plaintext**), plus setup checklist, user-level FUB requirements (CAN-SPAM signature, connected email, daily send limit ≥ 10), manual dry-test flow, and open questions for Viktor.
- `scripts/README.md` updated to point at the new artifact.
- Smoke-test status confirmed by direct DB query: never ran. Blocker was the missing email shell content; Viktor had built the trigger/tag wiring on his side.

### What did NOT ship
- iMessage shell (out of scope; LoopMessage parked)
- Any code changes to `lib/agent/` (the gap is FUB-side configuration)
- Brain App write. Drafts staged in `_brain-pending/` for paste-and-commit.

### Database state at session end (HGPG Core, ioypqogunwsoucgsnmla)
- `agent_enabled` = false (unchanged since 2026-05-06)
- `daily_send_cap` = 10
- 4 pending_review drafts from 2026-05-07 02:31 (Anthony Scott, Jay Miller, Seanna Mackey, Gerard Marmo). Keep or discard depending on smoke-test plan.
- 1 blocked draft id=6 (John Miller). Should be cleaned up; was Brian's session-4 manual approve attempt that the gate refused.
- 0 optouts, 0 cooldowns
- Cron `agent-daily-flush` firing nightly at 07:00 UTC, correctly no-op'ing while `agent_enabled=false` (cron_skipped events confirmed)

### Pickup notes for next session

**Sequence to unblock outbound:**

1. Brian or Viktor pastes shell content into FUB Automation 2.0 "Send Email" step (file: `scripts/fub-email-shell.md`)
2. Set email type to **plaintext**. HTML will collapse `\n\n` paragraph breaks; this is the most likely failure mode.
3. Add Step 2: Remove tag `agent_draft_email_ready` (idempotency)
4. Enable the automation
5. Brian runs manual dry-test (subject "Test from agent", body with paragraph break) per the verification section of the shell artifact
6. If dry test passes → run `scripts/session-5-smoke-test.sql` Section A→B→C with Brian as test recipient (fub_person_id 27764)
7. If smoke test passes → flip `agent_enabled = true`, ramp `daily_send_cap` from 10 to 25 over week 1, watch reject rate

**Things to keep on the radar:**
- iMessage shell artifact (`scripts/fub-imessage-shell.md`) when LoopMessage work resumes
- Interested-inbound surfacing in queue UI (logs only today)
- Week 1 metrics dashboard at `/agent/metrics`
- Set FUB UI visibility on custom fields 157/158/159 to admins-only (carryover from session 4)

---

## Locked queue

1. ⏳ Sellers guide Pixel QA — close out before treating as fully done
2. 🔄 Team Dashboard Tab 2 (Market Intelligence). Was next; FUB Agent jumped ahead this session.
3. 🎯 Buyer Alerts — first piece of MLS Dashboard Suite (~4-5 hr)
4. 🔍 Off-market / Expireds Finder — (~5-6 hr, replaces external scraper)
5. 🤖 FUB AI Agent: smoke test + go-live (next session)

### Smaller open items
- Signature Meta Pixel + CAPI (~30-45 min, copy buyers guide pattern)
- MLS Grid Media kickoff (parked alongside .net→.com migration)

### Parked
- #3 hgpg-transaction-monitor (audit checklist required)
- #6 deal notes UI (after CMA battle testing)
- MLS Grid Media
- #14 .net→.com migration
- Listing Report Portal MLS enrichment (low ROI — Team Dashboard now covers the team-facing need)

---

## Prior session: 2026-05-08 (PM) — Team Dashboard Tab 1 SHIPPED 🟢

### What got shipped
- New repo live: `HGPG1/hgpg-team-dash` (commit `e9d1366` initial scaffold)
- New Vercel project: `hgpg-team-dash` on team `team_FietQPKCmnyioG2n0FdteQCV`
- Live at: **`team.homegrownpropertygroup.com`** (CNAME via GoDaddy → cname.vercel-dns.com)
- Tab 1 (Inventory + Showings) end-to-end working:
  - Grid filtered by office key `CAR118249224`
  - Status filter pills (Live / Closed 90 / All / Canceled+Expired)
  - Sync freshness banner reading from `mls_sync_state`
  - `/inventory/[listingId]` drilldown with showings, ListTrac platform views, key dates
  - Banner when listing isn't onboarded into the Listing Report Portal
- Magic link auth + email allow-list via `TEAM_EMAILS` env var (Brian, Ashley, Brenda, Taylor, Closings)
- Brand-correct UI: Cooper Hewitt body, Sansita display, navy/steel/light-steel/off-white tokens, status pills (signature brown for Under Contract, navy for Active, etc.)
- Spec at `projects/team-dashboard.md` (commit `f1582a1`)

### Critical fixes shipped during deploy
- **Supabase RLS policies added** (migration `team_dash_read_mls_tables`): `mls_property`, `mls_member`, `mls_sync_state` had RLS on but zero policies, so anon key returned empty result sets. Added authenticated-role SELECT policies on all three.
- **Performance index added** (migration `team_dash_office_key_index`): MLS Grid table has 2.57M rows. No index on `list_office_key` was causing 8-second statement timeouts on the `?status=all` filter. Added composite `(list_office_key, standard_status, list_date DESC)` and `(listing_id)` indexes. Query time went from timing-out to ~36ms.

### Supabase auth config
- Site URL: kept as `https://listing-report-deploy.vercel.app` (do NOT change — Listing Report Portal uses Google OAuth and may rely on it; Brian found that callback URL was already in redirect list)
- Redirect URLs added:
  - `https://team.homegrownpropertygroup.com/**`
  - `http://localhost:3000/**`
- Existing Listing Report Portal redirect URLs left intact

### Key data discoveries
- **Office key** is `CAR118249224` — NOT R04075 (that's a Canopy display ID, doesn't appear in MLS Grid feed)
- 63 total team listings: 3 Active, 1 Active Under Contract, 1 Pending, 42 Closed, 14 Canceled, 2 Expired
- Active+UC+Pending: 1444 Society Hill Rd, 7807 Lamington Dr (Pending), 11230 Mallard Crossing Dr, 13002 Butters Way (UC), 00 Stratton Farm Rd
- 4 distinct list_agent_keys: Brian (CAR39987928), Ashley (CAR25452179), Brenda (CAR26251536), Taylor (CAR266730170)
- Vessie Lopez and Lauren McCarron are on the office key but haven't listed yet
- Address composition required — `unparsed_address` is null on most rows, build from `street_number + street_name + street_suffix`
- mls_property `listing_id` has `CAR` prefix; `listings.mls_number` does not. Strip on join: `REPLACE(listing_id, 'CAR', '')`
- MLS Grid Media table exists but zero rows synced. Hero photos fall back to portal `listings.hero_photo_url` then placeholder

---

## Prior session: 2026-05-08 (AM) — Strategic queue refresh + brain hygiene 🟢

### What got done
- Verified Pixel/CAPI rollout status across all consumer sites (Buyers, Sellers, NewCon shipped; Signature pending; TM/marketinganalyzer intentionally skipped)
- Pulled forward the locked queue from the 2026-05-06 strategic review

---

## Prior session: 2026-05-06 — Brain App MVP shipped 🟢

### What got built
- New Vercel project: `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- Live at: `https://brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Single-user lock: `BRIAN_EMAIL=brian@homegrownpropertygroup.com` allow-list

### Infra changes that affect other apps
- Resend custom SMTP wired into `HGPG Core` Supabase
- Supabase project renames for hygiene completed

### Deferred / Phase 2 for brain-app
- iPhone smoke test, Cooper Hewitt self-hosted, file rename/delete, diff view, cross-file search
