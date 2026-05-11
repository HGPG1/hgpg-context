<!-- Last Updated: 2026-05-11 -->

# Session Handoff

## Most recent session: 2026-05-11 — CMA engine math sweep complete + autosave 🟢

### State as Brian closes the day

Engine bug queue from the original 13-bug sweep is fully drained. Plus autosave shipped. Plus a bonus rate-tuning fix (Bug 14) for over-aggressive time-of-sale appreciation surfaced via real-world Cressingham verification.

`cma.homegrownpropertygroup.com` is in a materially different place than 24 hours ago. Three days of bug work (2026-05-07 through 2026-05-11) closed.

### PRs shipped today (2026-05-11)

| PR | Bugs | Description |
|----|------|-------------|
| #34 | Enhancement 1 | Autosave on /seller/adjust with editVersion counter + 500ms debounce. Status flips draft→presented on PDF export (not button click). |
| #35 | Bugs 6+12+13 | Outlier symmetry refactor + singleton-outlier pass (cap weight 0.25× when adjustedPrice >50% above/>40% below cluster median AND similarity <0.65 AND ≥1 quality flag) + time-of-sale appreciation (default 5.0%, +halfRate% at 91-180d, +fullRate% at 181-365d, stale-sale flag and exclusion >365d). |
| #36 (in-flight) | Bug 14 | Lower default appreciationRatePerYear from 5.0% to 3.0% calibrated to Charlotte metro 2024-2025 cooling market. Fix narrative mislabel where time-of-sale adjustments were rendering as "newer vintage" instead of explicitly time-of-sale. |

### Earlier today (sessions 2026-05-08 + 2026-05-11 morning)

All 11 original engine bugs shipped across PRs #25-#33:
- #25 Bug 5: PMV bounds invariant (Low ≤ anchor ≤ High; fallback anchor*0.93/1.08)
- #26 Bug 6 (original): Active comp weight drops to 0.03 when only 1 Sold; very-low confidence + banner when 0 Sold
- #27 Bug 4: Anchor sanity check; flag if no comp adjustedPrice within ±5%/$50K
- #28 Bug 7: Subject self-listing exclusion via parsed-parts compare + zip gate
- #29 Bugs 8+9: Distance-tiered cascade (subdivision → 1mi → 3mi → zip) + per-comp distance/direction display
- #30 Bug 10: Cross-state deprioritization (0.75× weight + 5pt similarity penalty + amber pill)
- #31 Bugs 1+3+11: Per-feature parity scoring (data-first via below_grade_finished_area, remarks regex fallback) + outlier counter-cluster protection + strategy band cap
- #32 hotfix: Removed below_grade_unfinished_area from COMP_SELECT (column doesn't exist in MLS Grid sync data; cost a 5-minute production outage)
- #33 Bug 2: GLA/basement separation per Fannie Mae URAR Form 1004; subject form gets 3 sqft fields; comp side reads structured columns; new $25/sqft finished + $10/sqft unfinished defaults

### Real-world verification status

**Candlestick (6022 Candlestick Ln, Bent Creek SC, luxury 6BR)**: PMV moved from $1,172K (broken math) → $932K (post-PR-33) → $1,020K (post-PR-35, Cutter Court correctly excluded as singleton outlier). $1,020K is defensible — Wyngate-anchored cluster math from 5 remaining comps. Brian's $1M instinct on this property was directionally correct. Report shippable to a real seller.

**Cressingham (5022 Cressingham, Indian Land SC, 5BR ranch)**: PMV moved from $659K (yesterday) → $704K (post-PR-35 with 5% appreciation default). Investigation showed math was mechanically correct (5046 Cressingham, 6430 Stirling, 6715 Kaci all 272-354 days old, each correctly getting 5% bumps) BUT 5% rate is too aggressive for Lancaster County 2025. Real Indian Land appreciation per Movoto/Redfin/Rocket is -1% to +3% YoY. Bug 14 lowers default to 3.0%. Post-Bug-14 expected PMV ~$680-685K, much more honest.

**Tyndale (215 Tyndale Ct, Somerset 28277)**: Not re-verified today after PR #35. Will need recheck after Bug 14 lands.

### Key engineering lessons captured today

