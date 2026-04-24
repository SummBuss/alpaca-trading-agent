# Alpaca Trading Agent — Autonomous 24/7 Stock Trading

A stateless, GitHub-backed autonomous trading agent running on Claude Code Remote Routines. It performs pre-market research via Perplexity, executes paper trades via Alpaca, manages risk intraday, and reports via ClickUp — fully autonomously, on a schedule, while you sleep.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                   Claude Desktop App                        │
│              (Remote Routines Engine)                       │
└─────────────────────┬───────────────────────────────────────┘
                      │ reads prompts from
                      ▼
┌─────────────────────────────────────────────────────────────┐
│              GitHub Repository (this repo)                  │
│  /memory/        — Agent's persistent brain                 │
│  /routines/      — Scheduled prompt files                   │
└──────┬──────────────┬──────────────┬────────────────────────┘
       │              │              │
       ▼              ▼              ▼
  Alpaca API    Perplexity API   ClickUp API
 (paper trades)  (research)    (notifications)
```

**Routine Schedule (Amsterdam Time / CET):**

| Routine | CRON | Time | Action |
|---------|------|------|--------|
| pre_market | `0 14 * * 1-5` | 14:00 M-F | Research & Action Plan |
| market_open | `45 15 * * 1-5` | 15:45 M-F | Execute trades |
| midday | `0 19 * * 1-5` | 19:00 M-F | Cut losers, tighten stops |
| market_close | `15 22 * * 1-5` | 22:15 M-F | EOD review & archive |
| weekly_review | `30 22 * * 5` | 22:30 Fri | Grade week, ClickUp report |

---

## Prerequisites

Before following the deployment steps, confirm you have:
- [ ] Claude Desktop App installed (not just the web version).
- [ ] A GitHub account with a personal access token that has `repo` scope.
- [ ] An Alpaca account with paper trading enabled (get keys at alpaca.markets).
- [ ] A Perplexity API key (get at perplexity.ai/api).
- [ ] A ClickUp API token and your Team ID (found in ClickUp Settings → Apps).

---

## Deployment Steps

### Step 1 — Initialize GitHub Repository

Open Claude Code (VS Code or terminal) in this project folder and give it this exact prompt:

```
Initialize this folder as a git repository, create a .gitignore that excludes .env files and any OS files, and push the first commit to a new GitHub repository named 'alpaca-trading-agent'. The commit message should be "init: scaffold trading agent architecture".
```

Claude Code will walk you through the GitHub push. Once complete, verify all files are visible on GitHub before proceeding.

> **Why GitHub?** The agent is stateless — it has no local persistent memory between runs. GitHub is the persistent brain. Every routine commits its memory file updates back to the repo so the next routine picks up where the last one left off.

---

### Step 2 — Create the Cloud Environment in Claude Desktop

1. Open the **Claude Desktop App** (standalone, not browser).
2. In the left sidebar, click **Routines** (or navigate to the Routines section).
3. Click **New Routine** → **Remote**.
4. Click **Add Cloud Environment**.
5. Name the environment exactly: `Trading`
6. Leave it open — you will add secrets to it in Step 3.

---

### Step 3 — Add API Keys to the Cloud Environment

> **CRITICAL SECURITY WARNING**
>
> Your API keys grant real financial access. Follow these rules without exception:
>
> - **NEVER** put API keys in a `.env` file in this repo.
> - **NEVER** commit API keys to GitHub — not even accidentally.
> - **NEVER** paste API keys into a routine prompt file.
> - **ALWAYS** store them exclusively inside the Claude Desktop Cloud Environment (Steps below). They are encrypted at rest and injected as environment variables at runtime, never exposed in plaintext.

Inside the `Trading` cloud environment you created in Step 2, add these exact environment variables:

| Variable Name | Where to Get It |
|---------------|-----------------|
| `ALPACA_API_KEY` | Alpaca dashboard → Paper Trading → API Keys |
| `ALPACA_SECRET_KEY` | Alpaca dashboard → Paper Trading → API Keys (shown once at creation) |
| `PERPLEXITY_API_KEY` | perplexity.ai/api → API Keys |
| `CLICKUP_API_TOKEN` | ClickUp → Settings → Apps → API Token |
| `CLICKUP_TEAM_ID` | ClickUp → your workspace URL (the number after `/app/`) |

Double-check all 5 variables are saved before proceeding.

---

### Step 4 — Auto-Schedule All Routines

Inside the Claude Desktop App:

1. Start a **new Claude Code session** (not a chat session — a Code session).
2. Point it at your **local project folder** (the one you pushed in Step 1) OR at the GitHub repo directly — the Remote Routine system will pull from GitHub.
3. Paste this exact prompt:

```
Look inside the routines folder inside of this project and help me set up all of these different schedule routine runs based on the CRON jobs and prompts provided. We will be working out of the GitHub repo and using the 'Trading' cloud environment.
```

Claude will read the CRON expressions from the comment headers in each routine file and create 5 scheduled Remote Routines connected to the `Trading` cloud environment. Confirm each one is created with the correct CRON and the correct source file.

---

### Step 5 — Enable Branch Push Permissions

> **This step is mandatory.** Without it, the agent cannot save its memory files back to GitHub and will lose all state between runs.

1. In the Claude Desktop App, navigate to your routine settings (Routines → [select a routine] → Settings or Permissions).
2. Find the toggle labeled **"Allow unrestricted branch pushes"** (or similar wording for Git write access).
3. **Toggle it ON** for all 5 routines.

Repeat for each of the 5 routines: `pre_market`, `market_open`, `midday`, `market_close`, `weekly_review`.

> **Why this matters**: Each routine ends with a `git push` to commit its memory updates. If branch push is disabled, the commit will fail silently, and the next routine will read stale data — leading to duplicate trades or missed stops.

---

## Verifying the Setup

After completing all 5 steps, do a manual sanity check:

1. **Alpaca connection**: Manually call `GET https://paper-api.alpaca.markets/v2/account` with your keys. You should see your paper account balance (default ~$100,000).
2. **Routine list**: In Claude Desktop, verify 5 routines appear with the correct schedules.
3. **First run**: If today is a weekday, wait for the 14:00 routine to fire and check that `memory/research.md` is updated on GitHub.
4. **ClickUp**: After the first trade executes (market_open routine), a task should appear in ClickUp.

