# Framework — Swing Trading Methodology

This document is the "why" behind `strategy.md`. It exists so that when the
agent (or you) adjusts `strategy.md` over time, the changes stay grounded in
an actual, named, publicly-documented approach — not ad hoc indicator
tweaking. If you can't point to which section of this document a strategy
change relates to, it's probably not a change to make.

## Style: swing trading, not day trading

Holding period target: **~5–15 trading days**, hard time-stop at **20
trading days**. Three runs a day — 9:30am (open), 2:30pm (midday, five
hours in), and 3:30pm (pre-close) — all inside regular hours, using
live regular-session prices rather than a stale or thin premarket quote,
and never after the close where a decision could not actually be filled.
These are for monitoring and new-signal generation, not intraday
scalping; nothing here is designed for sub-daily timeframes.

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

   **Why specifically the 52-week high** (the level `strategy.md`'s
   breakout trigger uses): at a 52-week high, essentially no one who
   bought in the past year is underwater, so there is no reservoir of
   trapped holders selling into strength just to get back to even —
   overhead supply is thin. This is Tier A evidence, not just
   practitioner lore: **George & Hwang (2004), "The 52-Week High and
   Momentum Investing," *Journal of Finance*** found that nearness to the
   52-week high predicts future returns *better than past returns
   themselves*, making it one of the more robust momentum variants. The
   2% window means the breakout is imminent or underway rather than
   already extended.

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

Every candidate is tested against both a bullish and a bearish template
on every run — so a broken company can be identified inside a healthy
market, and vice versa. The SPY regime doesn't decide which direction
gets examined; it decides which passes count as **counter-trend** and
therefore face much stricter gates (one position maximum, half size,
higher ADX bar, and no trading on degraded data).

That split reflects a genuine tension in the evidence. Cross-sectional
momentum (Jegadeesh & Titman) is inherently long *and* short at once, so
purely regime-gated trading leaves documented edge on the table. But that
research operates across hundreds of names per side; a six-slot book
taking a concentrated bet against the market's structural upward drift is
a different and worse proposition. Allowing counter-trend trades but
capping them at one, at half size, is the compromise — and the
`counter_trend` column in `trade_log.csv` exists specifically so this
allowance can be judged on its own record later rather than on argument.

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

## How well-established is this, really?

An honest audit, because "proven framework used by quants" conflates two
very different tiers of evidence.

### Tier A — robustly documented in peer-reviewed finance
These are among the most replicated findings in empirical finance, and
real quantitative funds do build on them:

- **Cross-sectional momentum** — Jegadeesh & Titman (1993). Replicated
  across decades, countries, and asset classes; added to the standard
  asset-pricing model as the fourth factor (Carhart, 1997).
- **Time-series momentum / trend following** — Moskowitz, Ooi & Pedersen
  (2012), across 58 instruments and 25 years.
- **Value, and especially value+momentum together** — Fama & French
  (1992); Asness, Moskowitz & Pedersen (2013), "Value and Momentum
  Everywhere," *Journal of Finance*.
- **Post-earnings-announcement drift** — Bernard & Thomas (1989).
- **Volatility-scaled position sizing** — standard practice across
  managed-futures/CTA funds; uncontroversial.

### Tier B — practitioner frameworks with real but weaker evidence
This is where Minervini sits, and the distinction matters:

- **Minervini's trend template / SEPA** — Mark Minervini is a real trader
  with verified competition results (1997 and 2021 U.S. Investing
  Championship). But he is a **discretionary** trader, not a quant. SEPA
  is a practitioner heuristic popularized through his books; it is **not**
  in the academic literature and is **not** what systematic funds run.
  What it *is*: a sensible, concrete operationalization of trend and
  momentum — Tier A ideas — into checkable rules.
- **Turtle Trading** — genuine documented results, but from 1980s
  **futures** markets, not equities, and widely believed to have degraded.
- **O'Neil / CANSLIM** — decades of use; independent validation is mixed.
- **Connors' RSI work** — backtested in his books, but over limited
  periods and criticized as prone to overfitting.
- **Qullamaggie** — a genuinely verified public track record, but n=1, in
  a particular era, concentrated in high-momentum names.

