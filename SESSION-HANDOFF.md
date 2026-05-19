<!-- Last Updated: 2026-05-19 -->

# Session Handoff

## Last session: 2026-05-19 — Buyers Guide Session 3 verified shipped 🟢

### What happened
- Brian flagged that Buyers Guide Session 3 was complete, but the brain still listed S3 as queued/parked.
- Pulled live state from production (`buyersguide.homegrownpropertygroup.com`) to verify what actually shipped. Confirmed: production was last deployed today 2026-05-19 13:45 UTC (09:45 ET). Session 3 IS live and accepting real traffic.
- Inspected the deployed JS bundle (`index-DHp9PF4a.js` and `agentDetection-R8ThZYk3.js`) to enumerate what was actually built.

### What's live (verified by bundle inspection + API probe + DB query)
- **Agent slug routes**: `/brian`, `/brenda`, `/ashley`, `/taylor` — each parses slug, persists to `localStorage` as `homegrown_assigned_agent_slug`, propagates via `AssignedAgentProvider` context
- **Reserved-routes table** in `agentDetection.ts` keeps `quiz, calculator, rent-vs-buy, neighborhoods, strategy, checklist, bonuses, thank-you, advisor, admin, agent-dashboard, api` from being misclassified as agent slugs
- **`AgentProfile` component** renders assigned agent's info in-page
- **Advisor Mode**: `/advisor/<path>` entry, `homegrown_advisor_mode` localStorage flag, persistent banner "Advisor Mode · Talking points visible to you only" with "Exit advisor mode" button
- **`AdvisorNote` components** embedded throughout buyer-facing pages, only render when `isAdvisorMode === true`
- **`assignedAgentSlug` propagation** to `/api/fub-lead`, `/api/exit-intent/submit`, `/api/bonus/unlock`
- **Server-side resolution**: each handler looks up `bg_agents WHERE slug = ?`, writes `bg_contacts.assigned_agent_id`, calls FUB `assignLeadToAgent`, returns `{success: true, contact: {...}, assignedAgent: {id, slug}}`
- **No new server endpoints** — agent list is statically baked into the bundle (hardcoded `[brian, brenda, ashley, taylor]`). No `/api/agents` collection endpoint. Tradeoff documented as lesson 19 in `projects/buyers-guide.md`.

### Verified real-traffic activity
4 leads with named `assigned_agent_id` captured today between 10:10 and 12:36 UTC, hitting agent IDs 30001 (Brian), 30003 (Ashley) via slug resolution. Full table in `projects/buyers-guide.md` "Verified live activity" section.

