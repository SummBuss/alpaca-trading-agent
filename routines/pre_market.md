# Routine: Pre-Market Research
<!-- CRON: 0 14 * * 1-5 -->
<!-- Runs: 14:00 Amsterdam time (CET/CEST), Monday–Friday -->
<!-- Purpose: Research catalysts, build Action Plan. NO trades. NO ClickUp unless API failure. -->

---

## Safety Protocol — Read Before Anything Else

You are an autonomous trading agent. You have one job right now: **research only**. You are NOT authorized to place any trades in this routine. If you feel an impulse to trade, stop. You will trade in the `market_open` routine only.

Your operating constraints are absolute:
- Paper trading only. API base URL: `https://paper-api.alpaca.markets`
- Never trust your own memory files as ground truth for portfolio state.
- If the Perplexity API fails **twice in a row**, execute the Graceful Failure Protocol below immediately.

---

## Step 1 — Verify Today is a Trading Day

Before doing anything else:
1. Check today's date and confirm it is a weekday (Monday–Friday).
2. Use the Alpaca API to check if today is a market holiday:
   ```
   GET https://paper-api.alpaca.markets/v2/calendar?start=TODAY&end=TODAY
   Authorization: Bearer $ALPACA_API_KEY
   ```
3. If the market is closed today, log: `[pre_market] Market closed today (holiday). Routine exiting cleanly.` and stop. Do not write to research.md.

---

## Step 2 — Read Strategy

Open and read `memory/strategy.md` in full. Internalize:
- The entry criteria (ALL must be met)
- The position sizing formula
- The risk rules table
- The reward:risk minimum (2:1)

You will use this as the filter for every ticker you research today.

---

## Step 3 — Market Environment Check

Use the Perplexity API to answer:
> "What is the current state of the US equity market today? What is SPY's trend relative to its 200-day SMA? What is the VIX level? Are there any major macro events or Fed speakers today or tomorrow that could move markets?"

**Perplexity API Call:**
```
POST https://api.perplexity.ai/chat/completions
Authorization: Bearer $PERPLEXITY_API_KEY
Body: { "model": "sonar", "messages": [{ "role": "user", "content": "<your question>" }] }
```

If this call fails, retry once after 30 seconds. If it fails a second time, execute **Graceful Failure Protocol**.

Record a 2–3 sentence macro summary.

---

## Step 4 — Catalyst Research (Perplexity)

Using Perplexity, research at least 3 potential trade candidates. For each candidate, ask:
> "Give me a fundamental analysis of [TICKER] as of today. Include: recent earnings results or upcoming earnings date, revenue growth rate, analyst rating changes in the last 30 days, any product launches or sector tailwinds, and why this stock may outperform the S&P 500 over the next 3–15 trading days."

Evaluate each candidate against the strategy.md entry criteria. Only proceed with tickers that pass ALL criteria:
- [ ] Strong, identifiable catalyst
- [ ] Price above 20-day EMA or breaking out on volume
- [ ] SPY is above its 200-day SMA (confirmed in Step 3)
- [ ] R:R ratio of at least 2:1 achievable

Discard any tickers that fail. Research more if needed until you have 2–3 high-conviction candidates.

If Perplexity fails twice on any individual call, log the failure, mark that ticker as SKIP in the Action Plan, and continue with remaining candidates.

---

## Step 5 — Build the Action Plan

For each surviving candidate, define:
- Entry trigger (specific price or condition, not vague)
- Target price (your upside estimate based on research)
- Downside / stop level (7% below entry)
- Reward:Risk ratio (must be ≥ 2.0 — if not, mark SKIP)

Then write the complete Action Plan into `memory/research.md`. Overwrite the entire file using the template structure in that file. Set "Research Date" to today's date.

In the "Tickers to AVOID Today" section, list any names with adverse news, earnings risk within 2 days, or broken thesis.

---

## Step 6 — Save and Commit

After writing research.md:
1. Stage and commit to GitHub: `git add memory/research.md && git commit -m "pre_market: research $(date +%Y-%m-%d)" && git push`
2. Log to console: `[pre_market] Complete. Action Plan written for [DATE]. Candidates: [TICKERS].`

Do NOT send any ClickUp message unless a failure occurred.

---

## Graceful Failure Protocol

**Trigger**: Any external API (Perplexity, Alpaca) fails 2 consecutive times.

Execute these steps in order — do not skip any:

1. **Stop all activity immediately.** Do not attempt further API calls.
2. **Log the error** to console with full error details:
   ```
   [CRITICAL] pre_market routine FAILED at [STEP].
   API: [Perplexity|Alpaca]
   Error: [full error message]
   Time: [timestamp]
   Action: Routine halted. No trades will be placed today.
   ```
3. **Send URGENT ClickUp message**:
   ```
   POST https://api.clickup.com/api/v2/team/{team_id}/task
   Authorization: $CLICKUP_API_TOKEN
   Body: {
     "name": "URGENT: Trading Agent API Failure — pre_market halted",
     "description": "The pre_market routine failed due to [API] error at [timestamp]. Error: [details]. No research was written. Market open routine should be manually suppressed today.",
     "priority": 1
   }
   ```
4. **Exit cleanly.** Do not write partial data to research.md.

---

*Routine version: 1.0 | Owner: autonomous-trading-agent*
