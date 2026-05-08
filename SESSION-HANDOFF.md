<!-- Last Updated: 2026-05-08 -->

# Session Handoff

## Last session: 2026-05-08 — Strategic queue refresh + brain hygiene 🟡

### What got done
- Verified Pixel/CAPI rollout status across all consumer sites:
  - ✅ charlotte-new-construction (original implementation)
  - ✅ charlotte-buyers-guide (shipped 2026-04-29, full 6-event chain + dedup)
  - ✅ charlotte-sellers-guide (shipped 2026-05-07, pixel ID 861295553661596, CAPI mirror, FUB UTM custom fields, AssessmentStarted + Lead + ScoreCompleted + QuizStarted + QuizCompleted, Meta-bypass path)
  - ❌ Signature (still pending — small open item)
  - ❌ TM + marketinganalyzer (intentionally skipped — internal/PIN-gated audiences)
- Pulled forward the locked queue from the 2026-05-06 strategic review (was lost from prior handoff)

### Pickup notes for next session

**Locked queue (post Brain App + Pixel rollout)**
1. ⏳ Sellers guide Pixel QA — close out before treating as fully done:
   - QA Path A: organic walkthrough of /home-selling-score confirming PageView + AssessmentStarted + Lead + ScoreCompleted in Test Events with browser+server dedup
   - QA Path B: Meta-bypass walkthrough with ?utm_source=meta confirming phone-required + skip 6-digit verify + FUB lead with meta-bypass tag
   - Register ScoreCompleted as Custom Conversion in Events Manager (Lead category, scoped to URL contains home-selling-score)
   - Wire Meta Lead Ads to FUB native integration for Instant Form path
   - After QA passes: remove META_TEST_EVENT_CODE env var from Vercel
2. 🎯 Buyer Alerts — first piece of MLS Dashboard Suite (#8 v2, ~4-5 hr)
   - LLM criteria parser (natural language to structured filters)
   - Matcher cron against fresh mls_property
   - Alert delivery via existing LoopMessage iMessage pipeline from TM
   - UX bottleneck is criteria capture flow — frictionless input is the make-or-break
3. 📊 Market Intelligence — second piece of MLS Dashboard Suite (~3-4 hr, leans on existing TM Recharts)
4. 🔍 Off-market / Expireds Finder — third piece (~5-6 hr, replaces the external scraper Brian is paying for)
5. 🤖 FUB AI Agent rebuild — biggest project, gated on Sendblue eval first
   - Sign up for Sendblue free sandbox
   - Test FUB integration (dashboard → Integrations → Follow Up Boss)
   - Book sales demo for AI Agent and Blue Ocean plans
   - Get LoopMessage second-sender quote for comparison
   - 30-45 min scoping session writing real spec (channel mix, decision authority, stale-lead scope, compliance)

**Smaller open items**
- Signature Meta Pixel + CAPI (~30-45 min, copy buyers guide pattern)
- MLS Grid Media kickoff (parked alongside .net→.com migration)

**Parked**
- #3 hgpg-transaction-monitor (audit checklist required)
- #6 deal notes UI (after CMA battle testing)
- MLS Grid Media
- #14 .net→.com migration
- Listing Report Portal MLS enrichment (low ROI — portal stable since 2026-04-03)

### Why MLS Dashboard Suite is the next big lever
MLS Grid is live with 2.57M rows replicating via cron, but only the CMA Engine consumes it. The infrastructure is already paid for, so every new tool built on top is cheap marginal value. Buyer Alerts is the highest deal-velocity tool in that suite.

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
