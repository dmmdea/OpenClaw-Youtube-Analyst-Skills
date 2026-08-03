# OpenClaw YouTube Analyst — Claude Code skills

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg) ![Claude Code Skills](https://img.shields.io/badge/Claude_Code-2_Skills-8A2BE2) ![YouTube Data API](https://img.shields.io/badge/YouTube-Data_API_v3-FF0000) ![Google Cloud](https://img.shields.io/badge/Google_Cloud-OAuth_bootstrap-4285F4)

Two complementary Claude Code skills that power a YouTube channel analysis pipeline:

| Skill | Version | Purpose | When to invoke |
|---|---|---|---|
| [`Youtube-Analyzer-Setup-Skill`](./Youtube-Analyzer-Setup-Skill/SKILL.md) | v2 (2026-04-28) | One-time bootstrap: Google Cloud project + OAuth client + YouTube Data API key + Google Ads Manager wrapper + developer token + secrets.env layout + 6 MCP registrations + smoke test. | "set up youtube analyst", "bootstrap youtube analytics pipeline", "connect my youtube channel", standing up a second node. |
| [`Youtube-Analyst-Runbook`](./Youtube-Analyst-Runbook/SKILL.md) | v3 (2026-04-28) | Day-to-day operation: scan new channels, rerun packaging + matched-pair analyses, join ad spend for organic-vs-paid split, write findings reports, route forward-roadmap tracks (B/C/D/E/F/H), weekly maintenance. | "scan this channel", "rerun packaging analysis", "matched pair analysis", "write findings report", "weekly rescan", "expand peer cluster", "build transcript features", "regression check", "wire to telegram", "how should I read these findings". |

The setup skill hands off to the runbook once bootstrap is complete. They ship as one repo because they share triggers, paths, memory references, and the same underlying pipeline.

See [CHANGELOG.md](./CHANGELOG.md) for version history.

## What the pipeline does

Brings together six MCP servers to take a YouTube channel from raw handle to evidence-backed packaging recommendations:

- `youtube` (yt-dlp) — transcripts, no API key
- `youtube-data` (YouTube Data API v3) — library scans, stats, outlier scoring
- `youtube-analytics` (YouTube Analytics + Reporting APIs) — owner-only revenue, retention, traffic sources, audience geography
- `google-ads` (Google Ads API + CSV ingest fallback) — ad spend per video, organic-vs-paid split
- `openclaw-quota` — free-tier budget tracker
- `brave-search` (optional) — competitor discovery

An optional 7th MCP — **`hailo-vision`** — adds NPU-accelerated thumbnail features (CLIP embeddings, OCR entity extraction) on Hailo-8L hardware. Auto-detected at runtime via `maybe_hailo_backend()`; analyst scripts fall back cleanly to OpenCV-only features when absent. The Hailo lane lives in a separate sibling repo: [`dmmdea/hailo-youtube-stack-mcp`](https://github.com/dmmdea/hailo-youtube-stack-mcp). Bootstrap that separately if you have the hardware — there's a 1060× speedup on repeat scans via the content-addressed Vision Cache.

Analysis scripts (in a separate `youtube-data` MCP repo) produce:

- Matched-pair test results (Cohen's d + bootstrap CI, SAMPLE_FLOOR=30) — within-channel top 10% vs bottom 50%
- Head-to-head packaging comparisons across peer channels (56-feature vector)
- Organic outlier scoring that separates content quality from paid boost (asymmetric framing: subject channel uses `organic_outlier_score` + collab-drop, peers use raw `outlier_score`)
- Findings reports following the format codified in Runbook Workflow 10 (effect size + CI on every claim, HYPOTHESIS-ONLY flags below sample floor, mandatory algorithm-drift disclaimer)

## Statistical discipline (non-negotiable)

This pipeline is an **analyst that explains**, not an oracle that predicts. Every output carries:

1. **Effect size** — Cohen's d or relative-lift %.
2. **Confidence interval** — 95% CI via bootstrap or t-test.
3. **Sample floor** — n ≥ 30 per cohort, otherwise tagged `HYPOTHESIS ONLY`.
4. **Matched-pair frame** — within-channel top 10% vs bottom 50% (cross-channel only when explicitly labeled).
5. **Time-normalized metrics** — never raw views; always views/age or view-to-subscriber.
6. **Algorithm-drift disclaimer** — every report carries the snapshot date.

What the agent will NOT do:

- Score thumbnails 0–100 (uncalibrated scoring is astrology)
- Tell you "do X to go viral" (charlatan territory)
- Claim to know what the algorithm wants

## Install

Each skill is a standard Claude Code skill: a single `SKILL.md` under `~/.claude/skills/<skill-name>/`.

```bash
mkdir -p ~/.claude/skills/
cp -r Youtube-Analyzer-Setup-Skill ~/.claude/skills/
cp -r Youtube-Analyst-Runbook     ~/.claude/skills/
```

Restart your Claude Code session. Invoke by saying any of the trigger phrases above.

For NPU acceleration, also follow the sibling repo: <https://github.com/dmmdea/hailo-youtube-stack-mcp>.

## Reading the skills directly

Both `SKILL.md` files are dense, copy-pasteable command-line playbooks. The **Runbook** documents 10 numbered workflows + Active tracks + Operational discipline + When-to-ask + Troubleshooting + 16 related-memory references. The **Setup-Skill** documents 8 numbered steps + smoke tests + a Dell verification record + Sibling skills + 8 related-memory references.

If you prefer to read rather than invoke, scan section headers (`## Workflow N — …`) to locate the workflow you want.

## License

MIT — see [LICENSE](./LICENSE). Use these skills to analyse your own channel; respect YouTube's and Google's terms when analysing channels you don't own.

## Related

- **Sibling repo:** [`dmmdea/hailo-youtube-stack-mcp`](https://github.com/dmmdea/hailo-youtube-stack-mcp) — Hailo-Stack-Skill, `hailo-vision` MCP, `openclaw_shared.cache.VisionCache`, DKMS kernel patch. NPU-accelerated thumbnail features, optional.
- **MCP server fleet** — `openclaw-mcp-servers` (the six MCPs above; public repo forthcoming).
- **Drive canonical:** `My Drive / AI Ecosystem / Skill and Gem Creation / Skills / _current / <skill-name>/SKILL.md`.
