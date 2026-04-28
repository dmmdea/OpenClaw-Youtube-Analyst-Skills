---
name: Youtube-Analyst-Runbook
description: Day-to-day operational runbook for the OpenClaw YouTube analyst pipeline — scanning new channels, running packaging + matched-pair analyses, joining ad spend for organic-vs-paid split, interpreting findings with the statistical-discipline contract, and weekly maintenance (OAuth re-auth, ads CSV refresh, quota check). Use when the user says "scan this channel", "analyze this youtube channel", "rerun packaging analysis", "matched pair analysis", "add peer channel", "run library scan", "refresh youtube analyst data", "ingest ads csv", "redo competitor analysis", "how should I read these findings", or anything else about EXECUTING analysis after the pipeline is set up (separate skill `Youtube-Analyzer-Setup-Skill` handles first-time bootstrap).
---

# YouTube Analyst Runbook

Operational playbook for the live pipeline. Assumes the setup skill has already run (6 MCPs connected, OAuth tokens on disk, secrets.env populated, Ads Manager wired up). Covers discovery → library scan → packaging analysis → matched-pair tests → honest interpretation → maintenance.

## When to use

- Scanning / adding a new channel to the analysis set: "scan @handle", "add peer channel X"
- Rerunning packaging/matched-pair after new videos, a fresh ads CSV, or a code change to features
- Interpreting findings before writing a report ("how should I read d=+1.35?", "is this result real?")
- Weekly maintenance: OAuth re-auth (7-day Testing-mode expiry), fresh Google Ads CSV export + ingest, quota check

**Out of scope:** first-time Google Cloud / Ads Manager / API key bootstrap → use `Youtube-Analyzer-Setup-Skill`.

## Paths you'll use a lot

```
~/openclaw-mcp-servers/youtube-data/          MCP + scripts/ dir
~/openclaw-mcp-servers/youtube-data/.venv/    venv for running scripts
~/openclaw-output/youtube-analyst/             all outputs
  ├─ week-1/<market>/<channel>/library.xlsx    one per scanned channel
  ├─ week-2/                                   packaging + matched-pair reports
  ├─ thumbnails/<channel-slug>/<video_id>.jpg  downloaded thumbnails
  ├─ ads-exports/                              Google Ads UI CSV dumps (Danmar only)
  ├─ roi/                                      per-video ROI joins
  └─ config/danmar_collab_video_ids.txt        optional whitelist, one video_id per line
```

Playbook spec (source of truth for the full pipeline design):
`/home/dmmdea/AI Ecosystem/OpenClaw/docs/superpowers/specs/2026-04-21-youtube-expert-analyst-playbook.md`

## Run scripts with the youtube-data venv

Every script must be invoked with the youtube-data MCP's venv (has pandas, openpyxl, cv2, google-api-python-client + the editable `openclaw_shared`). From `~/openclaw-mcp-servers/youtube-data/`:

```
cd ~/openclaw-mcp-servers/youtube-data
./.venv/bin/python -m scripts.<script_name>
```

`-m scripts.<name>` (not `./.venv/bin/python scripts/<name>.py`) is the correct invocation — scripts now import the sibling `scripts._danmar_ads_helper` module, and the `-m` form sets package context.

## Quota & rate-limit discipline

Two distinct limits are in play; conflate them at your peril.

**1. YouTube Data API quota** — global per-project, 10k units/day. Track via `openclaw-quota` MCP (wired into every Data API tool). Always check before a bulk run:

```
./.venv/bin/python -c "from openclaw_shared.quota_budget import QuotaBudget; print(QuotaBudget('youtube_data').status())"
```

Costs: `channel_info` 1 unit, `channel_videos` ⌈N/50⌉, `videos_batch_stats` ⌈N/50⌉. A single library scan ≈ 50 units; 8 channels ≈ 400 units — well under daily cap.

**2. YouTube caption-endpoint IP rate limit** — a per-IP throttle on `timedtext` (captions), unrelated to Data API quota. Symptoms when tripped:
- `youtube-transcript-api` raises `IpBlocked`
- `yt-dlp` returns `HTTP Error 429: Too Many Requests` even with `--impersonate chrome` (curl_cffi)

Notes:
- Single-video fetches during normal use are fine. ~10 back-to-back bulk pulls was enough to trip it on Dell 2026-04-21.
- yt-dlp's info-JSON endpoints (metadata, search) are NOT affected — only `timedtext`/captions.
- Reset window: 1–6 h, per-IP.

