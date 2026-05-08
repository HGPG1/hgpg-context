<!-- Last Updated: 2026-05-08 -->

# Buyers Guide

- **Status:** 🟢 Live
- **URL:** TBD — verify in production (likely `buyersguide.homegrownpropertygroup.com` or similar)
- **Stack:** React + Vite (migrated from Manus)

## Purpose

Companion to the Sellers Guide for buyer-side leads. Walks buyers through the home-buying process, captures leads, syncs to FUB.

## History

Originally built on Manus, migrated to React + Vite to gain control over deploy pipeline and avoid Manus-era constraints (custom domains, build limits, etc.).

## Open / pending

- **Original Manus URL** for the Buyers Guide is still unknown — needs confirmation. Once known, set up a 301 redirect from the Manus URL to the React+Vite URL so any external links don't break.
- **Meta Pixel + CAPI** rollout is pending here — playbook exists in `charlotte-new-construction-nextjs/META-PIXEL-CAPI-PLAYBOOK.md`, ~30 min per site
- **NeverBounce email validation** — when the validation API ships per `projects/neverbounce-validation.md`, this site is a candidate for adopting it on its lead capture forms

## Pickup notes

- The site IS live and capturing leads — primary risk if it breaks is missed buyer leads
- Keep parity with Sellers Guide branding (HGPG navy/steel palette, Cooper Hewitt + Sansita fonts, no green, no em dashes)
- This is a separate repo from `charlotte-sellers-guide-vercel` — do not conflate
