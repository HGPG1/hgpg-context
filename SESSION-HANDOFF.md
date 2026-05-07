<!-- Last Updated: 2026-05-07 -->

# Session Handoff

## Last session: 2026-05-07 — Sellers Guide Meta Pixel + CAPI fully verified 🟢

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
- `projects/sellers-guide.md` (or wherever this lives in brain) — Meta Pixel + CAPI status: 🟢 SHIPPED, all 5 original checklist items complete except 3 + 4 below
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

### Pickup notes for next session
- All env vars on `charlotte-sellers-guide-vercel` are healthy. Real FUB key, real Meta tokens, no test event code.
- Production deploy is on commit `8ea82cc` (or whatever's latest if anything else has shipped since)
- Pattern for future "FUB silently dropping fields" debugging: hit `GET /v1/customFields` with the rotated key to confirm exact API names match what your code is sending. The auth check trick `echo "len: ${#FUB_API_KEY}"` is a fast way to confirm your local env actually loaded the key (length 0 = empty, length ~38-40 = real)
- The CUSTOM_FIELD_MAP pattern in `api/fub-lead.js` is reusable for the other sites' FUB integrations — bring it to TM / marketing analyzer / signature when rolling pixel + CAPI to those sites

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
  - `ngdrliyjtqcwhhfrbxao` → "HGPG FUB Integration" (verify)
  - `wdheejgmrqzqxvgjvfee` → "HGPG Listing Reports + MLS" (verify)
  - `fkxgdqfnowskflgbuxhm` → "HGPG Signature + Relocation" (verify)
- Supabase `HGPG Core` redirect URLs added:
  - `https://brain.homegrownpropertygroup.com/**`
  - `http://localhost:3000/**`
  - (Existing tools.hgpg entries left intact)

### Bugs found and fixed mid-session
- Magic link redirected to `tools.homegrownpropertygroup.com` (Supabase Site URL fallback) — fixed by adding `/auth/callback` route handler that was missing from initial scaffold + pointing `emailRedirectTo` at it
- Supabase free SMTP rate limit (2/hr) hit during testing — fixed permanently by switching to Resend custom SMTP

### Project status updates
- `projects/brain-app.md` — status now 🟢 SHIPPED (was 🟡)
- `projects/hgpg-team-tools2.md` — Site URL in Supabase still points here for the broken app's eventual fix
- `projects/transaction-manager.md` — no changes today, but TM benefits from Resend SMTP upgrade

### Deferred / Phase 2 for brain-app
- iPhone smoke test (CodeMirror + iOS soft keyboard scroll behavior)
- Cooper Hewitt self-hosted (currently falling back to system sans)
- File rename and delete
- Diff view before save
- Cross-file search

### Pickup notes for next session
- Brain-app is live and working — use it for any future updates to `hgpg-context`
- Resend API key is in 1Password ("Supabase HGPG Core SMTP")
- Brain-app local dev: `cd ~/brain-app && npm run dev` on Mac mini (work machine)
- Brain-app local on iMac: same setup, repo at `~/Developer/brain-app` if rebuilt, otherwise needs fresh `gh repo clone HGPG1/brain-app` + `npm install` + `cp env.example .env.local`
- The `package-lock.json` may differ between iMac and Mac mini — push from whichever machine you most recently ran `npm install` on
