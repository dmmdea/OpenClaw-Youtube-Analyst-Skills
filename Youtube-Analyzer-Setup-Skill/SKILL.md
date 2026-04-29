---
name: Youtube-Analyzer-Setup-Skill
description: Bootstraps the full OpenClaw YouTube analyst pipeline on a fresh machine — Google Cloud project + OAuth client + YouTube Data API key + Google Ads Manager account + developer token + secrets.env layout + OAuth consent flow + MCP registrations + end-to-end smoke test. Use when the user says "set up youtube analyst", "bootstrap youtube analytics pipeline", "connect my youtube channel to openclaw", "connect google ads to openclaw", "install openclaw youtube analyst setup", or is standing up a second node (Aorus, Lenovo) and needs to replicate the Dell setup.
---

# YouTube Analyzer Setup

**Version:** v2 — 2026-04-28
**Scope:** YouTube analyst pipeline only (6 MCPs + OAuth + Google Ads wrapper + secrets layout). Optional NPU acceleration (Hailo-8L) lives in the sibling repo [`dmmdea/hailo-youtube-stack-mcp`](https://github.com/dmmdea/hailo-youtube-stack-mcp) — bootstrap that separately if/when you want it.

One-time bootstrap flow to stand up the full OpenClaw YouTube analyst pipeline. Captures all real friction encountered during the Dell installation 2026-04-21 so a fresh machine can reach end-to-end analysis without bespoke walkthroughs.

## When to use

Trigger on any of:
- "set up youtube analyst", "bootstrap youtube analytics pipeline"
- "connect my youtube channel to openclaw", "connect google ads to openclaw"
- "install openclaw youtube analyst setup"
- Standing up a second node and replicating the Dell setup

## What you get after running through

6 registered MCPs: `brave-search`, `youtube` (yt-dlp), `openclaw-quota`, `youtube-data` (Data API v3), `youtube-analytics` (Analytics API + Reporting API), `google-ads`. All 6 connected in `claude mcp list`. OAuth flow complete with 4 scopes. Ad-spend data accessible either via API (once Basic access lands) or via CSV ingest (immediately).

**Optional 7th MCP — `hailo-vision`:** for NPU-accelerated thumbnail features on Hailo-8L hardware. Owned by the sibling repo [`dmmdea/hailo-youtube-stack-mcp`](https://github.com/dmmdea/hailo-youtube-stack-mcp). Not part of this skill — bootstrap separately via that repo's `Hailo-Stack-Skill` if you have the hardware. The YouTube analyst auto-detects it at runtime per Runbook Workflow 9; no impact on the YouTube pipeline if absent.

## Prerequisites

- Python 3.12, `uv` or `pip`, Node.js (for `brave-search` npx MCP)
- The Google account that OWNS the YouTube channel being analysed (brand account, if applicable — see gotcha below)
- `~/openclaw-mcp-servers/` checked out / copied from the Dell reference (`_shared`, `youtube`, `youtube-data`, `youtube-analytics`, `google-ads`, `quota`)
- Existing Google Ads account with at least one campaign (for the Ads half of the setup)

## Step 1 — Google Cloud Console: project + APIs + OAuth client

1. Open https://console.cloud.google.com/
2. Create project `openclaw-youtube` (or select it if it exists)
3. **Enable 4 APIs** (each is a separate switch — missing any one causes confusing errors later):
   - YouTube Data API v3
   - YouTube Analytics API
   - YouTube Reporting API
   - Google Ads API ← **GOTCHA**: separate from the YouTube APIs, easy to miss. Symptom if missed: `Google Ads API has not been used in project X` error on the first live query.
4. **OAuth consent screen:**
   - User type: **External**
   - Publishing status: **Testing** (Production requires Google brand review)
   - Add the YouTube-channel-owning Google account as a **Test User**
   - **GOTCHA**: if the user's channel is on a **Brand Account** (separate from the personal Google account), add **that Brand Account's email** as the test user. Adding only the personal email will work for consent but the OAuth tokens will then target the wrong channel.
5. **Create OAuth 2.0 Client ID:**
   - Application type: **Desktop app** (NOT Web — the OAuth flow we use is the installed-app flow)
   - Name: e.g. `openclaw-youtube-desktop`
   - Download the client JSON → save to `/home/dmmdea/.openclaw/secrets/youtube_oauth_client.json`
   - `chmod 600 /home/dmmdea/.openclaw/secrets/youtube_oauth_client.json`

## Step 2 — YouTube Data API key (separate from OAuth)

1. Still in the `openclaw-youtube` project → APIs & Services → Credentials → Create credentials → API key
2. Restrict the key to **YouTube Data API v3** only (defence-in-depth; prevents accidental broad exposure)
3. Copy the key (looks like `AIzaSy...`)
4. Append to `/home/dmmdea/.openclaw/secrets.env`:
   ```
   YOUTUBE_API_KEY=AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
   ```

## Step 3 — Google Ads: Manager account wrapper (REQUIRED gotcha)

The Google Ads API Center — where developer tokens are issued — is **only available inside a Manager account**. A regular ad account cannot apply for a developer token directly.

1. Go to https://ads.google.com/home/tools/manager-accounts/
2. Click "Create a manager account"
3. Mode: **"Manage my other accounts"** (NOT "Manage my own advertising")
4. Give it a name (e.g. `DanmarMediaProductions`). Note the new **manager customer ID** (format `NNN-NNN-NNNN`).
5. From the Manager account, go to Accounts → Performance → `+` (add) → "Link existing account". Enter the sub-account's customer ID and send invite.
6. Sign in to the **sub-account** and accept the invite.
7. Note both:
   - **Manager customer ID** (login_customer_id — sent on every GAQL request as a header)
   - **Target sub-account customer ID** (the account that actually ran ads)

## Step 4 — Google Ads: developer token

1. From the Manager account → Tools & Settings (wrench icon) → Setup → **API Center**
2. Apply for **Basic access**:
   - Company type: **Advertiser**
   - Intended use: brief description (e.g. "Internal analytics — joining ad spend to YouTube performance for owned channels")
   - Approval typically lands in 1–3 business days via email
3. In the meantime, you'll see a **Test access** token already issued. Test access queries only sandbox accounts — it's useful for wiring verification, not real data.
4. Append the token to `/home/dmmdea/.openclaw/secrets.env`:
   ```
   GOOGLE_ADS_DEVELOPER_TOKEN=<token-string>
   ```

## Step 5 — `~/.openclaw/secrets.env` full template

```
# YouTube Data API (API-key auth, public-data endpoints)
YOUTUBE_API_KEY=AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

# Google Ads API (OAuth token + headers)
GOOGLE_ADS_DEVELOPER_TOKEN=<dev-token-from-API-Center>
GOOGLE_ADS_LOGIN_CUSTOMER_ID=<manager-customer-id-no-dashes>      # e.g. 1432139099
GOOGLE_ADS_CUSTOMER_ID=<target-sub-account-customer-id-no-dashes> # e.g. 2123100176

# Brave Search (optional — if registering brave-search MCP)
BRAVE_API_KEY=<brave-key>
```

```
chmod 600 /home/dmmdea/.openclaw/secrets.env
```

## Step 6 — Run OAuth consent flow

```
/home/dmmdea/openclaw-mcp-servers/youtube-analytics/.venv/bin/python \
  /home/dmmdea/openclaw-mcp-servers/youtube-analytics/scripts/oauth_auth.py
```

What it does: launches the installed-app OAuth flow (opens a browser), prompts for account choice, asks for consent to 4 scopes (`youtube.readonly`, `yt-analytics.readonly`, `yt-analytics-monetary.readonly`, `adwords`), writes tokens to `/home/dmmdea/.openclaw/secrets/youtube_oauth_tokens.json`.

**CRITICAL at the account-chooser screen:** pick the **brand account that owns the YouTube channel**, NOT the personal Google account. Getting this wrong silently queries the wrong channel and reports will be empty/confusing. If in doubt, abort the browser flow and re-run.

**7-day expiry (Testing mode):** the refresh token expires every 7 days until the OAuth consent screen is moved to **Production** (requires Google brand verification). Symptom when it expires: all `youtube-analytics` and `google-ads` tools return `invalid_grant`. Fix: re-run the same script. Consider a weekly reminder.

## Step 7 — Register all 6 MCPs

Run each of these. Each adds a single MCP entry in user-scope config (`~/.claude.json`).

```
# Quota tracker — must be registered before data-consuming MCPs
claude mcp add -s user openclaw-quota -- \
  /home/dmmdea/openclaw-mcp-servers/quota/.venv/bin/python \
  /home/dmmdea/openclaw-mcp-servers/quota/server.py

# YouTube Data API v3 (API-key, public data)
claude mcp add -s user youtube-data -- \
  /home/dmmdea/openclaw-mcp-servers/youtube-data/.venv/bin/python \
  /home/dmmdea/openclaw-mcp-servers/youtube-data/server.py

# YouTube Analytics + Reporting APIs (OAuth, owner data)
claude mcp add -s user youtube-analytics -- \
  /home/dmmdea/openclaw-mcp-servers/youtube-analytics/.venv/bin/python \
  /home/dmmdea/openclaw-mcp-servers/youtube-analytics/server.py

# Google Ads API + CSV-ingest fallback
claude mcp add -s user google-ads -- \
  /home/dmmdea/openclaw-mcp-servers/google-ads/.venv/bin/python \
  /home/dmmdea/openclaw-mcp-servers/google-ads/server.py

# yt-dlp based (transcripts, no API key)
claude mcp add -s user youtube -- \
  /home/dmmdea/openclaw-mcp-servers/youtube/.venv/bin/python \
  /home/dmmdea/openclaw-mcp-servers/youtube/server.py

# Optional — web search
claude mcp add -s user brave-search -- \
  npx -y @brave/brave-search-mcp-server
```

Confirm all 6 connected:
```
claude mcp list
```

## Step 8 — End-to-end smoke tests

All four checks should succeed before declaring setup complete:

1. `my_channel()` from `youtube-analytics` → returns the expected channel title/id. **If it returns a different channel**, re-run `oauth_auth.py` and pick the correct brand account.
2. `ads_list_accessible_customers()` from `google-ads` → should list at least the manager + sub-account. If it returns `test_access_limitation`, Basic access hasn't landed yet — CSV-ingest path is the fallback until the approval email arrives.
3. `youtube_channel_info("@YourHandle")` from `youtube-data` → returns channel details with 1 unit debited against the daily quota.
4. `library_scan.py` on the user's own channel:
   ```
   /home/dmmdea/openclaw-mcp-servers/youtube-data/.venv/bin/python \
     /home/dmmdea/openclaw-mcp-servers/youtube-data/scripts/library_scan.py @YourHandle
   ```
   Writes `library.xlsx` under `/home/dmmdea/openclaw-output/youtube-analyst/week-1/<channel>/`.

### Reference: Dell verification (2026-04-28)

End-to-end install verified live on Dell OptiPlex 7060 (`workernode`, 10.0.0.79). All 4 smoke tests passed against `@DanmarAutoReviews`:

- `my_channel()` returned the brand-account-owned channel (correct selection at OAuth picker).
- `ads_list_accessible_customers()` returned manager `143-213-9099` (DanmarMediaProductions) + sub-account `212-310-0176`.
- `youtube_channel_info("@DanmarAutoReviews")` returned channel details with 1 unit debited.
- `library_scan.py @DanmarAutoReviews` produced `library.xlsx` under `~/openclaw-output/youtube-analyst/week-1/<slug>/`.

Use this record as a baseline when running setup on a fresh node — same outputs, different channel handle.

## Troubleshooting cheat sheet

| Symptom | Cause | Fix |
|---|---|---|
| `Error 403: access_denied` at consent screen | Account not added as Test User | Cloud Console → OAuth consent screen → Test users → add |
| `The developer token is only approved for use with test accounts` | Basic access not yet approved | Wait for Google approval email. Use CSV-ingest path meanwhile |
| `Google Ads API has not been used in project X` | API not enabled | Cloud Console → APIs & Services → Library → enable Google Ads API (separate switch from YouTube APIs) |
| `The API Center is only available to manager accounts` | Tried to apply for developer token from a regular ad account | Create a Manager account wrapper first (Step 3) |
| OAuth picks wrong channel | Selected personal account instead of brand account | `oauth_auth.py --revoke` then re-run and select brand account at picker |
| Pyright "can't find module openclaw_shared" but runtime works fine | IDE venv config issue, not a real error | Each MCP dir has `pyrightconfig.json` with executionEnvironments; workspace-level `pyproject.toml` at `~/openclaw-mcp-servers/` can further stabilize it |
| `invalid_grant` after ~7 days | Testing-mode refresh-token expiry | Re-run `oauth_auth.py` |
| `library_scan.py` returns empty rows for a valid channel | Wrong handle / API key not loaded | Check `~/.openclaw/secrets.env` has `YOUTUBE_API_KEY`; restart the MCP session after edits |

## Reference paths in this repo

- Playbook spec: `/home/dmmdea/AI Ecosystem/OpenClaw/docs/superpowers/specs/2026-04-21-youtube-expert-analyst-playbook.md`
- Shared package: `/home/dmmdea/openclaw-mcp-servers/_shared/openclaw_shared/`
- Data dir tree: `/home/dmmdea/openclaw-output/youtube-analyst/{week-1,week-2,thumbnails,ads-exports,roi}`

## Related memory files

- `reference_openclaw_youtube_analytics_mcp.md` — Analytics MCP details + tools
- `reference_openclaw_google_ads_mcp.md` — Ads MCP details + Manager-wrapper gotcha
- `reference_openclaw_youtube_data_mcp.md` — Data API MCP
- `reference_openclaw_quota_mcp.md` — shared quota tracker
- `reference_openclaw_packaging_analysis.md` — feature extractors consumed by downstream analysis
- `reference_youtube_analyst_skills.md` — Drive + GitHub locations for the skill pair
- `feedback_skill_shipping_protocol.md` — Drive-first → GitHub → memory ship cycle (mandatory)
- `feedback_cross_reference_ads_before_citing_outliers.md` — analytical discipline for all outputs

## Sibling skills

- **`Youtube-Analyst-Runbook`** (this same repo, [`dmmdea/OpenClaw-Youtube-Analyst-Skills`](https://github.com/dmmdea/OpenClaw-Youtube-Analyst-Skills)) — day-to-day operations after bootstrap completes. Library scans, packaging analysis, matched-pair tests, weekly maintenance, findings interpretation.
- **`Hailo-Stack-Skill`** (sibling repo [`dmmdea/hailo-youtube-stack-mcp`](https://github.com/dmmdea/hailo-youtube-stack-mcp)) — optional NPU acceleration. Separate runbook for the Hailo-8L device, `hailo-vision` MCP, DKMS kernel patch, `openclaw_shared.cache.VisionCache` (1060× speedup on repeat scans), HEFs, OCR mode semantics. Bootstrap only if you have the hardware; the YouTube analyst pipeline auto-detects and uses it via `maybe_hailo_backend()`.