1. **Dual-Supabase architecture confusion**: Code defaulted to querying MLS project (`wdheejgmrqzqxvgjvfee`) for `cma_reports` and almost proposed a destructive migration. Reality: `cma_reports` lives in Core (`ioypqogunwsoucgsnmla`), 43 rows + RLS open via `cma_reports_open_all` policy. MLS data lives in Listing Reports project. Saved as Code memory: "diagnose, don't create."
2. **Verify Postgres columns before specifying** (lesson from PR #32 hotfix). Web Claude told Code `below_grade_unfinished_area` existed. Code shipped without checking. Comp search 500'd in production. Now standing rule: column check via Supabase MCP before specifying any new field.
3. **Status enum is `draft | presented | archived`** (not `draft | published` from generic UX spec). 'Presented' is canonical post-packet value. Use that naming in future specs.
4. **Carolina MLS field structure (verified, do not re-question)**:
   - `above_grade_finished_area` = GLA, appraisal-relevant
   - `below_grade_finished_area` = finished basement
   - `living_area = above_grade + below_grade` (SUM, includes basement)
   - **`below_grade_unfinished_area` does NOT exist** — comp-side unfinished basement is unrecoverable from feed. Subject-side is manual input only.
5. **Wyngate ground truth** (verified via Supabase 2026-05-08): 7026 Wyngate Place, 29720. above_grade=3,266, below_grade_finished=1,689, total=4,955. Year built 2020 (NOT a 2010s home — no age premium gap vs Candlestick subject).
6. **Diagnostic patches are infrastructure**: PR #23's verbose RPC error format saved hours diagnosing wrong-zip subject auto-fill timeout. Do NOT remove during cleanup.
7. **Appreciation defaults must reflect current market era**: 5% bakes in 2022-23 peak assumptions. Real Charlotte metro 2024-2025 is 0-3% YoY across submarkets. Default should be conservative; hot submarkets agents can dial up per-deal.

### Test report IDs (regression baselines)

- `b4278470-e018-466e-b5a5-71d71cd06b2b` — fresh 6022 Candlestick draft created today
- `3a84f12c-dc91-407b-ac93-5efaa5c4d564` — Candlestick draft used through 2026-05-08 verification
- `8023175d-37c8-45d7-95da-b9245abe4761` — original Candlestick (Bent Creek SC, luxury 6BR with finished basement + in-law-suite)
- 5022 Cressingham (Indian Land 29707, 5BR/4BA, no basement). Control test — math fixes should not move PMV more than ±2% absent realistic appreciation rate
- 215 Tyndale Ct (Somerset 28277). Tight-confidence strategy compression test
- 504 Redwine St (Monroe 28110, wrong-zip case)
- 5601 Medlin Rd (Monroe 28110, 1.84 acres semi-rural, thin-data tier-3 test)

### What's left (DO NOT ship more code without real-world reps)

1. **Bug 14 verification when it lands** — confirm Cressingham drops to ~$680-685K with 3.0% default. Per-comp narratives should say "time-of-sale appreciation" not "newer vintage."
2. **Re-run Tyndale after Bug 14** — confirm strategy spread stays tight and per-comp narratives correctly attribute time-of-sale lines.
3. **Soft Shell symmetry test on May 7 Candlestick run** — Bug 6 fix should make the 0.02 vs 0.08 split symmetric. Low priority since the May 7 draft is mostly an archival case now.
4. **Use the tool on a real listing for a week** — let real-world agent workflow surface the next priorities rather than speculation.

### Deferred enhancement queue (not blockers)

- **Build-year premium**: dropped from Bug 14 bundle on purpose. Charlotte metro doesn't reliably price 2019 vs 2020 differently. Revisit only if specific case demands.
- **Time-on-market adjustment for actives**: different from Bug 13's closed-comp appreciation. Lower priority.
- **Submarket-specific appreciation rates**: overengineering until a real workflow case demands it. Agents can dial per-deal in Rates UI for now.

### Mac mini Claude Code MCP setup

Still needs same MCP+GitHub PAT setup as iMac. When Brian works on Mac mini next:

    claude mcp add --transport http --scope user --header 'Authorization: Bearer NEW_PAT_HERE' github https://api.githubcopilot.com/mcp/

Generate separate PAT named "Claude Code MCP - Mac mini". Same fine-grained scope=user pattern.

`alias claude="claude --dangerously-skip-permissions"` already in iMac `~/.zshrc`. Mac mini still needs the alias added when Brian works there.

### Pickup checklist for next session

1. **Verify Bug 14 landed correctly** — re-save Cressingham, confirm PMV ~$680-685K with corrected narrative copy.
2. **Re-run Tyndale** — last regression test we owe.
3. **Stop coding for 24-48 hours**. Use the tool on a real listing. Let real-world feedback shape next priorities.
4. **Other repos for CLAUDE.md ship-it specs** (priority order, all deferred until CMA tool gets real-world reps):
   - `hgpg-transaction-manager`
   - `brain-app`
   - `charlotte-new-construction-nextjs`
   - `south-charlotte-report`
   - `homegrown-property-group-site`

### Web Claude Code on iOS — viable workflow path

Discovered today: Claude Code on the web (claude.ai/code) works from the iOS Claude app. Fire-and-forget pattern for bounded tasks. Useful when Brian is away from desk and wants to queue up small items. Terminal Claude Code remains better for iterative/verification work like today's bug sweep.

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