Mitigations for any bulk transcript job (transcript-feature track, recency cross-checks, peer-cluster expansion):
- Space requests ≥ 30 s apart
- Route through a mobile hotspot or VPN for fast bulk passes
- Wait it out (1–6 h) if you've already tripped it

The `youtube` MCP (`~/openclaw-mcp-servers/youtube/`) uses the same two transports, so it surfaces the same error — there's no "use a different MCP" escape hatch.

## Workflow 1 — Scan a new channel

```
cd ~/openclaw-mcp-servers/youtube-data
./.venv/bin/python -m scripts.library_scan @ChannelHandle
# OR a channel_id:
./.venv/bin/python -m scripts.library_scan UCxxxxxxxxxxxxxxxxxxxxxx
```

Writes `~/openclaw-output/youtube-analyst/week-1/<slug>/library.xlsx` with all videos + stats + `outlier_score`. Quota cost: 1 unit for channel resolve + ceil(N/50) units for the video list + ceil(N/50) for batched stats hydration. Check before running:

```
./.venv/bin/python -c "from openclaw_shared.quota_budget import QuotaBudget; print(QuotaBudget('youtube_data').status())"
```

Then download thumbnails (no API cost, CDN scrape):

```
./.venv/bin/python -m scripts.download_thumbnails @ChannelHandle <thumb_slug>
```

## Workflow 2 — Add a peer channel to competitor analysis

1. Scan the channel (Workflow 1).
2. Add it to the `CHANNEL_INFO` dict in `scripts/competitor_title_matched_pair.py` with market + tier.
3. For packaging comparison, also add an entry to `CHANNEL_SPECS` in `scripts/packaging_analysis.py`.
4. Rerun Workflow 3.

New peer channels do NOT get ad-spend data — they keep raw `outlier_score`. This asymmetry is intentional; see `feedback_cross_reference_ads_before_citing_outliers.md`.

## Workflow 3 — Rerun packaging + matched-pair after new data

Three scripts, always in this order (each reads artifacts from the previous):

```
cd ~/openclaw-mcp-servers/youtube-data
./.venv/bin/python -m scripts.title_matched_pair_analysis      # Danmar-only, organic score
./.venv/bin/python -m scripts.packaging_analysis               # Danmar recent vs peer outliers
./.venv/bin/python -m scripts.competitor_title_matched_pair    # Cross-channel d-matrix
```

Outputs land in `~/openclaw-output/youtube-analyst/week-2/`:
- `title_analysis.xlsx` + `FINDINGS.md` (within-channel top10% vs bottom50%)
- `packaging_analysis.xlsx` + `PACKAGING_FINDINGS.md` (head-to-head vs peers, 56-feature vector)
- `competitor_title_analysis.xlsx` + `COMPETITOR_FINDINGS.md` (universal-vs-Danmar-specific effects)

**Danmar uses `organic_outlier_score`; peers use raw `outlier_score`.** Collab videos (title mentions `@handle` or contains "saludo especial"/"ft."/"colaboración", or listed in `~/openclaw-output/youtube-analyst/config/danmar_collab_video_ids.txt`) are automatically dropped from Danmar cohorts.

## Workflow 4 — Refresh ads CSV (weekly or before a re-analysis)

