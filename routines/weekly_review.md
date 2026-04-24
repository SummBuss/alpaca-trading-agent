# Routine: Weekly Review & Strategy Assessment
<!-- CRON: 30 22 * * 5 -->
<!-- Runs: 22:30 Amsterdam time (CET/CEST), every Friday -->
<!-- Purpose: Grade week's performance, benchmark vs. SPY, suggest strategy updates, send ClickUp report. -->

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
Authorization: Bearer $ALPACA_API_KEY
```

Record: `portfolio_value`, `equity`

Also fetch account history for the week:
```
GET https://paper-api.alpaca.markets/v2/account/portfolio/history?period=1W&timeframe=1D
Authorization: Bearer $ALPACA_API_KEY
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

Use Perplexity to fetch SPY's week-to-date performance:
> "What was the S&P 500 (SPY ETF) percentage return this week (Monday open to Friday close)? Please give me the exact percentage."

```
POST https://api.perplexity.ai/chat/completions
Authorization: Bearer $PERPLEXITY_API_KEY
Body: { "model": "sonar", "messages": [{ "role": "user", "content": "<your question>" }] }
```

If Perplexity fails twice, note "SPY benchmark unavailable" and grade on absolute performance only.

Calculate:
```
PORTFOLIO_WEEK_RETURN   = (end_of_week_equity - start_of_week_equity) / start_of_week_equity × 100
SPY_WEEK_RETURN         = [from Perplexity]
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

## Step 6 — Strategy Improvement Proposals

Based on this week's analysis, identify up to 3 specific, actionable proposals to improve strategy.md. Each proposal must include:
- **What to change**: Specific rule or parameter (e.g., "Tighten hard stop from -7% to -5%")
- **Why**: Evidence from this week's trades
- **Expected impact**: How this would have changed outcomes

Format proposals clearly. These are PROPOSALS only. They are NOT implemented until the human approves and manually edits strategy.md.

---

## Step 7 — Compile and Send ClickUp Weekly Report

Send a comprehensive weekly summary to ClickUp:

```
POST https://api.clickup.com/api/v2/team/{team_id}/task
Authorization: $CLICKUP_API_TOKEN
Body: {
  "name": "Weekly Trading Report — Week of [DATE]",
  "priority": 2,
  "description": "[Use the full report template below]"
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
git add memory/ && git commit -m "weekly_review: week ending $(date +%Y-%m-%d) | grade: [GRADE] | alpha: [%]" && git push
```

Log: `[weekly_review] Complete. Grade: [GRADE]. Alpha: [%]. Report sent to ClickUp.`

---

## Graceful Failure Protocol

**Trigger**: Any API fails 2 consecutive times.

1. **Stop external calls.** Complete as much analysis as possible from local memory files.
2. **Log the error:**
   ```
   [CRITICAL] weekly_review routine FAILED at [STEP].
   API: [Perplexity|Alpaca|ClickUp]
   Error: [full error message]
   Time: [timestamp]
   ```
3. **Attempt ClickUp message** (even if the full report couldn't be built — send what you have with a note that analysis is partial):
   ```
   URGENT: Trading Agent — weekly_review FAILED (partial report)
   Error: [details]
   Partial analysis completed: [describe what was done before failure]
   Action required: Manual review of weekly performance.
   ```
4. **Exit cleanly.** Commit whatever analysis was written to memory.

---

*Routine version: 1.0 | Owner: autonomous-trading-agent*