### The honest characterization
This system is **a practitioner-flavored implementation of Tier A
factors** — not a replication of what a quant fund does. Real systematic
equity shops differ from this in nearly every operational respect: they
trade universes of thousands of names rather than a top-15 slice, derive
thresholds statistically instead of from round numbers (RSI 35-45, ADX
25, 25% of the 52-week high are all conventions, not optimized
parameters), run formal risk models and transaction-cost models, and
diversify across many small positions rather than concentrating in six.

### The short side is mirrored mechanically, but not evidentially

`strategy.md` mirrors every long rule for shorts — 52-week low replaces
52-week high, the range-position test inverts, breakdown replaces
breakout. The *mechanism* mirrors cleanly, and in one respect is stronger:
at a 52-week low everyone who bought in the past year is underwater, so
rallies meet motivated sellers escaping at break-even — more forceful
than the mere absence of sellers near a high.

But the **evidence** does not mirror, and the rules should not pretend it
does:

1. **Momentum's short leg is historically weaker than its long leg** —
   most of the documented winner-minus-loser spread comes from the
   winners.
2. **Stocks near 52-week lows include distressed names** whose payoffs
   are lottery-like. A refinancing, a buyout rumor, or a short squeeze
   can produce a violent upward move. The tail risk is fat and points
   *against* a short.
3. **The equity risk premium works against shorts.** Stocks drift upward
   over time, so a short position fights a structural tailwind that a
   long position rides.

This is why the system already leans against shorts structurally —
counter-trend gates whenever the regime is long, no outright short
selling (defined-risk inverse ETFs only), and a bear universe naturally
limited to ~13 names with liquid inverse products. Those are not
oversights to be "fixed" into symmetry; they are the asymmetry being
respected.

### Four caveats that genuinely bite
1. **Survivorship bias in the practitioner sources.** Minervini and
   Qullamaggie are known *because* they won. The many traders who used
   similar methods and didn't are invisible. Their records are evidence
   these methods *can* work, not that they work on average.
2. **Published anomalies decay.** McLean & Pontiff (2016) found returns
   to documented anomalies fall roughly 58% post-publication. Everything
   cited here is published.
3. **Momentum crashes — the single most important risk here.** Momentum's
   failure mode is not gentle. Daniel & Moskowitz, *Momentum Crashes*
   (Journal of Financial Economics), documents that these losses are not
   random: they cluster in "panic states — following market declines and
   when market volatility is high — and are contemporaneous with market
   rebounds." Momentum performs worst **not during the bear market but
   when it ends**. After a major decline, past-loser betas rise above 3
   while past-winner betas fall below 0.5, so a momentum book carries a
   large *negative* conditional beta into the recovery and is run over by
   it.

   This is why the risk level in `strategy.md` halts new entries when
   high VIX coincides with a prior decline. It is also the sharpest
   answer to an intuitive objection — high VIX marks fear, fear marks
   lows, so shouldn't we buy? For *the index*, the evidence says yes
   (SPX bought above VIX 30 has averaged ~23% one-year returns). For a
   *momentum strategy*, the same moment is the documented worst entry.
   The claims are about different strategies, not in conflict.

   Encouragingly, the paper finds these crashes are **partly
   forecastable**, and that a dynamic momentum strategy conditioned on
   forecast mean and variance roughly doubles static momentum's alpha and
   Sharpe. The risk-level layer is a crude version of that idea; a proper
   implementation would be a real upgrade.
4. **None of it is validated *here*.** The strongest caveat: no backtest
   of this specific synthesis, on this universe, with these thresholds,
   exists. Tier A evidence supports the ingredients; it says nothing
   about this recipe.

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
- **Paper P&L omits five real costs**, all of which bias it optimistic:
  fill certainty (limits are assumed to always fill at the quote),
  slippage beyond the spread, options contract fees, leveraged ETF
  expense ratios (~0.9%/yr), and dividends. The bid-ask spread itself
  *is* charged. See `DATA_SCHEMA.md` for the full accounting. Margin
  interest is moot — margin is disabled.
- **Universe is now the Tier 0 screener** (~96 names at last check), not
  the old hand-picked list — a real improvement, though the funnel still
  only carries the top 15 into Tier 1 for wall-clock reasons, so the
  cross-sectional ranking is computed over a narrow slice rather than the
  full market.
