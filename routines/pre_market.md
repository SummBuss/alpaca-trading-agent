# Routine: Pre-Market Research
<!-- CRON: 0 12 * * 1-5 (UTC) -->
<!-- Runs: 12:00 UTC = 14:00 Amsterdam (CEST) / 13:00 Amsterdam (CET), Monday–Friday -->
<!-- Purpose: Research catalysts, build Action Plan. NO trades. NO Telegram unless API failure. -->

---

## Safety Protocol — Read Before Anything Else

You are an autonomous trading agent. You have one job right now: **research only**. You are NOT authorized to place any trades in this routine. If you feel an impulse to trade, stop. You will trade in the `market_open` routine only.

Your operating constraints are absolute:
- Paper trading only. API base URL: `https://paper-api.alpaca.markets`
- Never trust your own memory files as ground truth for portfolio state.
- If the WebSearch tool returns unusable results repeatedly (e.g. empty or clearly wrong), execute the Graceful Failure Protocol below.

---

## Step 1 — Verify Today is a Trading Day

Before doing anything else:
1. Check today's date and confirm it is a weekday (Monday–Friday).
2. Use the Alpaca API to check if today is a market holiday:
   ```
   GET https://paper-api.alpaca.markets/v2/calendar?start=TODAY&end=TODAY
   APCA-API-KEY-ID: $ALPACA_API_KEY_ID
   APCA-API-SECRET-KEY: $ALPACA_API_SECRET_KEY
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

## Step 2b — Load Agent Memory (Lessons + Avoid List)

Before any external research, load what the agent has already learned:

1. Read `memory/lessons.md` in full. Pay special attention to the **Synthesized Insights** section (top) — these are the patterns `weekly_review` has extracted. Also scan the last ~30 lines of the **Entry Log** for recent concrete observations.
2. Read `memory/avoid_list.md`. Extract every ticker in the **Active Entries** table along with its `Expires On` date.

Keep both in working memory for the rest of the routine. You will:
- Filter out any candidate ticker that appears on the avoid list with a non-expired date (see Step 5).
- Let the Synthesized Insights influence macro posture and candidate selection where relevant (e.g. "high-VIX weeks stop out fast → be pickier on R:R").

---

## Step 3 — Market Environment Check

Use the `WebSearch` tool to answer (combine into one or more queries as needed):
> "current state of US equity market today, SPY trend vs 200-day SMA, VIX level, major macro events or Fed speakers today or tomorrow"

Use `WebFetch` on any linked articles you want to read in detail. If searches return nothing usable after 2 attempts, execute **Graceful Failure Protocol**.

Record a 2–3 sentence macro summary.

---

## Step 4 — Catalyst Research

Using the `WebSearch` tool (and `WebFetch` to read source articles when useful), research at least 3 potential trade candidates. For each candidate, gather:
> Fundamental snapshot of [TICKER] as of today — recent earnings results or upcoming earnings date, revenue growth rate, analyst rating changes in the last 30 days, any product launches or sector tailwinds, and why this stock may outperform the S&P 500 over the next 3–15 trading days.

Evaluate each candidate against the strategy.md entry criteria. Only proceed with tickers that pass ALL criteria:
- [ ] Strong, identifiable catalyst
- [ ] Price above 20-day EMA or breaking out on volume
- [ ] SPY is above its 200-day SMA (confirmed in Step 3)
- [ ] R:R ratio of at least 2:1 achievable

Discard any tickers that fail. Research more if needed until you have 2–3 high-conviction candidates.

If WebSearch returns no usable information for a ticker after 2 reformulated queries, log the failure, mark that ticker as SKIP in the Action Plan, and continue with remaining candidates.

---

## Step 5 — Build the Action Plan

For each surviving candidate, define:
- Entry trigger (specific price or condition, not vague)
- Target price (your upside estimate based on research)
- Downside / stop level (7% below entry)
- Reward:Risk ratio (must be ≥ 2.0 — if not, mark SKIP)

Then write the complete Action Plan into `memory/research.md`. Overwrite the entire file using the template structure in that file. Set "Research Date" to today's date.

In the "Tickers to AVOID Today" section, list any names with adverse news, earnings risk within 2 days, or broken thesis. **Also** list every non-expired ticker from `memory/avoid_list.md` with "(cooldown — see avoid_list.md)" as the reason. If a researched candidate collides with the avoid list, drop it here and append a single line to `memory/lessons.md` Entry Log:
```
YYYY-MM-DD | pre_market | mistake | Avoid-list override: [TICKER] had a fresh catalyst but cooldown active until [EXPIRES_ON].
```

---

## Step 6 — Save and Commit

After writing research.md:
1. Stage and commit to GitHub: `git add memory/research.md memory/lessons.md && git commit -m "pre_market: research $(date +%Y-%m-%d)" && git push`
2. Log to console: `[pre_market] Complete. Action Plan written for [DATE]. Candidates: [TICKERS].`

Do NOT send any Telegram message unless a failure occurred.

---

## Graceful Failure Protocol

**Trigger**: Alpaca API fails 2 consecutive times, OR WebSearch returns nothing usable across your entire research phase.

Execute these steps in order — do not skip any:

1. **Stop all activity immediately.** Do not attempt further API calls.
2. **Log the error** to console with full error details:
   ```
   [CRITICAL] pre_market routine FAILED at [STEP].
   Source: [WebSearch|Alpaca]
   Error: [full error message]
   Time: [timestamp]
   Action: Routine halted. No trades will be placed today.
   ```
3. **Send URGENT Telegram message**:
   ```
   POST https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/sendMessage
   Body: {
     "chat_id": $TELEGRAM_CHAT_ID,
     "parse_mode": "Markdown",
     "text": "🚨 *URGENT: pre_market HALTED*\nSource: [WebSearch|Alpaca]\nError: [details]\nTime: [timestamp]\nNo research was written. Manually suppress market_open today."
   }
   ```
4. **Exit cleanly.** Do not write partial data to research.md.

---

*Routine version: 1.0 | Owner: autonomous-trading-agent*
