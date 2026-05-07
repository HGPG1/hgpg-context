<!-- Last Updated: 2026-05-07 -->

# Session Handoff

## Last session: 2026-05-07 — Sellers Guide Meta Pixel + CAPI verified 🟢 + Brain audit reconciliation 🟢

Two distinct sessions today, both worth preserving in detail.

---

## SESSION A: Sellers Guide Meta Pixel + CAPI fully verified

### What shipped

- **Meta Pixel + CAPI on sellers guide is production-ready and verified end-to-end**
- Path A (organic) QA: PASSED — PageView, AssessmentStarted, Lead, ScoreCompleted all fire browser + server with matching `event_id` and Meta-side dedup confirmed
- Path B (Meta bypass) QA: PASSED — phone required, 6-digit verify skipped, FUB lead lands with `meta-bypass` tag and ALL 7 UTM/click custom fields populated
- `META_TEST_EVENT_CODE` env var removed from Vercel + redeployed — production traffic now flows to real Events Manager dashboards (not Test Events tab)

### Bug found + fixed mid-session

- `api/fub-lead.js` was sending FUB custom field **labels** (e.g., `"UTM Source"`) as object keys instead of FUB API **names** (e.g., `customUTMSource`). FUB silently drops unknown keys, so all 7 fields had been failing silently the whole time.
- Confirmed correct API names via `GET /v1/customFields`:
  - `customUTMSource`, `customUTMMedium`, `customUTMCampaign`, `customUTMContent`, `customUTMTerm`, `customFacebookClickID`, `customGoogleClickID`
- Patched in commit `8ea82cc` on `main`. Also added `X-System` / `X-System-Key` headers and a `DEBUG_FUB=1` env flag for surfacing FUB error bodies in API responses when needed.

### Infra changes

- **FUB API key rotated** — placeholder `fka_PLACEHOLDER` (which had been set as a Sensitive Vercel env var with empty value) replaced with real key on `charlotte-sellers-guide-vercel` project
- **Vercel "Sensitive" env var gotcha logged** — to UN-mark a var as Sensitive in Vercel, you must DELETE the var entirely and re-add it. There's no toggle. This was the root cause of repeated empty-key issues during diagnosis.
- **FUB integration system identifier** — sellers guide now identifies as `X-System: HGPG-SellersGuide` / `X-System-Key: sellers-guide-vercel` per FUB integration guide. Removes the rate-limit notice and gives us higher limits.

### Project status updates

- Meta Pixel + CAPI status on sellers guide: 🟢 SHIPPED, all 5 original checklist items complete except items 1-3 below
- FUB key swap blocker — RESOLVED. Brain notes flagged this as pending; can clear that flag.

### Deferred for next session

1. **Register `ScoreCompleted` as a Custom Conversion** in Meta Events Manager (Lead category, scoped to URL contains `home-selling-score`) — ~5 min
2. **Wire Meta Lead Ads → FUB native integration** for Instant Form path — ~15-20 min, touches both Meta and FUB
3. **Meta Pixel / CAPI rollout to remaining sites** (TM, marketing analyzer, signature) — playbook ready in `META-PIXEL-CAPI-PLAYBOOK.md` in sellers guide repo, ~30 min/site, each needs own Pixel ID

### Cleanup tasks (low priority)

- Delete QA test leads from FUB:
  - `qa-may7-fields-test@hgpg-test.com`
  - Any other QA leads created during Path A / Path B testing on May 7
- Remove "Phase 1 ads test markers" in code (flagged in original session intro as a future cleanup item)

### Pickup notes specific to Sellers Guide CAPI

