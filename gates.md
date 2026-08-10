# Gates — Hard Risk Limits

> **Finalized 2026-08-10.** These are real, deliberately-chosen numbers
> (Conservative risk profile, $10,000 paper account, leverage enabled at
> margin disabled) — not assistant-invented placeholders. They can still be revisited
> as the paper track record develops, but any change should be a
> deliberate decision, logged here, not a drift.

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
| Starting paper account size | $10,000 | User-specified 2026-08-10 (was a $25,000 placeholder before; changed with zero trades/P&L to date, so no historical inconsistency). Tracked in `positions.md`. |
| Max position size | 3% of current NAV per position | Applies to equity positions and to options premium at risk alike. |
| Max daily loss (halt trigger) | -2% of NAV in a single day | If hit, no further entries for the rest of that trading day; existing stops still apply. |
| Stop-loss floor | Every open equity position must have a stop no wider than 3% from entry | `strategy.md` may use a tighter ATR-derived stop; never wider than this. |
| Max new trades per run | 2 | Across equity and options combined. |
| Max open positions | 6 | Across equity and options combined. |
| Universe | Seed watchlist in `strategy.md` + top 3-5 from `TOP_GAINERS_LOSERS` | No penny stocks (<$5), no crypto. |
| Symbol exclusions | None yet | Add tickers here to hard-block them. |
| Regime filter (sets direction) | SPY above its 200-day SMA → **LONG mode** (long equity, calls, call debit spreads). SPY below → **SHORT mode** (long puts / put debit spreads **only**). | Existing positions are reviewed/exited in either mode. Grounded in `framework.md`'s trend-following basis — never open new longs against the broad trend, and never open new shorts with the broad trend against you. |
| **Short stock / margin** | **Both forbidden, absolutely** | Never short an equity outright and never borrow on margin, in any mode, for any reason. Short selling carries unbounded loss, borrow cost, and recall risk; margin adds forced liquidation. A system reviewing positions a few times a day cannot manage either. Bearish exposure is taken **only** by *buying* a 2x inverse index ETF (SDS/QID) with cash, where max loss is the amount paid. |

## Options gates

| Parameter | Value | Notes |
|---|---|---|
| Allowed structures | Long calls, long puts, vertical debit spreads **only** | Naked/short calls, naked/short puts, uncovered spreads, calendars, and diagonals are **not permitted** under any circumstance — undefined/large risk, out of scope for this system. Calls in LONG mode, puts in SHORT mode. |
| Effective premium cap at current NAV | **$300** per position ($10,000 × 3%) | Consequence worth knowing: a near-the-money contract on a $200+ underlying often exceeds this (a verified JPM ~45 DTE ATM call was $970). Debit spreads and lower-priced underlyings are the workable expressions. Never raise the cap to make a trade fit. |
| Max premium at risk per position | 3% of current NAV | Same ceiling as equity max position size — for a defined-risk long option or debit spread, premium paid *is* the max loss, so this single number fully bounds the downside. |
| Minimum days to expiration at entry | 30 days | Target ~45 DTE for a swing hold of up to 20 trading days. Never buy less time than the position might need. |
| Target delta range | 0.40 - 0.60 | Near-the-money to slightly OTM. |
| Minimum open interest | 500 contracts | Liquidity floor — illiquid options are how you get a terrible fill or can't exit. |
| Max bid-ask spread | 10% of the option's mid price | Second liquidity check. |

## Leverage gates — margin disabled, leveraged ETFs only

| Parameter | Value | Notes |
|---|---|---|
| **Margin / borrowing** | **Disabled (1.0x)** | Per user decision 2026-08-10: no margin, no borrowing, ever. Every position is bought with cash. Removes margin calls, forced liquidation, and margin interest from the system entirely. |
| Leveraged exposure method | **Buying 2x ETFs with cash only** | LONG: SSO (S&P) / QLD (Nasdaq). SHORT: SDS (S&P) / QID (Nasdaq). All verified liquid 2026-08-10. Max loss is the amount paid — these cannot go below zero or generate a liability. |
| Max open index-sleeve positions | **1** | Never hold a long and an inverse ETF at once — they are direct opposites and would cancel while paying two expense ratios. |
| Leveraged ETF exposure accounting | **Counts 2x toward gross exposure** | A 3%-of-NAV position in a 2x ETF is ~6% of effective market exposure. Count it that way, not at face value. |
| Max portfolio gross exposure | 1.1x of NAV | Computed with the 2x multiplier above applied to any leveraged ETF position. |
| Index leveraged ETF time-stop | **10 trading days** (vs 20 for ordinary stock) | These target 2x the *daily* return and reset daily, so multi-day holds are path-dependent and decay in chop — a flat round-trip in the index still loses money. Short leash by design. See `strategy.md`. |
| **Single-stock** leveraged ETF time-stop | **7 trading days** | Decay scales with σ², and single stocks are far more volatile than indices — roughly 36%/yr drag on a TSLA-like name, ~70%/yr on a COIN-like one, versus ~2.6%/yr on SPY. |
| Single-stock leveraged ETF volatility gate | Underlying **ATR(14) ≤ 4% of price** | Hard gate. Above it, decay is too steep for the holding period — trade the plain stock instead. Observed 2026-08-10: CONL (2x COIN) fell **92%** from its 52-week high while COIN itself did nothing remotely similar. |
| Leveraged ETF liquidity floor | 30-day average volume ≥ 1M shares | Many single-stock leveraged ETFs are thin; verify before using any not already listed in `strategy.md`. |

## Execution gates

| Parameter | Value | Notes |
|---|---|---|
| Order type | **Marketable limit only — never market orders** | Buys limit at the ask, sells limit at the bid. A market order surrenders price control and can fill far from the last print in a fast or thin market. |
| Simulated fill price | Buys at `ask_price`, sells at `bid_price` (options: `high_fill_rate_buy/sell_price`) | Never the midpoint or `last_trade_price` — both flatter paper P&L versus a real fill and hide the spread cost of every round trip. |
| Max bid-ask spread at entry | **0.5% of mid** (equity/ETF) | Wider than this on a name that passed the liquidity screen usually signals a halt, a news event, or stale data. Skip and log rather than crossing it. |
| Trading window | All runs inside regular hours: **10:30am, 2:30pm, 3:30pm ET** | Never pre-open (stale/thin quotes) and never post-close (a decision that cannot be filled until the next session at an unknown price). 10:30am rather than the bell avoids the widest spreads of the day. |

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
