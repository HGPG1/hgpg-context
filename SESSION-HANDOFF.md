<!-- Last Updated: 2026-05-08 -->

# Session Handoff

## Most recent session: 2026-05-08 — CMA adjustment engine bugs, full sweep 🟢

### State as Brian leaves the desk

Massive ship day on `hgpg-cma-tool`. Eight PRs merged to main, all live on `cma.homegrownpropertygroup.com`. Engine is materially better than this morning. Real verification still pending on Brian's desk after the road trip — see "Pickup checklist" below.

### CRITICAL — Verification still owed before sending CMAs to clients

The math fixes layered today need eye-on verification with real reports. Do NOT send CMAs to clients until Brian walks through:

1. **Open the Candlestick 3a84f12c draft** at `cma.homegrownpropertygroup.com/seller/adjust`. Manually input GLA=3805, finished basement=1250, unfinished basement=1000 (rough estimate, agent can refine). Confirm Wyngate's adjustment now shows three lines summing ~+$31K (vs pre-PR-33 +$6K). PMV should rise from $932K to ~$960-985K range — closer to Brian's $1M instinct.
2. **Re-save Cressingham** (5022 Cressingham). Subject is single-story no-basement. Basement fields should be 0. PMV stays at $659K (control test).
3. **Re-save Tyndale** (yesterday's tight-confidence Somerset). Should still produce strategies tightly capped within band.

If anything looks off, ping web Claude with what you see. Don't ship to clients until verification clean.

### What shipped today (8 PRs in sequence)

| PR | Bug | Description | Status |
|----|-----|-------------|--------|
| #25 | Bug 5 | Confidence bounds invariant (Low ≤ anchor ≤ High; fallback to anchor*0.93/1.08 if invalid) | ✅ live |
| #26 | Bug 6 | Active comp weighting drops to 0.03 when only 1 Sold; "very-low" confidence + banner when 0 Sold | ✅ live |
| #27 | Bug 4 | Anchor sanity check; flag if no comp's adjusted price within ±5%/$50K of weighted PMV | ✅ live |
| #28 | Bug 7 | Subject self-listing exclusion; address parts compare with postal_code gate, suffix-tolerant | ✅ live |
| #29 | Bugs 8+9 | Distance-tiered comp selection (subdivision → 1mi → 3mi → zip fallback) + per-comp distance/direction display | ✅ live |
| #30 | Bug 10 | Cross-state deprioritization; 0.75x weight + 5pt similarity penalty + amber pill flag for cross-state Sold comps | ✅ live |
| #31 | Bugs 1+3+11 | Per-feature parity scoring (data-first via `below_grade_finished_area`, remarks regex fallback) + outlier counter-cluster protection + strategy band cap | ✅ live |
| #32 | hotfix | Removed `below_grade_unfinished_area` reference from COMP_SELECT (column doesn't exist in MLS Grid sync data) | ✅ live |
| #33 | Bug 2 | GLA / basement separation per Fannie Mae URAR Form 1004; subject form gets 3 sqft fields; comp side reads structured `above_grade_finished_area` + `below_grade_finished_area`; rate table adds $25/sqft finished basement, $10/sqft unfinished | ✅ live |

### Key engine learnings (for future Claude Code prompts)

- **Carolina/Canopy MLS structure**: `mls_property` has `above_grade_finished_area` and `below_grade_finished_area`. **No `below_grade_unfinished_area` column** — comp-side unfinished basement is unrecoverable from feed. Subject-side is manual input only. `living_area = above_grade + below_grade` (sum, includes basement).
- **Wyngate ground truth (verified via Supabase)**: 7026 Wyngate Place, 29720, sold ~$910K. above_grade=3,266, below_grade_finished=1,689, total=4,955. Year built 2020 (NOT a 2010s home — no age premium gap vs subject).
- **Candlestick subject GLA confusion was the cause of $1M-vs-$932K disagreement**: with subject reported as 5,055 total (basement included) and Wyngate 4,955 total, the engine saw near-parity sqft. Reality: subject GLA is ~3,805 vs Wyngate's 3,266 = subject is LARGER above-grade by 539 sqft. PR 33 fixes this.
- **Verify Postgres columns before claiming they exist** (lesson from PR #32 hotfix). Web Claude told Code `below_grade_unfinished_area` was available; Code shipped, comp search 500'd in production. Always run column check via Supabase MCP before specifying.
- **MLS field naming gotcha**: Supabase columns are snake_case, MLS Grid spec is PascalCase, mapper translates. PostgREST silently returns empty when filtering on nonexistent column — error surfaces during SELECT instead.
- **Diagnostic patches are infrastructure**: PR #23 (yesterday) verbose RPC error format saved hours diagnosing the wrong-zip subject auto-fill timeout. Do NOT remove during cleanup.

### Wrong-zip subject auto-fill (RESOLVED yesterday + earlier today)

Two-part fix landed across yesterday evening through this morning:
- **Composite index** `idx_mls_prop_postal_streetnum` on `mls_property(postal_code, street_number)` — fixed strict-path RPC timeout (Postgres code 57014).
- **Statement timeout** raised to 30s for `authenticated` role — absorbs cold lambda + pooler latency variance.
- **Single-column index** `idx_mls_prop_streetnum` added in PR #24 (this morning) — composite index leads with postal_code so was unusable for wrong-zip lookup. Single-column btree fixes 19.6s seq scan → 8ms warm cache.
- **PR #24 wrong-zip fallback** lets agents type wrong zip; picker offers correct zip candidate; ZIP field auto-corrects on pick. Smoke test path: type "504 Redwine St" with ZIP "28210" — picker should offer 28110 candidate.

### Bugs/work remaining (priority order)

1. **PR 6 (Bug 3, outlier symmetry)** — LOWEST priority. Already partially addressed by counter-cluster protection in PR #31. Skip unless something specific surfaces.
2. **Enhancement 1 (autosave on /seller/adjust)** — currently saves only on "Generate Packet". Brian flagged this as UX risk. Should autosave drafts every change, with explicit publish on Generate. Lower priority but worth shipping after engine verification.
3. **Possible Bug 12 (build year adjustment)** — Brian observed Candlestick may benefit from age premium. Engine doesn't appear to apply year-built adjustment. Worth investigating after PR 33 verification — may already be acceptable if subject and Wyngate are both 2019-2020 builds.
4. **Possible Bug 13 (time-of-sale adjustment)** — appraisers add for older comps. Engine doesn't appear to apply. Investigate if Wyngate sold 6+ months ago and market has moved.

### Test reports to use as regression baselines

- `8023175d-37c8-45d7-95da-b9245abe4761` — 6022 Candlestick Lane (Bent Creek, Lancaster SC, luxury 6BR with finished basement + in-law-suite). Primary test case for feature parity, GLA/basement separation.
- `3a84f12c-dc91-407b-ac93-5efaa5c4d564` — Candlestick draft (created 18:49 today). The active draft for PR #33 verification.
- 5022 Cressingham (Indian Land 29707, 5BR/4BA, single-story no basement, healthy tier-1 same-subdivision data). Control test — math fixes should not move PMV from $659K.
- 215 Tyndale Ct (Somerset 28277, tight-confidence). Strategy compression test.
- 504 Redwine St (Monroe 28110, the wrong-zip case). Cross-state policy test.
- 5601 Medlin Rd (Monroe 28110, 1.84 acres, semi-rural). Thin-data tier-3 test.

### Brian's instincts to address in upcoming work

- **Strategy spreads**: was a real concern this morning, fixed by Bug 11 cap. Verify on desk that Aspirational stays under High and Event stays above Low across all 5 test reports.
- **$1M Candlestick instinct**: PR 33 should bring it within range. If post-PR-33 still feels low, investigate build year + time-of-sale adjustments (Bugs 12+13 noted above).
- **Cross-state lender concerns**: handled via 0.75x weight + amber pill flag in PR #30. Brian's "Waxhaw NC across state line" instinct was right; engine respects it but doesn't exclude.
- **Comp selection appraiser-grade**: handled via Fannie Mae B4-1.3-08 cascade (subdivision → 1mi → 3mi → zip) in PR #29. Distance + direction shown per comp.

### Mac mini Claude Code MCP setup

Still needs same MCP+GitHub PAT setup as iMac (which was completed yesterday evening). When Brian works on Mac mini next:

```
claude mcp add --transport http --scope user --header 'Authorization: Bearer NEW_PAT_HERE' github https://api.githubcopilot.com/mcp/
```

Generate separate PAT named "Claude Code MCP - Mac mini". Same fine-grained scope=user pattern as iMac.

### Default `claude` to dangerous mode

iMac alias added today: `alias claude="claude --dangerously-skip-permissions"` in `~/.zshrc`. Use `\claude` to bypass alias if ever needed. Mac mini still needs same alias added when Brian works there.

### Pickup checklist for next session

1. **Verify PR #33 on the desk** — open Candlestick 3a84f12c draft, manually input GLA 3,805 / finished basement 1,250 / unfinished basement 1,000. Confirm three sqft adjustment lines on Wyngate summing ~+$31K. PMV should rise from $932K to ~$960-985K range.
2. **Re-save Cressingham as control** — basement fields zero, PMV stays at $659K, no behavior change.
3. **Re-save Tyndale** — strategy compression test, Aspirational should cap inside the band.
4. **If verification clean**, the engine is essentially ready for client-facing CMAs. Ship Enhancement 1 (autosave) next as the UX gap that's most worth closing.
5. **If verification surfaces issues**, ping web Claude with what you see — likely Bug 12 (year-built) or Bug 13 (time-of-sale) territory.
6. **Mac mini Claude Code MCP setup** when Brian works there next.
7. Eventually decide which other repos get CLAUDE.md ship-it specs. TM is highest impact next.

### Other repos for CLAUDE.md ship-it specs (deferred)

- `hgpg-transaction-manager` (next priority)
- `brain-app`
- `charlotte-new-construction-nextjs`
- `south-charlotte-report`
- `homegrown-property-group-site`

---

## Previous session: 2026-05-06 — Brain App MVP shipped 🟢

### What got built
- New Vercel project: `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- New repo: `HGPG1/brain-app` (private)
- Live at: `https://brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Single-user lock: `BRIAN_EMAIL=brian@homegrownpropertygroup.com` allow-list
- GitHub auth: fine-grained PAT scoped to `HGPG1/hgpg-context`, contents:write only
- Round-trip verified: edit file in browser, commit lands on `main` with author `brian@homegrownpropertygroup.com`

### Brain App write API (added 2026-05-07 evening)
- POST `https://brain.homegrownpropertygroup.com/api/external/write`
- Authorization: Bearer token (in 1Password as "HGPG Brain Write Token")
- JSON body: `{ path, content, message?, branch? }`
- Authored as brian@homegrownpropertygroup.com
- 1MB content cap. Blocked paths: `.git/`, `.github/workflows/`, `.vercel/`, `node_modules/`, `.env*`, `package-lock.json`
- Used by web Claude sessions to update brain files without manual paste

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
- Brain-app is live and working, use it for any future updates to `hgpg-context`
- Resend API key is in 1Password ("Supabase HGPG Core SMTP")
- Brain-app local dev: `cd ~/brain-app && npm run dev` on Mac mini (work machine)
- Brain-app local on iMac: same setup, repo at `~/Developer/brain-app` if rebuilt, otherwise needs fresh `gh repo clone HGPG1/brain-app` + `npm install` + `cp env.example .env.local`
