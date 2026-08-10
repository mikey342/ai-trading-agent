# Strategy — Adaptive Swing Trading Playbook

This file operationalizes `framework.md` into concrete, checkable rules.
The agent **may edit this file** as it learns from `trade_journal.md`, but
every edit must be small, specific, and logged in the Changelog with a
cited rationale — never a wholesale rewrite in one run. See the
Adaptation policy at the bottom.

`gates.md` always wins in a conflict. This file never overrides a hard
limit.

## Universe

Benchmark/regime reference: **SPY** (not itself traded — used for the
regime filter and as the relative-strength baseline).

Seed watchlist (diversified across sectors so cross-sectional comparisons
mean something — edit freely, but keep it diversified):

| Symbol | Sector |
|---|---|
| AAPL, MSFT, NVDA, GOOGL | Technology |
| AMZN, COST, MCD | Consumer discretionary |
| JPM, V, MA | Financials |
| UNH, LLY, JNJ | Healthcare |
| HON, CAT | Industrials |
| XOM, CVX | Energy |
| META | Communication services |
| PG | Consumer staples |

Plus opportunistic candidates from `TOP_GAINERS_LOSERS` each run (top 3-5
only, to keep the funnel's Tier 1 pass tractable — see `framework.md`).

## Data sources — validated, tiered (see framework.md "The funnel")

Confirmed live against the actual connected accounts:

| Tool | Cost | Notes |
|---|---|---|
| `COMPANY_OVERVIEW` | Cheap, clean JSON | Has `50DayMovingAverage`, `200DayMovingAverage`, `52WeekHigh/Low`, P/E, PEG, ROE, margins, growth, Beta |
| `GLOBAL_QUOTE` | Cheap, clean JSON | Current price, volume, day change |
| `RSI`, `EMA`, `MACD`, `ATR` | **Expensive** — full history dumped to a file, not returned inline | Extract only the last 5 data points via `jq`/`tail`; never read the whole file (see `run_instructions.md`) |
| `TIME_SERIES_DAILY_ADJUSTED` | **Blocked** — premium-only on the current Alpha Vantage plan | Do not call it; if it's ever needed, that means the plan changed and this file should be revisited |
| `TIME_SERIES_WEEKLY_ADJUSTED` | Works but huge (100k+ chars) | Avoid unless a specific rule below calls for it |
| `NEWS_SENTIMENT` | Works but large (limit the `limit` param, e.g. 5-10) | Use only in Tier 3, per symbol |
| `EARNINGS_CALENDAR` | Cheap | Use per finalist symbol to check the blackout window |
| Robinhood `get_option_chains`, `get_option_quotes` | As needed | Tier 3, only if an options expression is being considered |

## Tier 1 — Trend template (full universe + top gainers/losers)

Pull `COMPANY_OVERVIEW` + `GLOBAL_QUOTE` for every candidate. A candidate
survives Tier 1 only if **all** of (Minervini-style trend template, adapted
to available fields):

1. Current price > `50DayMovingAverage` > `200DayMovingAverage`
2. Current price within 25% of `52WeekHigh`
3. Current price at least 25% above `52WeekLow`
4. Relative-strength proxy: `(price - 52WeekLow) / (52WeekHigh - 52WeekLow)` ≥ 0.6
   (i.e., trading in the upper 40% of its 52-week range — a cheap stand-in
   for a true relative-strength-vs-SPY line, which would need expensive
   historical series for every candidate; see `framework.md` limitations)
5. Also pull `GLOBAL_QUOTE` for SPY once per run (not per candidate) and
   confirm SPY's own price > its `200DayMovingAverage` (from
   `COMPANY_OVERVIEW` on SPY) — this is the regime filter from `gates.md`.
   If it fails, skip Tier 2/3 entirely for new entries this run (existing
   positions are still reviewed per the exit rules below).

## Tier 2 — Entry trigger (shortlist from Tier 1 only)

Pull `RSI` (14, daily) and `EMA` (8 and 21, daily) for Tier 1 survivors,
extracting the last 5 values of each. A candidate qualifies if **either**:

- **Breakout trigger** (Turtle-style): price within 2% of `52WeekHigh`
  AND EMA8 > EMA21
