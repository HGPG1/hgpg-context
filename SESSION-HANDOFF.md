<!-- Last Updated: 2026-05-17 -->

# Session Handoff

## Last session: 2026-05-17 — Brain capability audit + project instructions refresh 🟢

### What shipped

**Project instructions regenerated:**
- HGPG - CRM & Leads project instructions rewritten with full brain API surface (write + commit endpoints), GitHub App auth model, Lauren Williams removed, FUB MCP corrected as hosted (not Claude-Desktop-only), Twilio A2P language updated to "fully deprecated" matching Tech & Builds, standing rules added (MCP-first, Brain-first, HGPG-is-a-team), Terminal formatting block added.
- HGPG - Home Base project instructions rewritten with same brain API surface, build workflow updated to reflect that web/sandbox Claude can now commit directly (commit API, 2026-05-14), team list tightened to canonical names, standing rules added.

**Memory cleanup (entry #13 split into focused entries):**
- Old monolithic entry #13 (~8000 chars of mixed content) deleted along with patches #14 (1Password supersede) and #15 (web-sandbox-cannot-push supersede).
- Replaced by 14 focused entries (#13-#26), each under 500 chars, each on a single topic: people & phones, Transaction Manager, Sellers Guide, legacy decommissioned, on the horizon, repos, platforms, working principles, gotchas part 1, gotchas part 2, output rules, terminal formatting, cowork & dispatch, purpose & context.
- Memory entries #10 and #12 corrected in place (commit API documented, web/sandbox push restriction removed).
- Net memory state: 26 entries, all focused, no "supersedes" patches, room for ~4 more.

**CONTEXT.md refresh:**
- Old version (last updated 2026-05-15) was missing brain API surface, listed tc.homegrownpropertygroup.com as live (it's decommissioned), missing 8+ active projects (FUB AI Agent, team-dashboard, team-photo-sync, propstream-caller, new-construction sub-projects), still listed resolved blockers (GitHub PAT rotation, Mac Mini gh CLI auth).
- New version adds: "How to update the brain" section (write API + commit API + read API + manual paths), corrected active project list (11 active + claude skills), standing infra rules pulled inline, decommissioned section, recently completed bumped with brain-app/GitHub App/Resend SMTP/NeverBounce milestones.
- Commit `dfd98b5`.

### Open / parked

- **Tech & Builds project instructions** still contains the stale "Web/sandbox sessions cannot push to HGPG1 app repos. Brian pushes from his Mac via gh CLI" line. Most-likely-to-be-wrong-in-practice project. Worth regen for symmetry with CRM & Leads + Home Base.
- **Brain doesn't have a projects/leads.md or projects/crm.md.** CRM & Leads project instructions reference fetching leads.md "if present" but it doesn't exist. Active FUB campaign state has nowhere to live except memory or handoff. Worth creating a stub.
- **Token sitting in plaintext memory** is a minor risk. Brain write token is embedded in memory entry #12. Rotation playbook documented in brain-app.md. Awareness item only.
- **CONTEXT.md will need pruning over time.** Currently inline-lists 11 active projects plus sub-projects. Once the new-construction sub-project set settles or rolls up under the parent, can consolidate to keep this file scannable.

### Pickup notes for next session

- Memory is in clean state for the first time in months. If a future session needs to add memory, the 30-entry cap leaves ~4 slots. New entries should be focused and topic-scoped, NOT consolidated dumps.
- The brain write API + commit API work is now properly documented across: project instructions (Tech & Builds done, CRM & Leads done, Home Base done), memory (#10, #12), brain-app.md project file, and CONTEXT.md. Future sessions should not need re-education.
- raw.githubusercontent.com URLs may CDN-cache for minutes after a brain write. For fresh state during a session that just wrote, prefer `GET /api/files/<path>` with bearer auth.

---

## Previous session: 2026-05-17 — June 2026 geo-farming postcards built 🟡

### What shipped
- Three print-ready PDFs for Geosential June 1 drop:
  - `HGPG_postcard_bent_creek_2026-06.pdf`
  - `HGPG_postcard_bridgemill_2026-06.pdf`
  - `HGPG_postcard_queensbridge_2026-06.pdf`
- Spec: 5.5 x 8.5 in jumbo postcard, portrait, 0.125 in bleed, 300 DPI sRGB, two pages per file (front + back)
- All three QR codes scan-verified to farm-specific URLs with campaign tags (`bc-2026-06`, `bm-2026-06`, `qb-2026-06`)
- Full design system established for touches 2-12: typographic hero + grayscale photo, navy headline band, three stat tiles, steel intro panel, recent sales table, navy Quick Take callout, contact row, navy compliance footer with combined HGPG + Real Broker LLC lockup
- New brief at `marketing/farms/june-2026-introduction.md`

### Design decisions made
- **Portrait, not landscape** — brief specified portrait; design absorbed the content well; postal sorting handles slightly worse than landscape but acceptable. Test landscape on touch #2 (July) and compare scan rates.
- **Grayscale photo treatment** (not duotone) — cleaner editorial / market-brief feel, no blue tint, brand chrome pops on top. Stock photo placeholders for v1 (suburban brick home for Bent Creek, golf course for Bridgemill, gray craftsman for Queensbridge); swap for real neighborhood photos if Brian sources them before May 22.
- **Combined HGPG + Real Broker LLC lockup** on both front bottom strip (large, navy bg) and back compliance footer (small, navy-deep bg). Satisfies Real Broker compliance requirement.
- **Tagline kept as "Growth Starts Here, At the Roots."** — Brian raised "Your Neighborhood Realtors" as an alternative; deferred because (a) NAR Realtor trademark + Real Broker team-not-brokerage compliance risk, (b) we haven't earned the right on touch #1. Plan: keep this tagline through touch #4, drop tagline touches #5-8, revisit "Your Neighborhood Realtors" for touches #9-12 once data has done the credibility work.
- **Brian's title** = "Team Lead, Home Grown Property Group" everywhere on the card. NOT Broker/Owner (per project standing rule). Brief had it wrong; corrected silently.
- **Body text bumped from 28pt to 32pt** on both intro panel and Quick Take after Brian flagged readability. Body panel height expanded from 320 to 380, Quick Take from 200 to 230.

### MLS data findings (HGPG Listing Reports + MLS Supabase, `wdheejgmrqzqxvgjvfee`)
- Brief stats 2-of-3 accurate. Bridgemill brief was significantly wrong: brief said $820K / 19 / 98.1%, real trailing-12-month combined is $690K / 39 / 98.6%. Brief was likely citing partial-year or older data.
- **Bridgemill segments into two markets**: Single Family Residences (26 sales, $584K-$1.175M, median $780K) and Townhomes (10 sales, $390K-$486K, median $438K). Reporting a combined median misrepresents both audiences.
- **Decision: Bridgemill card uses SFH-only data.** Townhome segment is 4.6x smaller in volume and a different farming play. SFH-only stats: $780K / 26 / 98.6%. "Single Family Residences" segment label rendered under the recent sales section header. Townhomes excluded from the recent sales rows.
- **Bent Creek (Lancaster 29720 with some 29707)** and **Queensbridge (29707)** are 100% SFH. No segmentation needed. Brief stats matched reality exactly.
- **Header changed from "Q1 2026 Market Snapshot" to "Trailing 12-Month Snapshot"** — brief stats were trailing-12-month not Q1, so the header was misleading.
- **Section title changed from "Recent Sales, Last 90 Days" to "Recent Sales In Your Neighborhood"** — Queensbridge only had 3 sales in 90 days, needed to pull a 4th from January, so "Last 90 Days" was false advertising.
- **Data error caught**: `3145 Hartson Pointe Drive` shows close price $2,695 on 2026-04-27. Almost certainly a lease record miscategorized as a sale or a missing-zeros entry. Filtered with `close_price >= 100000`. Worth a Canopy ticket if not corrected upstream.

### Open / parked
- **Geosential upload** — PDFs are ready, Brian to upload to print.geosential.com by May 22 for June 1 drop. Quantities: Bent Creek 430, Bridgemill 600, Queensbridge 315, total 1,345.
- **Hero photos** — currently stock-with-grayscale placeholders. Brian to decide whether to swap to real neighborhood photos (Bent Creek gate, Bridgemill clubhouse, Queensbridge home) before upload. Stock-with-grayscale works as a design system if not swapped.
- **EHO icon** — custom-drawn approximation, not the official Equal Housing Opportunity vector. Acceptable for v1, swap to official mark when convenient.
- **Lander attribution check** — verify that `?c=bc-2026-06` / `bm-2026-06` / `qb-2026-06` query params land in FUB as lead source / source detail when forms are submitted on farm landing pages. Route to HGPG-Tech project to confirm.
- **Touch #2 (July) test plan** — flip orientation to landscape on all three farms, hold everything else constant, compare scan rates touch #1 vs touch #2 per farm. Three within-subject comparisons instead of cross-subject.

### Pickup notes for next session
- Source files at `/home/claude/postcards/` are ephemeral (session container). If a rebuild is needed in a future session: re-pull MLS data via Supabase MCP (same query patterns above), re-fetch fonts (Sansita Bold + Inter Variable from Google Fonts), re-extract HGPG mark from `1.png`/`3.png` and combined lockup from `7.png`. All build scripts are reproducible from the brief.
- Bridgemill SFH-only farming pattern should carry forward to touches 2-12. If Brian wants a Townhome variant added later, it gets its own card design with its own data.
- If Brian sources real neighborhood photos between sessions, regen heroes (`heroes_final.py`) and rebuild cards (`build_cards_final.py`). Photos should land at high resolution (>2000px wide), landscape orientation; grayscale treatment is applied automatically.

---

## Earlier session: 2026-05-06 — Brain App MVP shipped 🟢

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
