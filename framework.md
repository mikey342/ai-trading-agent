# Framework — Swing Trading Methodology

This document is the "why" behind `strategy.md`. It exists so that when the
agent (or you) adjusts `strategy.md` over time, the changes stay grounded in
an actual, named, publicly-documented approach — not ad hoc indicator
tweaking. If you can't point to which section of this document a strategy
change relates to, it's probably not a change to make.

## Style: swing trading, not day trading

Holding period target: **~5–15 trading days**, hard time-stop at **20
trading days**. Three runs a day (morning at market open, midday 5 hours
later, close — all using real, live regular-session prices, never a
stale or thin premarket quote) are for monitoring and new-signal
generation, not intraday scalping — nothing here is designed for
sub-daily timeframes.

## Sources checked (2026-08-09 research pass)

Beyond the five approaches below, a live research pass confirmed rather
than overturned the direction, and added two refinements:

- **O'Shaughnessy, *What Works on Wall Street*** — backtested 1954-2003:
  a combined Value + Momentum strategy returned ~17-18% CAGR vs. the
  S&P 500's ~11.5%, beating the market in 100% of rolling 10-year periods.
  Value alone and momentum alone both underperform the combination. This
  is why Tier 1 now includes a value/quality tiebreaker, not just trend —
  see `strategy.md`.
- **AQR, "Value and Momentum Everywhere" (Asness, Moskowitz, Pedersen)** —
  the same value+momentum combination effect holds across 8 asset classes
  and markets, not just US equities — evidence it's a real, persistent
  effect rather than a data-mined fluke in one dataset.
- **Qullamaggie (Kristjan Kullamägi)** — a contemporary breakout swing
  trader with a large, public, verifiable track record. Two things
  borrowed directly: (1) stop-loss sized off a volatility measure (average
  daily range, closely analogous to the ATR this system already uses),
  risking 0.25-1% of account per trade — validates the risk-per-trade
  sizing already in `strategy.md`; (2) a win rate as low as 20-35% can
  still be strongly profitable if winners are allowed to run via a
  trailing stop rather than exited at a fixed target — this is why
  `strategy.md`'s exit rules now prefer a trailing EMA stop over a fixed
  2R target once a trade is working.
- **Options practitioner consensus** (multiple sources) — 21 days-to-
  expiration is the standard checkpoint to force a close/roll decision on
  any options position, since gamma risk accelerates as expiration nears.
  Added to `strategy.md`'s options exit rules alongside the existing DTE-
  decay rule.

## The five approaches this system draws from

These are real, publicly documented systematic/discretionary methodologies
with track records — not proprietary secrets. `strategy.md` is a synthesis
of pieces from each, not a pure implementation of any one:

1. **Trend-following breakout (Turtle Trading / Donchian channels)** —
   Richard Dennis & William Eckhardt's "Turtle" system (1983, later
   published in detail by Curtis Faith). Enter on a breakout above a recent
   N-day high, size positions using ATR ("N" in Turtle terminology) so every
   position risks a similar dollar amount, exit on an ATR-based trailing
   stop. The core idea we borrow: **volatility-normalized position sizing
   and stops**, not fixed percentages.

