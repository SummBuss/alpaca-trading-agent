# Routine: Weekly Review & Strategy Assessment
<!-- CRON: 30 20 * * 5 (UTC) -->
<!-- Runs: 20:30 UTC = 22:30 Amsterdam (CEST) / 21:30 Amsterdam (CET), every Friday -->
<!-- Purpose: Grade week's performance, benchmark vs. SPY, suggest strategy updates, send Telegram report. -->

---

## Safety Protocol — Read Before Anything Else

This is a read-only analysis routine. You may NOT place orders. You may NOT modify strategy.md directly — you may only propose changes in your report. All strategy changes require human approval.

Your operating constraints are absolute:
- No trading activity in this routine.
- If any API fails twice, execute the Graceful Failure Protocol and send what partial analysis you have completed.

---

## Step 1 — Query Final Account State (Source of Truth)

```
GET https://paper-api.alpaca.markets/v2/account
APCA-API-KEY-ID: $ALPACA_API_KEY_ID
APCA-API-SECRET-KEY: $ALPACA_API_SECRET_KEY
```

Record: `portfolio_value`, `equity`

Also fetch account history for the week:
```
GET https://paper-api.alpaca.markets/v2/account/portfolio/history?period=1W&timeframe=1D
APCA-API-KEY-ID: $ALPACA_API_KEY_ID
APCA-API-SECRET-KEY: $ALPACA_API_SECRET_KEY
```

Record the equity values for each day this week to calculate week's P&L.

---

## Step 2 — Compile This Week's Trades

Read `memory/trade_log.md` in full. Extract:
- All closed trades in the "Closed Trades — Last 7 Days" table.
- All current open positions.

Also read the most recently archived trades in `memory/archive_log.md` (trades closed this week that may have already been archived by market_close routine).

Build a consolidated list of this week's closed trades.

---

## Step 3 — Benchmark Against SPY

Use the `WebSearch` tool to fetch SPY's week-to-date performance:
> "S&P 500 SPY ETF weekly return Monday open to Friday close week ending [DATE]"

Use `WebFetch` on a reputable source (Yahoo Finance, CNBC, MarketWatch) to confirm the exact percentage. If WebSearch returns no usable number after 2 attempts, note "SPY benchmark unavailable" and grade on absolute performance only.

Calculate:
```
PORTFOLIO_WEEK_RETURN   = (end_of_week_equity - start_of_week_equity) / start_of_week_equity × 100
SPY_WEEK_RETURN         = [from WebSearch]
ALPHA                   = PORTFOLIO_WEEK_RETURN - SPY_WEEK_RETURN
```

---

## Step 4 — Grade the Week

Apply the grading rubric from strategy.md:

| Grade | Condition |
|-------|-----------|
| A | Alpha ≥ +2% (outperformed SPY by more than 2%) |
| B | Alpha 0% to +2% (outperformed SPY) |
| C | Alpha near 0% (matched SPY, within ±0.5%) |
| D | Alpha negative (underperformed SPY) |
| F | Portfolio lost money while SPY gained |

Assign a grade. Write the reasoning.

---

## Step 5 — Analyze Best and Worst Trades

For each closed trade this week, calculate:
- Return % on that specific trade
- Whether the entry criteria from strategy.md were met
- Whether the exit was rule-driven (hard stop, trailing stop, target) or discretionary

Identify:
- **Best Trade**: Highest return %, explain what worked.
- **Worst Trade**: Lowest return % (or biggest loss), explain what went wrong and whether the rules were followed.

---

## Step 5b — Synthesize Lessons & Clean Avoid List

This is the **only** step in any routine authorized to autonomously rewrite parts of `memory/lessons.md` and `memory/avoid_list.md`. It is NOT authorized to edit `memory/strategy.md` — rule changes remain in Step 6 as proposals only.

**A. Synthesize the week's lessons.**
1. Read the last ~100 lines of `memory/lessons.md` Entry Log (this week plus a bit of prior context).
2. Identify up to 5 patterns that are each backed by at least 3 entry-log observations within the last 30 days. Examples: "Earnings-adjacent entries stopped out 4 of last 5", "High-VIX days produced no winning entries", "Tightening stop at +20% preserved gains 3/3 times".
3. Rewrite the **Synthesized Insights** section at the top of `memory/lessons.md` with these patterns. Replace any prior contents of that section; the Entry Log below remains append-only and untouched.
4. If fewer than 3 patterns are well-supported, write what you have and note "Insufficient data for additional patterns — will revisit next Friday."

