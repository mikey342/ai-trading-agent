# Strategy — Adaptive Swing Trading Playbook

This file operationalizes `framework.md` into concrete, checkable rules.
The agent **may edit this file** as it learns from `trade_journal.md`, but
every edit must be small, specific, and logged in the Changelog with a
cited rationale — never a wholesale rewrite in one run. See the
Adaptation policy at the bottom.

`gates.md` always wins in a conflict. This file never overrides a hard
limit.

## Universe — Tier 0: market-wide screener

Benchmark/regime reference: **SPY** (not itself traded — used for the
regime filter and as the relative-strength baseline).

Candidates come from a **saved Robinhood screener**, not a hardcoded
list:

- **scan_id:** `de1b1994-b5db-472a-9b79-c052f1215193`
  ("Swing Agent - Trend Candidates")
- **Call:** `run_scan` with that id — **one call**, returns live results
  with `Last`, `Market cap`, `RSI`, `Average directional index (14)`,
  `Average volume`, `Relative volume`, `% Change` already populated per
  row. No extra per-symbol calls needed at this stage.
- **Filters:** market cap > $2B, price > $5 (matches `gates.md`'s
  penny-stock exclusion), 30-day average volume > 1M shares (liquidity),
  **ADX(14) > 25** (a real trend exists — direction-agnostic, so this
  serves both long and short modes), RSI(14) between 25-75 (deliberately
  wide enough to surface oversold short-mode candidates as well as
  long-mode pullbacks), **relative options volume > 0.5** (ensures the
  name has a live options market — required for the options overlay —
  and surfaces the `Relative options volume` column for ranking).
- **Verified live 2026-08-10:** 96 matches under this filter set.

**Why this replaced the old hardcoded ~19-name watchlist:** verified live
2026-08-10, the screen returned **266 qualifying names**, and most of the
old watchlist (AAPL, MSFT, NVDA, GOOGL, META, JPM) **failed** it — ADX
below 25, i.e. not actually trending — while it surfaced names never on
the list (PANW, CRWD, FTNT, NET, NU...). A hand-picked mega-cap list was
simultaneously too narrow to find real leaders and padded with names that
didn't meet the strategy's own criteria. It also made the
relative-strength ranking nearly meaningless: ranking 19 pre-selected
mega-caps against each other is not a cross-sectional signal.

**Narrowing for the funnel:** ~96 is far more than Tier 1 can process.
Rank the scan results client-side (free — the scan already returned these
columns) by a blend of `Average directional index (14)` (trend strength)
and `Relative options volume` (unusual positioning — a public,
exchange-derived early-interest signal), then take the **top 15** into
Tier 1. Weight ADX primarily; use relative options volume as the
tiebreaker rather than the main sort, since unusual options activity is
suggestive, not directional on its own — it can precede a move either
way, and it is not a substitute for the trend template. If the scan
returns fewer than 15, use them all. Log the scan's total match count in
the journal so the universe's breadth on a given day is visible.

If `run_scan` fails, fall back to the previous behavior (screen a small
diversified set of liquid large-caps) and say so explicitly in the
journal — but treat that as degraded operation, not normal.

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

## Direction mode — set once per run, before Tier 1

The regime filter decides **which direction the run hunts in**, rather
than simply gating whether it trades at all:

**Every candidate is evaluated against *both* templates, every run.** A
stock cannot pass both (they are mutually exclusive — price cannot be
simultaneously above and below its own moving-average stack), and most
pass neither, which is correct. What the SPY regime determines is not
*whether* a direction is examined, but how much freedom a setup gets once
it passes:

| Setup direction vs. SPY regime | Classification | Treatment |
|---|---|---|
| Bullish setup while SPY **>** SMA(200) | **with-trend** | Normal rules |
| Bearish setup while SPY **<** SMA(200) | **with-trend** | Normal rules |
| Bearish setup while SPY **>** SMA(200) | **counter-trend** | Strict extra gates below |
| Bullish setup while SPY **<** SMA(200) | **counter-trend** | Strict extra gates below |

