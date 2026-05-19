<!-- Last Updated: 2026-05-19 -->

# Session Handoff

## Last session: 2026-05-19 — Variant E launch reconciled, Buyers Guide cleanup pass, brain-app /log endpoint shipped, IDXRE B2 verified 🟢

### What shipped

**Brain reconciliation (Variant E):**
- Variant E launched 2026-05-18 with full scoped package (1:1 + 4:5 + 9:16 creatives, IMG_5675, $10K+ closing-cost hook, "Learn More" CTA, `utm_content=variant-e-10k`).
- Brain entries fixed: `projects/charlotte-new-construction.md` (status 🟡→🟢, read schedule logged, Variant E in Done/shipped), `marketing.md` (new dedicated Variant E section with brief/creative/UTM/benchmark), `CONTEXT.md` (active campaigns line + Recently completed).

**Buyers Guide code cleanup (4 commits to charlotte-buyers-guide):**
- `11a0a4a4` — bump Last Updated header on `api/fub-lead.ts` 2026-05-12 → 2026-05-13
- `2b405da3` — bump Last Updated header on `api/exit-intent/submit.ts`
- `2e722609` — bump Last Updated header on `api/bonus/unlock.ts`
- `9c3168c7` — strip orphan `ADVISOR_MODE_GUIDE.md` reference from `api/_lib/agents.ts` (file never existed in repo)
- Retracted earlier false copy-bug claim on `/admin` page (it correctly renders `<h1>Admin</h1>` — I misread the minified bundle on first pass)

**Brain-app feature shipped:**
- `/api/external/log` endpoint live at `brain.homegrownpropertygroup.com` (commit `2b73da1a` in HGPG1/brain-app). Read-only git-log proxy for any HGPG1 repo, same auth/validation pattern as `/api/external/read`. Optional filters: `path`, `since`, `until`, `ref`, `per_page` (cap 100), `page`. POST or GET. Returns sha, short_sha, message, full_message, author_name, author_email, author_date. Closed the gap where Claude sessions needed `gh` from Brian's Mac for commit history.
- `operations.md` updated with full 5-endpoint surface documentation
- `projects/brain-app.md` updated with `/api/external/read` + `/api/external/log` sections, auth-model heading bumped to "all four endpoints"

**Buyers Guide brain reconciliation:**
- Full Session 3 commit-by-commit history captured via the new /log endpoint (8 commits since 2026-05-13)
- Surfaced **2 commits that hadn't been documented anywhere** before today:
  - `2af5340` (2026-05-19 06:20 ET) — Post-Session-3 polish: phone format, FUB tags, `/owner` route dropped from public allowlist
  - `58687cc` (2026-05-19 08:29 ET) — Fix FUB quiz tag keys to match canonical Quiz values. **Silent bug: 4 of 5 buyer-type tags + 1 of 4 lifestyle tags weren't being applied for ~7 days (Session 2 ship → 2026-05-19).** Any Quiz lead captured 2026-05-12 to 2026-05-19 missing buyer-type/lifestyle FUB tag.
- New brain file `projects/buyers-guide-advisor-notes.md` (commit `f380f02a`) — full 40-talking-point inventory pulled live from `AdvisorNote-C4olMM97.js`, organized by buyer-facing page (/quiz, /neighborhoods, /strategy, /checklist). 1 of 12 blocks approved this session ("Setting up the strategy conversation"). Review pass paused.
- `projects/buyers-guide.md` corrections: fixed Lesson #19 (agent allowlist baked in, but agent DATA fetched from Supabase via `/api/agent/by-slug` and `/api/agent/default` — earlier claim of "no /api/agents endpoint" was wrong). Added Lessons #22 (silent enum-map drift post-mortem) + #23 (intentional two-tier allowlist divergence between client and server).

**IDXRE B2 verification (no code shipped, all live API probes):**
- Brain `projects/idxre-2026-04.md` and `CONTEXT.md` updated with verification results (commits `3ea8114b`, `2dd51167`).
- KTS pause held — all 27 Hot leads frozen at 2026-05-17T17:28-31 UTC, no activity since. Strong indirect signal that the bulk-pause Automation 2.0 worked.
- All B1 tier counts match brain exactly: Hot=27, Warm=105, Cool=239, Cold=778, Pond 17 Dead=135.
- **B2 build verified live:** Templates 1166 + 1167 exist (created 2026-05-18 18:23 UTC). Automations 336 (Sellers) + 337 (Buyers) exist (created 2026-05-18 20:15-20:29 UTC).
- **Narrative drift found:** brain said Viktor was building B2 automations. Live data shows `createdById: 1 (Brian McCarron)` on both. Either Brian built them himself or Viktor was working under Brian's login.

