## 2026-05-05 — Correction: MLS Grid replication is fully live, not just unblocked

Earlier today I logged the MLS Grid token as "in hand, CMA tool unblocked" and queued wiring + cron verification for next session. That was wrong — the system is already live and has been running.

Current verified state of MLS Grid replication on Supabase wdheejgmrqzqxvgjvfee:
- mls_property: 2,573,569 rows
- mls_sync_state: cron */15 firing successfully on Property, Member, Office, OpenHouse (all status=success, last_run_at within 15 min of check)
- Media: status=never_run, gated separately or not yet triggered (worth checking next session if media is needed for CMA packets)

The token, the env vars, and the cron are all wired and operational. CMA Engine searchComps() in lib/mls/compSearch.ts can be exercised against live data right now.

Open follow-up only:
- Verify Media resource is intentionally not running, or kick it off if it should be

---

## 2026-05-05 — Outstanding list cleanup: Twilio A2P killed, MLS Grid token in hand

Two parked items resolved as no-longer-relevant or unblocked:

### Twilio A2P 10DLC — DEAD, do not resurrect
Twilio is fully deprecated for HGPG (memory line: "Twilio/A2P fully deprecated and removed from HGPG stack 2026-04. iMessage via LoopMessage is the only TM messaging path."). The TCR-rejected campaign that was blocked on adding SMS consent checkboxes is no longer relevant — there is no Twilio-based SMS path to resubmit to. Stop tracking. Future "do we need SMS for X" decisions are independent.

### MLS Grid token — RECEIVED, CMA tool unblocked
Canopy MLS Grid token from Bridgett Bouvier is in hand. The MLS Grid → Supabase replication built 2026-04-29 (HGPG1/hgpg-cma-tool, branch mls-grid-sync, 9 tables in wdheejgmrqzqxvgjvfee, cron */15) is no longer blocked on the API token. CMA Engine searchComps() in lib/mls/compSearch.ts can now run against live MLS data.

Open subitem: confirm token is wired into the right Vercel env var on hgpg-cma-tool project. Also worth confirming the cron is firing successfully now that auth is live (next session).

---

## 2026-05-05 — Sellers Guide FUB API key swapped from placeholder to real

charlotte-sellers-guide-vercel had FUB_API_KEY env var set to fka_PLACEHOLDER. Code at api/fub-lead.js and api/assessment/submit-ratings.js reads process.env.FUB_API_KEY and posts to api.followupboss.com/v1/events when set.

Action taken:
- Generated new FUB API key in app.followupboss.com/2/api named "Sellers Guide - 2026-05"
- Updated FUB_API_KEY env var in Vercel project charlotte-sellers-guide-vercel (Production, Preview, Development)
- Redeployed
- Verified end-to-end: completed full sellers guide assessment flow, lead landed in FUB

Operational note for future debugging:
- /api/verification/send-code and /api/assessment/create do NOT call FUB
- Only /api/assessment/submit-ratings (the final step in the flow) posts to FUB
- A submit that stops mid-flow will NOT show in FUB even if everything else is configured correctly

Per-project FUB key strategy: each Vercel project gets its own scoped FUB API key for cleaner audit/rotation. Currently:
- charlotte-new-construction-nextjs: rotated 2026-05-05 to its own key
- charlotte-sellers-guide-vercel: new key 2026-05-05 (this commit)

---

## 2026-05-05 — hgpg-transaction-monitor cleanup PARKED. Read before resuming.

Attempted teardown of remaining hgpg-transaction-monitor remnants today. Aborted before completion after a near-miss on tc-concierge (the LIVE system Don uses).

What happened:
- Brain notes said hgpg-transaction-monitor was decommissioned 2026-04-29 with cleanup pending on Vercel project + Apps Script + DNS
- During DNS cleanup, accidentally deleted both transactions.homegrownpropertygroup.com AND tc.homegrownpropertygroup.com records at GoDaddy
- tc.homegrownpropertygroup.com is the LLM proxy that the hourly tc-concierge Apps Script (in closings@ Google account) hits to send PDFs through Claude Haiku — ~44 emails/day, 100% extraction success
- Re-added tc DNS record immediately, verified via Cloudflare resolver. Production pipeline restored.
- Local Mac DNS cache stayed stale for an extended window (negative-cache stubborn), but Google DNS resolved fine the whole time, so Apps Script production was unaffected
- transactions.homegrownpropertygroup.com left without DNS for a brief window then re-added (decision pending whether to keep or kill)

