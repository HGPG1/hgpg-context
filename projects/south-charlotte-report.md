<!-- Last Updated: 2026-05-19 -->

# South Charlotte Report

- **Status:** 🟢 Live, daily content pipeline running (ops-mode, hands-off until it breaks)
- **Repo:** HGPG1/south-charlotte-report
- **Companion repo:** HGPG1/south-charlotte-scraper
- **Brand handle:** `@southcharlottereport`

## Purpose

Daily-cadence market content for the South Charlotte / Indian Land / Fort Mill / Waxhaw area. Generates a video + blog post + social posts + email digest from MLS data, automated end-to-end.

## Pipeline (as documented — needs repo-truth verification next time near the code)

Daily workflow runs via GitHub Actions: `morning_fetch.yml` → `run_morning_fetch_enhanced.py`

Output stages:
1. **MLS data fetch** from Canopy MLS Grid (same source as CMA Engine)
2. **Stat calculations** — new listings, sold, price trends, days on market
3. **AI content generation** — blog post + script for video
4. **Video synthesis** — HeyGen-driven, with H.264 + `+faststart` for Instagram Reels compatibility
5. **Distribution**:
   - Instagram Reels (strict format requirements)
   - YouTube
   - Blog post on `blog.homegrownpropertygroup.com` (WordPress/SiteGround)
   - Email digest

## Known discrepancies — needs repo verification

These conflict across brain files and prior session memory. Resolve next time Brian is near the repo:

- **Timing.** `marketing.md` and this file imply early-AM ET cron with "ready by 7am ET digest send." Prior session memory references "scraper pulls at 5 AM ET, posts at 6 AM ET" — possibly a v5.0 change. Check `morning_fetch.yml` cron.
- **Newsletter platform.** This file previously said digest sends "via Resend." `marketing.md` says "Newsletter: Beehiiv." Both could be true (ops alert digest via Resend, subscriber newsletter via Beehiiv) but unverified — check what the workflow actually sends and to whom.
- **Manus removal / v5.0.** Prior session memory references "Manus dependencies removed (v5.0)." Not reflected in this file. Check repo for a v5.0 tag/branch and what changed.
- **S3 image migration.** Prior session memory references "S3 image migration complete." Not in this file. Check whether images now live in S3 vs. WordPress media library.
- **Instagram token refresh.** Prior session memory notes a recent IG token refresh. Confirm where the long-lived token is stored (GitHub Actions secret name) so the next rotation isn't a fire drill.

## Failure handling

- HTML digest email sends regardless of downstream success
- Red failure banner + subject prefix on the digest when downstream fails
- HeyGen failure short-circuits IG/YouTube checks to prevent cascade false positives
- All errors land in the digest so Brian sees them in inbox without checking dashboards

## Key technical constraints

- **Instagram Reels:** `-movflags +faststart` is MANDATORY. Moov atom at tail hard-fails Instagram CDN. YouTube tolerates it but IG does not. This is hardcoded in the video normalization step.
- **Video spec:** H.264, known-good Reels spec for compatibility
- **Cron schedule:** see workflow YAML — runs in early morning ET so output is ready before the digest sends

## Audit findings — 2026-05-19 session

Brian asked for an audit of the flow + simplification suggestions. Captured here so they're not lost.

**Structural observations:**

1. **Monolithic script.** `run_morning_fetch_enhanced.py` does fetch → stats → AI gen → HeyGen → IG → YouTube → WordPress → email all in one run. The failure-handling layer (digest-anyway, red banner, HeyGen short-circuit) is a workaround for the absence of checkpoints. Splitting into discrete GitHub Actions jobs with artifacts would let any single step be re-run independently.

2. **Two repos for one product.** `south-charlotte-report` and `south-charlotte-scraper` were decoupled "so the scraper can be reused." CMA Engine also pulls Canopy MLS Grid but is not known to share the scraper. If scraper isn't actually reused elsewhere, the two-repo split is pure overhead (2x secrets, 2x deps, 2x Actions configs). Worth checking before any consolidation.

3. **HeyGen is the most fragile/slowest/most expensive step** AND it gates social distribution. Blog + email do most of the SEO/nurture work and don't need video to ship. Decoupling text-publish from video-publish on the cadence means a HeyGen outage no longer kills the day's content.

4. **Distribution fan-out from one Python script.** IG + YouTube + WordPress + email each have their own auth lifecycles. Recent IG token refresh is the tell — when these are all in one script, an IG outage masks as a pipeline outage. Worth considering whether IG/YT publishing belongs in a scheduler (Buffer/Later/Metricool) instead of direct API.

5. **Digest serves three audiences.** Ops monitoring + subscribers + inbox alerts in one email. Red-banner-on-failure means subscribers occasionally get a "BROKEN" email. Split it: ops to dm-brian relay, subscriber digest only when content is actually ready.

6. **No feedback loop.** Every other content surface has analytics or Pixel feedback. SCR runs daily blind. At minimum, UTMs on blog links in digest + YouTube descriptions so GA4/FUB can see whether SCR traffic ever converts.

**Proposed simpler shape:**

One workflow, three jobs:

    job: fetch_and_analyze       (5:00 AM ET, always runs)
      -> uploads market_data.json + blog_post.md + script.txt as artifacts

    job: publish_text            (5:30 AM ET, needs fetch_and_analyze)
      -> WordPress + Beehiiv/Resend digest
      -> independent of video pipeline

    job: produce_and_publish_video  (5:30 AM ET, needs fetch_and_analyze)
      -> HeyGen + IG + YouTube
      -> allowed to fail without blocking anything else

Two output paths, one shared input. Ops alerts go to dm-brian relay. Subscribers only see clean content. Same content, fewer failure modes.

**Status of audit work:** documented only. No code touched, no commits made. Next session can pick this up by reading the repo and confirming the discrepancies above before designing the refactor.

## Pickup notes

- Content quality has been steady; pipeline is mostly hands-off
- If output stops appearing, check GitHub Actions log first — failures usually surface as a clear error
- HeyGen API key + Resend API key (and possibly Beehiiv API key) in repo's GitHub Actions secrets
- The scraper repo (`south-charlotte-scraper`) is a separate dependency — verify whether it's actually reused before consolidating

## Open / pending

- **Repo-truth pass:** resolve the timing / newsletter platform / Manus-v5.0 / S3 / IG token discrepancies listed above. Next time Brian is near the code.
- **Audit decision:** Brian to decide whether the simpler 3-job shape is worth the refactor, or whether the current pipeline is "fine until it breaks."
- No active dev work otherwise. This is an ops-mode tool.
