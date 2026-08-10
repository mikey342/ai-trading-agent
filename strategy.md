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

### Value tiebreaker and Tier 2 cap

Published research (O'Shaughnessy's *What Works on Wall Street*, AQR's
"Value and Momentum Everywhere" — see `framework.md`) finds that momentum
combined with value outperforms momentum alone. So among Tier 1 survivors,
rank by a composite of `0.6 × relative-strength proxy (from #4 above) +
0.4 × (1 / PEGRatio)` (from `COMPANY_OVERVIEW`; if `PEGRatio` is missing or
negative, use `PERatio` inverted instead, or treat that term as neutral —
don't drop the candidate solely for a missing field). This is a
prioritization, not an additional pass/fail gate — it decides which
survivors advance to the more expensive Tier 2, capped at the **top 8**, to
keep the funnel's cost bounded even if an unusually large fraction of the
watchlist passes Tier 1.

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

1. **Momentum confirmation**: EMA8/EMA21 spread (already pulled in Tier 2 —
   no new call needed) is widening over its last 5 values, i.e. momentum
   is accelerating, not just present. `MACD` was the original design for
   this check but is **permanently premium-gated on the current Alpha
   Vantage plan** (confirmed live, not a transient rate-limit — see
   `gates.md` operational notes), so it is never called. If the plan is
   ever upgraded to unlock it, MACD confirmation can be reinstated as a
   second, redundant momentum check — but the EMA-spread check alone is
   sufficient and must not be treated as incomplete confirmation.
2. `NEWS_SENTIMENT` (limit 5-10, this symbol): sentiment not negative.
3. `EARNINGS_CALENDAR` (this symbol): no earnings report scheduled in the
   next 3 calendar days (blackout — this is event risk, not a rule this
   system is trying to trade).
4. `ATR` (14, daily): pull and extract the latest value — needed for
   sizing and the stop below.

A candidate that fails any Tier 3 check is dropped, not force-fit.

## Position sizing (equity) — risk-based, not equal-weight

1. `stop_distance = min(1.0-1.5 × ATR14, entry_price × gates.md stop-loss
   floor)` — always the *tighter* of the ATR-derived stop and the hard
   ceiling. Use 1.5x by default; 1.0x is the tighter end of the range used
   by contemporary breakout swing traders (see `framework.md`, Qullamaggie)
   and is the adaptive knob to tighten if stops are proving too loose.
2. `risk_per_trade = 1% of current NAV` (adaptive — this is the number to
   tune if position sizing needs revisiting, not the gates.md ceiling).
3. `shares = floor(risk_per_trade_dollars / stop_distance)`
4. `position_dollar_size = shares × entry_price`, capped at gates.md's max
   position size regardless of what the risk math above produced.
5. `R-target = entry_price + 2 × stop_distance` (2R) — not an automatic
   exit; see the trailing-stop rule below.

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
- **At entry**, compute the underlying stop and R-target using the exact
  same ATR-based math as the equity position sizing above, and write them
  into `positions.md`'s `Underlying stop` / `Underlying R-target` columns.
  These are fixed at entry and never recomputed on a later run — a future
  run comparing against a freshly-recalculated ATR could silently drift
  from what was actually decided at entry.
- **Every subsequent run**, compare the underlying's current price against
  those stored values using the exact same trailing-stop mechanic as the
  equity exit rules (fixed stop until the R-target is hit, then trail
  EMA21, one-way flag, never lower than the original stop).
- Exit the option position if: the underlying hits its current exit stop
  (per the trailing mechanic above), OR 21 days-to-expiration remaining is
  reached (standard practitioner checkpoint — force a close or roll
  decision, don't let gamma risk run unmanaged into expiration), OR 50% of
  the option's DTE at entry has elapsed with the R-target not yet reached
  — whichever comes first.

## Leverage overlay

If used, funds an equity position (never options — options are already
leveraged instruments) partially on margin, up to gates.md's max leverage
ratio. Same stop-loss/target/time-stop rules apply unchanged; leverage only
changes funding, not risk discipline.

## Exit rules (all positions, checked every run)

Mechanical trailing-stop logic (source: Qullamaggie's "let winners run"
approach — a 20-35% win rate can still be strongly profitable if winners
are held longer than losers; see `framework.md`). This replaces a fixed
take-profit with a state flag per position, tracked in `positions.md`:

- **Before the R-target is reached:** the exit stop is the original fixed
  `stop_distance` stop from entry. Nothing else moves it.
- **The instant price reaches the R-target:** record in `positions.md`
  that this position has "reached R-target" — this is a one-way flag, it
  never resets. From this point on, the exit stop becomes the rising
  EMA21 (equity: pulled fresh via `EMA` for open positions during the
  review step) instead of the original fixed stop. Never lower this
  trailing stop once set, and never let it fall below the original fixed
  stop even if EMA21 dips under it.

Close a position if **any** of:
1. Price hits the current exit stop (fixed stop pre-R-target, or trailing
   EMA21 stop post-R-target, per above)
2. Price closes back below `50DayMovingAverage` (trend template broken —
   the premise for the trade no longer holds). This always exits,
   regardless of R-target/trailing-stop state.
3. Held > 20 trading days without reaching the R-target (time-stop —
   RSI(14) > 75 pre-R-target is a signal to watch more closely, not an
   automatic exit on its own; the time-stop and trend-template rules
   above are what actually close a stalled position)
4. For options specifically: 21 days-to-expiration reached, OR 50% of the
   DTE at entry has elapsed with no progress toward the R-target —
   whichever comes first (see the options overlay section for the full
   rationale)

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
