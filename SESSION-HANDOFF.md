<!-- Last Updated: 2026-05-15 -->

# Session Handoff

## Last session: 2026-05-15 — New Construction incentives triage 🟢

### What got done
- Triaged `nc_incentives` table on the New Construction site (Supabase project HGPG Signature + Relocation, `fkxgdqfnowskflgbuxhm`).
- Started: 79 total / 45 active. Ended: 77 total / 35 active.
- 10 stale or duplicate rows soft-deactivated (preserved for audit, not deleted).
- 4 new rows added (M/I Homes 5.875% investment rate + Zero Down 2026; Lennar weekly-rotation placeholder; DR Horton division-wide $9,500 closing cost placeholder).
- Toll Brothers row extended through 6/30 with note about ~2-week refresh cycle.

### Created `projects/new-construction.md`
Captures data layer (table schemas, RLS), admin PIN/API surface, triage protocol, and the **builder refresh cadences** I worked out from web-search verification:
- Lennar = weekly rotation (Dream Savings, generic placeholder is the right move)
- Toll Brothers = 2-week rolling 2/1 buydown, division-wide
- DR Horton = monthly Red Tag events + community-specific buydowns (don't extrapolate division-wide)
- M/I Homes = stable named programs (easy to keep accurate)
- Meritage = slow rotation, "Blue Homes" = move-in-ready identifier
- Empire = community-specific (multiple rows are NOT dupes)

### Open / parked
- **M/I `85cca8ac`** — 4.875% Great Low Rate expires 5/29 (2 weeks out). Verify with M/I rep whether it rolls forward, or replace.
- **Meritage `3373880f`** — $3,000 May closing bonus expires 5/31.
- Consider an admin dashboard "Expiring This Week" widget so Brian can triage on-demand without waiting for the list to balloon.

### Pickup notes for next session
- New Construction site has its own brain file now: `projects/new-construction.md`. Read it before any triage work.
- For incentive triage: pull active list with builder name, group by builder, flag 3+ active rows, look for stale dates in description vs null `expiration_date`. Soft-deactivate, don't delete.
- Don't chase Lennar weekly specifics — keep the generic placeholder.

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

### Pickup notes for next session
- Brain-app is live and working — use it for any future updates to `hgpg-context`
- Resend API key is in 1Password ("Supabase HGPG Core SMTP")
- Brain-app local dev: `cd ~/brain-app && npm run dev` on Mac mini (work machine)
- Brain-app local on iMac: same setup, repo at `~/Developer/brain-app` if rebuilt, otherwise needs fresh `gh repo clone HGPG1/brain-app` + `npm install` + `cp env.example .env.local`
- The `package-lock.json` may differ between iMac and Mac mini — push from whichever machine you most recently ran `npm install` on