1. In Google Ads UI (https://ads.google.com), switch to the Manager account (143-213-9099), drill into 212-310-0176.
2. Export fresh CSVs for each report type — **Campaigns**, **Ads**, **Ad groups**, **Asset groups**, **Promotions** (from YouTube Studio). Leave default column sets.
3. Drop all CSVs into `~/openclaw-output/youtube-analyst/ads-exports/` (replace prior files).
4. When Basic access lands, the MCP path (`ads_video_performance`, `ads_daily_spend`, etc.) replaces the CSV path — analyst-side scripts don't need changes since the schema is identical.

Trigger a rerun of `top5_with_ads.py` + Workflow 3 after new CSVs land — the organic outlier scores and per-video ROI change with fresh data.

When Google Ads Basic access lands (email approval pending to dmmdea@hotmail.com), Workflow 4's CSV step is replaced by a live API path — see Track A in `~/openclaw-output/youtube-analyst/CONTINUATION_NON_HAILO.md` for the cutover procedure (smoke test, reconcile against CSV totals, swap `top5_with_ads.py` + `per_video_roi_generator.py` to live-query, keep CSV as cached fallback).

## Workflow 5 — Weekly OAuth re-auth (7-day Testing-mode expiry)

Symptom that it's time: any `youtube-analytics` or `google-ads` tool returns `invalid_grant`.

```
/home/dmmdea/openclaw-mcp-servers/youtube-analytics/.venv/bin/python \
  /home/dmmdea/openclaw-mcp-servers/youtube-analytics/scripts/oauth_auth.py
```

Critical gotcha: at the Google account-picker, pick the **brand account that owns the YouTube channel**, NOT the personal Google account. Wrong selection silently queries the wrong channel.

Fix the 7-day cycle permanently by moving the Cloud Console OAuth consent screen from **Testing** to **Production** (requires Google brand verification).

## Workflow 6 — Interpreting findings (statistical-discipline contract)

Every claim in a report must carry its confidence. These rules are non-negotiable:

1. **SAMPLE_FLOOR = 30 per cohort.** If either top or bottom cohort has fewer than 30 videos, the result is tagged `HYPOTHESIS ONLY, n<30`. Never promote a hypothesis-only finding to a confirmed conclusion in a report. Danmar currently has 68 scored videos → top cohort = 6. All Danmar-internal findings are hypothesis-only until the library grows past ~300.
2. **Cohen's d thresholds:** |d|<0.2 negligible, 0.2–0.5 small, 0.5–0.8 medium, 0.8+ large. Report the size, don't just say "there's an effect."
3. **CI excludes zero** = the effect direction is stable across bootstrap resamples (higher confidence). CI crosses zero = directionally uncertain.
4. **Cross-reference ads before citing outlier wins on Danmar.** Raw `outlier_score` on a promoted video conflates packaging quality with paid boost. Always also report `ad_views_claimed` and `ad_spend_usd` for any specific video called out as "working well". See `feedback_cross_reference_ads_before_citing_outliers.md`.
5. **Asymmetric framing.** Danmar findings are filtered through organic_outlier_score + collab-drop. Peer findings are raw. State this asymmetry in every report that compares the two.

## Workflow 7 — Quick recency cross-checks

Before writing advice based on historical patterns, verify the pattern still holds in recent content — creators self-correct. Example: in Week 1 analysis, Danmar's Colombia-peso pricing was flagged as a weakness; by the time of Week 2 it had already dropped from 15% to 0% in recent titles.

```
./.venv/bin/python -m scripts.recent_content_analysis     # last 30 videos only
```

If the historical weakness has already been fixed, drop it from the recommendations.

## Workflow 8 — Refreshing a specific subset

- **Just thumbnails, no stats refresh:** `./.venv/bin/python -m scripts.download_thumbnails @handle <slug>`
- **Just the Danmar top-5 with ads snapshot:** `./.venv/bin/python -m scripts.top5_with_ads`
- **Revenue-weighted report (market CPM-weighted findings):** `./.venv/bin/python -m scripts.revenue_weighted_report`
- **Resolve a batch of candidate handles → channel_ids:** `./.venv/bin/python -m scripts.resolve_candidates`

## Workflow 9 — Hailo-accelerated features (when device present)

Packaging analysis automatically picks up Hailo for richer thumbnail features (extra CLIP embeddings, OCR entity extraction) when `/dev/hailo0` exists. No manual switch — `openclaw_shared/features/thumbnail.py` calls `extract_thumbnail_features(path, backend=maybe_hailo_backend())`, which returns a Hailo backend if available and `None` otherwise. The OpenCV-only path remains the deterministic fallback; both paths emit the same dict schema, so downstream matched-pair stats don't care which produced the features.

Verify Hailo health before a bulk run that's expected to use it:

```
ls /dev/hailo*                          # expect /dev/hailo0
hailortcli fw-control identify          # expect Board=Hailo-8, Firmware 4.23.0
```

If either fails, packaging silently falls back to OpenCV — that's correct behaviour, but you'll want to know: `PACKAGING_FINDINGS.md` will show fewer feature columns than a Hailo-enabled run, and any `week-3-hailo/` deliverables will skip the Hailo-only columns.

Hailo runtime, DKMS driver, HEF management, kernel-patch lifecycle, and the OCR-quality pipeline are out of scope for this skill. They live in a separate `hailo-stack` skill (TBD as of 2026-04-28). Do **not** reach into `~/openclaw-mcp-servers/hailo-vision/` or `/usr/src/hailo_pci-*/` from analyst scripts — go through `maybe_hailo_backend()`.

## Operational discipline

Invariants that hold across every run, every track, every report:

- **Standalone-first.** Don't introduce cluster, DB, or external-service dependencies. If a track needs external data, fetch once and cache to `~/openclaw-output/youtube-analyst/cache/<source>/`. Every analysis must be reproducible on a single node.
- **Checkpoint to disk.** Anything multi-hour writes intermediate state under `~/openclaw-output/youtube-analyst/<workflow>/` so an interruption (kernel update, OAuth expiry, network blip) doesn't lose work. Subsequent runs check markers (`phase_N_done.marker`) before redoing expensive steps.
- **Quota first.** Call `openclaw-quota` before any YouTube/Brave op above single-call cost. If the budget is short, abort early — never half-run a bulk job and leave artifacts in inconsistent state.
- **Quality mode is default.** Local optimizes for quality; overnight runs are expected. Don't pessimize accuracy to fit a 10-minute window.
- **Don't regress W1/W2 deliverables.** Existing sheets and column names in `library.xlsx`, `title_analysis.xlsx`, `packaging_analysis.xlsx`, `competitor_title_analysis.xlsx`, `per_video_roi.xlsx` are reference points the user has built downstream Excel/Looker links against. **Add columns; never rename them.**
- **Re-auth weekly while OAuth is in Testing.** See Workflow 5. The 7-day refresh-token expiry is a hard floor; schedule a reminder.

## When to ask the user before acting

- Adding peers whose market/role isn't obvious from their handle
- Expanding the peer cluster beyond 20 channels (statistical noise vs cost trade-off)
- Any change to sheet/column naming conventions
- Cross-platform expansion (Meta, TikTok) — scope is weeks, not hours
- Shipping any skill update to GitHub (Drive → verify → GitHub → memory pipeline)
- When a re-run flips the sign of a previously validated finding — the data may be right, but verify before publishing

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `invalid_grant` on any owner-side MCP call | Testing-mode 7-day token expired | Workflow 5 — re-run `oauth_auth.py` |
| `ModuleNotFoundError: scripts._danmar_ads_helper` | Ran as `python scripts/foo.py` instead of `-m scripts.foo` | Use the `-m` form |
| `test_access_limitation` on every `ads_*` API call | Google hasn't granted Basic access yet | Keep using the CSV-ingest path (Workflow 4); check dmmdea@hotmail.com for the approval email |
| Packaging findings differ wildly between runs | Ads CSV was refreshed OR collab whitelist changed | Expected — rerunning with new data produces new cohorts |
| Danmar's top cohort shrinks to <6 videos | Too many collabs + promoted videos, library too small | Directional signals only; document `HYPOTHESIS ONLY` |
| Outlier score on a specific video looks wrong | Reference median window is 365 days by default | Override via `channel_outlier_scores(..., recent_window_days=N)` |
| Pyright "can't find `openclaw_shared.*`" in IDE | Venv not fully picked up | Each MCP has pyrightconfig with `venvPath` + `venv`; workspace-level `pyproject.toml` at `~/openclaw-mcp-servers/`. Reopen IDE. Runtime is unaffected. |

## Related memory files (read these before publishing any finding)

- `feedback_cross_reference_ads_before_citing_outliers.md` — the #1 analytical rule
- `project_user_channel_danmar_auto_reviews.md` — revenue + audience facts, biggest known packaging gaps
- `reference_openclaw_packaging_analysis.md` — feature extractors + matched-pair stats module
- `reference_openclaw_youtube_data_mcp.md` — Data API MCP tools + quota costs
- `reference_openclaw_youtube_analytics_mcp.md` — owner-side Analytics MCP + OAuth lifecycle
- `reference_openclaw_google_ads_mcp.md` — ad-spend MCP + CSV ingest fallback
- `reference_openclaw_quota_mcp.md` — shared quota tracker

## Sibling skill

`Youtube-Analyzer-Setup-Skill` handles one-time bootstrap (Cloud Console APIs, OAuth consent flow, Ads Manager wrapper, developer token, MCP registrations). If any of those is missing, run it first.