**B. Prune old entry log rows.**
- Delete Entry Log rows older than 90 days (count by date column). Note the deletion count in the console log.

**C. Clean the avoid list.**
- In `memory/avoid_list.md`, remove every row whose `Expires On` date is today or earlier.
- If a ticker appeared in the avoid list more than twice in the last 60 days (count across lessons.md `mistake` entries), note it in Step 6 as a candidate for permanent exclusion — do NOT add it to strategy.md yourself.

---

## Step 6 — Strategy Improvement Proposals

Based on this week's analysis, identify up to 3 specific, actionable proposals to improve strategy.md. Each proposal must include:
- **What to change**: Specific rule or parameter (e.g., "Tighten hard stop from -7% to -5%")
- **Why**: Evidence from this week's trades
- **Expected impact**: How this would have changed outcomes

Format proposals clearly. These are PROPOSALS only. They are NOT implemented until the human approves and manually edits strategy.md.

---

## Step 7 — Compile and Send Telegram Weekly Report

Send a comprehensive weekly summary to Telegram. Telegram `sendMessage` has a 4096-character limit — if the full report exceeds that, split into multiple messages (Part 1/2, Part 2/2).

```
POST https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/sendMessage
Body: {
  "chat_id": $TELEGRAM_CHAT_ID,
  "parse_mode": "Markdown",
  "text": "[Use the full report template below]"
}
```

**Report template:**
```
# Weekly Trading Report — [DATE RANGE]

## Performance Summary
- Portfolio Value (EOW): $[value]
- Week P&L: $[amount] ([%])
- SPY Return this week: [%]
- Alpha: [%]
- GRADE: [A/B/C/D/F]

## Trades This Week
| Ticker | Entry | Exit | P&L | P&L% | Exit Reason |
|--------|-------|------|-----|------|-------------|
[rows for each closed trade]

## Win/Loss
- Trades closed: [N]
- Winners: [N] ([%])
- Losers: [N] ([%])
- Avg winner: [%]
- Avg loser: [%]

## Best Trade
[TICKER]: [+X%] — [brief reason it worked]

## Worst Trade
[TICKER]: [-X%] — [brief reason, and whether rules were followed]

## Open Positions Heading into Next Week
[TICKER | Entry | Current P&L% | Trailing Stop]

## Strategy Improvement Proposals
1. [Proposal 1]
2. [Proposal 2]
3. [Proposal 3]
⚠️ These are proposals only. Human approval required before implementation.

## Notes
[Any unusual market conditions, API issues, or observations this week]
```

---

## Step 8 — Save and Commit

```
git add memory/ && git commit -m "weekly_review: week ending $(date +%Y-%m-%d) | grade: [GRADE] | alpha: [%] | lessons synthesized" && git push
```

Log: `[weekly_review] Complete. Grade: [GRADE]. Alpha: [%]. Report sent to Telegram.`

---

## Graceful Failure Protocol

**Trigger**: Any API fails 2 consecutive times.

1. **Stop external calls.** Complete as much analysis as possible from local memory files.
2. **Log the error:**
   ```
   [CRITICAL] weekly_review routine FAILED at [STEP].
   Source: [WebSearch|Alpaca|Telegram]
   Error: [full error message]
   Time: [timestamp]
   ```
3. **Attempt Telegram message** (even if the full report couldn't be built — send what you have with a note that analysis is partial):
   ```
   POST https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/sendMessage
   Body: {
     "chat_id": $TELEGRAM_CHAT_ID,
     "parse_mode": "Markdown",
     "text": "⚠️ *weekly_review FAILED (partial report)*\nError: [details]\nPartial analysis completed: [describe what was done before failure]\nAction required: Manual review of weekly performance."
   }
   ```
4. **Exit cleanly.** Commit whatever analysis was written to memory.

---

*Routine version: 1.0 | Owner: autonomous-trading-agent*
