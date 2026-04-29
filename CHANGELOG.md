# Changelog

All notable changes to the OpenClaw YouTube Analyst skill pair are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Each skill carries its own version (in the `**Version:**` line near the top of its `SKILL.md`) and the two skills evolve independently — entries below are grouped per ship, with the affected skill called out.

## [Runbook v3] — 2026-04-28

Commit: [`9178a68`](https://github.com/dmmdea/OpenClaw-Youtube-Analyst-Skills/commit/9178a68) (parent `e097cfd`)

### Added — Runbook
- **Workflow 10 — Writing a findings report.** Codifies the `FINDINGS.md` / `PACKAGING_FINDINGS.md` / `COMPETITOR_FINDINGS.md` format, required sections in order, mandatory rules (no scoring out of 100, no "do X to go viral", HYPOTHESIS-ONLY non-promotion, ad cross-reference, no column renames). Skeleton: most-recent `week-2/FINDINGS.md` as canonical template.
- **Active tracks** section. Forward-roadmap pointer listing tracks A–H with brief descriptions plus a trigger-phrase routing block. Cross-links `~/openclaw-output/youtube-analyst/SKILL_PLAN.md` (priority/effort) and `CONTINUATION_NON_HAILO.md` (per-track invariants). Track G (cross-platform) stays explicitly deferred per playbook §9 + §12.
- Version stamp `**Version:** v3 — 2026-04-28` after the H1.

### Changed — Runbook
- **Description frontmatter** extended with trigger phrases for tracks B/C/D/E/F/H ("weekly rescan", "expand peer cluster", "build transcript features", "regression check", "wire to telegram", etc.) so user phrases route correctly.
- **Related memory files** extended from 7 to 16 entries, organized into Doctrine / Skill cross-reference / MCP references / Project context / Active plan groups. Mirrors the 15-file ordered list from `CONTINUATION_NON_HAILO.md` plus `SKILL_PLAN.md` as meta-pointer.

233 → 297 lines (+70 / −6).

## [Setup-Skill v2] — 2026-04-28

Commit: [`e097cfd`](https://github.com/dmmdea/OpenClaw-Youtube-Analyst-Skills/commit/e097cfd) (parent `85276a3`)

### Added — Setup-Skill
- Version stamp `**Version:** v2 — 2026-04-28` and a scope clarifier noting NPU acceleration lives in sibling repo [`dmmdea/hailo-youtube-stack-mcp`](https://github.com/dmmdea/hailo-youtube-stack-mcp).
- **Optional 7th MCP** pointer in "What you get after running through" — `hailo-vision` is owned by the sibling repo's Hailo-Stack-Skill, auto-detected at runtime per Runbook Workflow 9, no impact on YouTube pipeline if absent.
- **Reference: Dell verification (2026-04-28)** record after Step 8 smoke tests — captures the end-to-end install verified live on `workernode/10.0.0.79` against `@DanmarAutoReviews` (all 4 smoke tests passing).
- New **Sibling skills** section pointing to Runbook (this same repo) and Hailo-Stack-Skill (sibling repo).

### Changed — Setup-Skill
- **Related memory files** extended with `reference_youtube_analyst_skills.md` and `feedback_skill_shipping_protocol.md`.

198 → 221 lines.

## [Runbook v2.1] — 2026-04-28

Commit: [`85276a3`](https://github.com/dmmdea/OpenClaw-Youtube-Analyst-Skills/commit/85276a3) (squash-merge of PR #1, parent `db01501`)

### Changed — Runbook
- **Workflow 9** — replaced placeholder `(TBD as of 2026-04-28)` with live link to sibling repo [`dmmdea/hailo-youtube-stack-mcp`](https://github.com/dmmdea/hailo-youtube-stack-mcp). Now references `openclaw_shared.cache.VisionCache` and the **1060× speedup on repeat scans**.

1 file, +1 / −1.

## [Runbook v2] — 2026-04-28

Commit: [`db01501`](https://github.com/dmmdea/OpenClaw-Youtube-Analyst-Skills/commit/db01501) (parent `4629aea`)

### Added — Runbook
- **Quota & rate-limit discipline** section after the venv setup. Covers both the YouTube Data API 10k/day quota AND the per-IP caption-endpoint throttle (`timedtext` IpBlocked / HTTP 429) observed during bulk transcript pulls on Dell 2026-04-21. Mitigations: ≥30 s spacing, hotspot/VPN, 1–6 h cooldown.
- **Workflow 9 — Hailo-accelerated features (when device present).** Auto-detect via `/dev/hailo0` + `hailortcli fw-control identify` health check. Defers stack management to a separate `hailo-stack` skill (TBD at the time of this ship; resolved in v2.1 to live in sibling repo).
- **Operational discipline** section capturing invariants from `CONTINUATION_NON_HAILO.md`: standalone-first, checkpoint to disk, quota first, quality-mode default, don't rename W1/W2 sheet columns, weekly OAuth re-auth.
- **When to ask the user before acting** section listing the six "ask first" cases (peer expansion >20 channels, cross-platform, sheet/column renames, sign-flip on validated findings, etc.).
- Forward pointer in **Workflow 4** to Track A in `CONTINUATION_NON_HAILO.md` for the Google Ads Basic-access live-API cutover (gated on email).

168 → 233 lines (+65 / −0).

## [v1] — 2026-04-22

Commit: [`4629aea`](https://github.com/dmmdea/OpenClaw-Youtube-Analyst-Skills/commit/4629aea)

### Added
- Initial publish of two complementary skills:
  - **Youtube-Analyzer-Setup-Skill** — one-time bootstrap (Cloud Console APIs, OAuth consent flow, Ads Manager wrapper, developer token, MCP registrations).
  - **Youtube-Analyst-Runbook** — day-to-day operations (library scans, packaging analysis, matched-pair tests, weekly maintenance, findings interpretation).
- 6 MCPs registered: `brave-search`, `youtube` (yt-dlp), `openclaw-quota`, `youtube-data` (Data API v3), `youtube-analytics` (Analytics + Reporting APIs), `google-ads`.
- MIT license, public repo at <https://github.com/dmmdea/OpenClaw-Youtube-Analyst-Skills>.
