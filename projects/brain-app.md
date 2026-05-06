<!-- Last Updated: 2026-05-06 -->

# Session Handoff

## Last session: 2026-05-06 — Brain App MVP + Phase 1.5 shipped 🟢

### What got built

**Brain App MVP**
- New Vercel project: `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- New repo: `HGPG1/brain-app` (private)
- Live at: `https://brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Single-user lock: `BRIAN_EMAIL=brian@homegrownpropertygroup.com` allow-list
- GitHub auth: fine-grained PAT scoped to `HGPG1/hgpg-context`, contents:write only
- Round-trip verified: edit file in browser → commit lands on `main` with author `brian@homegrownpropertygroup.com`

**Phase 1.5 UX polish (same day)**
- Dashboard redesigned with Pinned cards (4 hardcoded in `lib/config.ts`), Quick actions, Recently edited (top 5) — replaces duplicate file list
- Edit page sticky header with back arrow, breadcrumb, save status (30s polling: "Saved Xm ago" / "Unsaved changes" / "Saving...")
- Mobile hamburger drawer (300px, slides in from left, dimmed overlay, auto-closes on file tap, 200ms ease-out)
- Sidebar sorted by last-modified date desc within each section (Root, projects/, archive/)

### Infra changes that affect other apps

- Resend custom SMTP wired into `HGPG Core` Supabase (project `ioypqogunwsoucgsnmla`)
  - Sender: `noreply@homegrownpropertygroup.com`, name: HGPG
  - API key stored in Resend as "Supabase HGPG Core"
  - Rate limit went from 2/hr (Supabase default) to 30/hr (Resend default), can be raised
  - This affects ALL apps using this Supabase: TM, CMA, TC Concierge, brain-app
- Supabase project renames for hygiene:
  - `ioypqogunwsoucgsnmla` → "HGPG Core"
  - `ngdrliyjtqcwhhfrbxao` → "HGPG FUB Integration"
  - `wdheejgmrqzqxvgjvfee` → "HGPG Listing Reports + MLS"
  - `fkxgdqfnowskflgbuxhm` → "HGPG Signature + Relocation"
- Supabase `HGPG Core` redirect URLs added:
  - `https://brain.homegrownpropertygroup.com/**`
  - `http://localhost:3000/**`
  - (Existing tools.hgpg entries left intact)

### Bugs found and fixed mid-session

- Magic link redirected to `tools.homegrownpropertygroup.com` (Supabase Site URL fallback) — fixed by adding `/auth/callback` route handler that was missing from initial scaffold + pointing `emailRedirectTo` at it
- Supabase free SMTP rate limit (2/hr) hit during testing — fixed permanently by switching to Resend custom SMTP

### Project status updates

- `projects/brain-app.md` — status now 🟢 SHIPPED with full Phase 2 backlog documented
- `projects/hgpg-team-tools2.md` — Site URL in HGPG Core Supabase still points here for the broken app's eventual fix
- `projects/transaction-manager.md` — no code changes today, but TM benefits from Resend SMTP upgrade

### Deferred / Phase 2 for brain-app

- iPhone smoke test deeper pass (CodeMirror + iOS soft keyboard scroll behavior in real-world editing)
- Cooper Hewitt self-hosted (currently falling back to system sans, not on Google Fonts)
- File rename and delete
- Diff view before save
- Cross-file search
- Multi-user allow-list (current `BRIAN_EMAIL` check is single-string but `lib/auth.ts` is structured for easy expansion)
- Audit log (Supabase table tracking who edited what, when)
- Draft autosave (Supabase table, recovers unsaved work on browser close)
- Drawer Esc-to-close + focus trap (a11y polish)
- Sticky sidebar on long dashboards (currently scrolls past)
- Save-status precision tighter than 30s polling

### Pickup notes for next session

- Brain-app is live and working — use it for any future updates to `hgpg-context`
- Resend API key is in 1Password ("Supabase HGPG Core SMTP") — verify before next rotate
- Brain-app local dev: `cd ~/brain-app && npm run dev` on Mac mini (work machine)
- Brain-app on iMac: needs fresh `gh repo clone HGPG1/brain-app` + `npm install` + `cp env.example .env.local`
- The `package-lock.json` may differ between iMac and Mac mini — push from whichever machine you most recently ran `npm install` on
- Brain-app Phase 2 backlog lives in `projects/brain-app.md`
- Pinned files on dashboard configured in `lib/config.ts` — edit that array to change pins
- Stray bundle in `~/Downloads/fub-agent-tm-files/` is unrelated FUB lead-scoring agent work for `hgpg-transaction-manager`, NOT brain-app — leave alone, separate project for another session
- Cleanup tip: `rm ~/package-lock.json` to clear the harmless "multiple lockfiles" Next.js warning (leftover from earlier npm misuse in home dir)