2. **Trend template / stage analysis (Minervini's SEPA)** — Mark
   Minervini's published methodology (*Trade Like a Stock Market Wizard*):
   a stock must be in a confirmed uptrend (price > 50DMA > 150/200DMA, DMAs
   themselves rising, price within ~25% of its 52-week high) before it's
   even considered. We borrow the **multi-moving-average "stack" as a
   trend-quality filter**, applied before looking at any entry trigger.

3. **Relative strength / CANSLIM** — William O'Neil's methodology (*How to
   Make Money in Stocks*): rank candidates by price strength *relative to
   the market*, not in isolation, and prefer stocks already leading the
   market during the current move. We borrow **relative strength vs. SPY**
   as a required filter — a stock can look technically fine in isolation
   and still be a laggard.

4. **Mean-reversion-within-trend (Connors RSI pullback)** — Larry Connors'
   published, backtested research (*Short Term Trading Strategies That
   Work*): within an established uptrend, buy short-term oversold
   conditions (RSI pullbacks) rather than chasing strength, because pullback
   entries in a real uptrend have a statistically better risk/reward than
   breakout entries. We borrow **RSI pullback as one of two valid entry
   triggers** (the other being a breakout, per Turtle-style).

5. **Post-earnings-announcement drift (PEAD)** — this one is peer-reviewed
   academic finance, not a practitioner book: Bernard & Thomas (1989) showed
   stocks that beat earnings estimates continue drifting up for weeks
   afterward, and the market underreacts to the surprise. We borrow this as
   the **catalyst/news layer**: a recent earnings beat + non-negative news
   sentiment is a tailwind, not just a "don't trade into an unknown," and an
   upcoming (not-yet-reported) earnings date is treated as event risk to
   avoid, per the earnings blackout gate.

## The funnel: cheap screen → expensive confirmation

This isn't from a textbook — it's a practical constraint, though the
specifics changed on 2026-08-10 (see below). The general principle stays:
running expensive analysis on a full universe every run is wasteful and
budget-risky, so the workflow is explicitly tiered — see
`run_instructions.md` for the exact sequence:

- **Tier 1 (full universe, cheap):** batched `get_equity_quotes` +
  `get_equity_fundamentals` for price, valuation, and 52-week range, plus
  per-symbol `get_equity_technical_indicators` (sma, 50 and 200) for
  trend. Eliminate anything failing the trend template here before
  spending any more calls on it.
- **Tier 2 (shortlist only):** `get_equity_technical_indicators` for RSI
  and EMA (8/21). Confirms entry timing.
- **Tier 3 (finalists only, most expensive):** EMA8/EMA21 spread (reused
  from Tier 2, no new call) for final momentum confirmation,
  `NEWS_SENTIMENT` (Alpha Vantage) for the catalyst check,
  `get_earnings_results` for the earnings blackout, and — only if an
  options expression is being considered — Robinhood
  `get_option_chains`/`get_option_quotes` for strikes/liquidity.

**Update, 2026-08-10:** live testing found Robinhood's own data tools
(`get_equity_quotes`, `get_equity_fundamentals`, `get_equity_technical_
indicators`, `get_earnings_results`) cover almost all of the above, with
no observed daily cap (unlike Alpha Vantage's hard 25/day) and a built-in
`output: "latest"`/`"last:N"` trim server-side — unlike Alpha Vantage's
indicator endpoints (`RSI`, `MACD`, `EMA`, `SMA`, weekly/monthly time
series), which return entire multi-year histories with no compact option
(60-100k+ characters for a single symbol, confirmed live). So the tiers
above are now Robinhood-primary; Alpha Vantage is reserved for
`NEWS_SENTIMENT` (Robinhood has no news/sentiment equivalent) and
occasional deliberate deep-dives (institutional/insider holdings,
transcripts) — never a full-universe pull. `MACD` on Alpha Vantage is
also permanently premium-gated on the current plan (confirmed via a live
run, not a transient limit); Robinhood's own `macd` indicator type is a
working substitute if a second momentum check is ever wanted, though the
EMA-spread check already covers this need. See `strategy.md`'s "Data
sources" table for the exact tool-by-tool mapping.

This mirrors how real systematic desks actually operate under compute/data
budgets: broad cheap screens first, expensive analysis only on what
survives.

## Regime filter: don't fight the tape

New long entries (equity or options) require SPY above its 200-day SMA.
This is a simple, well-documented trend/regime filter (closely related to
Mebane Faber's published tactical asset allocation research) — momentum and
breakout systems specifically perform worse in broad downtrends. This lives
in `gates.md`, not here, because it's a hard portfolio-level control, not a
tunable preference.

## Direction: long and short, via cash-bought instruments only

The regime filter sets direction rather than merely gating activity: SPY
above its 200-day SMA → LONG mode; below → SHORT mode. This lets the
system stay useful in a downtrend instead of sitting idle for months.

Two hard constraints shape *how* direction is expressed: **no margin and
no short selling, ever.** Both introduce failure modes an unattended
system checking positions a few times a day cannot manage — unbounded
loss and squeezes on the short side, forced liquidation on the margin
side. Instead, bearish exposure is a *long* position in a 2x inverse
index ETF (SDS/QID), and leveraged bullish exposure is a long position in
a 2x ETF (SSO/QLD). Both are bought with cash; max loss is the amount
paid.

The cost of that safety is **volatility decay**: these funds target 2x
the *daily* return and reset each day, so a flat round-trip in the index
still loses money (index +10% then −9.09% → 2x fund −1.8%). They reward
sustained trends and punish chop, which is why the index sleeve only
fires with trend confirmation present and carries a 10-day time-stop
rather than 20.

## Options and leverage: same signal, different instrument

Options and leverage are **not a separate strategy** — they're an alternate
way to express the exact same equity signal from this framework, used only
when they make the trade more capital-efficient without changing its risk
character. Concretely:

- Only **defined-risk** structures are ever used: long calls, long puts,
  and vertical debit spreads. Max loss is always known and capped at the
  premium paid at entry. Naked/undefined-risk options (short calls, short
  puts, uncovered spreads) are explicitly out of scope — see `gates.md`.
- Leverage never comes from margin (disabled outright) — only from
  *buying* a 2x ETF with cash, as described in the section above.
- Neither options nor leveraged ETFs remove the need for the
  stop-loss/time-stop discipline in `strategy.md` — if anything, theta
  decay (options) and daily-reset volatility decay (leveraged ETFs) make
  the time-stop *more* binding than for a plain stock position.

## What's deliberately not here (yet)

- **No backtest.** Every methodology cited above has a published track
  record *in its original form, on its original universe and era*. None of
  that validates this specific synthesis, on this universe, with this
  data, starting now. Recommended next step before trusting this beyond
  paper trading — `get_equity_historicals` (20 years of OHLCV) is the
  path, see `DATA_SCHEMA.md`.
- **No options Greeks modeling.** Delta is used as a rough strike-selection
  heuristic (see `strategy.md`), but theta/vega/gamma exposure isn't
  tracked or limited beyond the DTE floor and defined-risk-only rule.
- **No modeling of leveraged-ETF decay.** The 10-day time-stop is a blunt
  guard against it, not a model of it. Realized decay depends on the
  actual volatility path and isn't projected anywhere.
- **No transaction costs or slippage** in the paper P&L (margin interest
  is moot — margin is disabled). Leveraged ETF expense ratios (~0.9%/yr)
  are also unmodeled.
- **Universe is now the Tier 0 screener** (~96 names at last check), not
  the old hand-picked list — a real improvement, though the funnel still
  only carries the top 15 into Tier 1 for wall-clock reasons, so the
  cross-sectional ranking is computed over a narrow slice rather than the
  full market.
