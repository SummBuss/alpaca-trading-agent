# Pre-Market Research

> Written by the `pre_market` routine. Read by the `market_open` routine.
> This file is overwritten fresh each trading day. It represents the current day's Action Plan only.
> Do NOT execute trades based on stale research (check the "Research Date" field before acting).

---

## Research Date
*(agent writes today's date here in format YYYY-MM-DD)*

## Market Environment
*(agent writes a 2–3 sentence macro summary: SPY trend, VIX level, key levels to watch)*

---

## Watchlist & Catalyst Research

### Ticker 1: [SYMBOL]
- **Catalyst**: *(e.g., earnings beat, analyst upgrade, sector rotation)*
- **Fundamental Case**: *(revenue growth, margins, competitive moat — 3–5 sentences)*
- **Entry Trigger**: *(specific price action signal or level to confirm before buying)*
- **Target Price**: $0.00
- **Downside Risk**: $0.00
- **Reward:Risk Ratio**: 0:1
- **Perplexity Source Summary**: *(key quotes or data points from research)*

### Ticker 2: [SYMBOL]
- **Catalyst**:
- **Fundamental Case**:
- **Entry Trigger**:
- **Target Price**: $0.00
- **Downside Risk**: $0.00
- **Reward:Risk Ratio**: 0:1
- **Perplexity Source Summary**:

### Ticker 3: [SYMBOL]
*(add if applicable)*

---

## Action Plan

> This section is the AUTHORITATIVE instruction set for the `market_open` routine.
> Each row is either EXECUTE or SKIP — the market_open routine does not deviate from this plan.

| Priority | Ticker | Action | Planned Allocation | Entry Condition | Target | Stop | Rationale |
|----------|--------|--------|--------------------|-----------------|--------|------|-----------|
| 1 | — | BUY/SKIP | 5% portfolio | — | — | — | — |
| 2 | — | BUY/SKIP | 5% portfolio | — | — | — | — |
| 3 | — | BUY/SKIP | 5% portfolio | — | — | — | — |

---

## Tickers to AVOID Today
*(Any names with adverse news, earnings risk, or broken thesis — market_open must skip these)*
- —

---

## Pre-Market Risk Flags
*(Any macro events, Fed speakers, or geopolitical factors that could invalidate the plan)*
- —

---

*Written by: pre_market routine*
*To be read by: market_open routine*
