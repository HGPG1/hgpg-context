<!-- Last Updated: 2026-05-12 -->

# Buyers Guide

**Status:** 🟢 Instrumentation shipped (Meta Pixel + CAPI + NeverBounce) on `claude/buyers-guide-instrumentation-nGCLI`. Pending: PR merge + env vars provisioned + redeploy + post-deploy verification. Manus 301 still pending (must be configured on Manus's side, not ours).

## URLs

- Production: https://buyersguide.homegrownpropertygroup.com
- Repo: `HGPG1/charlotte-buyers-guide` (private)
- Stack: React 19 + Vite 8 + Tailwind v4 + react-router-dom 7
- Output: `dist/`
- Vercel team: `team_FietQPKCmnyioG2n0FdteQCV`

## Routes (from `src/App.tsx`)

- `/` Home
- `/quiz` Buyer Quiz
- `/calculator` and `/rent-vs-buy` (alias) — Rent vs Buy Calculator
- `/neighborhoods`
- `/strategy`
- `/checklist`
- `/bonuses`
- `/thank-you`
- `/*` NotFound

Page-parity audit vs. the original Manus site is **deferred** — needs Brian's eyes on Manus (the `?code=...` magic-link gate blocks automated comparison).

## Lead capture

Two forms, both in `src/components/Layout.tsx`:

- **UnlockModal** — guide-unlock gate (name + email + phone) → fires `Lead` Pixel event + POSTs `/api/fub-lead`
- **ContactDialog** — "Get Started" CTA in header + footer (first/last/email/phone/message) → same flow

Both forms now call `validateEmail(email)` from `src/lib/validateEmail.ts` BEFORE the Pixel + FUB POST. Fail-closed on NeverBounce result `invalid` or `disposable`; allow-through on `valid`, `catchall`, `unknown`, or network failure (so a vendor outage doesn't kill the funnel).

Bonus-share modal (social share for bonus unlock) doesn't collect email and is not gated.

FUB ingestion via `api/fub-lead.ts` → POST to `/v1/events` (FUB Events API, not /v1/people), `source: "Charlotte Buyer's Guide"`. Returns 502 on FUB failure.

## Meta Pixel + CAPI

- **Pixel ID:** `1449157226505129` (HGPG - Buyers Guide). Distinct from Sellers Guide (`861295553661596`) per playbook (one pixel per site).
- **Browser:** `src/components/MetaPixel.tsx` loads `fbevents.js` and fires `PageView` on mount + each react-router location change. Reads `VITE_META_PIXEL_ID` at build time — if unset, component renders null and the fbevents loader never injects (Vite tree-shakes the dead path, which is why the pre-2026-05-12 bundle had zero `fbq` references).
- **Server CAPI:** `api/_lib/meta.ts` + invoked from `api/fub-lead.ts`. v21.0, SHA-256 hashed PII per spec, AbortController 5s timeout, fbp/fbc cookie passthrough, shared `event_id` for browser+server dedup. `isCapiConfigured()` returns false if `META_PIXEL_ID` or `META_CAPI_ACCESS_TOKEN` is unset → silently no-ops.
- **Events firing:**
  - `Lead` — UnlockModal submit, ContactDialog submit, GuideContext.unlockContent (browser + CAPI dedup)
  - `ViewContent` — Neighborhoods county tab change
  - `Search` — Quiz reaches results page
  - `Contact` — Calculator scenario click
  - `SubmitApplication` — bonus share / copy
  - `PageView` — base loader + each route change

## NeverBounce

- `api/validate-email.ts` — POST proxy. Reads `NEVERBOUNCE_API_KEY` (reuse Sellers Guide key per session decision 2026-05-12). Calls v4 single/check, returns `{ result, flags, suggested_correction }`. All infra failures (missing key, upstream timeout, non-200, non-success) resolve to `result: "unknown"` with a `server_error` tag — keeps lead capture alive during NeverBounce outages.
- `src/lib/validateEmail.ts` — client helper used by both forms. Fail-closed on `invalid`/`disposable` (returns `ok: false` + a user-facing message). Fail-open on everything else.

## Vercel env vars required

Must be set in **Production** scope on the Buyers Guide project before the new deploy goes out:

| Var | Value | Scope |
|---|---|---|
| `VITE_META_PIXEL_ID` | `1449157226505129` | Build-time |
| `META_PIXEL_ID` | `1449157226505129` | Runtime |
| `META_CAPI_ACCESS_TOKEN` | from Meta Events Manager → Settings → CAPI → Generate Access Token | Runtime |
| `META_CAPI_TEST_EVENT_CODE` | TEST code (Meta Events Manager → Test Events). Remove before scaling ads. | Runtime, optional |
| `NEVERBOUNCE_API_KEY` | reuse Sellers Guide value | Runtime |
| `FUB_API_KEY` | already set, confirm still present | Runtime |

## Vercel build config gotcha

`vercel.json` runs `npx vite build` directly — it does NOT run `tsc -b`. This means TS errors don't block the production deploy but do block the local `npm run build` gate (`tsc -b && vite build`). Two pre-existing TS errors in `Bonuses.tsx` (missing `@/` path alias) and `Calculator.tsx` (Recharts Tooltip formatter type widening) were fixed in commit `5c233ee` to satisfy the session gate.

## Post-deploy verification gates

After Brian pushes and Vercel redeploys with all env vars set:

    BUNDLE=$(curl -s https://buyersguide.homegrownpropertygroup.com/ | grep -oE 'src="[^"]*assets/index[^"]*"' | head -1 | sed 's/src="//;s/"$//')
    curl -s "https://buyersguide.homegrownpropertygroup.com$BUNDLE" > /tmp/bg.js
    grep -o fbq /tmp/bg.js | wc -l                  # > 0 (locally: 9)
    grep -c connect.facebook.net /tmp/bg.js          # 1
    grep -c 1449157226505129 /tmp/bg.js              # 1
    grep -c validate-email /tmp/bg.js                # >= 1

Note: don't grep for `neverbounce` in the client bundle — it lives only in the server-side proxy.

End-to-end: submit a test lead → confirm in Meta Events Manager Test Events that browser + server Lead events deduplicate (single row with both sources). Then confirm FUB person created with correct source.

## Pending

- 🟡 **Manus 301 redirect** — the original Manus URL is `https://homegrownbg-hyqbjnnt.manus.space/?code=Cgvt4g5HuhfxNgdswHwqb4`. Can NOT be redirected from our Vercel project because the host doesn't resolve to us. Must be configured on Manus's side (set destination URL on the app to `https://buyersguide.homegrownpropertygroup.com`, or take the app down with a forwarding URL).
- 🟡 **Page-parity audit vs. Manus** — deferred. Brian to send screenshots or a page list from the Manus app for a follow-up session.
- 🟡 **`META_CAPI_TEST_EVENT_CODE` cleanup** — remove from Vercel env once QA confirms dedup. Leaving it on routes all CAPI events to Test Events instead of production, blocking ad-spend scaling.
- 🟢 FUB Automation 2.0 — not in scope this session, parallel concern with Sellers Guide.

## Recent build history (most recent first)

- **2026-05-12** — feat(buyers): NeverBounce email validation proxy + form gates (fail-closed on invalid/disposable, fail-open on infra) + build-gate fixes for two pre-existing TS errors. Commit `5c233ee` on `claude/buyers-guide-instrumentation-nGCLI`. Pushed to origin in-session (sandbox push works after all).
- 2026-05-01 — SEO trim (homepage meta description 178 → 152 chars)
- 2026-05-01 — Lowercase email in schema + visible references
- 2026-05-01 — SEO/AEO: schema enrichment + footer link fix + author byline
- 2026-04-28 → 2026-05-01 — Meta Pixel + CAPI scaffolding shipped to repo (events: PageView, ViewContent, Search, Contact, Lead, SubmitApplication; CAPI mirror with `event_id` dedup). Inert in production until 2026-05-12 because env vars were never provisioned.

## Lessons noted

1. **Code-on-`main` does NOT equal shipped-in-production.** The Pixel + CAPI commits had been on main since late April but the live bundle had zero `fbq` references because `VITE_META_PIXEL_ID` was unset — Vite tree-shook the dead path. Always verify against a built bundle, not just the source.
2. **Build-gate divergence:** `vercel.json` `buildCommand` (`npx vite build`) differed from `package.json` build script (`tsc -b && vite build`). Means TS errors can land on main without breaking production. Either align them, or accept the divergence and run `npm run build` locally before merging.
3. **Sandbox push works.** Stale assumption corrected this session — `git push` from the sandbox succeeded against `HGPG1/charlotte-buyers-guide`. The "sandbox cannot push" line in prior kickoff prompts is no longer universally true; depends on which repo + which token is wired in for that session.