---

## File Structure

```
alpaca-trading-agent/
├── README.md                    ← This file
├── memory/
│   ├── strategy.md              ← Trading rules & risk parameters
│   ├── trade_log.md             ← Active positions + 7-day closed trades
│   ├── archive_log.md           ← All-time historical trade record
│   └── research.md              ← Daily pre-market Action Plan
└── routines/
    ├── pre_market.md            ← CRON: 0 14 * * 1-5
    ├── market_open.md           ← CRON: 45 15 * * 1-5
    ├── midday.md                ← CRON: 0 19 * * 1-5
    ├── market_close.md          ← CRON: 15 22 * * 1-5
    └── weekly_review.md         ← CRON: 30 22 * * 5
```

---

## Safety Architecture Summary

| Protocol | Implementation |
|----------|----------------|
| Source of Truth | Every routine queries Alpaca API first; memory files are secondary |
| Verify-Then-Execute | Agent writes out position sizing math explicitly before every order |
| Max position size | 5% of portfolio, calculated fresh from live balance |
| Hard stop | -7% triggers immediate market sell |
| Trailing stop | 10% on all positions, tightens to 5% at +20% gain |
| Context rot prevention | trade_log.md capped at 7 days; older trades auto-archived |
| Graceful failure | 2 API failures → stop, log, ClickUp URGENT alert, clean exit |
| Paper trading lock | Base URL hardcoded to paper API endpoint in all routines |

---

## Modifying the Strategy

To update trading rules (e.g., change the hard stop from -7% to -5%):
1. The weekly_review routine will propose changes in its ClickUp report.
2. Review the proposal manually.
3. Edit `memory/strategy.md` directly in VS Code.
4. Commit and push the change.
5. The agent will read the updated strategy at its next run.

The agent is explicitly prohibited from self-modifying strategy.md.

---

## Troubleshooting

| Issue | Likely Cause | Fix |
|-------|-------------|-----|
| research.md not updated after 14:00 | Routine didn't fire or Perplexity failed | Check routine logs in Claude Desktop; check ClickUp for URGENT alerts |
| No trades placed at 15:45 | research.md is stale (wrong date) or all tickers were SKIP | Check research.md date and Action Plan |
| trade_log.md not updating on GitHub | Branch push permission not enabled | Re-check Step 5 |
| Agent placed a trade at wrong size | Check math log in routine output | Review position sizing calc; verify PORTFOLIO_VALUE pulled from API |
| ClickUp not receiving messages | CLICKUP_API_TOKEN or CLICKUP_TEAM_ID wrong | Re-check environment variables in Step 3 |

---

*Generated by Claude Code | Strategy version 1.0 | Paper trading only*
