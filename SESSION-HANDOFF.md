<!-- Last Updated: 2026-05-11 -->

# Session Handoff

## Last session: 2026-05-11 — Brain audit: sellers guide state caught up to reality 🟡

### What this session was
Brian asked "are we ready for ads on the sellers guide" and CONTEXT.md gave a stale answer ("recently completed - rebrand to brand colors"). Pulled live state from Vercel commit history + Supabase to reconstruct what actually shipped between 5/2 and 5/7.

### What actually shipped (was not in brain)
- **2026-05-02** — Header/footer pattern matched to buyers guide. Contact-an-Agent lead-capture modal (writes to `/api/fub-lead`, tags `sellers-guide-2026`, `website-lead`, `home-score-contact`)
- **2026-05-04** — Schema/AEO enrichment across all 7 pages. SEO meta length fixes.
- **2026-05-05** — **Meta Pixel + CAPI shipped.** Pixel `861295553661596`. Server-side CAPI mirror `api/meta/capi.js` with shared `event_id` dedup. Events: `AssessmentStarted`, `Lead`, `ScoreCompleted`, `QuizStarted`, `QuizCompleted`. `utm_source=meta` bypasses 6-digit verify (paid-traffic-friendly path).
- **2026-05-06** — CAPI converted to ESM (`fix(capi): convert to ESM for type:module`). FUB UTM custom field bug fixed (use `customXXX` API names, not labels).
- **2026-05-07** — **Home Grown Selling Score v2** shipped. 46-item wizard replaced with 5×4 single-page flow. New `/api/assessment/submit` writes `seller_assessments` + forwards to FUB. NeverBounce email validation wired. Score curve recalibrated to cap at 80 ("Market Ready" 85+ intentionally unreachable). Category Breakdown dropped from results. Email re-validates on edit.

### Lead capture is live
Supabase project `fkxgdqfnowskflgbuxhm` (Signature + Relocation, NOT HGPG Core):

- `seller_assessments`: 6 rows, latest 2026-05-07
- `seller_assessment_ratings`: 238 rows, latest 2026-05-07
- `seller_verification_codes`: 15 rows

### Ad-readiness verdict
Technically wired. **Not yet QA'd post-CAPI-ESM-fix.** Before spend:

1. Verify in Meta Events Manager that `Lead` events appearing browser+server with working dedup
2. End-to-end test from `?utm_source=meta` URL — bypass works, FUB lead lands with right tags + populated UTM custom fields
3. Confirm `sellers-guide-2026` tag fires the intended FUB Automation 2.0
4. Spot-check NeverBounce rejection path
5. Spot-check Contact-an-Agent modal lead (separate code path)

### Brain updates committed this session
- `projects/sellers-guide.md` — NEW, full state doc
- `CONTEXT.md` — sellers guide moved out of "Recently completed" into "Active right now"

### Process note for future sessions
CONTEXT.md drifted ~1 week behind actual shipped state. The Brain App write API is live at `https://brain.homegrownpropertygroup.com/api/external/write` — Claude can write directly. **Default behavior going forward:** when a session ships material changes to a project, Claude proactively updates the relevant `projects/*.md` AND bumps CONTEXT.md status in the same session, not at the next ad-hoc audit.

### Pickup notes for next session
- Run the 5-step ad QA checklist above
- After QA green, set up Meta ad campaigns pointing at sellersguide.homegrownpropertygroup.com with `utm_source=meta` baked into the destination URLs
- Phase 1 ads test markers left in code (per 5/5 Pixel commit) — clean up post-launch
- `marketing.md` was not touched this session — when ads go live, document campaign IDs, ad set structure, and budget there