- All env vars on `charlotte-sellers-guide-vercel` are healthy. Real FUB key, real Meta tokens, no test event code.
- Production deploy is on commit `8ea82cc` (or whatever's latest if anything else has shipped since)
- Pattern for future "FUB silently dropping fields" debugging: hit `GET /v1/customFields` with the rotated key to confirm exact API names match what your code is sending. The auth check trick `echo "len: ${#FUB_API_KEY}"` is a fast way to confirm your local env actually loaded the key (length 0 = empty, length ~38-40 = real)
- The `CUSTOM_FIELD_MAP` pattern in `api/fub-lead.js` is reusable for the other sites' FUB integrations — bring it to TM / marketing analyzer / signature when rolling pixel + CAPI to those sites

---

## SESSION B: Brain audit + Priority 1+2 reconciliation

### What this session did

A full audit of `hgpg-context` against the actual state of HGPG infrastructure, project work, and ships from 5/5-5/7. Surfaced staleness and contradictions across most files. Drafted reconciliation for 9 files + archived 1.

### Files updated this session

**Supporting files:**
- `team.md` - phone disambiguation (3700 = main office FUB, 4701 = Brian's direct FUB), Lauren removed, FUB user status added per agent
- `infrastructure.md` - 4 active Supabase projects with correct names, brain-app + tools + new-construction subdomains added, Resend SMTP section added, MLS Grid path corrected, Suna deletion noted
- `marketing.md` - Sendblue decision locked (was "none selected"), MLS Grid framing corrected, Meta Pixel multi-site rollout table added, South Charlotte Report expanded, LoopMessage path documented
- `operations.md` - **Last-Updated header convention added** (every brain file gets `<!-- Last Updated: YYYY-MM-DD -->`), **auto-update trigger pattern documented** (bueno/wrapping up/new thread = Claude proactively preps brain push)
- `SESSION-HANDOFF.md` - this file

**Project files:**
- `projects/cma-engine.md` - PRs #14-22 added, MLS Grid replication path documented, GLA adjustment rate documented, outlier handling section added (was last updated April 19)
- `projects/transaction-manager.md` - Don's 5/1 + 5/6 Cluster A-D batches added, LoopMessage migration, milestone tracker, post-close FUB handoff, multi-agent routing, $395 fee toggle parking, /agent surface noted, local path corrected to ~/Documents
- `projects/fub-ai-agent.md` - sessions 2 + 3 added (approval queue, draft generator, voice file, classifier rebuild, rejection handling, Sendblue wiring), session 4 plan documented
- `projects/listing-report-portal.md` - GitHub auth blocker resolved (verified by 5/6 favicon push)

**Archived:**
- `projects/deals-tracker.md` -> `projects/archive/deals-tracker.md` (decommissioned 2026-05-01)

### Key generalizable lessons logged

- **FUB silently drops unknown custom field keys.** Always use API names (`customUTMSource`), never labels (`"UTM Source"`). Verify via `GET /v1/customFields`.
- **Vercel Sensitive env vars cannot be un-marked.** Delete + re-add is the only path.
- **Brain drift cost is real.** 5+ days of heavy shipping = brain notes wrong on most active projects. Vercel deployment list + commit history are the most reliable sources of truth when brain notes contradict.

---

## Recent ship history (5/5-5/7) - context

The brain wasn't tracking these in real time. They're now reflected across the updated project files.

**5/5 - HGPG Team Tools auth restoration**
- Rebuilt on HGPG Core Supabase + Google OAuth after `nfwjgsanvmwmrginhklz` Supabase deletion during Suna teardown
- See `projects/team-tools.md`

**5/5 - NeverBounce email validation**
- Real-time email validation on incentives funnel forms before lead lands in FUB
- See `projects/incentives-funnel.md`

**5/6 - Brain App MVP shipped**
- Live at `brain.homegrownpropertygroup.com`, Phase 1.5 polish included
- Resend SMTP wired into HGPG Core Supabase (rate limit 2/hr -> 30/hr, affects all apps on HGPG Core)
- See `projects/brain-app.md`

**5/6 - CMA Engine PRs #16-22**
- Outlier auto-exclusion + escape hatch + suppression banner
- Condition-tier inference, address parser, GLA methodology fix
- OUTLIER_MIN_CLUSTER_SIZE lowered from 4 to 3 (PR #17)
- Awaiting Taylor stress test
- See `projects/cma-engine.md`

**5/6 - Don's TM feedback Cluster A-D batch**
- 7 items resolved (DD fee placement, buyer inspection on seller side, no vendor in Setup Inspection, seller concessions in summary email, termination fee not picked up, seller concessions $3000 not picked up, seller info link expired)
- $395 fee toggle parked as build spec (refs commit b9fa0deb)

**5/6 - Other-side client email made optional**
- Seller/buyer wizards no longer require other-side email
- Format validation prevents typo'd addresses
- Transactional footer for cross-side recipient

**5/6 - Favicons rolled out to 9 sites**
- Canonical asset set in brain at `assets/favicon/`
- TM kept its wordmark exception
- Listing-report-portal favicon push verified GitHub auth working (resolved prior blocker)

**5/7 - Sellers Guide Meta Pixel + CAPI verified end-to-end** (Session A above)
- Production deploy on commit `8ea82cc`
- See deferred items + cleanup tasks above

**5/6-5/7 - FUB AI Agent sessions 2 + 3**
- Session 2: approval queue UI, draft generator, voice file, outbound gate, classifier rebuild, rejection handling
- Session 3: Sendblue wiring, FUB Automations 2.0 setup, inbound parser
- See `projects/fub-ai-agent.md` (now current)

---

## Infra reality (post-audit)

**Supabase (4 active):**
- HGPG Core (`ioypqogunwsoucgsnmla`) - TM, CMA, TC Concierge, brain-app, Team Tools, FUB AI Agent
- HGPG Listing Reports + MLS (`wdheejgmrqzqxvgjvfee`) - Listing Report Portal + MLS Grid replication
- HGPG FUB Integration (`ngdrliyjtqcwhhfrbxao`) - legacy FUB texting integration
- HGPG Signature + Relocation (`fkxgdqfnowskflgbuxhm`) - Signature site + NYC relocation

**Removed 2026-05-04:** `nfwjgsanvmwmrginhklz` (Suna self-host)

**Resend custom SMTP** wired into HGPG Core (5/6) - 30/hr rate limit, affects all HGPG Core apps.

---

## Known unresolved (carried forward)

- Sherlock 403 on TM (likely API key scope)
- NC office routing not implemented (swap to NC office ID when txn.state === 'NC')
- transaction-pdfs bucket needs to flip to signed URLs
- Lamington duplicate cleanup
- GitHub PAT exposed in chat history needs rotation
- Supabase listing-report-portal at 15GB (soft warning, "revisit" status)
- Twilio A2P TCR rejection (need SMS consent checkbox)
- `.net` Google Workspace migration (do not proactively remind)
- QA test leads in FUB from Sellers Guide testing - cleanup pending

---

## Pickup notes for next session

- **Auto-update triggers are now operational.** "bueno", "wrapping up", "new thread" will cause Claude to proactively prepare brain updates. Documented in `operations.md`.
- **Last-Updated headers** should be checked + bumped on any brain file edit going forward.
- **Brain App** at `brain.homegrownpropertygroup.com` is the canonical edit surface. Direct PAT writes to `hgpg-context/main`.
- **Priority 3 still open:** create spec files for Charlotte New Construction, Sellers Guide, Buyers Guide, South Charlotte Report, NC Scout. None of these have brain files despite being active.
- **CONTEXT.md** was rebuilt as a real project index earlier in the day (5/7) - it now reflects current ship state.
- **Sellers Guide Meta Pixel + CAPI deferred items** (Custom Conversion registration, Meta Lead Ads -> FUB integration, multi-site rollout) are the natural next builds — see Session A deferred list.
- The drafted file updates from this session need to be committed via brain-app or terminal. Single commit recommended.

---

## Previous session: 2026-05-06 — Brain App MVP shipped 🟢

### What got built
- New Vercel project: `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- New repo: `HGPG1/brain-app` (private)
- Live at: `https://brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Single-user lock: `BRIAN_EMAIL=brian@homegrownpropertygroup.com` allow-list
- GitHub auth: fine-grained PAT scoped to `HGPG1/hgpg-context`, contents:write only
- Round-trip verified: edit file in browser → commit lands on `main` with author `brian@homegrownpropertygroup.com`

### Infra changes that affect other apps
- Resend custom SMTP wired into `HGPG Core` Supabase (project `ioypqogunwsoucgsnmla`)
  - Sender: `noreply@homegrownpropertygroup.com`, name: HGPG
  - API key stored under "Supabase HGPG Core" in Resend
  - Rate limit went from 2/hr (Supabase default) to 30/hr (Resend default), can be raised
  - This affects ALL apps using this Supabase: TM, CMA, TC Concierge, brain-app
- Supabase project renames for hygiene:
  - `ioypqogunwsoucgsnmla` → "HGPG Core"
  - `ngdrliyjtqcwhhfrbxao` → "HGPG FUB Integration"
  - `wdheejgmrqzqxvgjvfee` → "HGPG Listing Reports + MLS"
  - `fkxgdqfnowskflgbuxhm` → "HGPG Signature + Relocation"
- Supabase `HGPG Core` redirect URLs added:
  - `https://brain.homegrownpropertygroup.com/**`
  - `http://localhost:3000/**`

### Bugs found and fixed mid-session
- Magic link redirected to `tools.homegrownpropertygroup.com` (Supabase Site URL fallback) — fixed by adding `/auth/callback` route handler that was missing from initial scaffold + pointing `emailRedirectTo` at it
- Supabase free SMTP rate limit (2/hr) hit during testing — fixed permanently by switching to Resend custom SMTP

### Deferred / Phase 2 for brain-app
- iPhone smoke test (CodeMirror + iOS soft keyboard scroll behavior)
- Cooper Hewitt self-hosted (currently falling back to system sans)
- File rename and delete
- Diff view before save
- Cross-file search

### Pickup notes
- Brain-app is live and working — use it for any future updates to `hgpg-context`
- Resend API key is in 1Password ("Supabase HGPG Core SMTP")
- Brain-app local dev: `cd ~/brain-app && npm run dev` on Mac mini (work machine)
- Brain-app local on iMac: same setup, repo at `~/Developer/brain-app` if rebuilt, otherwise needs fresh `gh repo clone HGPG1/brain-app` + `npm install` + `cp env.example .env.local`
- The `package-lock.json` may differ between iMac and Mac mini — push from whichever machine you most recently ran `npm install` on
