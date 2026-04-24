# Avoid List

> Tickers the agent should NOT take new positions in for a defined cooldown period.
> Maintained autonomously by `midday` and `market_close` (add entries on stop-outs / thesis breaks) and `weekly_review` (removes expired entries).
> Read by `pre_market` when building the Action Plan and by `market_open` as a final check before placing orders.

---

## Active Entries

| Ticker | Reason | Added On | Expires On | Source Routine |
|--------|--------|----------|------------|----------------|
| —      | —      | —        | —          | —              |

---

## Rules

- **Default cooldown**: 5 trading days from the event.
- **Extend** to 10 trading days if the same ticker hits a second stop within 30 days.
- **Never blacklist permanently** from this file; permanent bans belong in `strategy.md` (human approval required).
- **Expired entries** are removed by `weekly_review` each Friday.
- **Conflict handling**: if `pre_market` wants to enter a ticker on this list, the routine must SKIP it and log a `mistake`-tagged entry in `lessons.md` explaining why the avoid rule overrode the catalyst.

---

*This file is maintained by the agent. Humans may add entries manually, but entries without an `Expires On` date will be ignored by `weekly_review` cleanup.*
