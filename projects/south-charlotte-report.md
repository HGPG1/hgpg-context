<!-- Last Updated: 2026-05-19 -->

# South Charlotte Report

- **Status:** 🟢 Live, daily content pipeline running (ops-mode, hands-off until it breaks)
- **Repo:** HGPG1/south-charlotte-report
- **Companion repos:**
  - `south-charlotte-scraper` (discovery engine — RSS + government sites + social, output written to S3)
  - `south-charlotte-market-report` (separate repo, generates monthly market data; triggers monthly HGPG video pipeline in this repo via `repository_dispatch: market-report-ready`)
- **Brand handle:** `@southcharlottereport`
- **Current version:** v4.1.8 (per changelog, 2026-03-15). README header says "v5.0" but that's marketing — code self-identifies as v4.1.6 in monitoring output. v5.0 references the Manus-removal milestone, not a version bump.

## What it actually is

A two-track content pipeline, run entirely from GitHub Actions, no external orchestrator.

**Track 1 — Daily news content (`morning_fetch.yml` + `run_morning_fetch_enhanced.py`)**

- Cron: `0 10 * * *` UTC = **5:00 AM EST / 6:00 AM EDT** (workflow comment incorrectly says "6:00 AM EST" — it's actually 5 AM EST. Real bug, harmless but confusing.)
- Sources: RSS feeds (10+ local news outlets) + S3-scraped data (government websites + social media, fed in by `south-charlotte-scraper`)
- Two-stage brand-safety filter (Stage 1 on RSS title/description, Stage 2 on full article content) blocks crime, tragedy, controversial politics, negative economic news, historical retrospectives
- AI content gen via Claude (`ANTHROPIC_API_KEY`); `OPENAI_API_KEY` also in env (likely embeddings for dedup but unverified)
- Video synthesis via HeyGen (avatar + voice), H.264 + `-movflags +faststart` MANDATORY for Instagram
- Distribution per HANDOFF.md (current actual state):
  - **WordPress** (`blog.homegrownpropertygroup.com`) — full articles
  - **Instagram Reels** — 9:16 avatar videos
  - **YouTube Shorts for SCR news: NOT in use** (per HANDOFF)
- Daily digest email to `brian@homegrownpropertygroup.com` via **Gmail SMTP** (`GMAIL_SENDER_EMAIL` + `GMAIL_APP_PASSWORD`) — NOT Resend, NOT Beehiiv

**Track 2 — Monthly market video (`hgpg_market_video.yml` + `generate_hgpg_video.py`)**

- Triggered by `south-charlotte-market-report` repo via `repository_dispatch: market-report-ready` after that repo generates the monthly PDF
- Fallback cron: 1st of month at 15:00 UTC (10 AM EST / 11 AM EDT)
- Reads market data from S3 (URL passed in via `client_payload.market_data_s3_url`)
- AI script via Claude → HeyGen avatar video → published to HGPG YouTube only (uses `YOUTUBE_REFRESH_TOKEN_HGPG`, NOT the SCR YouTube token)
- No IG/WP/email — the monthly market video is YouTube-only, higher editorial bar

## Workflow inputs (manual triggers)

`morning_fetch.yml` supports `workflow_dispatch` with:
- `reprocess_date` (YYYY-MM-DD) — clears that day's S3 hashes so stories re-process
- `enable_wordpress` (true/false) — skip WP on re-runs to avoid duplicate posts
- `story_url` — process a specific article URL (skips RSS fetch)

All brand-safety filters apply even on manual triggers.

## Secrets in GitHub Actions

- AI: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `HEYGEN_API_KEY`, `HEYGEN_VOICE_ID`
- AWS: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_BUCKET`
- Instagram: `INSTAGRAM_ACCESS_TOKEN` (60-day expiry), `INSTAGRAM_ACCOUNT_ID` (17841402996871007), `FACEBOOK_PAGE_ID`
- YouTube: `YOUTUBE_CLIENT_ID`, `YOUTUBE_CLIENT_SECRET`, `YOUTUBE_REFRESH_TOKEN` (SCR channel, currently unused), `YOUTUBE_REFRESH_TOKEN_HGPG` (HGPG channel, monthly only)
- WordPress: `WORDPRESS_SITE_URL`, `WORDPRESS_USERNAME` (`Brian@HomeGrownPropertyGroup.com`), `WORDPRESS_APP_PASSWORD`, `WORDPRESS_CATEGORY_IDS`, `WORDPRESS_TAG_IDS`
- Email: `GMAIL_SENDER_EMAIL`, `GMAIL_APP_PASSWORD`, `DAILY_DIGEST_EMAIL`
- Misc: `BRIAN_PHONE`

## Failure handling

- HTML digest email sends regardless of downstream success
- Red failure banner + subject prefix on the digest when downstream fails
- HeyGen failure short-circuits IG checks to prevent cascade false positives
- All errors land in the digest so Brian sees them in inbox without checking dashboards

## Key technical constraints

- **Instagram Reels:** `-movflags +faststart` MANDATORY. Moov atom at tail hard-fails Instagram CDN. YouTube tolerates it but IG does not. Hardcoded in video normalization.
- **Video spec:** H.264, known-good Reels spec
- **S3 dedup:** Stories are hashed and the hash is stored in S3. `reprocess_date` workflow input clears the day's hashes to allow re-processing.
- **Background image rotation (v4.1.4):** 11 AI-generated background images for video, 7-day rotation window tracked in S3 (mirrors music rotation system).
- **Instagram token rotation:** 60-day expiry. Set a calendar reminder. HANDOFF.md says last refreshed 2025-04-05 (stale — Brian has rotated since).

## Brand-safety filter (the heart of the system)

The "Helpful Expert" persona depends on aggressive filtering. Two-stage process:

- **Stage 1 (fast)** on RSS title/description: out-of-area, national news, sensitive keywords
- **Stage 2 (deep)** on full article: immigration crises, child exploitation, crime, political protests, school violence, opinion segments

Hard-blocks (additive, version by version):
- Crime/tragedy: murder, shootings, robbery, heists, fatal accidents, infant mortality, present-tense violence verbs (shoots, kills, stabs, robs, beats, attacks, assaults), 'charged after' phrasing, hit-and-run, vehicle-pedestrian/child injury (v4.1.7/v4.1.8)
- Controversial social: political protests, activism, ICE detentions, abortion, gun control, school violence, immigration crises (v4.1.1)
- Negative economic: gas prices, rising costs, inflation, business struggles/bankruptcies
- Historical retrospectives (allows current year mentions 2025–2026, blocks lookbacks)
- Child exploitation (v4.1.2): comprehensive keyword set
- Off-brand viral/opinion (v4.1.3): "The Edge:" segments, theft/property crime, drug crime, police shortage

Prioritized:
1. Real estate (highest priority)
2. Community & events
3. Infrastructure

**Rule:** Filters in `utils/content_filter.py` are NOT to be relaxed without explicit Brian approval.

## Audit findings — 2026-05-19 session

Brian asked for an audit + simplification suggestions. Captured here before any code touched.

**Structural observations (now informed by actual repo, not assumptions):**

1. **The monolithic script is bigger than it looks.** `run_morning_fetch_enhanced.py` is **182KB** (single file). Plus `south_charlotte_v4_enhancements.py` (19KB) and `social_media_monitor.py` (15KB). The 182KB orchestrator does fetch → filter → AI gen → HeyGen → IG → WP → email all in one process. When step 5 fails, the whole 20-min run has to be debugged as one blob. The failure-handling layer (digest-anyway, red banner, HeyGen short-circuit) is a workaround for the absence of checkpoints.

2. **Two-repo split for the daily pipeline is justified.** `south-charlotte-scraper` is the discovery engine writing to S3; `south-charlotte-report` is the consumer. They're decoupled at S3, which is the right boundary. Don't consolidate.

3. **A third repo exists.** `south-charlotte-market-report` generates the monthly market PDF and triggers the HGPG video pipeline. Brain didn't mention this repo. Add to infrastructure.md.

4. **YouTube for SCR is dormant.** Per HANDOFF, SCR YouTube Shorts are not in use; only IG + WP. The `YOUTUBE_REFRESH_TOKEN` env var is still wired but unused. Consider removing or documenting the dormancy explicitly.

5. **HeyGen gates SCR video distribution but not the blog.** Blog post publishes regardless of HeyGen success. So the worst-case failure mode is "no Reel today, blog still ships" — better than I initially assumed in the un-grounded audit. The failure decoupling is partially done already.

6. **Distribution does fan out from one Python script** (IG + WP + Gmail). Recent IG token refresh issues are real because direct Graph API auth has a 60-day rotation cycle. A scheduler (Buffer/Later) would offload IG auth lifecycle but adds cost + reduces flexibility.

7. **The "digest serves three audiences" critique partially applies.** Currently digest is to Brian only (`brian@homegrownpropertygroup.com`), so subscriber-bleed isn't a real risk — there are no external subscribers on this digest. If the goal is to grow a subscriber audience, the Beehiiv reference in `marketing.md` might be aspirational. Worth clarifying.

8. **No traffic feedback loop.** No UTMs on blog links or YouTube descriptions. SCR runs daily blind — can't tell if posts convert.

9. **Workflow comment is wrong.** `morning_fetch.yml` cron `0 10 * * *` is labeled "6:00 AM EST" but it's actually 5 AM EST / 6 AM EDT. Minor doc bug.

**Proposed simpler shape (revised given ground truth):**

The original 3-job proposal still mostly applies but adjusted for what's actually here:

    job: fetch_and_filter        (10:00 UTC daily, always runs)
      -> RSS + S3 scraper data, brand-safety filter, write market_data + stories.json as artifacts

    job: publish_text            (10:30 UTC, needs fetch_and_filter)
      -> WordPress + Gmail digest
      -> independent of HeyGen/IG, ships even if video pipeline fails

    job: produce_and_publish_video  (10:30 UTC, needs fetch_and_filter)
      -> HeyGen + IG Reel
      -> allowed to fail without blocking text publish

Benefits over current:
- Can re-run video step independently if HeyGen flakes (currently requires full re-run + reprocess_date workaround)
- Filter step can be re-run independently if a brand-safety bug is detected
- Each step gets its own logs upload, easier to debug
- Crucially: NO functional change to output. Same content, same channels, same cadence.

Costs:
- Touches 182KB of orchestrator code. Real surgery, not a quick YAML reshuffle.
- Adds artifact-passing complexity between jobs
- Doesn't fix anything that's currently broken — pipeline is humming

**Honest assessment:** the pipeline works. Brian said it's ops-mode, hands-off until it breaks. The refactor I'm proposing is "code health" not "fix a problem." If the next breakage is HeyGen-specific and forces a full re-run, that's the trigger to do this work. Until then, it's optional.

## Pickup notes

- Content quality has been steady; pipeline is mostly hands-off
- If output stops appearing, check Daily Digest first, then GitHub Actions log
- Instagram token: 60-day rotation. Last documented refresh 2025-04-05 (HANDOFF), Brian has rotated since but date not captured. Worth logging when next rotated.
- Brand-safety filters in `utils/content_filter.py` are NOT to be relaxed without Brian approval
- Manual story re-process: `workflow_dispatch` with `story_url` set
- Manual day re-process: `workflow_dispatch` with `reprocess_date` + `enable_wordpress: false`

## Open / pending

- **Refactor decision:** Brian to decide whether the 3-job split is worth the surgery on the 182KB orchestrator, or whether to leave the pipeline alone until next breakage forces a touch.
- **Workflow comment bug:** `morning_fetch.yml` cron comment says "6:00 AM EST" but should be "5:00 AM EST / 6:00 AM EDT". Trivial 1-line fix, worth shipping next time touching the repo.
- **YouTube SCR dormancy:** decide whether to remove or document the unused `YOUTUBE_REFRESH_TOKEN` env var.
- **Beehiiv reference in `marketing.md`:** clarify whether this is aspirational or whether there's a separate Beehiiv newsletter outside the workflow. The daily digest is Gmail-only.
- **Add `south-charlotte-market-report` to `infrastructure.md`** active build repos list.
- **Add UTMs** to blog/YouTube links if traffic feedback is wanted.

No active dev work otherwise. Ops-mode tool.
