<!-- Last Updated: 2026-05-09 -->

# Session Handoff

## Last session: 2026-05-09 — Session 6 stopping point 🟡

### What got shipped (since last brain push commits b90d85a / ffcefd9)
- **FUB email template id 1156 created via API** (`Agent Draft Outreach (System)`). Subject `[-customAgentDraftEmailSubject-]`, body `[-customAgentDraftEmailBody-]`. Round-trip verified — both merge tags came back byte-identical via GET `/v1/templates/1156`. Created `2026-05-09T09:51:30Z` UTC / `05:51:30-04:00` ET.
- **`scripts/fub-email-shell.md`** updated with a "Template created" footer block (id, ISO timestamps, verification status, next-step instruction for Viktor).
- **Repo-root `CLAUDE.md`** created with a session-end checklist: push code, push brain via `POST https://brain.homegrownpropertygroup.com/api/external/write`, verify 2xx, no `_brain-pending/` as default staging.
- **`BRAIN_WRITE_TOKEN` moved to env var.** Value lives in `.env.local` (gitignored); `.env.local.example` shipped with the var name only. `.gitignore` updated with `!.env*.example` (the existing `.env*` rule was over-broad) and `_brain-pending/` (so fallback staging never lands in git).
- **Pre-flight checks both green** before Viktor's done:
  - GET `/v1/customFields` confirmed exact-name match for `customAgentDraftEmailBody` (157) and `customAgentDraftEmailSubject` (158). Capitalization, prefix, casing all align with `lib/agent/fubAgentConstants.ts`.
  - Supabase queue: 4 drafts in `pending_review`. Top 3 by combined_score: Gerard (draft 7, 37, viewed_listing_recent.email), Seanna (draft 8, 37, same), Jay (draft 9, 35, seller_report_engaged.seller). Anthony Scott is the 4th. Real warm-tier email drafts ready to push when smoke-test time comes.
- **`fub_cleanup/` decommissioned.** Stale FUB API key (`...wz1BQa`) sat in plaintext in an iCloud-synced dir for ~2 weeks. Verified against FUB Admin: Brian had already revoked it in earlier cleanup, so it was inert by the time it was caught. `fub_cleanup/.env` replaced with REVOKED marker; `fub_cleanup/README.md` got a status banner. Whole dir gitignored. Production FUB key (`...5j9xeR` in `.env.local` / `.env.vercel.local`) is separate and untouched.

### What did NOT ship
- `agent_enabled` still **false**. No outbound. Cron healthy and gated.
- Smoke-test SQL not run.
- Viktor's Automation 2.0 not yet wired to template 1156. Brian sent him the spec.
- iMessage shell still not built.
- Hot tier / iMessage seller variants / cooldown re-touch templates: deferred until after smoke test.

### Pickup notes for next session

**The smoke-test gate sequence (4 checkpoints, each with a kill switch — don't conflate "Viktor's done" with "ready to flip the switch"):**

1. **Manual test through the Automation.** Brian sets custom fields 157/158 by hand on a test person, saves, watches the Automation fire. Confirms the tag-trigger fires the Automation and the merge tags resolve.
2. **Confirm the email arrives.** Plaintext, with paragraph breaks intact, merge tags fully resolved (not literal `[-...-]` in the inbox). HTML-mode collapse on `\n\n` is the most likely failure here.
3. **Run `scripts/session-5-smoke-test.sql` Section A→B→C.** Synthetic Brian-only iMessage draft against fub_person_id 27764. Exercises draft generator → approve → push → cooldown → cleanup end-to-end against the agent code path.
4. **Push one real warm draft.** Gerard (draft id 7, score 37). First real outbound. Monitor.

If all 4 gates pass → flip `agent_enabled = true`, ramp `daily_send_cap` 10 → 25 over week 1, watch reject rate.

**Other things on the radar:**
- 4,340 unscored eligible leads still sitting. Once smoke test passes, run a scoring sweep to keep the queue fed past day 4 of the 10/day cap.
- Spec for Viktor: select template **id 1156** in Automation 2.0 → "Send Email" picker, set type to plaintext, add Step 2 "Remove tag `agent_draft_email_ready`" so the Automation doesn't re-fire.
- iMessage shell artifact (`scripts/fub-imessage-shell.md`) when LoopMessage work resumes.
- Interested-inbound surfacing in queue UI (logs only today).
- Week 1 metrics dashboard at `/agent/metrics`.
- Set FUB UI visibility on custom fields 157/158/159 to admins-only (carryover from session 4).

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