Critical lesson:
- Brain notes about decommissioned services may be stale or wrong. Closing Concierge was supposedly "503/down" per brain — turned out to be serving a live landing page. hgpg-transaction-monitor may have similar surprises.
- Before doing ANY DNS deletion at GoDaddy, double-check the subdomain you are targeting. Adjacent records in alphabetical order can be confused (tc and transactions sit close enough to slip).

Status of hgpg-transaction-monitor remnants:
- GitHub repo HGPG1/hgpg-transaction-monitor: archived (already done 2026-04-29)
- Supabase data: archived in-place as _archived_transaction_emails_2026_04_29 in wdheejgmrqzqxvgjvfee (already done 2026-04-29)
- Vercel project: STILL LIVE, NOT TOUCHED
- Apps Script in closings@: STILL LIVE, NOT TOUCHED
- DNS for transactions.homegrownpropertygroup.com: re-added, still resolving to Vercel

Next-session checklist before resuming this teardown:
1. Confirm Vercel project name matches (likely `hgpg-transaction-monitor`) — do NOT assume
2. Click into that Vercel project's domains list and confirm it lists transactions.homegrownpropertygroup.com (not anything else live)
3. Pull build logs to confirm no recent deploys / no recent traffic
4. Check the closings@ Apps Script trigger list — confirm there's nothing the closings inbox needs from that script
5. ONLY THEN: delete Vercel project, delete Apps Script, delete DNS

Estimated cost of leaving it as-is: ~$0/month (idle Vercel project, idle Apps Script). Cleanup is cosmetic, not financial. Low priority.

---

## 2026-05-05 — Closing Concierge standalone fully decommissioned

Closing Concierge standalone (concierge.homegrownpropertygroup.com) torn down. Superseded by TM-integrated concierge auto-trigger (commit 1fd2d02 from 2026-04-25).

Pre-teardown verification:
- Supabase API logs: zero hits in last 24hr
- Supabase postgres logs: only Supabase mgmt-api connections, zero application connections
- TM grep: no references to the standalone domain or repo
- Only data was 1 test row (123 main st, Brian as agent, dated April 1)

Teardown completed:
- Schema archived to docs/closing-concierge-schema-archive.md (commit 5f4020d) before deletion
- DNS for concierge.homegrownpropertygroup.com removed (Brian, GoDaddy)
- Supabase project mvhyurdomejzdfigmyuk deleted (Brian, dashboard)
- GitHub repo HGPG1/closing-concierge archived (gh CLI)
- Vercel project deleted (Brian, dashboard)

If TM concierge ever needs richer device/utility/move-in detail capture, the schema reference at docs/closing-concierge-schema-archive.md has the jsonb shapes (smart_devices, keypad_locks, utilities, garage_access, mail_forwarding) that worked.

---

## 2026-05-05 — GitHub PAT cleanup

Old PAT (ghp_y2iuUY...) had been embedded in origin URLs of three local repos:
- charlotte-sellers-guide-vercel
- hgpg-cma-tool
- nyc-to-charlotte-relocation

Audited all local repos for PAT references. Confirmed:
- No tracked code files reference any GitHub PAT
- The three repos above were the only ones with PAT-in-URL
- gh CLI keyring auth (Keychain) is doing all the actual work

Action taken:
- Stripped PAT from origin URLs of those three repos using `git remote set-url`
- Verified fetches work via gh CLI keyring without prompting
- Confirmed old PAT was already revoked at github.com/settings/tokens (was orphaned, no longer existed)

Result: no PATs in use anywhere. gh CLI on Mac Mini + iMac handles all GitHub auth via macOS Keychain.

