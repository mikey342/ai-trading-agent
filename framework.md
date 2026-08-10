# Framework — Swing Trading Methodology

This document is the "why" behind `strategy.md`. It exists so that when the
agent (or you) adjusts `strategy.md` over time, the changes stay grounded in
an actual, named, publicly-documented approach — not ad hoc indicator
tweaking. If you can't point to which section of this document a strategy
change relates to, it's probably not a change to make.

## Style: swing trading, not day trading

Holding period target: **~5–15 trading days**, hard time-stop at **20
trading days**. Two runs a day (premarket + close) are for monitoring and
new-signal generation, not intraday scalping — nothing here is designed for
sub-daily timeframes.

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

This isn't from a textbook — it's a practical constraint. Some Alpha
Vantage endpoints (`RSI`, `MACD`, `EMA`, `SMA`, weekly/monthly time series)
return **entire multi-year histories with no compact option**, saved to a
file rather than returned inline, because the payload is too large for a
single response (60–100k+ characters observed for a single symbol). Calling
these for a large universe on every run would be both slow and likely to
hit API rate limits. So the workflow is explicitly tiered — see
`run_instructions.md` for the exact sequence:

- **Tier 1 (full universe, cheap):** `COMPANY_OVERVIEW` + `GLOBAL_QUOTE`
  per candidate. This alone gives trend (50DMA/200DMA are already fields in
  `COMPANY_OVERVIEW` — no separate SMA call needed), valuation, quality,
  growth, and current price. Eliminate anything failing the trend template
  here before spending any more calls on it.
- **Tier 2 (shortlist only, medium cost):** `RSI` and `EMA` (8/21) for
  survivors of tier 1, extracting only the most recent value from the saved
  file (never read the whole file — see `run_instructions.md`). Confirms
  entry timing.
- **Tier 3 (finalists only, most expensive):** `MACD` for final momentum
  confirmation, `NEWS_SENTIMENT` for the catalyst check, and — only if an
  options expression is being considered — Robinhood `get_option_chains` /
  `get_option_quotes` for strikes/liquidity.

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

## Options and leverage: same signal, different instrument

Options and leverage are **not a separate strategy** — they're an alternate
way to express the exact same equity signal from this framework, used only
when they make the trade more capital-efficient without changing its risk
character. Concretely:

- Only **defined-risk** structures are ever used: long calls, long puts,
  and vertical debit spreads. Max loss is always known and capped at the
  premium paid at entry. Naked/undefined-risk options (short calls, short
  puts, uncovered spreads) are explicitly out of scope — see `gates.md`.
- Leverage (margin) is capped at a low ratio (see `gates.md`) applied on
  top of the same equity signal — it doesn't change what's being traded,
  just the funding of it.
- Neither options nor leverage removes the need for the stop-loss/time-stop
  discipline in `strategy.md` — if anything, theta decay makes the time-stop
  *more* binding for options than for the equivalent stock position.

## What's deliberately not here (yet)

- **No backtest.** Every methodology cited above has a published track
  record *in its original form, on its original universe and era*. None of
  that validates this specific synthesis, on this specific ~15–20 stock
  universe, using Alpha Vantage's data, starting now. Recommended next step
  before trusting this beyond paper trading.
- **No options Greeks modeling.** Delta is used as a rough strike-selection
  heuristic (see `strategy.md`), but theta/vega/gamma exposure isn't
  tracked or limited beyond the DTE floor and defined-risk-only rule.
- **No transaction costs, slippage, or margin interest** in the paper P&L.
- **Small, hand-picked universe** (~15–20 large/liquid names), not a real
  cross-sectional universe (e.g., full S&P 500) — constrained by API
  budget. Relative strength and trend-template filtering are directionally
  useful here but statistically noisier than they'd be on a larger universe.