### What's parked

**Buyers Guide:**
- AdvisorNote review pass: 1 of 12 blocks approved, 11 remaining. Pickup at "Seller's market notes" next. Source-of-truth file in repo is `src/data/advisor-content.ts`. Approval marker format: `✅ APPROVED YYYY-MM-DD` next to block heading. Rework marker: `🔴 REVIEW:` prefix on a line.
- Mobile smoke test of AdvisorMode (iPhone/iPad) — test plan in advisor-notes file. Entry URL pattern: `buyersguide.homegrownpropertygroup.com/advisor/<page>`. Persistent device-level localStorage flag. Tap "Exit advisor mode" to clear.
- Session 4 (optional): Interactive Map + Market Pulse + bonus PDFs. ~1.5 hrs.
- Manus app takedown after S4.
- ⚠️ **FUB quiz tag backfill potential** — leads captured 2026-05-12 → 2026-05-19 may be missing buyer-type / lifestyle tags. Bg_quiz_results.buyer_type and lifestyle_priority are intact in Supabase; FUB tags are what's missing. Worth a one-off backfill script if you want clean historical data.

**IDXRE B2 launch gates (UI-only, can't be done via API):**
1. Verify audience filters on Automations 336 + 337: `IDXRE-Cool OR IDXRE-Cold + IDXRE-Seller` (336); same + `IDXRE-Buyer` (337). Hot/Warm exclusion on both.
2. Date-stamp tag baked in (`IDXRE-2026-05-XX-B2`).
3. Final copy eyeball on subjects + bodies (templates 1166/1167 read live this session, body content captured in `projects/idxre-2026-04.md` verification section).
4. Flip automations from draft → live.
5. Pick launch day: Tue-Thu this week = today/tomorrow/Thursday.

**Variant E observation window:**
- Day 3-4 first read: 2026-05-21 → 2026-05-22 (Thu-Fri)
- Day 5-7 decision point: 2026-05-23 → 2026-05-25 against Variant C's $4.63 CPL benchmark
- Success bar: E within 1.5x C's CPL at day 3-4 (≤ ~$7 CPL)

**Open from earlier sessions today (preserved):**
- Pipeboard token blocker — `PIPEBOARD_API_TOKEN` env var still needs to be added to Vercel TM project before the Meta Ads dashboard becomes useful. The Meta Ads drill-down session today shipped UI; the data source is still blocked on the token.

### Pickup notes for next session

- **`/api/external/log` endpoint is live and bearer-protected.** Use it to fetch commit history for any HGPG1 repo when reconciling brain entries. Same token, same pattern as `/read`. Operations.md documents the full surface.
- **AdvisorNote review file** lives at `projects/buyers-guide-advisor-notes.md`. The 11 unreviewed blocks are listed in source order; "Seller's market notes" is next.
- **IDXRE B2 is unblocked from a build standpoint.** Only UI-side gates remain. Brian can flip the automations live whenever he's done with the audience-filter verification.
- **The Viktor narrative on IDXRE B2 needs reconciliation** — was Viktor actually involved in the B2 build, or was that a brain memory artifact from an earlier plan? Worth a one-line note in idxre-2026-04.md once clarified.
- **Phone Capture Rate Review** — run window opened 2026-05-19 (today), still untouched. Query plan in `projects/new-construction-phone-capture-rate-review.md`.
- **TC Concierge real-life extraction review** — Don's been running real deals; classifications/extractions need a quality check.

## Previous session: 2026-05-19 — Meta Ads dashboard drill-down shipped 🟢

### What shipped
- TM /meta-ads now supports campaign → ad set → ad drill-down navigation. Click any non-unknown row to drill in. Breadcrumb at top for navigation back up. Ad level is terminal.
- Hide unknown status toggle in the tab row. Defaults ON. Count chip shows when unknown rows exist (e.g. "Hide unknown (12)"). Unknown rows are deleted/archived objects with no current Pipeboard metadata — usually noise.
- API extended: `/api/meta-insights` takes optional `level` (campaign|adset|ad) + `parent_id` + `parent_kind`. Backwards-compatible (no params = campaign-level as before).
- Server-side 60s response cache keyed by `${level}:${parentKind}:${parentId}:${days}` to absorb rapid drill-down clicks without burning Pipeboard's 200/hour rate limit.
- Fail-soft metadata: if Pipeboard rejects the get_adsets / get_ads tool calls, status pills fall back to "unknown" but the insights table still renders.
- Two files edited, both deployed cleanly on `dpl_CyGSmEx4DLUC1FFNu3LaVjaeCoDv` (commit `334ba6c`):
  - `app/api/meta-insights/route.ts`
  - `app/meta-ads/MetaAdsDashboard.tsx`

### Earlier in the same session (mid-day 2026-05-19)
- Built and shipped the initial /meta-ads dashboard (campaign-level only at that point). See `projects/transaction-manager.md` Recent Ships > Meta Ads Dashboard.
- Hit the Vercel "Sensitive" env var gotcha: PIPEBOARD_API_TOKEN was marked Sensitive on creation, which causes the runtime to receive empty strings for the var. Fix: delete + re-add WITHOUT the Sensitive toggle. Diagnosed via a temporary `/api/debug-env` route (now removed). Memory updated with this gotcha for future env-var debugging.

### Follow-ups during the same session

**Unknown statuses at adset/ad drill-down (FIXED).** Initial deploy showed all rows as "UNKNOWN" status at ad set and ad levels. Root cause: Pipeboard's MCP tools (`get_adsets`, `get_ads`) silently ignore scoping args like `campaign_id` / `adset_id` — they return the first N unscoped objects. So lookups for the actual drilled-into IDs missed the cache and fell back to "unknown". Fix: switched to fetching account-wide metadata maps (one per level) cached for 5 min, look up by ID locally. Account is small enough that wide fetches are cheap.

**Drill-down showing wrong ads (FIXED).** After the metadata fix, drilling into an ad set showed ALL ads in the account (with correct statuses now). Root cause: Pipeboard's MCP insights tool also ignores the `filtering` array param. It was returning every ad in the account with activity in the date range. Fix: switched to scoping via `object_id` (mirrors Meta's native `/<adset_id>/insights` URL-path scoping pattern). Kept `filtering` array as belt-and-suspenders. Verified working: drilling into a real campaign now shows only the 8 ads in that campaign, with correct statuses, leads, CPL.