Rationale for allowing counter-trend at all: classic cross-sectional
momentum (Jegadeesh & Titman — Tier A evidence, see `framework.md`) is
inherently long *and* short simultaneously, and genuinely broken stocks
exist in bull markets. Rationale for constraining it hard: that research
assumes hundreds of names per side, while this book has six slots. A
concentrated counter-trend bet is not the diversified factor the research
validates, and shorting into a market with a structural upward drift has
negative expected value on average.

### Counter-trend gates (all required, on top of every normal check)

1. **At most one counter-trend position open at a time**, out of the
   6-position book.
2. **ADX(14) > 30** (versus the baseline 25) — the counter-direction
   trend must be unusually well established, not marginal.
3. **A more extreme relative-strength reading**: ≤ **0.25** of the
   52-week range for a counter-trend bearish setup (vs. 0.4 normally),
   ≥ **0.75** for a counter-trend bullish setup (vs. 0.6).
4. **No degraded data.** Every Tier 3 check must return real values —
   in particular, `news_sentiment` must **not** be `UNAVAILABLE`. If the
   Alpha Vantage quota is exhausted, no counter-trend trade is taken that
   run, full stop. A counter-trend bet made blind to news is exactly the
   trade most likely to be on the wrong side of a catalyst.
5. **Half size**: `risk_per_trade = 0.5% of NAV` rather than 1%.

If any of these fails, the setup is not downgraded to a normal trade —
it is dropped, and logged as a counter-trend rejection.

### The index sleeve is never counter-trend

The 2x index ETFs (SSO/QLD/SDS/QID) always follow the regime: long-side
only while SPY is above its SMA(200), inverse-side only while below.
Buying SDS in a confirmed uptrend is a bet against the primary trend
itself, which is a categorically worse proposition than identifying one
broken company inside a healthy market. Counter-trend applies to
individual stocks — idiosyncratic breakdowns — never to the index.

**Never short a stock, and never use margin — in any mode, for any
reason.** Both are disabled outright in `gates.md`. Short selling has
unbounded loss (a squeeze can run indefinitely) plus borrow cost and
recall risk, and a system that reviews positions a few times a day cannot
manage that. Bearish exposure is taken **only** by *buying* an inverse
ETF with cash, where max loss is the amount paid. If a qualifying name has
no sufficiently liquid bear ETF, take no trade on it; do not substitute a
different instrument or reach for a thin one.

Both modes use identical machinery — trend template, entry triggers, ATR
sizing, R-targets, trailing stops — just mirrored. Everything below is
written for LONG mode; the SHORT mode mirror is stated alongside each
rule. The mirrored Tier 1/2 rules below apply to the **index** (SPY/QQQ)
in SHORT mode, since that's the only thing being traded then.

## Tier 1 — Trend template (top 15 from the Tier 0 scan)

Pull `get_equity_quotes` (all 15 in one batched call) +
`get_equity_fundamentals` (batched, 10 per call — 2 calls). Then, per
candidate, `get_equity_technical_indicators` (type=sma, period=50,
output=latest) and (type=sma, period=200, output=latest) — these are
one-symbol-per-call and are the expensive part of Tier 1 (~30 calls for
15 names). Robinhood has shown no daily cap, but each call costs wall-clock
time, which is why Tier 0 narrows to 15 first. A candidate survives Tier 1
only if **all** of (Minervini-style trend template, adapted to available
fields):

**LONG mode:**
1. Current price > SMA(50) > SMA(200)
2. Current price within 25% of `high_52_weeks`
3. Current price at least 25% above `low_52_weeks`
4. Relative-strength proxy: `(price - low_52_weeks) / (high_52_weeks -
   low_52_weeks)` ≥ 0.6 (i.e., trading in the upper 40% of its 52-week
   range — a cheap stand-in for a true relative-strength-vs-SPY line,
   which would need a full historical series for every candidate; see
   `framework.md` limitations)

**SHORT mode (exact mirror — a confirmed *downtrend*, not merely a
long-mode failure):**
1. Current price < SMA(50) < SMA(200)
2. Current price within 25% of `low_52_weeks`
3. Current price at least 25% below `high_52_weeks`
4. Relative-weakness proxy: `(price - low_52_weeks) / (high_52_weeks -
   low_52_weeks)` ≤ 0.4 (lower 40% of its 52-week range)