### Brain updates pushed this session
- `projects/buyers-guide.md` — status flipped to `🟢 Sessions 1+2+3 complete`, full Session 3 plumbing section added, Sessions 3 listing moved from Pending → Recent build history, 3 new lessons appended (#19 baked-in agent list, #20 reserved routes table, #21 localStorage attribution model). Commit `470432e3`.
- `CONTEXT.md` — "Last updated" bumped to 2026-05-19; "Recently completed" gained Buyers Guide S3 line; "Parked" entry rewritten to reflect only S4 outstanding. Commit `bf7dac68`.

### What's still pending on Buyers Guide
- **S3 post-deploy verification** (now in `projects/buyers-guide.md` Pending): confirm `/admin` and `/agent-dashboard` reserved routes render their intended UI (or were left as placeholders for a future session). Confirm advisor mode banner displays correctly on iPad and iPhone. Confirm `AdvisorNote` copy is reviewed before any agent uses it in a buyer consult.
- **Session 4 (optional)**: Interactive map, Market Pulse, bonus PDFs (~1.5 hrs)
- **Manus app takedown**: after S4

### Pickup notes for next session
- One test row was written to `bg_contacts` during this verification session (`probe-discardme@example.invalid`, id 13) and cleaned up via direct Supabase DELETE. No residue.
- If Brian wants the brain to track Session 3 commit SHAs (vs just the high-level "shipped"), need access to `HGPG1/charlotte-buyers-guide` git log — the project file lists the S2 commits but not S3. Worth fetching via gh CLI next time we're touching the repo.
- The Pipeboard token blocker from the TM Meta Ads dashboard session is still outstanding — Brian needs to add `PIPEBOARD_API_TOKEN` to Vercel TM project env vars before that dashboard becomes useful.

## Previous session: 2026-05-19 — Meta Ads dashboard added to TM 🟢

### What shipped
- New TM route `/meta-ads` — internal Meta Ads performance dashboard, replacing manual CSV exports from Ads Manager (Pipeboard token now lives server-side, never in browser)
- New TM API route `/api/meta-insights` — server-side Pipeboard JSON-RPC proxy. Reads `PIPEBOARD_API_TOKEN` from env, resolves Pipeboard tool names once per process lifetime (1hr TTL), fail-soft on the campaigns call (status pills fall back to "unknown" if Pipeboard rejects that tool but insights still renders)
- Brian-gated end-to-end: server component redirects non-Brian sessions, API route returns 403 to non-Brian. Defense in depth so the token never gets exercised by anyone else
- Sidebar nav links added (`app/layout.tsx` desktop, `components/MobileNav.tsx` mobile), gated on `user.email === brian@...` matching the `/agent` pattern
- All 5 files committed via brain commit API. Final deploy `dpl_4LxM3f61sWuhUS9pZ1F7jXy7Ui1T` READY at production

### Files
- `app/api/meta-insights/route.ts` (new)
- `app/meta-ads/page.tsx` (new, server component)
- `app/meta-ads/MetaAdsDashboard.tsx` (new, client component, Tailwind matching TM conventions)
- `app/layout.tsx` (edit: nav link)
- `components/MobileNav.tsx` (edit: nav link)

### Architecture notes
- The UI is a 1:1 port of the Ads-project prototype (`meta_ads_dashboard.html`) with the token panel stripped. CTR normalization, lead extraction from `actions[]`, status pill colors, sort logic, CSV export logic all lifted verbatim
- Pipeboard endpoint: `https://meta-ads.mcp.pipeboard.co/` (JSON-RPC 2.0, Bearer auth, 200/hour rate limit). Account `act_31445287`
- 4 date ranges: 7 / 14 / 30 / 90 days (mapped to `last_7d`/`last_14d`/`last_30d`/`last_90d` Pipeboard presets)
- 5s refresh throttle on the Refresh button. Tab switches always force-fetch (no throttle)
- Page renders nothing useful until `PIPEBOARD_API_TOKEN` env var is set on Vercel — error box will display `PIPEBOARD_API_TOKEN not configured on server`

### Pickup notes for next session
- **Blocking: `PIPEBOARD_API_TOKEN` not yet set on Vercel TM project.** Brian to add via Vercel dashboard (Production scope minimum; Preview if testing on branch deploys). Token format starts with `pk_`. Get from pipeboard.co/api-tokens. Once set, the page works immediately on next request (tool name cache is per-process, so cold-start picks it up).
- One intermediate deploy errored (`dpl_8trNCqM7m2Yc4mHYLJPqgayeqCtz`) because `page.tsx` committed before `MetaAdsDashboard.tsx`. The 5 commits were sequential single-file calls so intermediate states existed briefly. Subsequent deploy fixed it. Not a problem in practice but a reason to bundle related files into a single push when possible (or commit them in dependency order).
- Future enhancements parked: ad set / ad creative drilldown, custom date range picker, scheduled performance digests (e.g. weekly email to Brian), multi-account support if Brian ever runs ads from a second account.
- Token rotation procedure if Pipeboard token expires: refresh at pipeboard.co/api-tokens, replace `PIPEBOARD_API_TOKEN` in Vercel, trigger a redeploy (or wait for the next push; tool-name cache invalidates per cold start).

## Earlier session: 2026-05-19 — Variant E scoping confirmed, creative render deferred 🟡

### What happened
- Brian asked whether Variant E (New Construction Incentives campaign) had been scoped here. It had — in an earlier same-day session.
- Confirmed Variant E is fully scoped and the brand-styled brief PDF is already shipped at Drive ID `1U6Voq92rqVSFDIrTFiborWCBKFbiLqk3` (`variant-e_brief.pdf`, created 2026-05-18 19:44 UTC, in Brian's My Drive root).
- Brief covers: dollar-anchor angle ($10K+ closing-cost hook), audience/placement table mirroring Variant C, 5 copy variants with headlines + descriptions, creative dimension specs (1080x1080, 1080x1350, 1080x1920), UTM table with `utm_content=variant-e-10k`, and the day-3/5/7 test plan against C and D.
- CTA in shipped brief is "Learn More" (not "Get Offer" — earlier draft in this thread proposed Get Offer but the locked brief uses Learn More; go with what shipped).

### What's still parked
- **Creative render is NOT done.** The brief references three PNG/JPG creatives (1:1, 4:5, 9:16) built around photo IMG_5675 (black polo, kitchen island, warm closed-mouth smile — same photo used in Variant D).
- IMG_5675 could not be located via Drive search this session. Folder ID `14G9ECCBAGgJJjau-IglmtDIOBPKlwouF` (51-photo lifestyle folder per brain memory) returned empty on multiple queries. Either the folder ID rotated, permissions changed, or the connector isn't indexing images in that folder.
- Three new photos uploaded to project this session (IMG_5681, IMG_5711, IMG_5720) — all Work Hard Be Kind shirt, disqualified per brand rule against competing personal branding on clothing.

### Pickup notes for next session
- **First move:** get IMG_5675 to Claude. Options that worked / didn't:
  - Drive search by name = empty
  - Drive search by parent folder ID = empty
  - Direct file ID would work via `Google Drive:download_file_content` if Brian pastes a link
  - Or Brian uploads IMG_5675 directly to the chat
- **Once photo is in hand:** render three creatives matching the locked brief spec — 1080x1080 square, 1080x1350 portrait, 1080x1920 vertical. Output both PNG (quality) and JPG (size fallback). Composition per brief: photo top with navy gradient, single hero number `$10,000+`, locality strip (Indian Land · Fort Mill · Waxhaw · Lake Wylie), HGPG + Real Broker lockup, CTA pill.
- **Audit before Brian launches:** confirm pixel ID `1880396459290092`, URL destination, UTM tags match brief table, Special Ad Category: Housing applied.
- **Day 3-4 first read; day 5-7 decision point** against Variant C's CPL. Success bar: E within 1.5x C's CPL at day 3-4.

### Audit note for marketing.md
- Variant E status in `marketing.md` should reflect: brief shipped, creative render pending, not yet launched. If the file currently lists Variant E as "launched" or "live," that's wrong — only the brief is done. Next session should reconcile.

### Drive connector note
- Multiple name-based and parent-based Drive searches returned empty this session. May be a connector indexing issue rather than missing files. Worth a smoke test next session — try `name contains 'IMG'` against a known-good folder to confirm the connector is actually searching image files.

---

## Last session: 2026-05-06 — Brain App MVP shipped 🟢

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