- **Pullback trigger** (Connors-style): RSI(14) between 35-45 AND price
  still > `50DayMovingAverage` (pullback within an intact uptrend, not a
  breakdown) AND EMA21 is flat-to-rising over the last 5 values (trend not
  rolling over)

## Tier 3 — Confirmation (finalists only, cap at 3 per run per gates.md)

For candidates passing Tier 2:

1. `MACD` (12/26/9, daily): MACD line > signal line, or a bullish crossover
   within the last 3 of the 5 extracted values.
2. `NEWS_SENTIMENT` (limit 5-10, this symbol): sentiment not negative.
3. `EARNINGS_CALENDAR` (this symbol): no earnings report scheduled in the
   next 3 calendar days (blackout — this is event risk, not a rule this
   system is trying to trade).
4. `ATR` (14, daily): pull and extract the latest value — needed for
   sizing and the stop below.

A candidate that fails any Tier 3 check is dropped, not force-fit.

## Position sizing (equity) — risk-based, not equal-weight

1. `stop_distance = min(1.5 × ATR14, entry_price × gates.md stop-loss floor)`
   — always the *tighter* of the ATR-derived stop and the hard ceiling.
2. `risk_per_trade = 1% of current NAV` (adaptive — this is the number to
   tune if position sizing needs revisiting, not the gates.md ceiling).
3. `shares = floor(risk_per_trade_dollars / stop_distance)`
4. `position_dollar_size = shares × entry_price`, capped at gates.md's max
   position size regardless of what the risk math above produced.
5. Take-profit = `entry_price + 2 × stop_distance` (2R reward:risk).

## Options overlay (defined-risk only — see gates.md for hard limits)

Used only to express a signal that already passed Tiers 1-3, when it makes
the trade more capital-efficient. Never a standalone signal source.

- Only long calls, long puts, or vertical debit spreads. Never short/naked.
- Minimum 30 days to expiration, target ~45 DTE for a swing hold of up to
  20 trading days (never buy less time than you expect to need).
- Target delta 0.40-0.60 (near-the-money to slightly OTM).
- Liquidity gate (hard, see `gates.md`): adequate open interest, bid-ask
  spread within limit.
- Max premium at risk = gates.md's options premium cap. This is the entire
  max loss for the position — no additional stop-loss math needed, the
  defined-risk structure already caps it.
- Exit: at the equity-equivalent stop/target price, OR when 50% of the
  option's remaining DTE (at entry) has elapsed, whichever comes first —
  protects against theta decay eating a thesis that hasn't played out yet.

## Leverage overlay

If used, funds an equity position (never options — options are already
leveraged instruments) partially on margin, up to gates.md's max leverage
ratio. Same stop-loss/target/time-stop rules apply unchanged; leverage only
changes funding, not risk discipline.

## Exit rules (all positions, checked every run)

Close a position if **any** of:
1. Price hits the recorded stop-loss
2. Price hits the recorded take-profit
3. RSI(14) crosses above 75 (extreme overbought)
4. Price closes back below `50DayMovingAverage` (trend template broken —
   the premise for the trade no longer holds)
5. Held > 20 trading days with no progress toward target (time-stop)
6. For options specifically: 50% of remaining DTE elapsed with no progress

## Adaptation policy

Every run, after updating the journal:
- Look at the last 10 closed trades in `trade_journal.md`.
- If a specific, named parameter above (RSI pullback band, EMA periods,
  ATR stop multiplier, risk-per-trade %, relative-strength threshold, DTE/
  delta targets) is associated with a losing pattern across ≥5 of the last
  10 trades, propose ONE small, specific adjustment — not a wholesale
  rewrite, and never a change to which methodologies are used (that's a
  `framework.md` conversation with the human, not something to self-modify).
- Log the change below: date, what changed, why (cite the trades), what
  you'll watch for.
- Never touch a `gates.md`-governed limit to "fix" a losing streak.

## Changelog

- (seed) Rewritten from a generic RSI/MACD/SMA gate system into a swing
  trading framework grounded in Turtle/Minervini/O'Neil/Connors/PEAD
  methodologies, per user request. Not yet run — no track record to
  evaluate yet.