Project instructions in Claude still list the revoked PAT — Brian to update manually in project settings.

---

## 2026-05-05 — Team Tools auth restored

Team Tools (tools.homegrownpropertygroup.com) was bricked since 2026-05-04 when Supabase project nfwjgsanvmwmrginhklz was deleted during Suna teardown. The dead URL was hardcoded in src/lib/supabase.ts.

Fix shipped today (commit d93befe on HGPG1/hgpg-team-tools2):
- Refactored src/lib/supabase.ts to read VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY from env vars
- Pointed at primary Supabase ioypqogunwsoucgsnmla
- Created agent_profiles table with RLS policies
- Created Google Cloud project HGPG Team Tools (project number 429376800551)
- Set OAuth consent screen to Internal (Workspace-only enforcement)
- Created OAuth Web Client, wired into Supabase Google provider
- Set Site URL and Redirect URLs in Supabase auth config
- Added VITE_* env vars to Vercel project hgpg-team-tools2
- Forced no-cache rebuild to bust stale build cache (initial redeploy served old hardcoded URL)

Auth verified end-to-end with @homegrownpropertygroup.com Workspace account. Personal Gmail correctly rejected at Google layer.

Repo was archived during the Suna cleanup; unarchived as part of this fix.

See projects/team-tools.md for the canonical reference.

---

# SESSION-HANDOFF

**Last session:** 2026-05-05 (afternoon)
**Length:** ~3 hours, single-track focused build.
**State at end:** Shipped to production. Monitoring for 2-4 weeks before next iteration.

---

## What happened this session

### Shipped: NeverBounce email validation on /incentives

Real-time email validation now live on `newconstruction.homegrownpropertygroup.com/incentives` (all 3 form variants: hero, card_modal, secondary). Catches invalid + disposable emails at the form before they reach FUB.

**What it does:**
- Debounced 500ms blur validation hits NeverBounce v4 single/check via server-side route
- 5 result codes mapped:
  - valid -> green check, allow submit
  - invalid -> red X, block submit ("This email address doesn't exist")
  - disposable -> red X, block submit ("Please use a permanent email address")
  - catchall -> yellow warning, allow submit
  - unknown -> yellow warning, allow submit (fail-open philosophy)
- Spelling-mistake flag detection: when NeverBounce flags a typo but doesn't return a suggested_correction, UI softens to yellow warning ("This looks like a typo - please double-check your email") and allows submit
- Suggested-correction button when NeverBounce provides one (e.g. "Use brian@gmail.com")
- Server-side fail-open: NeverBounce timeout/error/missing key returns 'unknown' so form never breaks

**Architecture:**
- `src/app/api/validate-email/route.ts` - NEW. Server-side proxy to NeverBounce. Only file that references `NEVERBOUNCE_API_KEY`. 10s AbortController timeout, format-checks server-side before hitting API to avoid wasting credits.
- `src/app/incentives/IncentivesClient.tsx` - added validation state, debounced blur effect, sequence counter to ignore late responses, submit gate, ValidationStatusLine UI
- `src/app/api/incentives-lead/route.ts` - passes `email_validation_status` through to FUB layer
- `src/lib/fubLight.ts` - writes raw NeverBounce result code to `customEmailValidationStatus` custom field; auto-tags `Email-Invalid` or `Email-Disposable` based on result

**Critical design decision: UI permissive, analytics honest.** The typo case (`result: invalid` + `flags: ["spelling_mistake"]`) shows yellow warning in UI but sends raw `invalid` to FUB. Display state and analytical state are separated - `validationStatus` (UI) vs `rawResult` (FUB payload). This preserves the ability to filter FUB later for typo'd emails while not blocking real humans who fat-fingered.

### Infrastructure changes

- New env var: `NEVERBOUNCE_API_KEY` in Vercel for `charlotte-new-construction-nextjs` project. Marked Sensitive (Production + Preview only, no Dev pulls).
- New FUB custom field: `customEmailValidationStatus` (text, label "Email Validation Status", id 149)
- New auto-tags in incentives flow: `Email-Invalid`, `Email-Disposable`
- FUB API key rotated 2026-05-05 (lost the first new key, generated a second). Old keys revoked. Current key lives in Vercel only.

