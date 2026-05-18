<!-- Last Updated: 2026-05-18 -->

# Session Handoff

## Last session: 2026-05-18 — Variant E shipped, Sellers Guide CONV diagnosed, week-3 read 🟢

### What got done
- **Weekly performance read** on Viktor's analysis (New Construction). Agreed CPL spike is real ($14.68 → $27.37, +86%) but disagreed with framing: CPM only +5%, so this is CTR/CVR saturation not auction failure. Reach grew 83% on a single creative against a limited interest pool — textbook signature.
- **Variant E creative built** end-to-end (3 sizes: 1080x1080, 1080x1350, 1080x1920; PNG+JPG). Concept: dollar-anchor typography card, no portrait. Hook: "$10,000+ in closing costs, paid by the builder." Brand-compliant (navy/steel, no green, Sansita+Inter, Real Broker LLC compliance line).
- **5 copy variants written** (Direct, Buyer-math, Local specificity, Objection-handled, Short) each with headline and description within Meta char limits.
- **Variant E brief PDF shipped** (5 pages): why-now, audience+placement table, copy variants, creative specs with previews, UTM table, test plan day-by-day, pre-launch checklist.
- **Sellers Guide CONV diagnosed** (lander: `/home-selling-score/`). 9 LPVs / 0 leads in 3 days. Three diagnoses ranked by likelihood: (1) intent mismatch — cold traffic on deep-funnel page with 20-question quiz before any conversion event; (2) form friction — 6 required fields including phone + full address; (3) known optimization-event soft-fail (raw Lead instead of custom conversion, parked for Day-30 rebuild). Recommended actions: pull AssessmentStarted + ScoreCompleted from Events Manager today to triangulate which diagnosis is right, make phone optional this week, decide on destination switch by day 7-10.

### Deliverables (in /mnt/user-data/outputs/)
- variant-e_1x1_square.png / .jpg
- variant-e_4x5_portrait.png / .jpg
- variant-e_9x16_vertical.png / .jpg
- variant-e_brief.pdf (5 pages, brand-styled)

### Key decisions made this session
- $10K is the anchor number (conservative — holds across builders). To be cross-checked against at least one current MLS listing before launch.
- Variant E uses typography lead, not portrait. All 3 uploaded photos were Work Hard Be Kind shirts (competing personal branding = brand rule violation). IMG_5675 from Drive (Variant D asset) was the only clean option but not in project files; chose no-portrait to ship today instead of blocking on Drive access.
- Variant E should launch ALONGSIDE reactivated Variant D, both in same ad set as C.
- Sellers Guide CONV stays live for now (3 days is below Meta Learning Phase minimum). Don't kill, but diagnose the funnel today.

### Open / parked
- [ ] **Brian: verify $10K+ figure against a current builder MLS listing** before uploading Variant E
- [ ] **Brian: pull AssessmentStarted + ScoreCompleted event counts** from Events Manager → Pixel 861295553661596 (Sellers Guide). That tells us which CONV diagnosis is correct.
- [ ] **Decide automation owner for Variant D leads** (Viktor vs. Claude CRM project) — still parked from prior session
- [ ] **Tech & Builds project: make phone field optional** on `/home-selling-score/` form (highest-ROI single change for CONV campaign)
- [ ] **Day 5-7 read after Variant E launches:** compare C vs D vs E on CTR, freq, CPL; pause worst performer
- [ ] **Day 30 rebuild** for Sellers Guide CONV ad set (custom conversion baked in from creation) — known soft-fail
- [ ] **Variant F queued** if E underperforms — angle candidates already noted (urgency, comparison, objection-handled)

