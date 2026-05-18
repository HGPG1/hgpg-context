<!-- Last Updated: 2026-05-18 (later) -->

# Session Handoff

## Last session: 2026-05-18 (paid social) — Variant E shipped + Sellers Guide CONV mystery solved 🟢

### What got done

**Weekly performance read on New Construction campaign.** Viktor's analysis was numerically correct but framing was off. Real story: CPM only +5% w/w, so the CPL spike ($14.68 → $27.37) is a CTR/CVR saturation problem (single creative against doubled reach), not an auction failure. CTR drop (4.51% → 3.37%) is the leading indicator; CPL is downstream. Zero-lead Mon/Thu is Poisson noise at this volume.

**Variant E shipped.** Dollar-anchor typography concept, no portrait. Hook: "$10,000+ in closing costs, paid by the builder." All 3 sizes built (1080x1080, 1080x1350, 1080x1920; PNG+JPG). 5 copy variants written. Brand-styled brief PDF (5 pages) delivered. Decided no-portrait because all 3 uploaded photos were Work Hard Be Kind shirt = competing personal branding, and IMG_5675 from Drive wasn't accessible in this session. Brian sent spec to Viktor for build. To launch alongside reactivated Variant D in same ad set as C.

**Sellers Guide CONV mystery solved.** Pulled the Events Manager funnel:
- PageView 192 → AssessmentStarted 28 (14.6%) → ScoreCompleted 20 (71% of starters) → Lead 14 (70% of completers)
- Last Lead: 3 days ago (May 15) with red triangle warning
- The lander funnel is actually HEALTHY below the click. 70% form completion of starters is strong. Form friction was NOT the bottleneck — my earlier diagnosis was wrong.
- Top-of-funnel is the issue: only 14.6% of page visitors tap "Start the Score."

