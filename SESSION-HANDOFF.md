<!-- Last Updated: 2026-05-08 -->

# Session Handoff

## Last session: 2026-05-08 (PM) — Team Dashboard scaffolded + Tab 1 built 🟡

### What got built
- New repo scaffolded locally: `hgpg-team-dash` (Next.js 16, Tailwind v4, Supabase SSR auth, Recharts ready)
- Spec committed to brain at `projects/team-dashboard.md` (commit f1582a1)
- Tab 1 (Inventory + Showings) end-to-end:
  - `/inventory` grid filtered by office key `CAR118249224`
  - Status filter pills (Live / Closed 90 / All / Canceled+Expired)
  - Sync freshness banner reading from `mls_sync_state.last_synced_at`
  - `/inventory/[listingId]` drilldown with showing list, ListTrac platform views, key dates, banner when listing isn't onboarded into the Listing Report Portal
- Magic link auth + email allow-list via `TEAM_EMAILS` env var
- Brand-correct UI: Cooper Hewitt body, Sansita display, navy/steel/light-steel/off-white tokens, status pills (signature brown for Under Contract, navy for Active, etc.)
- Production build verified clean, no warnings, all 9 routes resolved
- Tarball at `/mnt/user-data/outputs/hgpg-team-dash.tar.gz` ready for Brian to download + push from Mac