**Vercel "Sensitive" env var gotcha rediscovered.** During initial /meta-ads setup, PIPEBOARD_API_TOKEN was added with the Sensitive toggle ON (which is default in some Vercel UI flows). Sensitive vars come into the runtime as empty strings (length 0, no error). Diagnostic pattern: `/api/debug-env` route that reports `present`, `length`, `truthy` for the var. Fix: delete + re-add the var with Sensitive OFF. Added to memory.

### Diagnostic patterns worth keeping
- `/api/debug-env` (already removed): reports env var presence/length/truthy without leaking values. Re-add with same shape if any future env-var-related not-configured error.
- `/api/debug-meta-ads` (still in repo as of commit `36cdabb`, needs removal): tests Pipeboard tool list + tool-call variants to diagnose which scoping params actually work. Useful template for other MCP integrations where you need to figure out what the wrapper actually honors.

### Pickup notes for next session
- Drill-down assumes Pipeboard's `get_adsets` and `get_ads` tools accept Meta-style scoping args (`campaign_id`, `adset_id`). If those don't scope server-side, the function still works (we filter the metadata client-side via the map), but it'll pull more data than needed. If you see ad set views taking >10s, check Pipeboard tool docs and add explicit filtering arg.
- Insights `filtering` param uses Meta Graph syntax: `[{"field":"campaign.id","operator":"EQUAL","value":"123"}]`. Pipeboard should pass this through. If drill-down returns empty data, the filtering shape might need adjusting per Pipeboard's preferences — fall back to client-side filter as a safety net.
- The 60s response cache is per-lambda-instance. Cold starts get a fresh cache. For Brian's single-user pattern this is fine.

### Open for future enhancements (parked)
- Custom date range picker (currently only 7/14/30/90)
- Scheduled performance digests (weekly email to Brian with top-line numbers)
- Click-through to Meta Ads Manager from each row (open the actual campaign in Ads Manager UI for editing)
- Trend sparklines on KPI cards
- "Compare to previous period" toggle

## Earlier session: 2026-05-19 — Buyers Guide Session 3 verified shipped 🟢

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

## Earlier session (2): 2026-05-19 — Meta Ads dashboard added to TM 🟢

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

## Earlier session (3): 2026-05-19 — Variant E scoping confirmed, creative render deferred 🟡

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
