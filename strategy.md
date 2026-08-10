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

Confirmed live against the actual connected accounts. **Robinhood is now
the primary source for almost everything** — it has no observed daily
cap (unlike Alpha Vantage's hard 25/day), returns clean values with a
built-in `output: latest`/`last:N` trim (no huge-payload/file-extraction
workaround needed), and its indicator values cross-checked against Alpha
Vantage's (JPM 50DMA: $333.00 from both, confirmed 2026-08-10). Alpha
Vantage is reserved for the handful of things only it provides.

| Tool | Provider | Notes |
|---|---|---|
| `get_equity_quotes` | Robinhood | Current price. Batches up to 20 symbols in one call. |
| `get_equity_fundamentals` | Robinhood | PE, PB, market cap, 52-week high/low (with dates), sector. Batches up to 10 symbols per call. No PEG — use PE ranked cross-sectionally within the candidate set instead. |
| `get_equity_technical_indicators` | Robinhood | `sma`/`ema`/`rsi`/`atr`/`macd`/`bollinger`/`vwap`/`obv`, one symbol per call, but `output: "latest"` or `"last:N"` returns exactly what's needed — no extraction step required. |
| `get_earnings_results` | Robinhood | Trailing 8 quarters actual-vs-estimate EPS + next report date, one symbol per call. Covers both the earnings-blackout check and a real surprise-history read for the PEAD catalyst layer. |
| `get_option_chains`, `get_option_quotes` | Robinhood | Tier 3+, only if an options expression is being considered, or reviewing an open options position. |
| `NEWS_SENTIMENT` | Alpha Vantage | The one thing Robinhood has no equivalent for. Limit the `limit` param (5-10). Use sparingly — Tier 3 finalists only, capped at 3/run by gates.md. |
| `MARKET_STATUS` | Alpha Vantage | Cheap, used once per run for the open/closed check. If Alpha Vantage's daily quota is exhausted, treat the market as open during normal US trading hours as a fallback rather than skipping the run entirely — this check doesn't need Alpha Vantage specifically. |
| `TIME_SERIES_DAILY_ADJUSTED`, `MACD` (Alpha Vantage) | Alpha Vantage | **Blocked** — premium-only on the current plan. Never call either; Robinhood's `get_equity_technical_indicators` (type=macd) covers the MACD need if it's ever wanted again. |
| `INSTITUTIONAL_HOLDINGS`, `INSIDER_TRANSACTIONS`, `EARNINGS_CALL_TRANSCRIPT`, `EARNINGS_ESTIMATES` | Alpha Vantage | Not used in the regular funnel — reserved for occasional, deliberate deep-dives on a specific name, never a full-universe pull. |

## Tier 1 — Trend template (full universe + top gainers/losers)

Pull `get_equity_quotes` (all candidates in one batched call, up to 20) +
`get_equity_fundamentals` (batched, up to 10 per call — so 2 calls for the
full watchlist). Then, per candidate, `get_equity_technical_indicators`
(type=sma, period=50, output=latest) and (type=sma, period=200,
output=latest) — these are one-symbol-per-call, so this is the expensive
part of Tier 1, budget-wise, but Robinhood has shown no daily cap so far
(unlike Alpha Vantage's hard 25/day). A candidate survives Tier 1 only if
**all** of (Minervini-style trend template, adapted to available fields):

1. Current price > SMA(50) > SMA(200)
2. Current price within 25% of `high_52_weeks`
3. Current price at least 25% above `low_52_weeks`
4. Relative-strength proxy: `(price - low_52_weeks) / (high_52_weeks -
   low_52_weeks)` ≥ 0.6 (i.e., trading in the upper 40% of its 52-week
   range — a cheap stand-in for a true relative-strength-vs-SPY line,
   which would need a full historical series for every candidate; see
   `framework.md` limitations)
5. Also pull `get_equity_quotes` for SPY once per run (not per candidate)
   and `get_equity_technical_indicators` (SPY, sma, period=200,
   output=latest) to confirm SPY's price > its 200-day SMA — this is the
   regime filter from `gates.md`. If it fails, skip Tier 2/3 entirely for
   new entries this run (existing positions are still reviewed per the
   exit rules below).

### Value tiebreaker and Tier 2 cap

Published research (O'Shaughnessy's *What Works on Wall Street*, AQR's
"Value and Momentum Everywhere" — see `framework.md`) finds that momentum
combined with value outperforms momentum alone. So among Tier 1 survivors,
rank by a composite of `0.6 × relative-strength proxy (from #4 above) +
0.4 × (1 / pe_ratio, ranked cross-sectionally within this run's candidate
set — lower P/E ranks higher)` (from `get_equity_fundamentals`; Robinhood
doesn't expose PEG, so this uses raw P/E relative to peers in the same
run rather than growth-adjusted P/E — a cruder value signal, noted as a
limitation, not silently upgraded to something it isn't). This is a
prioritization, not an additional pass/fail gate — it decides which
survivors advance to the more expensive Tier 2, capped at the **top 8**.

## Tier 2 — Entry trigger (shortlist from Tier 1 only)

Pull `get_equity_technical_indicators` for RSI (period=14, output="last:5")
and EMA (period=8 and period=21, output="last:5") for Tier 1 survivors —
`output` trims the response server-side, no extraction step needed. A
candidate qualifies if **either**:

- **Breakout trigger** (Turtle-style): price within 2% of `high_52_weeks`
  AND EMA8 > EMA21
- **Pullback trigger** (Connors-style): RSI(14) between 35-45 AND price
  still > SMA(50) (pullback within an intact uptrend, not a breakdown)
  AND EMA21 is flat-to-rising over the last 5 values (trend not rolling
  over)

## Tier 3 — Confirmation (finalists only, cap at 3 per run per gates.md)

For candidates passing Tier 2:

1. **Momentum confirmation**: EMA8/EMA21 spread (already pulled in Tier 2 —
   no new call needed) is widening over its last 5 values, i.e. momentum
   is accelerating, not just present. `MACD` (Alpha Vantage) was the
   original design for this check but is **permanently premium-gated on
   the current Alpha Vantage plan** (confirmed live, not a transient
   rate-limit — see `gates.md`). Robinhood's `get_equity_technical_
   indicators` (type=macd) is a working substitute if a second, redundant
   momentum check is ever wanted — but the EMA-spread check alone is
   sufficient and must not be treated as incomplete confirmation.
2. `NEWS_SENTIMENT` (Alpha Vantage, limit 5-10, this symbol): sentiment
   not negative. The one step in this whole funnel that still needs
   Alpha Vantage — Robinhood has no news/sentiment tool.
   **Graceful degradation (important):** if this call fails because the
   Alpha Vantage daily quota is exhausted, do **not** drop the candidate
   and do **not** invent a sentiment value. Proceed with the trade if
   every other check passes, and record `news_sentiment: UNAVAILABLE` in
   both `trade_log.csv` and the journal entry. Rationale: a hard block
   here would mean a free-tier quota limit silently becomes a permanent
   no-trade bug (the same class of failure as the premium-gated `MACD`
   issue). The trend template and earnings blackout still provide partial
   protection against bad-news names. This is a real, accepted reduction
   in safety, which is exactly why it must be logged every time rather
   than passed over silently.
3. `get_earnings_results` (Robinhood, this symbol): no earnings report
   scheduled in the next 3 calendar days (blackout — event risk, not a
   rule this system is trying to trade). Also gives 8 quarters of
   actual-vs-estimate EPS — a consistent beat streak is a soft positive
   for the PEAD catalyst read, a miss streak a soft negative.
4. `get_equity_technical_indicators` (Robinhood, type=atr, period=14,
   output=latest): needed for sizing and the stop below.

A candidate that fails any Tier 3 check is dropped, not force-fit.

## Entry price and chase-protection

The funnel (Tiers 1-3) can take several minutes to run across the full
universe — this is a multi-step agent doing sequential tool calls and
reasoning, not a low-latency system, and that's fine for a swing horizon
of days. But it means the price read during Tier 3 confirmation
(`signal_price`) can be stale by the time a trade is actually about to be
simulated. Two rules to keep this honest:

1. **Always re-fetch a fresh quote** (`get_equity_quotes`/
   `get_option_quotes`) immediately before simulating a fill in step 6 of
   `run_instructions.md` — call this `entry_price`. Never reuse a quote
   pulled earlier in the funnel as the simulated fill price.
   **Fill-price realism:** for options, simulate buys at
   `high_fill_rate_buy_price` and sells at `high_fill_rate_sell_price`
   (both returned by `get_option_quotes`), not at `mark_price` — mark is
   the midpoint and systematically flatters paper P&L versus what a real
   order actually fills at. For equity, use the quote price but treat it
   the same way: never assume a better-than-midpoint fill.
2. **Chase-protection**: if `entry_price` has already moved more than
   `0.5 × stop_distance` (see sizing below) beyond `signal_price` in the
   favorable direction since Tier 3 confirmed the setup, **do not chase
   it** — drop the trade this run and log it under "Rejected by gates" as
   a chase-protection skip, not a data/gate failure. The setup can
   requalify on a later run if it's still valid then. This is exactly the
   protection against "the stock already jumped too high by the time we'd
   execute" — better to skip a trade than enter at a materially worse
   reward:risk than what was actually evaluated.
   Moving in the *unfavorable* direction isn't a rejection — just size
   and set the stop/R-target off the real `entry_price`, not the stale
   `signal_price`.

## Position sizing (equity) — risk-based, not equal-weight

1. `stop_distance = min(1.0-1.5 × ATR14, entry_price × gates.md stop-loss
   floor)` — always the *tighter* of the ATR-derived stop and the hard
   ceiling, computed off the fresh `entry_price` above, not `signal_price`.
   Use 1.5x by default; 1.0x is the tighter end of the range used by
   contemporary breakout swing traders (see `framework.md`, Qullamaggie)
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
- **Tool flow is three steps** (verified 2026-08-10, see
  `tool_verification.md`): `get_option_chains` (chain id + expirations) →
  `get_option_instruments` (contract UUIDs, filtered by expiration/type/
  strike) → `get_option_quotes` (delta, open_interest, bid/ask, mark,
  Greeks). Chains carry no contracts — the middle step isn't skippable.
  Every field the options gates need is confirmed present in the quote
  response, so no gate here is aspirational.
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
2. Price closes back below SMA(50) (trend template broken —
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