### Key data discoveries
- **Office key** is `CAR118249224` — NOT R04075 (which is Canopy display ID, doesn't exist in the MLS Grid feed)
- Active listings count at build: 3 Active, 1 Under Contract, 1 Pending, 42 Closed lifetime, 14 Canceled, 2 Expired
- Active+UC+Pending: Society Hill Rd (Vessie listing, $200K), Lamington Dr ($625K Pending), Mallard Crossing ($295K), Butters Way ($689,999 UC), Stratton Farm Rd ($600K Active 405-day)
- Address composition required — `unparsed_address` is null on most rows, build from `street_number + street_name + street_suffix`
- mls_property `listing_id` has `CAR` prefix; `listings.mls_number` does not. Strip on join: `REPLACE(mls_property.listing_id, 'CAR', '') = listings.mls_number`
- MLS Grid Media table exists but has zero rows synced (parked work item). Hero photos fall back to portal `listings.hero_photo_url` then placeholder
- All 6 active HGPG members on office key: Brian, Ashley (×2 records), Brenda, Taylor, Lauren (closings), Vessie

### Pickup notes for next session

**Step 1: Brian pushes the scaffolded repo from Mac**

    cd ~/Documents
    tar -xzf ~/Downloads/hgpg-team-dash.tar.gz
    cd hgpg-team-dash
    gh repo create HGPG1/hgpg-team-dash --private --source=. --remote=origin
    git add -A
    git commit -m "feat: initial scaffold + Tab 1 inventory + showings"
    git push -u origin main

**Step 2: Vercel project setup**
- Import repo from GitHub at vercel.com/new under team `team_FietQPKCmnyioG2n0FdteQCV`
- Env vars to set:
  - `NEXT_PUBLIC_SUPABASE_URL` = `https://wdheejgmrqzqxvgjvfee.supabase.co`
  - `NEXT_PUBLIC_SUPABASE_ANON_KEY` = (from Supabase project HGPG Listing Reports + MLS)
  - `TEAM_EMAILS` = `brian@homegrownpropertygroup.com,ashley@homegrownpropertygroup.com,brenda@homegrownpropertygroup.com,taylor@homegrownpropertygroup.com,closings@homegrownpropertygroup.com`
- Add domain (proposed: `team.homegrownpropertygroup.com` — confirm with Brian; alternatives `office.` or `inside.`)

**Step 3: Supabase Auth config**
- Add `https://team.homegrownpropertygroup.com/**` and `http://localhost:3000/**` to redirect URLs on the `wdheejgmrqzqxvgjvfee` project
- Confirm Resend SMTP is wired (it should be — same project as HGPG Core)

**Step 4: Smoke test then build Tab 2**
- Magic link round-trip from Brian's email
- Verify all 5 active+UC+pending listings render with data
- Verify Lamington drilldown shows showing data (it has portal row + ListTrac data)
- Then start Tab 2 (Market Intelligence) build

### Open follow-ups specific to team dashboard
- **Vessie Lopez** appears on office key but isn't in the team roster. Confirm whether her listings should display unconditionally or be filtered out. Current build shows everything tied to `CAR118249224`.
- **MLS Grid Media sync** still parked — until it runs, hero photos only appear for the ~7 listings already in the Listing Report Portal
- **Subdomain choice** — `team`, `office`, or `inside`?

---

## Locked queue (post Brain App + Pixel rollout)

1. ⏳ Sellers guide Pixel QA — close out before treating as fully done:
   - QA Path A: organic walkthrough of `/home-selling-score` confirming PageView + AssessmentStarted + Lead + ScoreCompleted in Test Events with browser+server dedup
   - QA Path B: Meta-bypass walkthrough with `?utm_source=meta` confirming phone-required + skip 6-digit verify + FUB lead with meta-bypass tag
   - Register ScoreCompleted as Custom Conversion in Events Manager (Lead category, scoped to URL contains home-selling-score)
   - Wire Meta Lead Ads to FUB native integration for Instant Form path
   - After QA passes: remove `META_TEST_EVENT_CODE` env var from Vercel
2. 🔄 Team Dashboard — Tab 1 BUILT today, awaiting deploy. Tab 2 (Market Intelligence) is next session
3. 🎯 Buyer Alerts — first piece of MLS Dashboard Suite (~4-5 hr) — still in queue but moved behind Team Dashboard
4. 🔍 Off-market / Expireds Finder — (~5-6 hr, replaces external scraper)
5. 🤖 FUB AI Agent rebuild — gated on Sendblue eval (signup, FUB integration test, sales demo, LoopMessage 2nd-sender quote, real spec)

### Smaller open items
- Signature Meta Pixel + CAPI (~30-45 min, copy buyers guide pattern)
- MLS Grid Media kickoff (parked alongside .net→.com migration)

### Parked
- #3 hgpg-transaction-monitor (audit checklist required)
- #6 deal notes UI (after CMA battle testing)
- MLS Grid Media
- #14 .net→.com migration
- Listing Report Portal MLS enrichment (low ROI — portal stable since 2026-04-03, dashboard now covers the team-facing need anyway)

---

## Prior session: 2026-05-08 (AM) — Strategic queue refresh + brain hygiene 🟢

### What got done
- Verified Pixel/CAPI rollout status across all consumer sites:
  - ✅ charlotte-new-construction (original implementation)
  - ✅ charlotte-buyers-guide (shipped 2026-04-29, full 6-event chain + dedup)
  - ✅ charlotte-sellers-guide (shipped 2026-05-07, pixel ID 861295553661596, CAPI mirror, FUB UTM custom fields, AssessmentStarted + Lead + ScoreCompleted + QuizStarted + QuizCompleted, Meta-bypass path)
  - ❌ Signature (still pending — small open item)
  - ❌ TM + marketinganalyzer (intentionally skipped — internal/PIN-gated audiences)
- Pulled forward the locked queue from the 2026-05-06 strategic review (was lost from prior handoff)

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
  - Rate limit went from 2/hr (Supabase default) to 30/hr (Resend default), can be raised
  - This affects ALL apps using this Supabase: TM, CMA, TC Concierge, brain-app
- Supabase project renames for hygiene:
  - `ioypqogunwsoucgsnmla` → "HGPG Core"
  - `wdheejgmrqzqxvgjvfee` → "HGPG Listing Reports + MLS"
  - `fkxgdqfnowskflgbuxhm` → "HGPG Signature + Relocation"
  - `ngdrliyjtqcwhhfrbxao` → "HGPG FUB Integration"

### Deferred / Phase 2 for brain-app
- iPhone smoke test (CodeMirror + iOS soft keyboard scroll behavior)
- Cooper Hewitt self-hosted (currently falling back to system sans)
- File rename and delete
- Diff view before save
- Cross-file search