### Disagreements with Viktor's analysis (for context)
- Zero-lead Mon/Thu is Poisson noise at this volume, not actionable
- CTR drop is the leading indicator; CPL is downstream
- C and D produced identical CPL ($26 vs $28) on 7-8 conversions each — that's within noise, not a meaningful difference
- TRAF campaign delivering "0 leads" is correct behavior (it's optimized for traffic, not leads) — should not be flagged as a concern

### Performance snapshot (week of 2026-05-12 to 2026-05-18)
- Spend: $410.52 (+115% w/w)
- Leads: 15 (+15% w/w)
- CPL: $27.37 (+86% w/w, still under $50 target)
- CTR: 3.37% (-25%)
- CPM: $32.81 (+5%)  ← key tell: it's not an auction problem
- Frequency: 2.32 (+12%)


## Previous session: 2026-05-17 — IDXRE-2026-04 scored, tier+segment tagged, Dead triaged 🟡

### What got done
- Ran fub_idxre_engagement.py: 1,289 scored leads from pond 16 (1,460 active)
  - Hot 27 / Warm 105 / Cool 239 / Cold 778 / Dead 135
- Built and ran fub_idxre_tier_tag.py: 1,149 leads tagged IDXRE-Hot/Warm/Cool/Cold
- Built and ran fub_idxre_source_segment.py: 1,284 leads tagged IDXRE-Seller (902, 70%) / IDXRE-Buyer (352, 27%) / IDXRE-Unknown (30, 2%)
- Built and ran fub_idxre_dead_cleanup.py: 135 Dead leads moved to new pond 17 "IDXRE - Triage" with bounce-type tags (19 Y_BAD_EMAIL, 114 Y_SOFT_BOUNCE, 2 Y_IDXRE_UNSUBSCRIBED)
- Ran rescue scoring: 134 of 135 Dead have phone, 113 marked "RE-INCLUDE B2 + call as backup"
- Ran contamination audit: 92% of pond 16 (1,184 leads) received non-IDXRE emails during the campaign window. 3,491 contamination events.
- Created pond 17 "IDXRE - Triage" (owner userId 12 Owner Account)

### Scripts shipped (in ~/Documents/hgpg-transaction-manager/fub_cleanup/)
- fub_idxre_engagement.py - pulls emEvents, joins pond, scores tiers
- fub_idxre_tier_tag.py - per-person PUT with mergeTags
- fub_idxre_source_segment.py - source-based Seller/Buyer/Unknown tagging
- fub_idxre_dead_cleanup.py - tag and move Dead leads to triage pond
- fub_idxre_dead_rescue_score.py - local CSV analysis, recovery prioritization
- fub_idxre_contamination_audit.py - finds non-IDXRE sends to pond 16 members
- contamination_summary_from_csv.py - read existing audit CSV, no API calls

### Major discoveries
- FUB REST API has NO /people/bulkUpdate endpoint - sequential per-person PUT required (0.05s sleep, 250 req/10s with X-System-Key)
- listAutomations endpoint gated (HTTP 403) - all Automation 2.0 audit must be manual UI
- KTS legacy Action Plans got auto-converted by FUB to Automations 2.0 and are STILL ACTIVELY SENDING to pond 16 members
- IDXRE pool composition is 70% sellers / 27% buyers / 3% unknown - campaign messaging mismatched
- Top 27 Hot tier leads are 85% sellers (Expired/Withdrawn from MyPlusLeads)
- Top contaminating campaigns: 1613/1612/1602 are historical FUB/Beacon mass sends (already over, no future risk), KTS Expired #7/#8/#9 are active Automation 2.0 (must pause)

### Key clarifications captured to memory
- Y_UNSUBSCRIBED tag is Ylopo-era artifact, NOT current-program unsub. Use Y_IDXRE_UNSUBSCRIBED for current campaign opt-outs
- Pond 9 "Brian Excluded Contacts" is family/personal, NOT marketing suppression
- userId 12 is "Owner Account" - generic placeholder, not a real person
- House accounts (NEVER for active assignment): 6, 12, 15. Real agents only: 1, 18, 21

### Open Monday tasks (in priority order)
- [ ] FUB UI: Pause or add pond-16 exclusion to every active KTS-named Automation 2.0 (KTS Expired #6-#12, KTS Back to Website #5-#8, KTS Nurture Buyer/Seller, KTS Reviews, KTS Auto Valuation, KTS New Buyer)
- [ ] Build 5 smart lists in FUB UI: IDXRE-Hot, IDXRE-Warm, IDXRE-Cool, IDXRE-Cold, IDXRE-Triage pond
- [ ] Decide B2 angle (currently parked): listing recovery for sellers vs. buyer-search refresh
- [ ] Optionally tag the 27 Hot leads with DNC/Bounce flags for safer outreach planning

### Hold list
- B2 send remains PARKED until KTS automations are stopped or excluded
- Hot tier outreach PARKED until contamination-aware re-scoring decides which clicks were actually IDXRE vs. KTS

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
