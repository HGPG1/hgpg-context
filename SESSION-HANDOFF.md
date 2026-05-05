# SESSION-HANDOFF

**Last session:** 2026-05-05
**Length:** Long. Multi-track infrastructure cleanup + project restructure.
**State at end:** Clean. Ready for focused work next session.

---

## What happened this session

### Infrastructure cleanup
- Migrated GitHub auth from PAT-embedded URLs to gh CLI on both Macs (Mac mini + iMac). Token in macOS Keychain. No more PAT rotation drama.
- Tore down Suna fully: DNS (`ai.homegrownpropertygroup.com`), Railway deployment, Supabase project `nfwjgsanvmwmrginhklz`.
- Decommissioned Manus FUB AI Agent: unpublished from Manus, deleted Railway services + project, archived `HGPG1/fub-ai-agent` repo. Cancelled Railway plan entirely (no recurring charge).
- Archived 20+ cruft GitHub repos (versioned predecessors with `1`, `2`, `3`, `part2` suffixes).
- Deleted two cruft Vercel projects (`adoring-wilbur`, `project-lisqf`).
- Resolved sellers guide drift: `sellersguide.homegrownpropertygroup.com` is canonical, `sellers.` subdomain removed, empty placeholder repo `charlotte-sellers-guide-2026` archived.
- Clarified Listing Report Portal infrastructure: source repo is `hgpg-listing-report`, Vercel project is named `listing-report-deploy`. The naming mismatch is cosmetic but documented.

### Documentation
- Manus FUB AI Agent handoff doc preserved at `archive/fub-agent-handoff.md` in brain repo and at `~/Library/Mobile Documents/com~apple~CloudDocs/HGPG-Cowork/fub-agent-handoff.md`.
- Brain app spec drafted at `projects/brain-app.md` for future build.
- Brain repo (CONTEXT.md + infrastructure.md) updated and pushed multiple times.

### Workflow
- `brain` shell function deployed on both Macs. Auto-pulls files from `~/Downloads/` matching brain file patterns, copies into local repo, prompts for commit message, pushes. Friction reduction is live.

### Claude project sidebar restructure
- Audited all Claude projects, consolidated overlap.
- 5 HGPG projects (renamed for consistency, all use `HGPG - <name>` style):
  - HGPG - Home Base (catch-all, formerly `Project- Home Base`)
  - HGPG - Tech & Build
  - HGPG - CRM & Leads
  - HGPG - Marketing & Content (absorbed Content Pipeline)
  - HGPG - Ads (formerly `HGPG- Meta Ads`, scope expanded)
- Modernized Homie project to fetch live brain repo instead of relying on uploaded master memory file.
- Configured Home projects (personal home improvement) with new instructions.
- Archived 4 projects: Dev & Tech, Real Estate Ops, Content Pipeline, How to use Claude.

---

## What's open for next session

### Quick win (~30 min)
**Team Tools auth fix** at `tools.homegrownpropertygroup.com`.
- Diagnosed: was wired to the deleted Suna Supabase (`nfwjgsanvmwmrginhklz`), so login is broken.
- Source repo: private, `HGPG1/hgpg-team-tools2` (Vite app).
- Fix: clone repo, swap Supabase env vars to point at primary `ioypqogunwsoucgsnmla`, set up Google OAuth on that Supabase, redeploy.
- Whole team uses it (5 people), but no urgency.

### Major builds
**FUB lead automation rebuild** at `leads.homegrownpropertygroup.com`.
- Spec preserved at `archive/fub-agent-handoff.md` in brain repo.
- Reuse existing infra: Vercel project `fub-texting-integration`, Supabase `ngdrliyjtqcwhhfrbxao`, GitHub repo `HGPG1/fub-texting-integration`.
- Schema already designed (15 tables including bulk campaigns, opt-out tracking, auto-reply rules).
- Multi-day build estimated.

**Brain app** at `brain.homegrownpropertygroup.com`.
- Spec at `projects/brain-app.md` in brain repo.
- 60-90 min Claude Code session estimated.
- Pre-build checklist in spec: GitHub PAT, Supabase project decision, GoDaddy DNS, empty repo creation.

### Minor decisions
- MLS Listing Dashboard project — currently parked. Decision depends on whether actionable MLS alerts arrive in Gmail. Audit Gmail filters first.
- A2P 10DLC resubmission — still blocked. Need SMS consent checkbox on contact forms before retry.

---

## State of the brain repo

- Clean and current.
- `CONTEXT.md` reflects all completed teardowns and open items.
- `infrastructure.md` reflects current Vercel + Supabase + GitHub state.
- `projects/brain-app.md` has full spec for future build.
- `archive/fub-agent-handoff.md` has Manus-era spec for FUB rebuild reference.

---

## How to start next session

1. Open the relevant Claude project in claude.ai (HGPG - Tech & Build for builds, HGPG - CRM & Leads for FUB work, etc.).
2. The project will fetch CONTEXT.md automatically.
3. Reference this handoff for "where we left off."
4. If tackling Team Tools auth: see the quick win section above. Use the prompt prepared in last session for kickoff.

---

## Notes / lessons from this session

- **Heredoc paste hazards:** zsh + Messages app + heredoc syntax is brutal. Multiple times the paste got corrupted with `EOFcat` collisions or stray characters. The `brain` shell function setup hit this twice. Lesson: prefer single-command appends or text-editor edits over chained heredocs in Terminal.
- **Don't trust "cruft" labels without verification:** I had `listing-report-deploy` flagged as cruft for deletion. It was actually serving the live Listing Report Portal. Lesson: verify subdomain ownership in Vercel before deleting any project, even one that "looks" abandoned.
- **Project instructions had Manus-era stale references:** Several old projects still referenced "migrating from Manus" as in-progress. The migration is done. New project instructions reflect that.
