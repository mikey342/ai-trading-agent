# Data Schema — trade_log.csv

Machine-readable companion to `trade_journal.md`. The journal holds
narrative reasoning (good for a human reading back through decisions);
this CSV holds the same decisions as queryable rows (good for computing
win rate, expectancy, R-multiple distribution, or checking whether a
specific rule is actually earning its place).

**Append-only.** One row per action. Never rewrite past rows.

## When to write a row

- **Every position opened** — `action=open`, with the full indicator
  snapshot that justified it (see below).
- **Every position closed** — `action=close`, with `exit_rule`,
  `realized_pnl`, and `r_multiple`.
- **Every candidate rejected at Tier 3 or at the gates** —
  `action=reject`, with as much of the snapshot as was gathered and the
  reason in `notes`. Rejections matter: a strategy that would have made
  money on the trades it *declined* is a strategy with a miscalibrated
  filter, and that's invisible if only fills are recorded.

## Columns

| Column | Applies to | Meaning |
|---|---|---|
| `trade_id` | all | Stable id; reuse the same id on the `close` row that matches an `open`. Format: `SYMBOL-YYYYMMDD-N`. |
| `timestamp_utc` | all | ISO-8601 UTC. |
| `run_type` | all | `morning` / `midday` / `close` / `monitor`. |
| `action` | all | `open` / `close` / `reject`. |
| `symbol` | all | Underlying ticker. |
| `sleeve` | all | `stock` (individual name from the Tier 0-3 funnel) or `index` (2x ETF market-direction bet). |
| `instrument` | all | `equity`, `leveraged_etf`, or `option`. |
| `mode` | all | `LONG` or `SHORT` — the direction mode the run was in, set by SPY vs its SMA(200). |
| `qty` | open/close | Shares, or contracts for options. |
| `effective_exposure` | open/close | Notional **after** applying the 2x multiplier for leveraged ETFs; equals `notional` otherwise. This is what gross-exposure limits are checked against. |
| `adx14` | all | Trend strength from the Tier 0 scan. |
| `rel_options_volume` | all | Relative options volume from the Tier 0 scan — unusual-positioning signal, used as a ranking tiebreaker, never as a directional signal on its own. |
| `fill_price` | open/close | Simulated fill. Options use `high_fill_rate_buy_price`/`sell_price`, not mark (see `strategy.md`). |
| `notional` | open/close | `qty × fill_price` (× 100 for options). |
| `stop`, `r_target` | open | Fixed at entry, never recomputed later. |
| `atr14`, `rsi14`, `ema8`, `ema21`, `sma50`, `sma200` | all | Indicator snapshot **at decision time**. This is the core of "why did we take this trade" in machine-readable form. |
| `pct_52w_range` | all | `(price - low52) / (high52 - low52)` — the relative-strength proxy. |
| `pe_ratio` | all | From `get_equity_fundamentals`. |
| `composite_rank` | all | The value+momentum tiebreaker score from `strategy.md`. |
| `trigger` | open/reject | `breakout` or `pullback` — which entry rule fired. |
| `news_sentiment` | all | Score, or the literal `UNAVAILABLE` when the Alpha Vantage quota was exhausted (see `strategy.md`'s degradation rule). Never fabricate this. |
| `earnings_days_away` | all | Days to next earnings report; blackout is <3. |
| `regime` | all | `risk-on` / `risk-off` — SPY vs its 200-day SMA. |
| `strike`, `expiration`, `delta_at_entry`, `iv_at_entry`, `dte_at_entry`, `underlying_price` | options only | Blank for equity rows. |
| `exit_rule` | close | Which rule fired: `stop`, `trailing_stop`, `trend_break`, `time_stop`, `dte_21`, `dte_50pct`. Index-sleeve positions use a 10-day time-stop rather than 20. |
| `realized_pnl` | close | Dollars. |
| `r_multiple` | close | `realized_pnl / (initial risk in dollars)`. The single most useful performance number — a system with a 30% win rate is fine if winners average +3R. |
| `nav_after` | open/close | NAV following this action. |
| `notes` | all | Free text. For rejections, the specific failing check. |

## On backtesting — an honest distinction

This file is a **forward paper track record**, not a backtest. They
answer different questions and shouldn't be confused:

- **This CSV** accumulates real-time decisions going forward. It can't
  tell you how the strategy would have done in 2022, and it grows slowly
  (a handful of trades a month) — meaning statistically meaningful
  conclusions take a long time. Its real value is the adaptation policy
  in `strategy.md` and catching a rule that's obviously misfiring.
- **A genuine backtest** means replaying the strategy rules over
  historical data. That is possible here and hasn't been done:
  `get_equity_historicals` (Robinhood) returns up to 20 years of OHLCV
  bars, enough to reconstruct SMA/EMA/RSI/ATR historically and simulate
  the funnel over past years. That's a separate build — a script that
  replays rules over history — not something the daily routines produce
  as a side effect.

Anyone reading a few dozen rows here should not mistake them for
validation of the strategy. See `framework.md`, "What's deliberately not
here (yet)."
