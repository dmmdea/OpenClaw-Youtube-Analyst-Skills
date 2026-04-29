# OpenClaw YouTube Analyst — Claude Code skills

Two complementary Claude Code skills that power a YouTube channel analysis pipeline:

| Skill | Purpose | When to invoke |
|---|---|---|
| [`Youtube-Analyzer-Setup-Skill`](./Youtube-Analyzer-Setup-Skill/SKILL.md) | One-time bootstrap: Google Cloud project + OAuth client + YouTube Data API key + Google Ads Manager wrapper + developer token + secrets.env layout + 6 MCP registrations + smoke test. | "set up youtube analyst", "bootstrap youtube analytics pipeline", "connect my youtube channel", standing up a second node. |
| [`Youtube-Analyst-Runbook`](./Youtube-Analyst-Runbook/SKILL.md) | Day-to-day operation: scan new channels, rerun packaging + matched-pair analyses, join ad spend for organic-vs-paid split, apply the statistical-discipline contract, weekly maintenance (OAuth re-auth, ads CSV refresh, quota check). | "scan this channel", "rerun packaging analysis", "matched pair analysis", "refresh analyst data", "how should I read these findings". |

The setup skill hands off to the runbook skill once bootstrap is complete. They're shipped as one repo because they share triggers, paths, memory references, and the same underlying pipeline.

## What the pipeline does

Brings together six MCP servers to take a YouTube channel from raw handle to evidence-backed packaging recommendations:

- `youtube` (yt-dlp) — transcripts, no API key
- `youtube-data` (YouTube Data API v3) — library scans, stats, outlier scoring
- `youtube-analytics` (YouTube Analytics + Reporting APIs) — owner-only revenue, retention, traffic sources, audience geography
- `google-ads` (Google Ads API + CSV ingest fallback) — ad spend per video, organic-vs-paid split
- `openclaw-quota` — free-tier budget tracker
- `brave-search` (optional) — competitor discovery

Analysis scripts sit in a separate `youtube-data` MCP repo and produce matched-pair test results (Cohen's d + bootstrap CI, SAMPLE_FLOOR=30), head-to-head packaging comparisons vs peers, and organic outlier scoring that separates content quality from paid boost.

## Install

Each skill is a standard Claude Code skill: a single `SKILL.md` under `~/.claude/skills/<skill-name>/`.

```
mkdir -p ~/.claude/skills/
cp -r Youtube-Analyzer-Setup-Skill ~/.claude/skills/
cp -r Youtube-Analyst-Runbook     ~/.claude/skills/
```

Restart your Claude Code session. Invoke by saying any of the trigger phrases in the tables above.

## Reading the skills directly

Both SKILL.md files are dense, copy-pasteable command-line playbooks under ~200 lines each. If you prefer to read them as reference rather than invoke them, start with the Table of Contents at the top of each file.

## License

MIT. Use these skills to analyse your own channel; respect YouTube's and Google's terms when analysing channels you don't own.

## Related

- `openclaw-mcp-servers` — the six MCP servers referenced above (public repo forthcoming)
- Google Drive canonical copy: `My Drive / AI Ecosystem / Skill and Gem Creation / Skills / _current / <skill-name>/SKILL.md`
