# Routine: Market Open — Execute Trades
<!-- CRON: 45 15 * * 1-5 -->
<!-- Runs: 15:45 Amsterdam time (CET/CEST), Monday–Friday -->
<!-- Purpose: Query live portfolio, execute Action Plan from research.md, set stops, log trades. -->

---

## Safety Protocol — Read Before Anything Else

You are executing real (paper) trades with real simulated capital. Every number you use MUST come from the Alpaca API — never from memory, never from a cached file. One math error = one wrong position size. You will write out every calculation explicitly before submitting any order.

Your operating constraints are absolute:
- Paper trading only. API base URL: `https://paper-api.alpaca.markets`
- Max 5% of portfolio per position. Calculate this fresh from the Alpaca API right now.
- If the Alpaca API fails **twice in a row**, execute the Graceful Failure Protocol immediately.
- If research.md is dated before today, STOP. The pre_market routine did not run successfully. Do not trade.

---

## Step 1 — Verify Market is Open

```
GET https://paper-api.alpaca.markets/v2/clock
Authorization: Bearer $ALPACA_API_KEY
```

Check that `is_open` is `true`. If the market is not open, log `[market_open] Market is not open at runtime. Exiting cleanly.` and stop.

---

## Step 2 — Query Live Portfolio State (Source of Truth)

Make the following Alpaca API calls. These numbers override EVERYTHING in memory files.

**Account balances:**
```
GET https://paper-api.alpaca.markets/v2/account
Authorization: Bearer $ALPACA_API_KEY
```
Record these values from the response:
- `portfolio_value` → PORTFOLIO_VALUE
- `buying_power` → BUYING_POWER
- `cash` → CASH

**Open positions:**
```
GET https://paper-api.alpaca.markets/v2/positions
Authorization: Bearer $ALPACA_API_KEY
```
Record: ticker, qty, avg_entry_price, current_price, unrealized_pl, unrealized_plpc for each position.

Count open positions → OPEN_POSITION_COUNT

If OPEN_POSITION_COUNT ≥ 5, log `[market_open] Max positions (5) already held. No new entries today.` and skip to Step 6.

---

## Step 3 — Read and Validate the Action Plan

Open `memory/research.md`. Check:
- [ ] "Research Date" equals today's date (YYYY-MM-DD). If NOT, log a warning and STOP — do not trade on stale research.
- [ ] The Action Plan table is populated and at least one row is marked BUY (not SKIP).

If the file is stale or the Action Plan is empty, log the issue and exit cleanly. Do not send ClickUp message for this (it is not an API failure).

---

## Step 4 — Verify Math Before Every Order

For EACH ticker marked BUY in the Action Plan, do the following calculation and write it out explicitly in your thought process BEFORE submitting the order:

```
PORTFOLIO_VALUE      = $[value from Alpaca API in Step 2]
MAX_POSITION_VALUE   = PORTFOLIO_VALUE × 0.05 = $[result]
CURRENT_ASK_PRICE    = $[fetched from Alpaca quote or market data]
SHARES_TO_BUY        = FLOOR(MAX_POSITION_VALUE ÷ CURRENT_ASK_PRICE) = [integer]
TOTAL_COST           = SHARES_TO_BUY × CURRENT_ASK_PRICE = $[result]
TOTAL_COST ≤ BUYING_POWER?  [YES / NO — if NO, reduce shares until YES]
REWARD:RISK RATIO    = (TARGET - ASK) ÷ (ASK - STOP) = [ratio] — must be ≥ 2.0
```

If R:R is < 2.0, mark that ticker SKIP and do not place the order.

Also confirm the entry condition from research.md is currently met (e.g., price above trigger level). If not met, mark SKIP.

---

## Step 5 — Execute Orders

For each ticker that passed Step 4:

**Place market buy order:**
```
POST https://paper-api.alpaca.markets/v2/orders
Authorization: Bearer $ALPACA_API_KEY
Body: {
  "symbol": "[TICKER]",
  "qty": [SHARES_TO_BUY],
  "side": "buy",
  "type": "market",
  "time_in_force": "day"
}
```

Wait for fill confirmation (poll `GET /v2/orders/{order_id}` until `status == "filled"`).

Record: `filled_avg_price` → FILL_PRICE

**Immediately set 10% trailing stop:**
```
POST https://paper-api.alpaca.markets/v2/orders
Authorization: Bearer $ALPACA_API_KEY
Body: {
  "symbol": "[TICKER]",
  "qty": [SHARES_TO_BUY],
  "side": "sell",
  "type": "trailing_stop",
  "trail_percent": "10",
  "time_in_force": "gtc"
}
```

Confirm the trailing stop order is accepted. Record the trailing stop order ID.

If any order fails, log the error and do NOT retry the same order. Move to the next ticker.

---

## Step 6 — Update trade_log.md

For every new position opened, append a row to the **Active Positions** table in `memory/trade_log.md`:

| Field | Value |
|-------|-------|
| Ticker | [SYMBOL] |
| Entry Date | Today's date |
| Entry Price | FILL_PRICE |
| Shares | SHARES_TO_BUY |
| Cost Basis | FILL_PRICE × SHARES_TO_BUY |
| Current Price | FILL_PRICE (will be updated at midday/close) |
| Trailing Stop % | 10% |
| Stop Price | FILL_PRICE × 0.90 |
| Notes | Entry catalyst from research.md |

Also update "Alpaca Account Balance at last sync" and timestamp.

---

## Step 7 — ClickUp Notification (ONLY if a trade was placed)

If at least one order was filled, send a ClickUp message:
```
POST https://api.clickup.com/api/v2/team/{team_id}/task
Authorization: $CLICKUP_API_TOKEN
Body: {
  "name": "Trade Executed — [DATE]",
  "description": "Positions opened today:\n[list each: TICKER, shares @ fill price, trailing stop set at 10%]\n\nPortfolio value: $[PORTFOLIO_VALUE]\nBuying power remaining: $[BUYING_POWER after orders]",
  "priority": 3
}
```

If NO trades were placed (all tickers skipped), do NOT send ClickUp.

---

## Step 8 — Save and Commit

```
git add memory/trade_log.md && git commit -m "market_open: trades $(date +%Y-%m-%d)" && git push
```

Log: `[market_open] Complete. [N] positions opened. [N] skipped. Portfolio: $[VALUE].`

---

## Graceful Failure Protocol

**Trigger**: Alpaca API fails 2 consecutive times on any call.

1. **Stop all activity immediately.** If a buy order was submitted but not confirmed, do NOT submit the trailing stop. Log the orphaned order ID.
2. **Log the error:**
   ```
   [CRITICAL] market_open routine FAILED at [STEP].
   API: Alpaca
   Error: [full error message]
   Time: [timestamp]
   Orphaned orders (if any): [order IDs]
   Action: Routine halted. Manual review required.
   ```
3. **Send URGENT ClickUp message** (priority 1):
   ```
   URGENT: Trading Agent — market_open FAILED
   Details: [error], [timestamp]
   Orphaned orders needing manual review: [IDs or 'none']
   Immediate action: Log into Alpaca paper account and verify all open orders/positions.
   ```
4. **Exit cleanly.** Do not partially update trade_log.md.

---

*Routine version: 1.0 | Owner: autonomous-trading-agent*