**Real cause discovered: Vercel preview domain leaking pixel events.** Viktor's AEM audit confirmed pixel `861295553661596` is healthy and AEM-enabled, BUT it's also receiving events from `charlotte-sellers-guide-vercel.vercel.app` (Vercel preview deployments running with production env vars). Every branch push fires the pixel. This pollutes:
- Event Match Quality (preview traffic = no real users/cookies)
- Lead optimization signal (Meta can't separate real Leads from preview Leads)
- iOS attribution (preview domain not in AEM)

Meta has been optimizing CONV against polluted signal for the entire campaign window. The "Confirm domains that belong to you" warning in Diagnostics (auto-resolved, "no longer detected") was the symptom we missed.

### Deliverables (in /mnt/user-data/outputs/)
- variant-e_1x1_square.png/.jpg
- variant-e_4x5_portrait.png/.jpg
- variant-e_9x16_vertical.png/.jpg
- variant-e_brief.pdf (5 pages)

### Key diagnoses
- New Construction: audience saturation, fresh angle needed → Variant E addresses this
- Sellers Guide CONV: Vercel preview leak corrupting optimization signal → block list + Tech & Builds env-gating

### Open / parked

- [ ] **Brian: add `charlotte-sellers-guide-vercel.vercel.app` to pixel block list** (pixel `861295553661596`). Step-by-step provided in session.
- [ ] **Tech & Builds: env-gate pixel injection** in `charlotte-sellers-guide` Vercel project so pixel only fires on production. Pattern in `META-PIXEL-CAPI-PLAYBOOK.md`.
- [ ] **Brian: cross-check $10K+ Variant E figure** against one current builder MLS listing before Viktor pushes live. If $15K consistent, ping for regenerate.
- [ ] **Viktor: build Variant E in same ad set as C and D**, reactivate D, keep C. Spec sent. Confirm pixel firing + UTMs in FUB lead source data after launch.
- [ ] **Day 1-2 post-fixes:** watch Lead event in Sellers Guide pixel. Expect 24-48hr Meta optimization recovery.
- [ ] **Day 3+ post-fixes:** if CONV still flat after clean signal, THEN do the destination switch (`/home-selling-score/` → `/`). Holding this decision until preview leak is resolved — there's a real chance switch isn't needed.
- [ ] **Day 5-7 (May 23-25):** C/D/E performance read. Pause worst performer, scale winner.
- [ ] **Day 30:** Sellers Guide CONV ad set rebuild with custom conversion baked in (known soft-fail).
- [ ] **Variant F queued** if E underperforms (urgency, comparison, or objection-handled angles).
- [ ] **Decide automation owner for Variant D leads** (Viktor vs Claude CRM project) — still parked.
- [ ] **Brian: form friction fix on /home-selling-score/ is DEPRIORITIZED** — funnel proves it works at 70% completion. Don't change form until preview leak resolved.

### Performance snapshot (week of 2026-05-12 to 2026-05-18) — New Construction
- Spend: $410.52 (+115% w/w)
- Leads: 15 (+15% w/w)
- CPL: $27.37 (+86% w/w; still under $50 target)
- CTR: 3.37% (-25%)
- CPM: $32.81 (+5%) ← key tell: not an auction problem
- Frequency: 2.32 (+12%)

### Sellers Guide CONV snapshot (3 days live, May 15-17)
- Spend: $47.05
- Clicks: 17
- LPVs: 9 (53% LPV rate vs TRAF's 99% — cold traffic on deep-funnel page)
- AssessmentStarted: 28 (across whole 28-day pixel window)
- ScoreCompleted: 20 (71% of starters)
- Lead: 14 historical (mostly TRAF + organic), 0 from CONV in 3 days
- Diagnosis: Vercel preview leak pollution + cold-traffic-on-deep-funnel intent mismatch


## Previous session: 2026-05-18 (late) — Brain hygiene + full status sweep 🟢

### What got done

**CDN staleness audit.** `raw.githubusercontent.com` was caching 5/04 CONTEXT and 5/06 SESSION-HANDOFF. `/api/files` (bearer-authed) showed both were actually fresher (CONTEXT 5/17, handoff had the 5/18 evening entry). Lesson: trust `/api/files` over `raw.githubusercontent.com` for fresh reads; CDN caches for minutes after writes.

**Status sweep across every active project file.** Walked all 25 project files in the brain, surfaced everything mid-flight that CONTEXT.md wasn't showing. Resolved each item with Brian:

| Item | Outcome |
|---|---|
| Buyer Alerts (LoopMessage patch DRAFTED) | Keep parked, wait for clean Items 1+2+3 session |
| Charlotte New Construction (Variant D running) | Variant E under consideration |
| Buyers Guide migration (S1+S2 done, S3+S4 queued) | Brian thought further along; live-site probe + Mac mini `git status` confirmed S3+S4 NOT built. Stays parked. |
| FUB AI Agent week 1 ramp | Humming, daily_send_cap=10 manual approve |
| TM $395 fee toggle | Keep parked |
| DocuSign migration off zipForms | Keep parked, re-evaluate pain before 2026-06-13 |
| Sellers Guide FUB Automation 2.0 | Likely ready, validation gates on first real ad lead from Friday campaign |
| NC Quiz Scoring calibration | Added to recheck list (two dead inputs: commute, firstTimeBuyer) |
| Phone Capture Rate Review | Tomorrow's docket (window opens 2026-05-19) |
| NC "For Builders" footer link | Keep parked |
| Incentives Funnel Phase 2 (general) | Keep on recheck list |
| Incentives Funnel Phase 2 calendar checkpoint | Tomorrow check (calendar target ~2026-06-02) |
| TC Concierge real-life classifications | Brian to share live deals next session for extraction quality review |
| Sherlock 403 on TM | RESOLVED, removed from blockers |
| GitHub auth on Mac mini | RESOLVED, removed from blockers |
| CRM brain stub (`projects/crm.md`) | Recheck list — needs Active campaigns + Recent FUB Work entries backfilled |

### CONTEXT.md rewrite

Reorganized into four explicit buckets so the index is scannable at a glance:
- **What's active right now** — projects actively iterating, with one-line state per project
- **Recently completed** — chronological, last ~2 weeks
- **Parked** — explicit list of things NOT to pick up without a trigger
- **On the docket for next session** — Phone Capture Rate Review, Incentives Funnel calendar checkpoint, TC Concierge real-life review
- **On the recheck list** — lower-urgency items to keep on radar (Incentives Phase 2 design, NC Quiz calibration, CRM brain stub buildout)

Posted via `/api/external/write` (commit `357c723`).

### Buyers Guide forensics

Brian thought Sessions 3+4 might be done. Three checks confirmed they aren't:
1. Live-site probe of `/brian`, `/ashley`, `/map`, `/market-pulse`, `/advisor`, `/admin` — all returned 200 but with identical 12,739-byte SPA-fallback HTML (`lesson #12 — Vercel + Vite SPA fallback caches index.html per URL`)
2. Bundle inspection — page chunks shipped are Home/Calculator/Quiz/Bonuses/Neighborhoods/Checklist/ThankYou/NotFound. No AgentProfile, Advisor, Admin, Map, or MarketPulse chunks.
3. Mac mini `git status` — clean, on `main`, matches `origin/main`. Last commit `16d1e79` is Session 2 quiz fix. No unpushed work, no S3/S4 branches anywhere.

### Pickup notes for next session

- Brain is in sync. Trust `/api/files` over `raw.githubusercontent.com` for fresh reads.
- Tomorrow morning: KTS pause verification → toggle templates 1166/1167 to Shared → review Viktor's automation builds → decide on B2 activation (see 5/18 evening entry below).
- Phone Capture Rate Review window now open (2026-05-19 to 2026-05-26).
- Brian to share live TC Concierge classifications for extraction-quality review.

## Earlier session: 2026-05-18 — KTS contamination shut down, B2 templates shipped, Viktor handoff 🟢

### What shipped today

**Cleanup ops**
- Built `bulk` Automation 2.0 in FUB - manual trigger, pauses Action Plans + 4 target Automations 2.0
- Bulk-applied to 6,625 expired leads (Stage = Expired/Withdrawn). Completed: 6,603/6,625
- Surgically paused 3 lingering KTS automations at the automation level:
  - `*KTS Back to Website` (45 enrolled, the only ACTIVE one)
  - `*KTS Nurture FSBO` (70)
  - `*KTS FSBO - SMS` (52)
- Hit 9 stragglers manually in `*KTS Nurture Buyer`

**B2 templates created via FUB API (Owner Account / userId 12)**
- Template ID 1166: "IDXRE-B2-Seller — Just Checking In"
  Subject: "Just checking in, %contact_first_name%"
  Audience: ~693 leads (IDXRE-Cool|Cold + IDXRE-Seller)
  Tone: Personal, short, no agenda. 4 sentences. Reply-driven CTA.
- Template ID 1167: "IDXRE-B2-Buyer — Market Shifted"
  Subject: "%contact_first_name%, the Charlotte market shifted a bit"
  Audience: ~304 leads (IDXRE-Cool|Cold + IDXRE-Buyer)
  Tone: Market data lead + soft listing offer. Reply-driven CTA.

**Handed off to Viktor**
- Brief to build 2 Automation 2.0s (IDXRE-B2-Sellers, IDXRE-B2-Buyers)
- Both end INACTIVE for Brian review before activation
- Includes audience filters, 3-step structure (send email + 2 tag steps)
- Bonus verification step optional (audience size validation)

### Critical discoveries
1. **Template IDs vs Automation IDs are confusable in emEvents data.** Audit showed "KTS Expired #7/#8/#9" with IDs 583/584/585 - those are TEMPLATE IDs. The real automation is one: `*KTS Nurture Expired` (ID 72). Verified via `getTemplate`.
2. **`getTemplate` API can return empty actionPlans/automations arrays even when relationship exists in UI** (saw this with template 362 → KTS Nurture Buyer). Don't fully trust the API for this lookup.
3. **"Inactive" automations keep processing in-flight enrollments** until paused at automation level OR people removed individually. Critical nuance.

### Pre-bulk baselines (for tomorrow's verification)
- *KTS Nurture Expired: 1,756 enrolled → target tomorrow under 200
- *KTS Expired - SMS: 820 enrolled → target tomorrow under 100
- *KTS Back to Website: 45 enrolled → should be 0 (paused entirely)
- *KTS Nurture FSBO: 70 enrolled → should be 0 (paused entirely)
- *KTS FSBO - SMS: 52 enrolled → should be 0 (paused entirely)

### B2 status: READY TO LAUNCH (post-verification)

Once tomorrow's verification confirms KTS pause completed:
1. Toggle both new templates to Shared (currently isShared: false - API default)
2. Review Viktor's automation builds (screenshots in his report)
3. Verify audience filter is correct or applied at bulk-apply step
4. Activate both automations
5. Bulk-apply audience filter:
   - B2-Sellers: IDXRE-Cool|Cold + IDXRE-Seller, NOT (Y_BAD_EMAIL|Y_IDXRE_UNSUBSCRIBED|Y_SOFT_BOUNCE|IDXRE-Hot|IDXRE-Warm)
   - B2-Buyers: same logic with IDXRE-Buyer
6. Date-stamp tag both automations apply: `IDXRE-2026-05-19-B2` (update if launch date slips)

### Decisions made today
- Pond 9 stays family/personal only - NOT campaign suppression
- Y_UNSUBSCRIBED is Ylopo-era - use Y_IDXRE_UNSUBSCRIBED for current opt-outs
- userId 12 = "Owner Account" - generic placeholder (now in protected list with 6 & 15)
- IDXRE Hot tier outreach PARKED until contamination-aware re-scoring possible
- Hot tier dossier finding generalizes: pool is 70% sellers, NOT primarily buyer-search audience
- B2 angle: personal note for sellers, market-data lead for buyers, short for both
- Both B2 automations end INACTIVE; Brian activates manually after verification

### Open questions
- Did Viktor's audience filtering work in Automation 2.0's UI, or did he build with bulk-apply filtering? (His report will say)
- KTS Automated Valuation Engagement - Ylopo: was it added to bulk step? Worth confirming.
- Template 362 (KTS Nurture Buyer parent) had only 9 enrolled - all hit manually. Worth a future deep dive on what triggered it to enroll expireds.

### Scripts shipped today
- `fub_expired_send_audit.py` - new, per-lead audit from FUB CSV export joined to last N days of emEvents

### Tomorrow's checklist
1. AM: Check KTS automation counts in FUB Admin > Automations
2. AM: Toggle templates 1166 + 1167 to Shared
3. AM: Review Viktor's build screenshots
4. Decide: launch B2 immediately or wait additional day
5. If launching: activate both automations, bulk-apply, monitor for issues
6. Optional: build 5 smart lists in FUB UI (Hot/Warm/Cool/Cold/Triage pond)

## Earlier session: 2026-05-17 — IDXRE-2026-04 scored, tier+segment tagged, Dead triaged

### What got done
- Ran fub_idxre_engagement.py: 1,289 scored leads from pond 16 (1,460 active)
  - Hot 27 / Warm 105 / Cool 239 / Cold 778 / Dead 135
- Built and ran fub_idxre_tier_tag.py: 1,149 leads tagged IDXRE-Hot/Warm/Cool/Cold
- Built and ran fub_idxre_source_segment.py: 1,284 leads tagged IDXRE-Seller (902, 70%) / IDXRE-Buyer (352, 27%) / IDXRE-Unknown (30, 2%)
- Built and ran fub_idxre_dead_cleanup.py: 135 Dead leads moved to new pond 17 "IDXRE - Triage" with bounce-type tags
- Ran rescue scoring: 134 of 135 Dead have phone, 113 marked "RE-INCLUDE B2 + call as backup"
- Ran contamination audit: 92% of pond 16 received non-IDXRE emails during campaign window
- Created pond 17 "IDXRE - Triage" (owner userId 12 Owner Account)

### Scripts shipped (in ~/Documents/hgpg-transaction-manager/fub_cleanup/)
- fub_idxre_engagement.py
- fub_idxre_tier_tag.py
- fub_idxre_source_segment.py
- fub_idxre_dead_cleanup.py
- fub_idxre_dead_rescue_score.py
- fub_idxre_contamination_audit.py
- contamination_summary_from_csv.py

### Major discoveries
- FUB REST API has NO /people/bulkUpdate endpoint - sequential per-person PUT required (0.05s sleep)
- listAutomations + listAutomationsPeople endpoints gated (HTTP 403)
- KTS legacy Action Plans got auto-converted by FUB to Automations 2.0
- IDXRE pool is 70% sellers / 27% buyers - campaign messaging was mismatched
- Top 27 Hot tier leads are 85% sellers (Expired/Withdrawn from MyPlusLeads)

## Earlier session: 2026-05-06 — Brain App MVP shipped 🟢

### What got built
- New Vercel project: `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- New repo: `HGPG1/brain-app` (private)
- Live at: `https://brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Single-user lock: `BRIAN_EMAIL=brian@homegrownpropertygroup.com` allow-list
- GitHub auth: fine-grained PAT scoped to `HGPG1/hgpg-context`
- Round-trip verified: edit file in browser → commit lands on `main`

### Pickup notes
- Brain-app is live and working — use it for any future updates to `hgpg-context`
- Resend API key is in 1Password ("Supabase HGPG Core SMTP")