### NeverBounce account state

- Free 10 credits exhausted during testing
- Bought $10 starter pack (~1,250 verifications)
- API key prefix: `private_xxx` (newer NeverBounce format, works identically to legacy `secret_xxx`)
- API call uses `address_info=1` to populate `suggested_correction` field for typos

### Decisions made

- **Bad-email tag automation strategy:** just monitor for now. Will revisit after 2-4 weeks of real volume to see actual invalid/disposable rates before deciding suppress-vs-triage workflow.
- Skipped Email-Typo as a separate tag - `customEmailValidationStatus` field preserves the distinction, and tags are for action triggers (which we don't need yet).
- FUB Automations 2.0 has no "Custom Field Updated" trigger - code-side tagging is the right approach when behavior needs to fire on field-write at lead creation.

### Gotchas captured

- Vercel "Sensitive" env vars exclude Dev environment - local `npm run dev` cannot test FUB or NeverBounce, must test on Preview deploys
- Next.js service workers cache aggressively on Preview deploys - requires unregister + hard reload (or incognito) to see new bundles. Caused ~30 min of misdirected debugging today; bookmark this.
- NeverBounce returns `flags: ["spelling_mistake"]` for typos but only fills `suggested_correction` when `address_info=1` query param is set
- FUB tags are created on demand - no FUB-side setup needed before sending new tag strings via API
- FUB doesn't expose custom field API names in the UI; use `GET /v1/customFields` to find them
- Random gibberish at @gmail.com often returns `valid` from NeverBounce (Gmail accepts mail at any address before bouncing) - use bogus DOMAINS for invalid testing, not bogus users at real domains

---

## What's open for next session

### Monitoring (passive, ~2-4 weeks)

NeverBounce on /incentives is live. Let real volume build up, then revisit:
- What % of leads come through as invalid / disposable
- Whether typo rate justifies a "review and find correct contact" agent task automation
- Whether disposable rate is high enough to warrant triggering "Suppress-From-Email" tag automatically

### Future Phase 2 - NeverBounce (deferred until 50+ leads through funnel)

- Industry/competitor email blocklist (Stephen Cooley RE, KW, Compass, etc.)
- Decision needed: hard-block vs flag-and-allow for industry leads
- Rollout to other forms: TM, marketing analyzer, Signature, sellers guide, buyers guide. Each site needs its own approach (`NEVERBOUNCE_API_KEY` env var per project, server-side route, validation UI). Playbook = use this incentives implementation as the template.

### Quick win (~30 min)

**Team Tools auth fix** at `tools.homegrownpropertygroup.com`.
- Diagnosed: was wired to the deleted Suna Supabase (`nfwjgsanvmwmrginhklz`), so login is broken.
- Source repo: private, `HGPG1/hgpg-team-tools2` (Vite app).
- Fix: clone repo, swap Supabase env vars to point at primary `ioypqogunwsoucgsnmla`, set up Google OAuth on that Supabase, redeploy.
- Whole team uses it (5 people), but no urgency.

### Major builds

**FUB lead automation rebuild** at `leads.homegrownpropertygroup.com`.
- Spec preserved at `archive/fub-agent-handoff.md` in brain repo.
- Reuse existing infra: Vercel project `fub-texting-integration`, Supabase `ngdrliyjtqcwhhfrbxao`, GitHub repo `HGPG1/fub-texting-integration`.
- Schema already designed (15 tables including bulk campaigns, opt-out tracking, auto-reply rules).
- Multi-day build estimated.

**Brain app** at `brain.homegrownpropertygroup.com`.
- Spec at `projects/brain-app.md` in brain repo.
- 60-90 min Claude Code session estimated.
- Pre-build checklist in spec: GitHub PAT, Supabase project decision, GoDaddy DNS, empty repo creation.

### Minor decisions

- MLS Listing Dashboard project - currently parked. Decision depends on whether actionable MLS alerts arrive in Gmail. Audit Gmail filters first.
- A2P 10DLC resubmission - still blocked. Need SMS consent checkbox on contact forms before retry.

---

## State of the brain repo

- Clean and current.
- `CONTEXT.md` reflects all completed teardowns and open items.
- `infrastructure.md` reflects current Vercel + Supabase + GitHub state.
- `projects/brain-app.md` has full spec for future build.
- `archive/fub-agent-handoff.md` has Manus-era spec for FUB rebuild reference.

---

## How to start next session

1. Open the relevant Claude project in claude.ai (HGPG - Tech & Build for builds, HGPG - CRM & Leads for FUB work, etc.).
2. The project will fetch CONTEXT.md automatically.
3. Reference this handoff for "where we left off."

---

## Previous session - 2026-05-05 (morning, infrastructure cleanup)

### What happened

**Infrastructure cleanup:**
- Migrated GitHub auth from PAT-embedded URLs to gh CLI on both Macs (Mac mini + iMac). Token in macOS Keychain. No more PAT rotation drama.
- Tore down Suna fully: DNS (`ai.homegrownpropertygroup.com`), Railway deployment, Supabase project `nfwjgsanvmwmrginhklz`.
- Decommissioned Manus FUB AI Agent: unpublished from Manus, deleted Railway services + project, archived `HGPG1/fub-ai-agent` repo. Cancelled Railway plan entirely (no recurring charge).
- Archived 20+ cruft GitHub repos (versioned predecessors with `1`, `2`, `3`, `part2` suffixes).
- Deleted two cruft Vercel projects (`adoring-wilbur`, `project-lisqf`).
- Resolved sellers guide drift: `sellersguide.homegrownpropertygroup.com` is canonical, `sellers.` subdomain removed, empty placeholder repo `charlotte-sellers-guide-2026` archived.
- Clarified Listing Report Portal infrastructure: source repo is `hgpg-listing-report`, Vercel project is named `listing-report-deploy`. The naming mismatch is cosmetic but documented.

**Documentation:**
- Manus FUB AI Agent handoff doc preserved at `archive/fub-agent-handoff.md` in brain repo and at `~/Library/Mobile Documents/com~apple~CloudDocs/HGPG-Cowork/fub-agent-handoff.md`.
- Brain app spec drafted at `projects/brain-app.md` for future build.
- Brain repo (CONTEXT.md + infrastructure.md) updated and pushed multiple times.

**Workflow:**
- `brain` shell function deployed on both Macs. Auto-pulls files from `~/Downloads/` matching brain file patterns, copies into local repo, prompts for commit message, pushes. Friction reduction is live.

**Claude project sidebar restructure:**
- Audited all Claude projects, consolidated overlap.
- 5 HGPG projects (renamed for consistency, all use `HGPG - <name>` style):
  - HGPG - Home Base (catch-all, formerly `Project- Home Base`)
  - HGPG - Tech & Build
  - HGPG - CRM & Leads
  - HGPG - Marketing & Content (absorbed Content Pipeline)
  - HGPG - Ads (formerly `HGPG- Meta Ads`, scope expanded)
- Modernized Homie project to fetch live brain repo instead of relying on uploaded master memory file.
- Configured Home projects (personal home improvement) with new instructions.
- Archived 4 projects: Dev & Tech, Real Estate Ops, Content Pipeline, How to use Claude.

### Lessons captured

- **Heredoc paste hazards:** zsh + Messages app + heredoc syntax is brutal. Multiple times the paste got corrupted with `EOFcat` collisions or stray characters. The `brain` shell function setup hit this twice. Lesson: prefer single-command appends or text-editor edits over chained heredocs in Terminal.
- **Don't trust "cruft" labels without verification:** I had `listing-report-deploy` flagged as cruft for deletion. It was actually serving the live Listing Report Portal. Lesson: verify subdomain ownership in Vercel before deleting any project, even one that "looks" abandoned.
- **Project instructions had Manus-era stale references:** Several old projects still referenced "migrating from Manus" as in-progress. The migration is done. New project instructions reflect that.
