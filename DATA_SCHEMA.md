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
| `run_type` | all | `morning` / `midday` / `pre-close` / `monitor`. |
| `action` | all | `open` / `close` / `reject`. |
| `symbol` | all | Underlying ticker. |
| `sleeve` | all | `stock` (individual name from the Tier 0-3 funnel) or `index` (2x ETF market-direction bet). |
| `instrument` | all | `equity`, `leveraged_etf`, or `option`. |
| `direction` | all | `bullish` or `bearish` — which template the setup passed. |
| `counter_trend` | all | `true` if the setup ran against the SPY regime (bearish while SPY > SMA(200), or bullish while below), `false` otherwise. Counter-trend trades are capped at 1 open, sized at half risk, and require undegraded data — so this column is the key one for later checking whether that allowance earned its place or just added losses. |
| `sector` | all | From `get_equity_fundamentals`. Drives the max-3-per-sector concentration cap, and lets performance be sliced by sector later. |
| `ai_theme` | all | `true` if the symbol is in `ai_theme.md` (which grants a 1.15× ranking tilt), else `false`. **This is the column that settles whether the AI tilt earns its place** — compare win rate and average R for `true` vs `false` once enough trades close. Tilt on evidence, not conviction. |
| `risk_level` | all | `NORMAL` / `CAUTION` / `STRESSED` at decision time — the worse of SPY-vs-50DMA and VIX. Records the conditions a trade was taken under, so a bad stretch can be checked against the regime it happened in. |
| `mode` | all | `LONG` or `SHORT` — the SPY direction regime, set by SPY vs its SMA(200) with 3-day confirmation. Note this no longer gates which direction is *examined* (both always are); it determines which passes count as counter-trend. |
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
| `relative_volume` | all | Last COMPLETED session volume ÷ 30-day average, from `get_equity_fundamentals`. The breakout gate: ≥1.4 (O'Neil's 40% above average). Computed, never read from the scan's column, which uses `session="all"`. |
| `momentum_test_would_pass` | all | Whether the retired EMA-spread test would have passed. **Logged only, gates nothing** — kept so the pre-registered question stays answerable: did the invented rule actually predict anything? |
| `trigger` | open/reject | Which entry rule fired. LONG: `breakout_52w`, `breakout_20d`, `pullback`. SHORT: `breakdown_52w`, `breakdown_20d`, `rally_to_resistance`. When both breakout triggers qualify, record the stronger one (`breakout_52w`). **Slice performance by this column** — `breakout_20d` is the weaker level signal and was added to fill a coverage gap; if it underperforms the other two once trades close, it should be removed rather than kept on the argument that introduced it. |
| `news_sentiment` | all | Score, or the literal `UNAVAILABLE` when the Alpha Vantage quota was exhausted (see `strategy.md`'s degradation rule). Never fabricate this. |
| `social_score` | all | Stocktwits `sentiment.score` (0-100) at decision time, or `NO_COVERAGE`. **Never `bullish_pct`** — on thin names that reads as false consensus (IMAX showed "100% bullish" against a canonical score of 47). |
| `social_label` | all | Stocktwits `sentiment.label` (Extremely Bearish … Extremely Bullish), or `NO_COVERAGE`. |
| `social_msg_volume` | all | Stocktwits `message_volume.score` (0-100 normalized) — the "is anyone talking about this" measure. Expect `NO_COVERAGE` on roughly two-thirds of candidates; the screener selects mid-caps in strong trends, which is not where retail chatter lives. |
| `earnings_days_away` | all | Days to next earnings report; blackout is <3. |
| `regime` | all | `risk-on` / `risk-off` — SPY vs its 200-day SMA. |
| `strike`, `expiration`, `delta_at_entry`, `iv_at_entry`, `dte_at_entry`, `underlying_price` | options only | Blank for equity rows. |
| `exit_rule` | close | Which rule fired: `stop`, `trailing_stop`, `trend_break`, `time_stop`, `dte_21`, `dte_50pct`. Time-stop limits by sleeve: 10 trading days for ordinary stock, 8 for an index-sleeve ETF, 5 for a single-stock leveraged ETF. |
| `realized_pnl` | close | Dollars. |
| `r_multiple` | close | `realized_pnl / (initial risk in dollars)`. The single most useful performance number — a system with a 30% win rate is fine if winners average +3R. |
| `nav_after` | open/close | NAV following this action. |
| `notes` | all | Free text. For rejections, the specific failing check. |

## What the paper P&L does and doesn't capture

Everything logged here is modeled as if trading for real, and the numbers
are not idealized: prices are live at the actual decision time, buys fill
at the **ask** and sells at the **bid** (never the midpoint), so the full
round-trip spread is charged on every trade. Orders are modeled as
marketable limits, and every gate — chase protection, spread, depth,
position caps — is applied exactly as it would be live.

Five things are **not** modeled, and every one of them makes the paper
record look better than reality would:

1. **Fill certainty.** Every marketable limit is assumed to fill
   instantly at the quoted price. Real ones partially fill, or miss
   entirely when the book moves first.
2. **Slippage beyond the quote.** The spread is charged; the extra drift
   a real order can suffer in a fast market is not.
3. **Options contract fees** (~$0.03–0.65 per contract in regulatory
   fees). Equities and ETFs are commission-free, options are not.
4. **Leveraged ETF expense ratios** (~0.9%/yr) — non-trivial even on a
   7–10 day hold.
5. **Dividends.** A real position crossing an ex-date collects the
   payment; the paper book does not.

**The paper book is entirely separate from the real brokerage account.**
Market data is read from Robinhood; actual positions are never read and
never touched, and no order-placement tool is ever called.

Practical reading: expect live results to run modestly worse than the
paper record. A strategy showing only a thin paper edge most likely has
no real one once these costs land.

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
