<!-- Last Updated: 2026-05-18 -->

# Session Handoff

## Last session: 2026-05-18 — KTS contamination shut down, B2 unblocked 🟢

### What shipped today
- **Built and bulk-applied `bulk` Automation 2.0** that pauses Action Plans + 2 target Automations 2.0 (`*KTS Nurture Expired`, `*KTS Expired - SMS`) for any enrolled person
- **Tested on Jenny Austin (personId 30569)** - confirmed Nurture Expired flipped Running → Paused
- **Bulk-applied to 6,625 leads** (FUB filter: Stage = Expired/Withdrawn). Processing in background, expected complete by 2026-05-19 morning
- **Built `fub_expired_send_audit.py`** - per-lead audit script that joins FUB CSV export to last N days of emEvents (in `~/Documents/hgpg-transaction-manager/fub_cleanup/`)

### Critical discovery: template IDs vs automation IDs
The IDs 583, 584, 585, etc. that appeared in our 2026-05-17 contamination audit as `*KTS Expired #7, #8, #9` are **email template IDs, NOT automation IDs**.

Verified via `getTemplate`:
- Template 583 `*KTS Expired #14` → parent: `*KTS Nurture Expired` (automation ID 72)
- Template 531 `*KTS Automated Valuation Engagement 5` → parent: `*KTS Automated Valuation Engagement - Ylopo` (automation ID 47)

So the actual list of contaminating Automations 2.0 is ~3, not 6+. All emails like KTS Expired #6/#7/#8/#9 etc. come from the single `*KTS Nurture Expired` automation.

### Pre-bulk baselines (for tomorrow's verification)
- `*KTS Nurture Expired`: 1,756 enrolled
- `*KTS Expired - SMS`: 820 enrolled
- Target tomorrow: both under 200/100 respectively. Anything above ~300/200 = bulk didn't process fully.

### Open question worth investigating
- Template 362 `*KTS Nurture Buyer #13 - blog` sent 78 emails to 42 leads in our expired pool but has no parent Action Plan or Automation 2.0 listed in `getTemplate`. Orphaned? Sent manually as mass action? Worth checking before B2 launches.

### B2 status: UNBLOCKED (once bulk completes)
With KTS contamination paused, B2 send is no longer at risk of contamination compounding. Open decisions:
- Seller-angle for 693 Cool+Cold IDXRE-Seller leads (listing recovery messaging)
- Buyer-angle for 304 Cool+Cold IDXRE-Buyer leads (refined search/market refresh)
- 20 IDXRE-Unknown parked for manual review

### Tomorrow checks (before launching B2)
1. Verify `*KTS Nurture Expired` count is near zero (under 200)
2. Verify `*KTS Expired - SMS` count is near zero (under 100)
3. Spot-check 5-10 leads across the bulk audience - automations widget should show Paused
4. Investigate orphaned template 362 if not already explained
5. Confirm `*KTS Automated Valuation Engagement - Ylopo` was added to bulk step (action item from this session)

### Updated brain memory entries
- userId 12 = "Owner Account" (generic placeholder, owns ponds 14-17)
- Pond 17 = "IDXRE - Triage" (owned by userId 12)
- FUB REST API has NO /people/bulkUpdate - use per-person PUT
- listAutomations + listAutomationsPeople are gated (403)
- IDXRE Hot tier is 85% sellers, contradicting initial buyer-search framing
- 92% of IDXRE pool (1,184 leads) was contaminated by non-IDXRE sends during B1

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
- Top contaminating campaigns: 1613/1612/1602 are historical FUB/Beacon mass sends (already over), KTS Expired #7/#8/#9 are active Automation 2.0 (must pause)

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

### Pickup notes
- Brain-app is live and working — use it for any future updates to `hgpg-context`
- Resend API key is in 1Password ("Supabase HGPG Core SMTP")