**Earnings blackout runs here, not at Tier 3.** Make **one**
`get_earnings_calendar` call (Robinhood, `days=7`, no market-cap filter)
at the start of Tier 1 and drop any candidate reporting within the next 3
calendar days. This replaces the old per-finalist `get_earnings_results`
blackout check: one market-wide call covers everything, and filtering here
means no expensive Tier 2/3 work is ever spent on a name about to report.
`get_earnings_results` is still called at Tier 3 for finalists, but now
only for its 8-quarter actual-vs-estimate surprise history (the PEAD
catalyst read), not for the blackout.

**Test every candidate against both templates.** A name that merely fails
the long template does **not** thereby qualify as a short — it must
independently pass the full short template. Most candidates will qualify
for neither, which is correct. Any that passes is then classified
with-trend or counter-trend per the table above, and counter-trend
setups must additionally clear all five counter-trend gates.

(The SPY regime check that sets the mode happens once per run, before
Tier 1 — see "Direction mode" above and step 1 of `run_instructions.md`.)

### Value tiebreaker and Tier 2 cap

Published research (O'Shaughnessy's *What Works on Wall Street*, AQR's
"Value and Momentum Everywhere" — see `framework.md`) finds that momentum
combined with value outperforms momentum alone. So among Tier 1 survivors,
rank by a composite of `0.6 × relative-strength proxy (from #4 above) +
0.4 × (1 / pe_ratio, ranked cross-sectionally within this run's candidate
set — lower P/E ranks higher)` (from `get_equity_fundamentals`; Robinhood
doesn't expose PEG in this endpoint, so this uses raw P/E relative to
peers in the same run rather than growth-adjusted P/E — a cruder value
signal, noted as a limitation, not silently upgraded to something it
isn't). This is a prioritization, not an additional pass/fail gate — it
decides which survivors advance to the more expensive Tier 2, capped at
the **top 8**.

**In SHORT mode, invert both terms**: rank by `0.6 × (1 −
relative-strength proxy) + 0.4 × (pe_ratio ranked cross-sectionally —
*higher* P/E ranks higher)`. The same research logic runs in reverse:
expensive, weak names are the better short candidates, just as cheap,
strong ones are the better longs.

## Tier 2 — Entry trigger (shortlist from Tier 1 only)

Pull `get_equity_technical_indicators` for RSI (period=14, output="last:5")
and EMA (period=8 and period=21, output="last:5") for Tier 1 survivors —
`output` trims the response server-side, no extraction step needed. A
candidate qualifies if **either**:

**LONG mode:**
- **Breakout trigger** (Turtle-style): price within 2% of `high_52_weeks`
  AND EMA8 > EMA21
- **Pullback trigger** (Connors-style): RSI(14) between 35-45 AND price
  still > SMA(50) (pullback within an intact uptrend, not a breakdown)
  AND EMA21 is flat-to-rising over the last 5 values (trend not rolling
  over)

**SHORT mode (mirrored):**
- **Breakdown trigger**: price within 2% of `low_52_weeks` AND EMA8 <
  EMA21
- **Rally-to-resistance trigger**: RSI(14) between 55-65 AND price still
  < SMA(50) (a bounce inside an intact downtrend, not a genuine reversal)
  AND EMA21 is flat-to-falling over the last 5 values

## Tier 3 — Confirmation (finalists only, cap at 3 per run per gates.md)

For candidates passing Tier 2:

1. **Momentum confirmation**: the EMA8/EMA21 spread (already pulled in
   Tier 2 — no new call needed) must show momentum *accelerating*, not
   merely present.

   **This test applies to BREAKOUT entries only — never to pullback
   entries.** For a breakout, compute `|EMA8 − EMA21|` for each of the
   last 5 sessions; the most recent value must be strictly greater than
   all four prior values. Anything else fails.

   **For a pullback entry, this test is skipped entirely, and running it
   would be a logic error.** A pullback *is* a narrowing spread — EMA8
   falling back toward EMA21 is the definition of the setup — so
   requiring the spread to be at a 5-session maximum would reject every
   pullback that ever occurs. The equivalent confirmation for a pullback
   is already performed at Tier 2: EMA21 must be flat-to-rising over its
   last 5 values, which is what establishes the trend is intact rather
   than rolling over. Confirmed as a real defect on 2026-08-10, when the
   strict test rejected 4 of 4 Tier 2 survivors and would have
   permanently blocked the pullback path had one fired.

   This wording replaces an earlier "spread is widening over its last 5
   values," which was ambiguous enough to be unusable: on real BAC data
   (1.093 → 1.133 → 1.184 → 1.163 → 1.147) a *net* reading passes
   (+0.054 across the window) while a *trend* reading fails (two
   consecutive narrowing sessions). Two runs could reach opposite
   conclusions from identical numbers, which disqualifies it as a
   mechanical rule. The stricter reading is the deliberate choice: a
   breakout whose momentum is already decelerating is exactly the one to
   decline. On that BAC data this test **fails**, and the trade is
   dropped.

   Direction still matters for the sign: in LONG mode EMA8 must be above
   EMA21, in SHORT mode below — the magnitude test above applies to the
   absolute spread once that direction check passes.

   `MACD` (Alpha Vantage) was the
   original design for this check but is **permanently premium-gated on
   the current Alpha Vantage plan** (confirmed live, not a transient
   rate-limit — see `gates.md`). Robinhood's `get_equity_technical_
   indicators` (type=macd) is a working substitute if a second, redundant
   momentum check is ever wanted — but the EMA-spread check alone is
   sufficient and must not be treated as incomplete confirmation.
2. `NEWS_SENTIMENT` (Alpha Vantage, limit 5-10, this symbol): in LONG
   mode, sentiment not negative; in SHORT mode, mirrored — sentiment not
   *positive* (don't short into good news). The one step in this whole
   funnel that still needs Alpha Vantage — Robinhood has no news tool.
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
3. `get_earnings_results` (Robinhood, this symbol): 8 quarters of
   actual-vs-estimate EPS — a consistent beat streak is a soft positive
   for the PEAD catalyst read, a miss streak a soft negative. (The
   earnings *blackout* already ran at Tier 1 via one market-wide
   `get_earnings_calendar` call, so this is purely the surprise-history
   read.)
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
   order actually fills at. For equity and ETFs, simulate **buys at the
   `ask_price` and sells at the `bid_price`** from `get_equity_quotes` —
   never at `last_trade_price` or the midpoint. This bakes the full
   bid-ask spread into every round trip, which is a real cost the system
   would otherwise silently ignore.

## Order entry — limit orders, never market

Fills are simulated today, but the simulation must reflect how the order
would actually be placed, or the paper record measures a strategy nobody
could execute.

**Always a marketable limit order. Never a market order.** A market order
surrenders all price control and, in a fast or thin market, fills far
from the last print. A marketable limit fills essentially as quickly
while capping the worst price accepted:

- **Buying:** limit at the current **ask**. Marketable, so it fills
  immediately in normal conditions, but it cannot fill above that price
  if the market jumps mid-order.
- **Selling:** limit at the current **bid**, same logic inverted.
- **Never chase a partial fill** by re-submitting higher. If the limit
  doesn't fill, the setup can requalify on the next run — this is the
  same discipline as the chase-protection rule above.

**Timing.** The morning run fires at the **9:30am ET open** and executes
as soon as its analysis completes (typically ~15 minutes later). An
earlier draft delayed this to 10:30am to avoid the widest spreads of the
day; that reasoning was borrowed from day trading and doesn't hold here.
On a 5-15 day swing with a ~3% stop and a ~6% target, a few basis points
of spread is noise — while waiting an hour lets breakout setups run away,
after which chase-protection correctly rejects them. The net effect of
waiting was systematically missing the fastest-moving setups in exchange
for a rounding error of execution cost. Wide spreads are instead handled
*dynamically* by the 0.5% spread check below, which skips a specific bad
fill rather than avoiding an entire time window.

**Liquidity sanity check before any entry — two parts:**

1. **Spread width.** If the bid-ask spread exceeds **0.5% of the mid
   price** for an equity or ETF, skip the trade and log it. A spread that
   wide on a name that passed a liquidity screen usually means something
   is wrong — a halt, a news event, or stale data — and crossing it eats
   a meaningful fraction of the expected edge on a 2R trade.
2. **Depth.** Call `get_equity_price_book` (Robinhood, Level 2, up to 4
   symbols) and confirm the resting size at the best ask (for a buy, or
   best bid for a sell) is **at least the order quantity**. Spread width
   alone is not sufficient: a book can show a tight-looking top-of-book
   price backed by almost no size, and an order larger than that size
   walks up the book to materially worse prices. Verified example
   (2026-08-10, market closed): JPM's best bid was $356.10 for **1
   share** against a $367.39 ask — a 3.1% effective spread that a
   quote-only check could easily have waved through, while SSO showed
   679/779 shares at a $0.02 spread. If depth at the top level is
   insufficient, skip the trade rather than accepting the slippage.
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
2. `risk_per_trade = 0.4% of current NAV` for a with-trend setup, or
   **0.2% for a counter-trend setup**. These are set so the ATR math and
   the 15%-of-NAV position cap in `gates.md` land in roughly the same
   place — see that file's "Why 15% / 0.4%" note. The previous 1% / 0.5%
   pairing was ~13× larger than the old 3% cap could express, which made
   this entire sizing calculation dead code. If you tune this, re-check
   the position cap alongside it; they are a set, not independent knobs.
3. `shares = floor(risk_per_trade_dollars / stop_distance)`
4. `position_dollar_size = shares × entry_price`, capped at gates.md's max
   position size regardless of what the risk math above produced.
5. `R-target = entry_price + 2 × stop_distance` (2R) — not an automatic
   exit; see the trailing-stop rule below.

**SHORT mode sizing** is identical arithmetic with the signs flipped:
the stop sits **above** entry (`entry_price + stop_distance`) and the
R-target **below** (`entry_price − 2 × stop_distance`). `stop_distance`
itself, the 1%-of-NAV risk budget, and the `gates.md` ceilings are
unchanged. Note that in SHORT mode the position is *always* expressed as
long puts or a put debit spread — so the actual capital at risk is the
premium, and these underlying levels are the trigger prices used to
decide when to close, exactly as described in the options overlay below.

## Options overlay (defined-risk only — see gates.md for hard limits)

Used only to express a signal that already passed Tiers 1-3, when it makes
the trade more capital-efficient. Never a standalone signal source.

- Only long calls, long puts, or vertical debit spreads. Never short/naked.
  In LONG mode use calls / call debit spreads; in SHORT mode use puts /
  put debit spreads — and in SHORT mode options are the **only** allowed
  expression, since shorting stock is forbidden outright.
- Minimum 30 days to expiration, target ~45 DTE for a swing hold of up to
  20 trading days (never buy less time than you expect to need).
- Target delta 0.40-0.60 in absolute value (puts quote negative delta —
  a −0.50 put satisfies the same gate a +0.50 call does).
- **Affordability at the current account size:** with a $10,000 NAV and
  `gates.md`'s 3% cap, max premium per position is **$300**. A single
  near-the-money contract on a $200+ underlying routinely costs more than
  that (a verified JPM ~45 DTE ATM call was $970). So in practice the
  workable expressions here are **debit spreads** (the short leg cuts net
  debit substantially) or contracts on lower-priced underlyings. If
  nothing fits within the premium cap, use equity in LONG mode — and in
  SHORT mode, **take no trade at all**, since equity shorting is
  forbidden. Do not loosen the cap to make an options trade fit.
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

## Index sleeve — leveraged ETFs (no margin, ever)

**Margin and short selling are both disabled entirely.** Leveraged
exposure comes exclusively from *buying* 2x ETFs with cash. Max loss is
what you paid; there is no borrowing, no margin call, no forced
liquidation, no borrow fee, and no squeeze risk.

| Mode | Instrument | Exposure |
|---|---|---|
| LONG | **SSO** (S&P 500) or **QLD** (Nasdaq-100) | 2x long |
| SHORT | **SDS** (S&P 500) or **QID** (Nasdaq-100) | 2x inverse |

All four verified live and liquid on 2026-08-10 (see
`tool_verification.md`). This sleeve is a **market-direction** bet and is
separate from the stock-selection funnel — different signal source,
different rules.

### Signal source
Apply the trend template to the underlying index (SPY for SSO/SDS, QQQ
for QLD/QID) — not to the leveraged ETF itself, whose own moving averages
are distorted by leverage and daily resets. SPY's price and SMA(200) are
already fetched for the regime check, so this costs little extra.

**Use the index-adjusted template below, not the stock one.** Required:
price > SMA(50) > SMA(200), price within **10%** of its 52-week high, and
a relative-strength proxy ≥ 0.6. The stock template's "at least 25% above
the 52-week low" rule is **deliberately dropped here.**

Why: that rule comes from Minervini's template, which was built for
*individual stocks* — names that routinely travel 40%+ in a year. A broad
index does not. Verified 2026-08-10: SPY's entire 52-week range was only
~19–23% top to bottom, so the 25%-above-low test was **mathematically
unsatisfiable** and blocked the index sleeve on every single run the
system made. Applying a single-stock rule unmodified to an index is a
category error, not a strict filter. For an index, trading above both
major moving averages and near its 52-week high is already sufficient
trend evidence.

- LONG mode: SPY (or QQQ) passes the long trend template and an entry
  trigger fires → buy **SSO** (or QLD).
- SHORT mode: the index passes the *inverted* template and a short
  trigger fires → buy **SDS** (or QID).

### Hard rules for this sleeve
- **At most one index position open at a time.** Never hold a long and an
  inverse ETF simultaneously — they are direct opposites and would simply
  cancel while paying two expense ratios.
- **Size off the ETF's own ATR**, not the index's. A 2x ETF's ATR is
  roughly double the index's in percentage terms, so ATR-based sizing
  automatically halves the position — which is the correct behavior, not
  something to override.
- **Counts double toward gross exposure.** A 3%-of-NAV position in a 2x
  ETF carries ~6% of effective market exposure; account for it that way
  against `gates.md` limits.
- **Tighter time-stop: 10 trading days**, versus 20 for ordinary stock
  positions. See the decay note below — time is a materially larger enemy
  here.

### Single-stock leveraged ETFs — both directions (with gates)

Most large caps have **both** a bull and a bear ETF, so the stock sleeve
can express a bearish single-name view without shorting, margin, or
options. All verified live 2026-08-10 (30-day average volumes):

| Underlying | Bull ETF | Vol | Bear ETF | Vol | Bear leverage |
|---|---|---|---|---|---|
| NVDA | NVDL | 13.5M | **NVD** | 78.8M | −2x |
| TSLA | TSLL | 76M | **TSLQ** | 5.9M | −2x |
| AMZN | AMZU | 3.3M | **AMZD** | 13.6M | −1x |
| AAPL | AAPU | 2.2M | **AAPD** | 11.4M | −1x |
| MSFT | MSFU | 7.7M | **MSFD** | 1.7M | −1x |
| META | METU | 5.4M | METD | 744K | ❌ below floor |
| GOOGL | GGLL | 1.7M | GGLS | 368K | ❌ below floor |
| COIN | CONL | 18.6M | CONI | 144K | ❌ below floor |

**Two asymmetries that must not be glossed over:**

1. **Bear-side leverage varies.** Several are only **−1x**, not −2x
   (AAPD, MSFD, AMZD, and TSLS). Always read the fund's own description
   via `get_equity_fundamentals` before sizing — never assume the bear
   ticker mirrors the bull ticker's leverage. Position sizing is ATR-based
   off the ETF itself, so this is handled automatically *if* the right
   instrument is measured, and badly wrong if it isn't.
2. **Inverse funds decay much faster.** The daily-reset drag is
   `(L² − L)/2 × σ²`, which means **−2x carries three times the drag of
   +2x** (3σ² vs 1σ²) — the same penalty as a +3x fund. Observed:
   CONI (−2x COIN) fell $141.65 → $52.64 and TSLQ (−2x TSLA) fell
   $51.45 → $25.17. Treat −2x single-stock funds as the most
   decay-prone instrument in this system.

**Bear side is often much thinner.** Three of eight fail the 1M-share
liquidity floor outright. If a name's bear ETF is below the floor, there
is simply no bearish expression for that name — take no trade rather than
reaching for an illiquid one.

**But the decay problem is far worse here than for index ETFs**, and the
numbers below are not hypothetical.

For a 2x daily-reset fund, the expected drag is approximately **σ² per
year** — it scales with the *square* of volatility, so a stock twice as
volatile as the index decays four times as fast:

| Underlying | Approx. annualized vol | Approx. 2x drag/yr | Drag over a 10-day hold |
|---|---|---|---|
| SPY | ~16% | ~2.6% | ~0.1% |
| TSLA | ~60% | ~36% | ~1.4% |
| COIN | ~85% | ~70% | ~2.8% |

Observed 2026-08-10, holding through a full drawdown: **CONL fell 92%**
from its 52-week high ($52.40 → $4.07) — Coinbase itself did nothing of
the sort — while TSLL fell 67% and METU 59%. Over a disciplined 5-15 day
swing hold the drag is tolerable; those figures are what happens to
someone who holds through chop and doesn't honor the time-stop.

**Rules for the single-stock variant:**
- Only permitted when the underlying's **ATR(14) as a percentage of price
  is ≤ 4%**. Above that, the decay is too steep for the holding period —
  trade the plain stock instead. This is what keeps a CONL-type outcome
  out of the book.
- **7-trading-day time-stop** — shorter than the index sleeve's 10, since
  drag is several times larger.
- Requires 30-day average volume ≥ 1M shares (all listed above qualify;
  re-check before using any ETF not on this list, since many single-stock
  leveraged ETFs are thinly traded).
- Counts **2x** toward gross exposure, same as the index sleeve.
- The **signal still comes from the underlying stock** passing the full
  Tier 1-3 funnel. The ETF is only the expression — never screen or
  signal off the leveraged ETF itself, whose own moving averages and
  52-week range are distorted by leverage and decay.
- Prefer plain equity by default. Use the leveraged version only for a
  setup that passed every tier cleanly, and never as a way to
  "make up" for a small position size.

### Volatility decay — why the time-stop is tighter
Every one of these funds targets **2x the *daily*** return, and resets
each day. Over a multi-day hold, the realized return is path-dependent
and is *not* 2x the period return. If the index rises 10% then falls
9.09% (net flat), the 2x fund returns +20% then −18.18% — ending at
**−1.8%**. Choppy, directionless markets bleed these instruments even
when the index goes nowhere.

The flip side: in a sustained trend, daily compounding works strongly in
your favor. Observed 2026-08-10 — SSO ran from its March low of $48.63 to
$71.68 (**+47%**) during the uptrend, while SDS fell from $80.50 to
$53.35 (**−34%**) over the same stretch and set its 52-week low three
days ago.

Two consequences the rules encode: (1) this sleeve is only used when
ADX-style trend confirmation is present — never in chop, which is exactly
what these instruments punish; (2) inverse leveraged ETFs decay
structurally in a rising market, so a SHORT-mode index position is a
tactical position with a short leash, never something to sit in and wait
on.

## Exit rules (all positions, checked every run)

Mechanical trailing-stop logic (source: Qullamaggie's "let winners run"
approach — a 20-35% win rate can still be strongly profitable if winners
are held longer than losers; see `framework.md`). This replaces a fixed
take-profit with a state flag per position, tracked in `positions.md`:

- **Before the R-target is reached:** the exit stop is the original fixed
  `stop_distance` stop from entry. Nothing else moves it.
- **The instant price reaches the R-target:** record in `positions.md`
  that this position has "reached R-target" — this is a one-way flag, it
  never resets. From this point on, the exit stop becomes the EMA21
  (pulled fresh during the review step) instead of the original fixed
  stop. The trailing stop only ever moves in the favorable direction —
  in a LONG it ratchets **up** and never down; in a SHORT it ratchets
  **down** and never up — and it never moves past the original fixed
  stop in the unfavorable direction, even if EMA21 does.

Close a position if **any** of:
1. Price hits the current exit stop (fixed stop pre-R-target, or trailing
   EMA21 stop post-R-target, per above). LONG: price falls to the stop.
   SHORT: price rises to it.
2. The trend template breaks — LONG: price closes back **below** SMA(50);
   SHORT: price closes back **above** SMA(50). The premise for the trade
   no longer holds. This always exits, regardless of R-target/trailing-
   stop state.
3. Held > 20 trading days without reaching the R-target (time-stop — an
   extreme RSI reading pre-R-target, >75 in a LONG or <25 in a SHORT, is
   a signal to watch more closely, not an automatic exit on its own; the
   time-stop and trend-template rules above are what actually close a
   stalled position)
4. For options specifically: 21 days-to-expiration reached, OR 50% of the
   DTE at entry has elapsed with no progress toward the R-target —
   whichever comes first (see the options overlay section for the full
   rationale)

## Adaptation policy

Every run, after updating the journal:
- Look at the last 10 closed trades in `trade_journal.md`.
- **Check the `counter_trend` column specifically.** If counter-trend
  trades are consistently losing while with-trend trades are not, say so
  plainly in the journal and propose tightening or suspending them — that
  allowance was made on a contested argument and should be judged on its
  own record. The reverse also holds: if they are clearly working, note it.
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
- **2026-08-10 (first live-market day, 0 trades, 4 logged rejects).** Two
  defects fixed, both found by the system's own logging rather than by
  reasoning about the rules:
  1. *Tier 3 momentum test now applies to breakout entries only.* As
     written it required the EMA spread to be at a 5-session maximum,
     which a pullback entry can never satisfy — a pullback is a narrowing
     spread by definition. It would have permanently blocked one of the
     two entry paths. Today's 4 rejects were all breakouts and were all
     genuinely decelerating, so the breakout test itself is **left
     unchanged**: one day is not evidence to loosen a rule, and doing so
     would be exactly the overfitting the adaptation policy forbids.
  2. *Index sleeve now uses an index-adjusted trend template.* The
     stock template's 25%-above-52-week-low rule was unsatisfiable for
     SPY (whose full-year range is ~19–23%) and blocked the index sleeve
     on every run.

  Watch next: whether the breakout momentum test keeps rejecting
  everything once more sessions accumulate. See the pre-registered
  decision rule below.

### Open question — breakout trigger vs. momentum acceleration

There is a plausible structural conflict between Tier 2's breakout
trigger and Tier 3's acceleration test, and it should be resolved by
data rather than argument.

**The mechanism.** `EMA8 − EMA21` is a rate-of-change proxy (the same
construction as MACD). It widens when price *accelerates*, flattens when
price rises at a steady rate, and narrows when price rises but
decelerates — so it peaks *before* price peaks. Tier 2 fires on a level
condition (price within 2% of a 52-week high); Tier 3 demands a
derivative condition (rate of change at a 5-session maximum). Those
coincide only in a violent thrust. In the common pattern — a grind up
into old resistance, a pause, then a break — price sets new highs while
the spread compresses.

Observed 2026-08-10: all three distinct names (TECH, TXG, IMAX) were at
or near new 52-week highs with decelerating spreads. Rejections were
individually correct; the question is whether the combination is
*systematically* unsatisfiable.

**Pre-registered decision rule** — fixed in advance to prevent post-hoc
rationalization:

- **Trigger for change:** ≥ 15 breakout candidates reach Tier 3 across at
  least 10 trading sessions, **and** the momentum test rejects ≥ 90% of
  them. (Both conditions; a small sample or a short window does not
  count.)
- **If triggered, the replacement is already chosen:** confirm breakouts
  with **volume expansion** instead of spread acceleration — require
  `Relative volume ≥ 1.5` from the Tier 0 scan, which costs nothing extra
  since the scan already returns it. This is what O'Neil and Qullamaggie
  actually use to confirm a breakout ("volume surge on the upside break")
  — a better-grounded criterion than a momentum oscillator for this
  specific setup. The acceleration test would then be retired for
  breakouts and remain retired for pullbacks.
- **If not triggered:** change nothing. A low pass rate is not by itself
  a defect — a selective filter that only admits genuinely accelerating
  breakouts may be doing exactly its job.

Track this by counting `action=reject` rows in `trade_log.csv` whose
`trigger` is `breakout` and whose `notes` cite the momentum test.
