<!-- Last Updated: 2026-05-11 -->

# Session Handoff

## Last session: 2026-05-11 — Sellers Guide ad-readiness QA 🟢 PASSED + 3 production bugs fixed

### Outcome
Sellers Guide is **ad-ready**. End-to-end QA from `?utm_source=meta` URL verified: Pixel + CAPI dedup, NeverBounce, FUB ingestion with full tag set, UTM custom-field attribution, fub_event_id round-trip. Started the session believing this had all shipped clean; found 3 production bugs in the process and fixed them all.

### Three bugs fixed today

1. **Score page had no Pixel init** (commits `~5627dd2`)
   - `/home-selling-score/` is the primary conversion funnel — Lead/AssessmentStarted/ScoreCompleted fire there
   - Page had event-firing helpers but never loaded `fbevents.js` and never called `fbq('init', ...)`
   - Every browser-side Pixel event was a silent no-op (the `if (typeof window.fbq === "function")` check was failing closed)
   - Fix: added the same Pixel `<head>` block from `/index.html` and rewrote `fireFbqEvent` to delegate to `window.hgpgTrack` so it fires browser + CAPI with shared event_id

2. **`/api/assessment/submit` FUB call was fire-and-forget** (commits `~5627dd2`)
   - Handler used `fetch(...).catch(...)` without `await`
   - On Vercel serverless, lambda teardown killed the in-flight FUB POST before completion
   - Result: every v2 lead returned 200 to the user but never reached FUB. All 5 existing rows from 5/2-5/7 had `fub_event_id=null`.
   - Fix: awaited the FUB fetch. Now ~2-3s response time but lead reliably reaches FUB.

3. **`/api/fub-lead` discarded FUB's response body** (commit `527bb7b`)
   - Returned `{success: true}` even though FUB returned the person object with `id`
   - Caller had no way to record the FUB person ID for cross-system lookup
   - Fix: parse FUB response, return `{ success, fub_person_id, fub_person, fub_status }`. `/api/assessment/submit` then writes the person ID back to `seller_assessments.fub_event_id`.

### QA result (single-pass, real submission from phone with utm_source=meta)

- Lead in DB: `schema_version=v2-2026-05-07`, `tier_name=Solid Bones`, `email_validation_status=catchall`, all 4 UTMs + `metaBypass: true` in `utms_json`, `fub_event_id=31928`, `fub_posted_at` populated
- FUB person 31928 landed with tags: `sellers-guide-2026`, `website-lead`, `home-selling-score`, `meta-bypass`, `utm:meta`. Stage=Lead, Source=Sellers Guide 2026.
- Meta Test Events: PageView, ScoreCompleted, Lead all processed with shared `event_id` (browser side visible; CAPI side firing per Vercel logs at matching timestamps; single row per event_id in Test Events = dedup working)
- NeverBounce ran and persisted result
- Meta-bypass flow worked (no 6-digit verify step)

### Outstanding before scaling ad spend

- **FUB Automation 2.0 not built** on `sellers-guide-2026` tag. Leads land tagged correctly but no automatic drip/assignment fires. Initial ad launch is fine (want manual eyes on first leads anyway), but blocks scaling. **Next session: build the Automation.**
- `/process/` and `/pricing/` routes return 404. If nav anywhere on the sellers guide links to them, audit before sending paid traffic.
- "Phase 1 ads test markers" comments left in code from 5/5 CAPI commit — cleanup later, not blocking.

### Data cleanup completed

- Removed 4 QA test rows from `seller_assessments` (Brian+test, Brian+qaverify, Brian+pid, Brian+momotest)
- Brian manually deleted FUB persons 31927 and 31928

### Process note

This session is exactly why the "Brain Updates Are Default" rule was added at start of session. CONTEXT.md was a week stale; sellers-guide had no project file; the score page Pixel was broken in production for an unknown number of days. Going forward Claude proactively writes brain updates at end of every material-change session, unprompted.

### Pickup notes for next session

- **Build FUB Automation 2.0** on `sellers-guide-2026` tag — drip cadence + agent assignment + whatever else you want. This unblocks ad scale.
- After Automation is built, set up Meta ad campaigns. Destination URLs should bake `?utm_source=meta&utm_medium=<paid|organic-test>&utm_campaign=<campaign-name>` into ad creative.
- Phase 1 ads test markers in code (per 5/5 Pixel commit) — clean up post-launch
- `marketing.md` in brain repo was NOT updated this session — when ads go live, document campaign IDs, ad set structure, and budget there
- Consider auditing `/process/` and `/pricing/` route 404s before spend
