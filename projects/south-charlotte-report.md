<!-- Last Updated: 2026-05-08 -->

# South Charlotte Report

- **Status:** 🟢 Live, daily content pipeline running
- **Repo:** HGPG1/south-charlotte-report
- **Companion repo:** HGPG1/south-charlotte-scraper

## Purpose

Daily-cadence market content for the South Charlotte / Indian Land / Fort Mill / Waxhaw area. Generates a video + blog post + social posts + email digest from MLS data, automated end-to-end.

## Pipeline

Daily workflow runs via GitHub Actions: `morning_fetch.yml` → `run_morning_fetch_enhanced.py`

Output stages:
1. **MLS data fetch** from Canopy MLS Grid (same source as CMA Engine)
2. **Stat calculations** — new listings, sold, price trends, days on market
3. **AI content generation** — blog post + script for video
4. **Video synthesis** — HeyGen-driven, with H.264 + `+faststart` for Instagram Reels compatibility
5. **Distribution**:
   - Instagram Reels (with strict format requirements)
   - YouTube
   - Blog post on `blog.homegrownpropertygroup.com`
   - Email digest via Resend

## Failure handling

- HTML digest email sends regardless of downstream success
- Red failure banner + subject prefix on the digest when downstream fails
- HeyGen failure short-circuits IG/YouTube checks to prevent cascade false positives
- All errors land in the digest so Brian sees them in inbox without checking dashboards

## Key technical constraints

- **Instagram Reels**: `-movflags +faststart` is MANDATORY. Moov atom at tail hard-fails Instagram CDN. YouTube tolerates it but IG does not. This is hardcoded in the video normalization step.
- **Video spec**: H.264, known-good Reels spec for compatibility
- **Cron schedule**: see workflow YAML — runs in early morning ET so output is ready by 7am ET digest send

## Pickup notes

- Content quality has been steady; pipeline is mostly hands-off
- If output stops appearing, check GitHub Actions log first — failures usually surface as a clear error
- HeyGen API key + Resend API key in repo's GitHub Actions secrets
- The scraper repo (`south-charlotte-scraper`) is a separate dependency — they're decoupled so the scraper can be reused for other content

## Open / pending

- No active dev work. This is an ops-mode tool: it runs, it works, it gets fixed when it breaks.
