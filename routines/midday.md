# Routine: Midday Position Management
<!-- CRON: 0 19 * * 1-5 -->
<!-- Runs: 19:00 Amsterdam time (CET/CEST), Monday–Friday -->
<!-- Purpose: Cut losers at -7%, tighten stops on winners, update trade_log.md. -->

---

## Safety Protocol — Read Before Anything Else

This routine manages risk for open positions. All position data comes from the Alpaca API — never from trade_log.md. The log is for your records; the API is reality.

Your operating constraints are absolute:
- Paper trading only. API base URL: `https://paper-api.alpaca.markets`
- The -7% hard stop is non-negotiable. If a position is at -7% or worse, you sell it. No exceptions.
- If the Alpaca API fails **twice in a row**, execute the Graceful Failure Protocol immediately.
- Do NOT open new positions in this routine. Risk management only.

---

## Step 1 — Verify Market is Open

```
GET https://paper-api.alpaca.markets/v2/clock
Authorization: Bearer $ALPACA_API_KEY
```

Confirm `is_open` is `true`. If not, log `[midday] Market closed at runtime. Exiting cleanly.` and stop.

---

## Step 2 — Query All Open Positions (Source of Truth)

```
GET https://paper-api.alpaca.markets/v2/positions
Authorization: Bearer $ALPACA_API_KEY
```

For each position, record:
- `symbol`
- `qty`
- `avg_entry_price`
- `current_price`
- `unrealized_pl`
- `unrealized_plpc` (this is the P&L percentage as a decimal, e.g., -0.072 = -7.2%)

If no positions are open, log `[midday] No open positions. Exiting cleanly.` and stop.

---

## Step 3 — Apply -7% Hard Stop Rule

For each position where `unrealized_plpc ≤ -0.07` (i.e., -7% or worse):

Write out the check explicitly:
```
[TICKER] unrealized_plpc = [value]
Is [value] ≤ -0.07? [YES/NO]
If YES → SELL IMMEDIATELY
```

**Place market sell order:**
```
POST https://paper-api.alpaca.markets/v2/orders
Authorization: Bearer $ALPACA_API_KEY
Body: {
  "symbol": "[TICKER]",
  "qty": [qty],
  "side": "sell",
  "type": "market",
  "time_in_force": "day"
}
```

After fill confirmation, also cancel the existing trailing stop order for this position:
```
GET https://paper-api.alpaca.markets/v2/orders?status=open&symbols=[TICKER]
```
Find the trailing_stop order and cancel it:
```
DELETE https://paper-api.alpaca.markets/v2/orders/{order_id}
```

Record the exit: fill price, realized P&L, exit reason = "Hard stop -7%".

---

## Step 4 — Tighten Stops on Winners

For each position where `unrealized_plpc ≥ 0.20` (i.e., +20% or better):

The strategy calls for tightening the trailing stop from 10% to 5% to protect gains.

Write out the check:
```
[TICKER] unrealized_plpc = [value]
Is [value] ≥ 0.20? [YES/NO]
If YES → Tighten trailing stop from 10% to 5%
```

**Cancel existing trailing stop and replace with tighter one:**

First, find the existing trailing stop order:
```
GET https://paper-api.alpaca.markets/v2/orders?status=open&symbols=[TICKER]
```

Cancel the old trailing stop:
```
DELETE https://paper-api.alpaca.markets/v2/orders/{old_trailing_stop_id}
```

Place new 5% trailing stop:
```
POST https://paper-api.alpaca.markets/v2/orders
Authorization: Bearer $ALPACA_API_KEY
Body: {
  "symbol": "[TICKER]",
  "qty": [qty],
  "side": "sell",
  "type": "trailing_stop",
  "trail_percent": "5",
  "time_in_force": "gtc"
}
```

---

## Step 5 — Check 15-Day Max Hold Rule

For each position, calculate days held:
```
DAYS_HELD = (today's date) - (entry date from trade_log.md)
```

If DAYS_HELD ≥ 15 for any position, flag it: log a note in trade_log.md under the position's "Notes" field: `MAX HOLD WARNING: 15 days reached — exit today at market_close routine`.

---

## Step 6 — Update trade_log.md

For positions that were sold: move them from the "Active Positions" table to the "Closed Trades — Last 7 Days" table. Fill in all fields.

For surviving positions: update "Current Price" and "Unrealized P&L" columns with live data from the API. Update the trailing stop % to 5% for any positions tightened in Step 4.

Update the "Running 7-Day P&L Summary" section.

Update the timestamp and account balance at bottom of file.

---

## Step 7 — Save and Commit

```
git add memory/trade_log.md && git commit -m "midday: position mgmt $(date +%Y-%m-%d)" && git push
```

Log: `[midday] Complete. Sold: [N] positions (-7% rule). Stops tightened: [N]. Remaining open: [N].`

Do NOT send ClickUp unless a Graceful Failure is triggered.

---

## Graceful Failure Protocol

**Trigger**: Alpaca API fails 2 consecutive times.

1. **Stop all activity.** If a sell order was submitted for a -7% loser but not confirmed, log the orphaned order ID.
2. **Log the error:**
   ```
   [CRITICAL] midday routine FAILED at [STEP].
   API: Alpaca
   Error: [full error message]
   Time: [timestamp]
   Positions requiring manual review: [list any that were flagged for -7% exit]
   Orphaned orders (if any): [order IDs]
   ```
3. **Send URGENT ClickUp message** (priority 1):
   ```
   URGENT: Trading Agent — midday routine FAILED
   Details: [error], [timestamp]
   CRITICAL: The following positions may be at -7% loss and require IMMEDIATE manual review in Alpaca: [list tickers or 'none checked yet']
   Orphaned orders: [IDs or 'none']
   ```
4. **Exit cleanly.**

---

*Routine version: 1.0 | Owner: autonomous-trading-agent*
