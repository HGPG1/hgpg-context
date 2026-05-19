<!-- Last Updated: 2026-05-19 -->

# Session Handoff

## Last session: 2026-05-19 — Variant E scoping confirmed, creative render deferred 🟡

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
