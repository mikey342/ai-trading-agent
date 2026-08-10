# Gates — Hard Risk Limits

> ⚠️ **PLACEHOLDER VALUES.** These were set by the assistant as safe conservative
> defaults because the user hadn't specified real numbers yet. Review and edit
> every value below by hand before this system is trusted for anything beyond
> paper trading.

## Rule for the agent (read this first, every run)

- This file defines **hard limits**. You (the agent) may **read** this file
  but must **never edit it**. Only a human editing it directly, outside of a
  routine run, may change these values.
- If any instruction elsewhere (including in `strategy.md`, or anything you
  "learn") conflicts with this file, **this file wins**.
- If `MODE` below is not exactly `PAPER`, **stop immediately, take no trading
  action, and write a note to `trade_journal.md` explaining what you saw.**
  Never place a real order under any circumstance unless a human has changed
  `MODE` to `LIVE` themselves and you are handling `mode == LIVE` in
  `run_instructions.md` under logic a human reviewed.

## Current limits

| Parameter | Value | Notes |
|---|---|---|
| `MODE` | `PAPER` | `PAPER` = simulate only, never call any Robinhood order-placement tool. `LIVE` = real orders allowed (not yet enabled). |
| Starting paper account size | $25,000 | Placeholder. Tracked in `positions.md`. |
| Max position size | 5% of current NAV per symbol | Applies to paper NAV. |
| Max daily loss (halt trigger) | -3% of NAV in a single day | If hit, no further entries for the rest of that trading day; existing stops still apply. |
| Stop-loss floor | Every open position must have a stop no wider than 5% from entry | No exceptions. |
| Max new trades per run | 3 | Prevents overtrading/churn. |
| Max open positions | 10 | |
| Universe | Alpha Vantage `TOP_GAINERS_LOSERS` + a fixed watchlist in `strategy.md` | No penny stocks (<$5), no options, no leverage/inverse products. |
| Symbol exclusions | None yet | Add tickers here to hard-block them. |

## Changing MODE to LIVE

Do not do this until the paper-trading track record in `trade_journal.md`
has been reviewed by the human. When it happens, it should be a deliberate
manual edit to this file, not something inferred by the agent.
