# Trading Strategy — Fundamental Long/Swing

## Mission
Beat the S&P 500 annualized return (benchmark: SPY) through disciplined fundamental long/swing trading on US equities. Paper trading only until track record validates live deployment.

---

## Core Philosophy
- **Fundamental-first**: Enter positions backed by earnings momentum, revenue growth, sector tailwinds, or identifiable catalysts (earnings beats, product launches, macro shifts).
- **Swing horizon**: Hold positions 3–15 trading days. Not a day-trading strategy. Not buy-and-hold.
- **Quality over quantity**: Maximum 5 open positions at any time. Concentration in high-conviction ideas.
- **Asymmetric risk**: Target minimum 2:1 reward-to-risk ratio on every trade. Only trade setups where upside target ≥ 2× downside stop.

---

## Entry Criteria (ALL must be met)
1. Strong fundamental catalyst (earnings surprise, analyst upgrade, sector rotation, macro tailwind).
2. Price action confirms the thesis: stock holding above key moving averages (20-day EMA preferred) or breaking out of consolidation on above-average volume.
3. Sector/market environment is not in a confirmed downtrend (SPY above 200-day SMA).
4. Position does not breach portfolio concentration rules (see Risk Rules below).

---

## Risk Rules — NON-NEGOTIABLE

| Rule | Value | Notes |
|------|-------|-------|
| Max allocation per position | **5% of total portfolio value** | Calculated at time of order using actual Alpaca balance |
| Trailing stop (all positions) | **10%** | Set immediately after fill; never removed |
| Hard stop / cut losers | **-7% from entry price** | Exit immediately; no averaging down |
| Max concurrent open positions | **5** | No new entries if 5 positions already open |
| Paper trading only | **ENFORCED** | Base URL must be `https://paper-api.alpaca.markets` |
| No leverage / margin | **ENFORCED** | Only use `cash` buying power; never enable margin |
| No options, crypto, or futures | **ENFORCED** | US equities only |

---

## Position Sizing Formula
```
Buying Power       = (query Alpaca /v2/account → buying_power field)
Max Position Value = Buying Power × 0.05
Shares to Buy      = FLOOR(Max Position Value ÷ Current Ask Price)
```
The agent MUST write out this calculation explicitly before submitting any order.

---

## Exit Rules
1. **Trailing stop triggered**: Alpaca auto-exits. Log the fill.
2. **-7% hard stop**: Agent checks midday and market-close. If unrealized P&L% ≤ -7%, place market sell order immediately.
3. **Profit target reached**: If unrealized P&L% ≥ 20%, tighten trailing stop to 5% to protect gains.
4. **Thesis broken**: If the fundamental catalyst that drove entry is invalidated (e.g., earnings miss post-entry), exit regardless of P&L.
5. **15-day max hold**: Exit any position held longer than 15 trading days to prevent thesis drift.

---

## Benchmark & Grading
- Weekly grade compares closed-trade P&L vs. SPY return over same period.
- A: Outperformed SPY by >2% | B: Outperformed by 0–2% | C: Matched SPY | D: Underperformed | F: Loss while SPY gained

---

## Strategy Review Schedule
- Reviewed every Friday during `weekly_review` routine.
- Agent may propose updates; changes require human approval before taking effect.

---

*Last reviewed: (agent updates this field during weekly_review)*
*Version: 1.0*
