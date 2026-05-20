<!-- Last Updated: 2026-05-20 -->

# Session Handoff

## Last session: 2026-05-20 — Meta insights endpoint fixed, Variant E breakout, FUB attribution gap surfaced 🟢🟡

### What got verified
- `/api/meta-insights` on closings.homegrownpropertygroup.com is returning correctly-windowed data. 7d/14d/30d all distinct, no lifetime-cache pollution. Bug fixed, 60s TTL holding.
- Bearer token `META_INSIGHTS_TOKEN` works end-to-end. Stored in HGPG - Ads project instructions.
- Pipeboard Pro trial active through ~2026-05-28. Decision pending after this read.

### Headline findings
**Account totals (pre-D-pause snapshot, locked for tomorrow's delta read):**
- 7d:  $608.87 spend / 110 leads / $5.54 CPL
- 14d: $856.43 spend / 162 leads / $5.29 CPL
- 30d: $1198.83 spend / 253 leads / $4.74 CPL
- CPL drifting up wk/wk pre-pause. Variant D was the drag.

**Variant E (10K hook) is the breakout, day 2:**
- E - 9:16 vertical: $84.17 spend, 37 leads, $2.27 CPL, 3.28% CTR
- E - 1:1 square: $1.60 spend, no delivery
- E - 4:5 portrait: $3.98 spend, no delivery
- Meta auctioned budget into vertical placement. Square + portrait got no delivery and were left running (Meta isn't burning budget on them).
- E vertical beats Variant C (control workhorse) by 1.86x on CPL with only ~18% of total spend.

**Variant C (LocalPride):** $240.60 / 57 leads / $4.22 CPL. Still the workhorse.
**Variant D (Incentives):** $125.66 / 12 leads / $10.47 CPL. PAUSED 2026-05-20 PM.

**Sellers Guide — Meta vs FUB attribution gap surfaced:**
- Meta reports: CONV ad set $83.42 / 4 leads / $20.86 CPL (under $35 target)
- FUB actual: 1 real lead tagged sellers-guide-2026 (Daniela Portillo id 32123, in this morning 03:38 UTC)
- 1 test record (id 32043 Test Mctest) was neutralized to Trash with `DELETE-ME-test-data` tag — Brian to sweep in UI
- True CPL is closer to $83 (1 real lead / $83 spend) than $20.86
- Cause is unknown: pixel firing on quiz-start instead of form-submit, OR webhook dropping leads, OR Meta double-counting attribution. Routed to Tech & Builds for diagnosis.

### Actions taken this session
- Pulled clean 7d/14d/30d campaign-level + ad-level data from fixed endpoint
- Verified FUB sellers-guide-2026 tag lead flow: 2 records, 1 real, 1 test
- Neutralized FUB test record id 32043 (stage=Trash, tags stripped to DELETE-ME-test-data)
- Brian paused Variant D ad (HGPG_New_Con_VariantD_LocalPride_2026-05-11, id 52503410311963)
- Drafted Tech & Builds handoff covering (a) SG-2026 Sellers Guide Nurture automation rebuild and (b) pixel/webhook attribution diagnosis — Brian shipped to T&B

### Blocked / pending
- **SG-2026 Sellers Guide Nurture automation — Send Email steps did not persist (FUB React Save bug).** Templates 1158-1165 are all created and shared. Automation shell exists but is unusable. Sequence to rebuild lives in T&B handoff. **Do NOT activate IDXRE-B2 sellers automation or the buyers automation until this is rebuilt AND pixel/webhook gap is diagnosed.**
- Cold Nurture (Auto #332) is complete and unaffected.
- Stoppers (Unsubscribed, Y_DNC_REGISTRY_TRUE) confirmed working.
- Pixel event trigger needs verification (quiz-start vs form-submit) — T&B owns
- Webhook delivery logs needed for 2026-05-13 through 2026-05-20 — T&B owns

### Day 5-7 decision tree for Variant E (~May 23-25)

E vertical needs to clear two bars at the day 5-7 read for "new control" status:

1. **Volume bar:** must hold sub-$3 Meta CPL across at least 3x current spend (≥$250 cumulative) AND ≥100 leads. If CPL drifts to $3-5 at scale, E is still a strong B-variant but not the new control.
2. **Stability bar:** CTR must stay ≥3.0% (currently 3.28%). Below 3.0% with rising CPL means creative fatigue is already starting on a fresh ad.

Outcomes:
- Both bars cleared → E vertical is the new control, demote C to support, build E variants for square/portrait that actually get delivery (the current ones don't)
- Volume bar only → E is a strong B-variant, keep both at parity in the rotation
- Neither bar → E was a 2-day fluke, revert to C as control, retire E

### Recommendations sent to Brian today
1. ✅ Pause Variant D — DONE
2. ✅ Pause E square (52509166115763) and E portrait (52509170611363) — DONE 2026-05-20 PM. Ad set is now C + E vertical only.
3. ⛔ Do NOT switch Sellers Guide destination URL — CONV is hitting Meta target; FUB gap is upstream attribution, not the lander
4. ⛔ Do NOT activate FUB sellers automation yet — blocked on T&B rebuild + attribution diagnosis
5. Day 5-7 re-read scheduled for ~May 23-25 against the decision tree above

### Pickup notes for next session
- **Tomorrow (2026-05-21) midday pull:** compare against today's account totals snapshot (above) to measure D-pause impact. Specifically watch (a) does E vertical absorb D's freed spend, (b) does account CPL drop back toward $4.74, (c) does C's lead count stay flat or grow.
- Endpoint pattern: `curl -H "Authorization: Bearer $META_INSIGHTS_TOKEN" "https://closings.homegrownpropertygroup.com/api/meta-insights?days=7&level=campaign"` — also takes `level=adset&parent_id=<cid>&parent_kind=campaign` and `level=ad&parent_id=<cid>&parent_kind=campaign`
- Token lives in HGPG - Ads project instructions
- Meta MCP rollout flag still off on all three ad accounts — endpoint is the workaround that actually works now
- Variant E ad IDs for next pull: 52509166115763 (1x1), 52509170611363 (4x5), 52509170629763 (9x16 — the breakout)
- One real Sellers Guide lead in FUB: Daniela Portillo (id 32123). She's the canary for the rebuilt nurture sequence when T&B unblocks it.
- Pipeboard Pro renewal decision before ~May 28. Recommendation: keep it. The fix paid for itself in one session.

### Parked
- Day 5-7 Variant C vs E CPL re-read (~May 23-25) — decision tree above
- T&B: SG-2026 Sellers Guide Nurture automation rebuild
- T&B: pixel event trigger diagnosis + webhook delivery log pull
- FUB IDXRE-B2 sellers and buyers automation activation (BLOCKED on T&B)
- Pipeboard Pro renewal decision (~May 28)
- Geo-farming postcards blocked on Canopy MLS data (May 22 upload deadline — separate workstream)
- Brian to sweep FUB UI for `DELETE-ME-test-data` tagged records when time permits
- Ad set state as of session close: C ACTIVE ($4.22 CPL), E vertical ACTIVE ($2.44 CPL), D + E square + E portrait all PAUSED

---

## Prior session: 2026-05-06 — Brain App MVP shipped 🟢

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
