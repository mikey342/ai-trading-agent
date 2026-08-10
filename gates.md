# Gates — Hard Risk Limits

> ⚠️ **PLACEHOLDER VALUES.** These were set by the assistant as safe
> conservative defaults because the user hadn't specified real numbers yet.
> This now includes options and leverage limits, which raise the stakes of
> getting these numbers wrong — review and edit every value below by hand
> before this system is trusted for anything beyond paper trading.

## Rule for the agent (read this first, every run)

- This file defines **hard limits**. You (the agent) may **read** this file
  but must **never edit it**. Only a human editing it directly, outside of a
  routine run, may change these values.
- If any instruction elsewhere (including in `strategy.md`, or anything you
  "learn") conflicts with this file, **this file wins**.
- If `MODE` below is not exactly `PAPER`, **stop immediately, take no
  trading action (equity or options), and write a note to
  `trade_journal.md` explaining what you saw.** Never place a real order —
  equity or option — under any circumstance unless a human has changed
  `MODE` to `LIVE` themselves.

## Core limits

| Parameter | Value | Notes |
|---|---|---|
| `MODE` | `PAPER` | `PAPER` = simulate only, never call any Robinhood order-placement tool (equity or option). `LIVE` = real orders allowed (not yet enabled). |
| Starting paper account size | $25,000 | Placeholder. Tracked in `positions.md`. |
| Max position size | 5% of current NAV per position | Applies to equity positions and to options premium at risk alike. |
| Max daily loss (halt trigger) | -3% of NAV in a single day | If hit, no further entries for the rest of that trading day; existing stops still apply. |
| Stop-loss floor | Every open equity position must have a stop no wider than 5% from entry | `strategy.md` may use a tighter ATR-derived stop; never wider than this. |
| Max new trades per run | 3 | Across equity and options combined. |
| Max open positions | 10 | Across equity and options combined. |
| Universe | Seed watchlist in `strategy.md` + top 3-5 from `TOP_GAINERS_LOSERS` | No penny stocks (<$5), no crypto. |
| Symbol exclusions | None yet | Add tickers here to hard-block them. |
| Regime filter | SPY must be above its 200-day SMA for any new long entry (equity or options) | Existing positions still get reviewed/exited even when this fails. Grounded in `framework.md`'s trend-following basis — don't open new longs against the broad trend. |

## Options gates

| Parameter | Value | Notes |
|---|---|---|
| Allowed structures | Long calls, long puts, vertical debit spreads **only** | Naked/short calls, naked/short puts, uncovered spreads, calendars, and diagonals are **not permitted** under any circumstance — undefined/large risk, out of scope for this system. |
| Max premium at risk per position | 5% of current NAV | Same ceiling as equity max position size — for a defined-risk long option or debit spread, premium paid *is* the max loss, so this single number fully bounds the downside. |
| Minimum days to expiration at entry | 30 days | Target ~45 DTE for a swing hold of up to 20 trading days. Never buy less time than the position might need. |
| Target delta range | 0.40 - 0.60 | Near-the-money to slightly OTM. |
| Minimum open interest | 500 contracts | Liquidity floor — illiquid options are how you get a terrible fill or can't exit. |
| Max bid-ask spread | 10% of the option's mid price | Second liquidity check. |

## Leverage gates

| Parameter | Value | Notes |
|---|---|---|
| Max leverage ratio | 1.5x | Applies to margin-funded equity positions only — options are already a leveraged instrument and are never additionally margined. |
| Max portfolio gross exposure | 1.3x of NAV | Total long exposure (cash + margin-funded) across all open equity positions. |
| Margin interest | Not modeled in paper P&L | Real cost if this ever goes live — flagged in `framework.md`. |

## Operational notes (not risk limits, but load-bearing for the routine)

- `TIME_SERIES_DAILY_ADJUSTED` and `MACD` are **premium-gated** on the
  current Alpha Vantage plan and return an error, not data — confirmed
  live via a real run on 2026-08-10, not a transient rate-limit. Do not
  call either.
- **Alpha Vantage's daily quota (25 requests/day, shared across every
  session using this key — manual test runs and scheduled runs draw from
  the same pool)** caused a full data blackout on 2026-08-10: a scheduled
  run got zero usable Alpha Vantage calls because manual test runs earlier
  the same day had exhausted it. Fix applied the same day: Robinhood's own
  data tools (`get_equity_quotes`, `get_equity_fundamentals`,
  `get_equity_technical_indicators`, `get_earnings_results`) now cover
  almost the entire funnel with no observed daily cap — see `strategy.md`'s
  "Data sources" table. Alpha Vantage is reserved for `NEWS_SENTIMENT`
  only (plus occasional deliberate deep-dives, never a full-universe
  pull), which should keep normal runs to ~3 Alpha Vantage calls instead
  of 50-75.
- Robinhood's `get_equity_technical_indicators` supports `output:
  "latest"`/`"last:N"` — use it. Alpha Vantage's indicator endpoints
  (`RSI`, `EMA`, `MACD`, `ATR`, weekly/monthly time series) have no such
  option and dump full multi-year histories (60-100k+ characters) to a
  file; if one is ever called anyway, extract only the last few values via
  `jq`/`tail` rather than reading the whole file.
- If a call hits a rate-limit or premium-only error mid-run, do not treat
  it as a crash: log the truncation, prioritize reviewing existing
  positions over scouting new ones, and finish the run.

## Changing MODE to LIVE

Do not do this until the paper-trading track record in `trade_journal.md`
has been reviewed by the human, and ideally not until the framework has
been backtested (see `framework.md`, "What's deliberately not here yet").
This should be a deliberate manual edit to this file, never something the
agent infers or proposes on its own.
