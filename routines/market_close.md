# Routine: Market Close — End of Day Review & Archiving
<!-- CRON: 15 20 * * 1-5 (UTC) -->
<!-- Runs: 20:15 UTC = 22:15 Amsterdam (CEST) / 21:15 Amsterdam (CET), Monday–Friday -->
<!-- Purpose: Review day, archive old trades, write EOD summary, commit all memory. -->

---

## Safety Protocol — Read Before Anything Else

The market is now closed. This is a read-and-archive routine. You may NOT place any new orders. Any trailing stops Alpaca auto-executed after market close are already reflected in the account — you will verify them via API.

Your operating constraints are absolute:
- Paper trading only. All reads from `https://paper-api.alpaca.markets`
- Do NOT submit any new orders in this routine.
- If the Alpaca API fails **twice in a row**, execute the Graceful Failure Protocol.

---

## Step 1 — Query Final Day State (Source of Truth)

Pull the day's final state from Alpaca:

**Account:**
```
GET https://paper-api.alpaca.markets/v2/account
APCA-API-KEY-ID: $ALPACA_API_KEY_ID
APCA-API-SECRET-KEY: $ALPACA_API_SECRET_KEY
```
Record: `portfolio_value`, `equity`, `last_equity`, `buying_power`

**Open positions (what survived the day):**
```
GET https://paper-api.alpaca.markets/v2/positions
APCA-API-KEY-ID: $ALPACA_API_KEY_ID
APCA-API-SECRET-KEY: $ALPACA_API_SECRET_KEY
```

**Today's filled orders (what was executed today):**
```
GET https://paper-api.alpaca.markets/v2/orders?status=closed&after=[TODAY_ISO]&limit=50
APCA-API-KEY-ID: $ALPACA_API_KEY_ID
APCA-API-SECRET-KEY: $ALPACA_API_SECRET_KEY
```

Record all filled orders: symbol, side, filled_qty, filled_avg_price, filled_at.

---

## Step 2 — Reconcile trade_log.md Against API

Compare the "Active Positions" table in `memory/trade_log.md` against the live positions from Step 1.

For any position that was in trade_log.md but is NO longer in the Alpaca response (it was closed — either by trailing stop or hard stop):
- Find the corresponding closed order from Step 1's order list.
- Calculate realized P&L: `(fill_price - entry_price) × shares`
- Move this entry from "Active Positions" to "Closed Trades — Last 7 Days" with all fields filled in.

For surviving positions: update current price and unrealized P&L to end-of-day values.

---

## Step 3 — Archive Trades Older Than 7 Days

Scan the "Closed Trades — Last 7 Days" table in `memory/trade_log.md`.

For each row where `Exit Date < (today - 7 days)`:

1. Append the row to `memory/archive_log.md` under the "Archived Trades" table.
2. Remove the row from `memory/trade_log.md`.

After archiving, update the "All-Time Performance Summary" in `archive_log.md`:
- Increment Total Trades count.
- Add realized P&L to Total Realized P&L.
- Recalculate Overall Win Rate (wins / total trades).
- Update Best Trade and Worst Trade if applicable.

---

## Step 4 — Write End of Day Summary

Append an EOD summary block to the bottom of `memory/trade_log.md` (overwrite the previous day's summary if it exists):

```
## End of Day Summary — [DATE]

**Portfolio Value**: $[equity from Alpaca]
**Day's P&L**: $[equity - last_equity] ([%])
**Open Positions**: [N]
**Trades Closed Today**: [list: TICKER exit_reason P&L]
**Trades Archived Today**: [N] trades moved to archive_log.md

**Notable Events**:
- [Any -7% stops hit, trailing stops triggered, or 15-day exits]
- [Any positions added, if market_open ran]

**Tomorrow's Prep Note**:
- [Flag any positions approaching -7% or 15-day limit for pre_market to be aware of]
```

---

## Step 4b — End-of-Day Reflection (Append Lessons)

Based on today's activity (all positions opened, closed, or currently held), append 1–3 focused observations to `memory/lessons.md` Entry Log. Keep each ≤160 chars. Use these tags:
- `execution` — slippage, partial fills, gap opens, unusual liquidity
- `macro` — if SPY/VIX/sector behavior was notable
- `thesis` — for closed trades: did the catalyst play out as expected?
- `win` / `mistake` — only if something decisively worked or decisively didn't

Example lines:
```
2026-04-24 | market_close | thesis | NVDA +3% on analyst upgrade as planned; volume 1.5x avg confirmed conviction.
2026-04-24 | market_close | mistake | AMD entered at 163.50 but gapped to 165.80 — entry criteria needed a wider slippage tolerance.
2026-04-24 | market_close | macro | VIX closed 22.4, above 20 threshold; consistent with chop in SPY.
```

If nothing notable happened (routine day, no trades closed, no unusual behavior), write ONE entry tagged `macro` summarizing the market day in one line. Never skip this step entirely — the entry log should have at least one line per trading day.

---

## Step 5 — Save and Commit All Memory

Commit all memory file changes in one atomic commit:

```
git add memory/trade_log.md memory/archive_log.md memory/lessons.md && git commit -m "market_close: EOD $(date +%Y-%m-%d) | portfolio: $[VALUE]" && git push
```

Log: `[market_close] Complete. Portfolio: $[VALUE]. Open positions: [N]. Archived: [N] trades.`

Do NOT send Telegram unless a Graceful Failure is triggered. (Weekly Telegram summary happens in weekly_review.)

---

## Graceful Failure Protocol

**Trigger**: Alpaca API fails 2 consecutive times.

1. **Stop all activity.**
2. **Log the error:**
   ```
   [CRITICAL] market_close routine FAILED at [STEP].
   API: Alpaca
   Error: [full error message]
   Time: [timestamp]
   Impact: trade_log.md and archive_log.md may be out of sync with actual account state.
   ```
3. **Send URGENT Telegram message**:
   ```
   POST https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/sendMessage
   Body: {
     "chat_id": $TELEGRAM_CHAT_ID,
     "parse_mode": "Markdown",
     "text": "🚨 *URGENT: market_close FAILED*\nError: [details]\nTime: [timestamp]\nImpact: EOD reconciliation did not complete. trade_log.md may be stale.\nAction: manually verify positions in Alpaca and check archive_log.md."
   }
   ```
4. **Exit cleanly.** Do not write partial updates to memory files.

---

*Routine version: 1.0 | Owner: autonomous-trading-agent*
