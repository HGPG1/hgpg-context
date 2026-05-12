<!-- Last Updated: 2026-05-11 -->

# Buyers Guide

- **Status:** 🟢 Live (capturing leads)
- **URL:** `https://buyersguide.homegrownpropertygroup.com` (verified 2026-05-11)
- **Stack:** React + Vite (migrated from Manus)

## Purpose

Companion to the Sellers Guide for buyer-side leads. Walks buyers through the home-buying process, captures leads, syncs to FUB.

## History

Originally built on Manus, migrated to React + Vite to gain control over deploy pipeline and avoid Manus-era constraints (custom domains, build limits, etc.).

## Verified state of instrumentation (2026-05-11)

Live bundle grep at `https://buyersguide.homegrownpropertygroup.com/assets/index-*.js` (~278KB):

- `fbq` references: **0**
- `connect.facebook.net` references: **0**
- 15-16 digit Pixel ID candidates: **none**
- `neverbounce` references: **0**
- `/api/meta/capi` and `/api/validate-email` endpoints: **return SPA index.html fallback** (i.e. routes do not exist; Vercel rewrites swallow them)

All three instrumentation items below are genuinely pending. None are partially shipped.

## Open / pending

- **Meta Pixel + CAPI** rollout. Playbook in `charlotte-new-construction-nextjs/META-PIXEL-CAPI-PLAYBOOK.md`. Buyers Guide needs its own Pixel ID (per playbook: one Pixel per site).
- **NeverBounce email validation** on lead capture forms. Sellers Guide already has the pattern shipped — mirror it. NeverBounce API key can be reused from Sellers Guide's Vercel env (single account, single key works across sites) or provisioned fresh for per-site quota visibility.
- **Original Manus URL** still unknown. Once known, 301 redirect via `vercel.json` to preserve any external inbound links. Do not block Pixel + NeverBounce on this; ship those even if the Manus URL never surfaces.

## Pending instrumentation session — kickoff prompt

Next session is queued: ship Pixel + CAPI + NeverBounce + (optionally) Manus 301. Estimated 2-3 hours single sitting. Paste this prompt into a fresh Claude Code session running in the Buyers Guide repo:

    You are picking up the Buyers Guide lead-capture instrumentation. The site is live at https://buyersguide.homegrownpropertygroup.com (React + Vite SPA, migrated from Manus) and is currently MISSING three things that the Sellers Guide already has shipped. Today's verified state: the production JS bundle has zero references to Meta Pixel, NeverBounce, or any CAPI/validate-email endpoint. Goal is to ship all three in this session.

    CONTEXT TO LOAD FIRST

    Read these in order before writing any code:

    1. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/CONTEXT.md
    2. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/SESSION-HANDOFF.md
    3. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/projects/buyers-guide.md
    4. https://raw.githubusercontent.com/HGPG1/hgpg-context/main/projects/sellers-guide.md (reference for what shipped + the patterns to mirror)
    5. Repo root: find and read META-PIXEL-CAPI-PLAYBOOK.md. If not in this repo, check the New Construction repo (charlotte-new-construction-nextjs) — that is where Brian noted it lives.

    DELIVERABLES

    1. Meta Pixel + CAPI (browser pixel + server-side conversions API with event_id dedup)
       - Use a Buyers-Guide-specific Pixel ID (NOT the New Con or Sellers Guide Pixel — each site gets its own per the playbook)
       - Fire Lead event on every form submission
       - Server endpoint /api/meta/capi mirrors the browser event using the same event_id for dedup
       - Hash PII (email, phone) per Meta CAPI spec before sending server-side

    2. NeverBounce email validation
       - Add /api/validate-email proxy route (server-side, NeverBounce API key in Vercel env)
       - Wire into all form submit handlers BEFORE the form fires the lead-capture POST
       - Fail-closed UX: if email is invalid or risky, block submit and surface a clear error
       - Mirror the Sellers Guide pattern exactly

    3. Manus URL 301 redirect — ONLY IF the original Manus URL is known. Brain currently says it is unknown. Ask Brian for it; if he does not have it, skip this and note it as still-pending in the handoff. Do not block the other two.

    VERIFICATION GATES

    Before declaring done, all of the following must pass:

    - `npm run build` clean
    - `npx tsc --noEmit` clean
    - After deploy, hit the prod bundle and grep: `fbq` count > 0, `neverbounce` references > 0, Pixel ID present as a 15-16 digit string
    - POST to /api/validate-email returns real NeverBounce JSON (not the SPA index.html fallback)
    - POST to /api/meta/capi returns real CAPI response
    - Submit a test lead end-to-end, confirm: NeverBounce validates, browser Pixel fires, CAPI fires server-side with matching event_id, FUB receives the lead via the existing Events API integration
    - Use Meta Events Manager Test Events to confirm browser + server events deduplicate (not double-counting)

    CONSTRAINTS

    - Last-Updated comment at top of every new/edited file
    - Commit author brian@homegrownpropertygroup.com (NOT brian@hgpg.com)
    - Push from this Mac (sandbox cannot push)
    - No em dashes in user-facing copy; HGPG brand colors only (#2A384C navy, #A0B2C2 steel blue, #D1D9DF light steel, #F0F0F0 off-white, no green)
    - All Vercel env vars must be added to Production environment, not just Preview
    - Update projects/buyers-guide.md and SESSION-HANDOFF.md via the brain write API at session end. Brain write API: POST https://brain.homegrownpropertygroup.com/api/external/write with Bearer token from Brian's memory + JSON {path, content, message}

    START BY

    Reading the four brain files above, then `git pull --rebase origin main`, then locating the META-PIXEL-CAPI-PLAYBOOK.md. Tell me what Pixel ID, what env vars I need to provision, and what the order of operations will be. Then begin.

## Pickup notes

- The site IS live and capturing leads — primary risk if it breaks is missed buyer leads. Land instrumentation in a way that preserves lead capture even if Pixel/CAPI errors out (NeverBounce can fail-closed; Pixel + CAPI should fail-open so a Meta outage doesn't kill the funnel).
- Keep parity with Sellers Guide branding (HGPG navy/steel palette, Cooper Hewitt + Sansita fonts, no green, no em dashes).
- This is a separate repo from `charlotte-sellers-guide-vercel` — do not conflate.
