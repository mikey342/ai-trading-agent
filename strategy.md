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
  penny-stock exclusion), 30-day average volume > 250K shares (lowered
  from 1M on 2026-08-11 — see the dollar-volume note below),
  **ADX(14) > 25** (a real trend exists — direction-agnostic, so this
  serves both long and short modes), RSI(14) between 25-75 (deliberately
  wide enough to surface oversold short-mode candidates as well as
  long-mode pullbacks).
- **Sorted ADX(14) descending** (changed from market-cap desc on
  2026-08-11) so that the scan's 200-row response cap aligns with the
  funnel's own primary ranking axis — the top 15 by ADX is then always
  inside the returned rows.
- **Verified live 2026-08-11:** 396 matches under this filter set,
  200 returned (response cap), truncating at ADX ≈ 31.2.

#### ⚠ Removed 2026-08-11: `relative options volume > 0.5`

This filter was **time-inconsistent and silently crippling the morning
run.** Robinhood computes it as
`optionsTotalDayVolume / optionsTotalVolumeAvg(30d)` — *today's
partially-accumulated* options volume over a *full-day* 30-day average.
The numerator grows through the session while the denominator does not,
so the same `> 0.5` threshold means something completely different
depending on when the scan runs:

| Run time | Session elapsed | Observed matches |
|---|---|---|
| 09:57 ET | ~7% | **4** |
| 12:57 ET | ~54% | **51** |
| (same day, filter removed) | — | **396** |

Measured 2026-08-11: the four names that survived the 09:57 scan read
0.628 / 0.817 / 0.633 / 0.934 — all clustered just above the threshold,
i.e. the filter was selecting *names running ~7x normal early options
pace*, not trend candidates. The same names by midday: TSLA 0.633 →
2.619, RKLB 0.934 → 2.326. The 2026-08-10 runs showed the identical
split (7 matches in the morning, ~92-96 at midday/close).

**This is not fixable by tuning the threshold.** At 09:57 `> 0.5` selects
the top few percent by early options flow; by 15:57 it means "about half
of normal pace," which is nearly everything. No single value is correct
across all three run times, so the filter was removed rather than
re-tuned. This is the same defect already documented for volume
confirmation in `gates.md` ("never use a partial intraday volume"), which
had been applied to the client-side check but never to the screener's own
filter set.

Nothing load-bearing was lost: liquidity is enforced by market cap >$2B,
price >$5, 30d avg volume >250K, and the client-side $20M/day
dollar-volume screen. The options overlay checks its own chain liquidity
at selection time regardless.

### ⚠ The scan's ADX is not the same measure as a direct ADX call

**Verified 2026-08-11.** The Tier 0 screener computes ADX with
`session="all"` — its own filter expression is
`adx(candlePeriod="1d", candleCount=14, session="all")` — which includes
extended-hours bars. `get_equity_technical_indicators` defaults to
`bounds="regular"`. The two disagree materially, in **both** directions:

| Symbol | Scan ADX | Direct (regular) |
|---|---|---|
| TECH | 59.71 | 72.03 |
| FTNT | 33.80 | 26.05 |

**Rule: the scan's ADX may be used for screening and ranking; it must
never satisfy a gate.**

- **Screening / ranking (fine):** the `ADX > 25` scan filter and the
  Tier 0 ranking. Both are relative and coarse, so a consistent
  alternative measure is acceptable.
- **Gates (must re-measure):** the counter-trend `ADX > 30` requirement,
  and any future ADX-dependent threshold. Call
  `get_equity_technical_indicators` (type=adx, period=14, interval=day,
  output=latest) for that symbol and use *that* value.

Why it matters concretely: FTNT clears the counter-trend ADX gate on the
scan value (33.80) and fails it on the direct value (26.05) — the same
stock, opposite decisions. Note also that TSLA was cleared through the
counter-trend gates on 2026-08-10 using a scan-derived "ADX 30.56"; under
a regular-session reading it may not have qualified. No trade resulted,
so nothing needs unwinding, but it shows the gate was being evaluated on
the wrong measure.

There is a deeper reason to prefer regular-session here: this system
trades only regular sessions, prices fills off regular-session quotes,
and refuses to run pre-open precisely because extended-hours data is thin
and unrepresentative. Letting extended-hours bars decide a trend-strength
gate contradicts that choice.

### Liquidity: use dollar volume, not share volume

Share volume is a poor liquidity measure across price levels — a $500
stock trading 200K shares moves $100M/day, while a $10 stock trading 1M
shares moves only $10M/day. The share-based filter passed the cheap one
and rejected the far more liquid one.

The scanner has no dollar-volume filter, but it returns both `Last` and
`Average volume`, so **compute `Last × Average volume` client-side and
drop anything below $20M/day.** Free, and a far better measure.

Measured 2026-08-11 (pre-filter-removal, 116-match set): only **3** fell below $20M/day
(TIMB $9M, EFXT $10M, TFSL $15M). The `market cap > $2B` filter was
already doing nearly all the liquidity work — the 1M-share floor was
excluding roughly 20 names while adding almost no protection. $20M/day is
still ~13,000× the maximum position size, so this is a guard against
genuinely illiquid names, not a binding constraint.

**Why this replaced the old hardcoded ~19-name watchlist:** verified live
2026-08-10, the screen returned **266 qualifying names**, and most of the
old watchlist (AAPL, MSFT, NVDA, GOOGL, META, JPM) **failed** it — ADX
below 25, i.e. not actually trending — while it surfaced names never on
the list (PANW, CRWD, FTNT, NET, NU...). A hand-picked mega-cap list was
simultaneously too narrow to find real leaders and padded with names that
didn't meet the strategy's own criteria. It also made the
relative-strength ranking nearly meaningless: ranking 19 pre-selected
mega-caps against each other is not a cross-sectional signal.

**Narrowing for the funnel:** the returned set (up to 200 rows) is far
more than Tier 1 can process. Rank the scan results client-side (free —
the scan already returned these columns) by `Average directional index
(14)` (trend strength), then take the **top 15** into Tier 1.

> **Known limitation (2026-08-11).** The scan matches ~396 names but
> returns only 200. Sorting by ADX desc makes this harmless for the
> ADX rank itself, but the **AI tilt is applied before the rank**, so an
> `ai_theme.md` name whose ADX falls below the truncation boundary
> (≈31.2 on 2026-08-11) cannot be tilted into the top 15 because it is
> never returned. On 2026-08-11 only **6 of 95** theme names appeared in
> the returned rows (BWXT, CRWD, FTNT, GFS, KLAC, NTAP). It is **not yet
> established** how many of the remaining 89 were legitimately filtered
> out by ADX>25 / RSI 25-75 versus merely truncated — do not assume
> either. Resolving this properly means querying `ai_theme.md` names
> directly rather than depending on scan visibility, which is a Tier 0
> redesign, not a bug fix, and is left as an open decision.

**AI theme tilt.** Multiply the ranking score by **1.15** for any symbol
listed in `ai_theme.md`. This is a tilt, not a filter: it makes AI names
likelier to reach the top 15 without excluding anything else, and a
non-AI name still displaces an AI name if it ranks higher on merit. The
boost is deliberately modest — large enough to surface the theme, too
small to promote a weak setup over a strong one. See `ai_theme.md` for
why tilting beats filtering here. Rank on ADX; the former relative-options-volume
tiebreaker was removed with that filter on 2026-08-11 (see above). If the scan
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

## Regime: two independent dimensions

Direction and risk level are separate questions and are measured
separately. **Direction** says which way to hunt; **risk level** says how
much to commit.

### Direction — SPY vs its 200-day SMA, 3-day confirmed

Unchanged as the primary signal, and deliberately slow. This answers
"what is the market's structural regime?" — not "when do I enter," which
the 8/21 EMAs and the momentum test already handle. Faber's 10-month SMA
model (≈200 days) cut maximum drawdown from 46% to under 13% while
staying invested ~70% of the time at under one round trip per year; that
is the benchmark this filter is drawn from.

**New: require 3 consecutive daily closes on the far side before
flipping.** A single close through the 200DMA is the classic whipsaw, and
published work on 200DMA timing finds 3-5 days of confirmation the right
balance between cutting false flips and arriving too late. Confirmation
applies *only* to the direction flip, never to the risk levels below —
those should react immediately.

### Risk level — SPY vs its 50-day SMA, and VIX

The 200DMA's weakness is lag: SPY can fall 10-15% before crossing it, and
for that entire window the system would still treat bullish setups as
with-trend and make bearish ones *harder*. That is backwards during a
decline. The risk level catches deterioration far earlier without
flipping the regime on noise.

| Level | Trigger (worst applies) | Effect |
|---|---|---|
| **NORMAL** | SPY > SMA(50) **and** VIX < 20 | Full rules |
| **CAUTION** | SPY < SMA(50), **or** VIX 20–25, **or** VIX ≥ 25 while SPY is still above its SMA(200) | Max **1** new trade this run; no new index-sleeve entries; **counter-trend disabled** |
| **STRESSED** | VIX ≥ 25 **and** SPY below its SMA(200) | **No new entries of any kind.** Existing positions are still reviewed and exited normally |

Always take the **worse** reading. Existing positions are managed
identically at every level — risk level gates *new* commitment only,
never exits.

### Why STRESSED requires *both* high VIX and a prior decline

A reasonable objection: high VIX marks fear, fear marks lows, so isn't
that when we should be buying? For **the index**, yes — the evidence is
strong. Buying SPX with VIX > 30 has historically produced ~23% average
one-year returns, ~12.4% over six months, positive 70–83% of the time,
and in the 30–40 VIX band three-week forward returns were positive 81.5%
of the time.

But that evidence is about **buying the index**, not about **running a
momentum strategy**. Daniel & Moskowitz, *Momentum Crashes* (Journal of
Financial Economics), finds momentum's rare catastrophic losses occur
specifically in "panic states — following market declines and when market
volatility is high — and are contemporaneous with market rebounds."
Momentum performs worst **not during the bear market but when it ends**.
The mechanism: after a major decline, past-loser betas rise above 3 while
past-winner betas fall below 0.5, so a momentum book carries a large
*negative* conditional beta into the rebound and gets run over by it.

So the moment that is excellent for buying SPY is the documented worst
moment for buying a 52-week-high breakout. The two claims don't conflict;
they describe different strategies.

**The refinement** is that Daniel & Moskowitz specify a *conjunction* —
high volatility **following declines**. A VIX spike inside an intact
uptrend is a shakeout, not a panic state, and halting entirely there
costs opportunity for no documented reason. Hence STRESSED requires both
conditions; high VIX alone with the primary trend intact is CAUTION.

**What we knowingly give up:** the sharp recoveries off panic lows. That
is a real cost, and it is accepted deliberately — capturing it would
require a *mean-reversion* sleeve (buy the index on fear, hold weeks to
months), which is a different edge on a different horizon from anything
here. Worth building someday; not worth bolting onto an unvalidated
momentum system now.

**VIX thresholds are calibrated to observed distribution, not guessed.**
Measured over the 12 months to 2026-08-11: VIX spent most weeks at
14–20, ran 20–24 in three elevated stretches, and reached 26–31 (peak
35.3) in the March 2026 selloff. Current level 15.46.

**The March 2026 episode is why this layer exists.** VIX crossed 25 in
early March — roughly *three weeks before* SPY bottomed on 3/30. SPY fell
about 17%, so the 200DMA cross fired too, but considerably later. VIX
gave the earlier warning, which is precisely the lag the direction signal
cannot fix on its own.

**Honest caveat:** these thresholds are conventions fitted to one year of
data, not backtested parameters. 20 and 25 are round numbers that match
the observed regimes; they are not optimized. If VIX's own baseline
shifts materially — a sustained high-volatility era where 22 is the new
normal — these need revisiting, since a fixed threshold would then read
"caution" permanently and disable counter-trend for months.

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
   trend must be unusually well established, not marginal. **Measure this
   with a direct `get_equity_technical_indicators` call, never the Tier 0
   scan's ADX column** — the two use different session bounds and
   disagree by 7-12 points in both directions. See "The scan's ADX is not
   the same measure" above.
3. **A more extreme relative-strength reading**: ≤ **0.25** of the
   52-week range for a counter-trend bearish setup (vs. 0.4 normally),
   ≥ **0.75** for a counter-trend bullish setup (vs. 0.6).
4. **No degraded data.** Every Tier 3 check must return real values —
   in particular, `news_sentiment` must **not** be `UNAVAILABLE`. If the
   Alpha Vantage quota is exhausted, no counter-trend trade is taken that
   run, full stop. A counter-trend bet made blind to news is exactly the
   trade most likely to be on the wrong side of a catalyst.
5. **Half size**: `risk_per_trade = 0.2% of NAV` rather than 0.4%.

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

**LONG mode — three triggers, any one qualifies:**
- **`breakout_52w`** (O'Neil / George & Hwang): price within 2% of
  `high_52_weeks` AND EMA8 > EMA21 AND **volume confirmation** (below).
  The strongest level signal — no overhead supply at all.
- **`breakout_20d`** (original Turtle): price today exceeds the **prior
  bar's** 20-day Donchian **upper** channel AND EMA8 > EMA21 AND
  **volume confirmation**. A weaker level signal than a 52-week high, but
  it carries the same volume requirement.
- **`pullback`** (Connors-style): RSI(14) between 35-45 AND price still >
  SMA(50) (pullback within an intact uptrend, not a breakdown) AND EMA21
  is flat-to-rising over the last 5 values (trend not rolling over).

### Volume confirmation — the breakout gate (replaces the momentum test)

**Both breakout triggers require `relative_volume ≥ 1.4`** — the most
recent **completed** daily session's volume divided by the 30-day average
volume, both from `get_equity_fundamentals`.

**Why 1.4:** O'Neil's rule is that a valid breakout comes on volume
40–50% above average — the "footprints of big money," institutional
demand arriving. This is not folklore: **O'Neil Global Advisors published
quantitative research over 1995–2021** finding breakouts produce ~1.1%
alpha and 3.2% returns over three months, with **breakouts accompanied by
sharp volume increases significantly outperforming**. O'Neil's summary:
*price without volume is meaningless.*

**Two implementation details that matter:**

1. **Use the last COMPLETED session, never a partial one.** During a live
   run today's volume is only partial, so comparing it to a full-day
   average would understate every name and reject everything. This
   matches how the rest of the funnel already behaves — all daily
   indicators are computed through the prior close.
2. **Compute it, don't read the scan's `Relative volume` column.** The
   scan uses `session="all"`; the same session-bounds discipline that
   applies to ADX applies here. Fundamentals give regular-session volume.

**What this replaced, and why.** Tier 3 previously gated breakouts on a
momentum-acceleration test — the EMA8/EMA21 spread having to be the
widest of its last five sessions. **That rule was invented, not
researched.** It began as an ambiguous "spread widening" phrase, and the
precise form was chosen for determinism, not derived from evidence. It
rejected 5 of 5 breakout candidates across two sessions.

The change is made on **provenance, not on those rejections.** The
pre-registered threshold (≥15 candidates over ≥10 sessions) governs
whether a *grounded* rule is too strict; it was never a reason to keep an
*ungrounded* one. Volume confirmation replaces it because it has a
26-year quantitative study behind it, a citable threshold, and it sits
with the trigger rather than as a separate layer — the original Turtle
system had no third confirmation stage at all.

**The old test is still computed and logged**, just not gating, so the
pre-registered question can still be answered later: would it have helped?

**Evidence it is doing real work, not just letting everything through**
(measured 2026-08-11 on Monday's completed session): the two names that
reached Tier 3 under the old rule broke toward 52-week highs on
*below-average* volume — ACA at 0.68× and HQY at 0.85× — precisely the
weak breakouts O'Neil declines. Meanwhile BFLY (1.50×) and NTAP (1.46×)
showed genuine participation. The rule selects **different** names, not
merely **more** of them.

**Why `breakout_20d` was added (2026-08-11).** The first two triggers left
a **dead zone**: a stock 10% off its high, basing for six weeks, RSI ~52,
breaking out of its range fires *neither* — too far from the high for
`breakout_52w`, not oversold enough for `pullback`. That is a textbook
swing setup falling straight through the middle. It is also worth noting
the Turtles' actual system used 20- and 55-day channel breakouts, not
52-week highs; adopting only the stricter O'Neil level had quietly
dropped the entry that Turtle sizing was designed around.

**Labeling when triggers overlap.** A stock at a new 52-week high is
necessarily at a new 20-day high, so `breakout_52w` is a strict subset of
`breakout_20d`. When both fire, label it **`breakout_52w`** — the
stronger signal. Recording which trigger fired is what will later show
whether the weaker 20-day breakout earns its place or just adds losing
trades.

**SHORT mode mirrors all three**, using the Donchian **lower** channel for
`breakdown_20d`, and the 52-week low for `breakdown_52w`.

**SHORT mode (all three mirrored):**
- **`breakdown_52w`**: price within 2% of `low_52_weeks` AND EMA8 < EMA21
- **`breakdown_20d`**: price today falls below the **prior bar's** 20-day
  Donchian **lower** channel AND EMA8 < EMA21 AND `relative_volume` ≥ 1.4
- **`rally_to_resistance`**: RSI(14) between 55-65 AND price still <
  SMA(50) (a bounce inside an intact downtrend, not a genuine reversal)
  AND EMA21 is flat-to-falling over the last 5 values

**The short mirror is "short the bounce," not "short the dip."** Note the
RSI band for a short entry (55-65) sits *above* neutral, not below — the
setup wants the stock to have rallied into resistance so the entry is
against strength that is likely to fail. A weak RSI reading is
explicitly *not* a short signal: selling something that has already
fallen hard invites the snapback. Worked example, 2026-08-10: TSLA passed
the bearish trend template but sat at RSI 41.4 and 15% above its 52-week
low — too little bounce for a rally entry, too far off the low for a
breakdown. Correctly declined. (41.4 would have been a valid *long*
pullback reading, which is exactly the asymmetry: the mirror of buying a
dip is shorting a rally, not shorting a decline.)

## Tier 3 — Confirmation (finalists only, cap at 3 per run per gates.md)

For candidates passing Tier 2:

1. **Momentum spread — LOGGED, NOT GATING** (changed 2026-08-11).
   Compute `|EMA8 − EMA21|` over the last 5 sessions and record whether
   the most recent value is the widest, into `trade_log.csv`'s
   `momentum_test_would_pass` column. **Never reject a candidate on it.**

   Breakout confirmation now happens at Tier 2 via **volume** (see
   "Volume confirmation" above), which carries a 26-year quantitative
   study where this test had nothing behind it. The old rule was
   *invented*: it started as an ambiguous "spread is widening" phrase and
   its precise form — most recent value strictly the widest of five — was
   chosen by the assistant for determinism, not derived from evidence. It
   then rejected 5 of 5 breakout candidates across two sessions
   (TECH, TXG, IMAX on 08-10; ACA, HQY on 08-11).

   **It was retired on provenance, not on those rejections.** The
   pre-registered threshold (≥15 candidates across ≥10 sessions) governs
   whether a *grounded* rule is too strict. It was never a reason to keep
   an *ungrounded* one, and treating a 5-name sample as justification
   would have been exactly the reasoning that threshold exists to
   prevent.

   Keeping it logged preserves the original question: once trades close,
   compare outcomes where this test would have passed versus failed. If
   it turns out to carry predictive value, it can be reinstated **on
   evidence** rather than on the intuition that produced it.

   (It never applied to pullbacks in any case — a pullback *is* a
   narrowing spread, so a 5-session-maximum requirement would have
   rejected every pullback that ever occurred.)

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
5. **`get_sentiment` (Stocktwits) — logged, never gating.** Record
   `score`, `label`, and message volume into `trade_log.csv`; write
   `NO_COVERAGE` when the call returns an empty data array, which happens
   for roughly two-thirds of candidates. **This never blocks or approves
   a trade.** Coverage correlates with "is this a retail favorite," which
   is not a quality this system selects for — TECH, the top-ranked
   candidate on 2026-08-10, had no coverage at all. Use `score`/`label`,
   never `bullish_pct` (see `tool_verification.md` for why that field
   lies on thin names).

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

## Instrument selection — which vehicle expresses the signal

Once a candidate clears Tier 3, decide *how* to express it. This is a
mechanical rule, not a judgement call.

**The governing insight: under ATR-based sizing, leverage does not
increase your risk — it increases your capital efficiency.** A 2x ETF has
roughly 2x the ATR, so `stop_distance` doubles and the share count
halves. Same dollar risk, half the capital. Leverage therefore only earns
its place when **the 15% position cap prevents reaching the risk
target** — never as a way to "bet bigger."

### The decision procedure

Compute both share counts first (you already do this for sizing):

```
shares_risk = floor(risk_budget / stop_distance)      # 0.4% NAV (0.2% counter-trend)
shares_cap  = floor(0.15 x NAV / price)               # the 15% position cap
```

**1. If `shares_risk <= shares_cap` → PLAIN EQUITY.**
The risk budget binds, meaning the position already reaches its intended
risk within the cap. Leverage here would *overshoot* the risk target,
and adds decay and an extra failure mode for nothing. This is the common
case and the default.

**2. If `shares_risk > shares_cap` → the cap is binding and the position
is under-risked. Consider a leveraged ETF**, but only if *all* hold:
   - A mapped ETF exists for that name (see the table below) clearing
     the 100K-share floor, with the order ≤ 1% of its 30-day ADV
   - Underlying **ATR ≤ 4% of price** (the decay gate in `gates.md`)
   - Leverage **verified per instrument** — several bear ETFs are −1x,
     not −2x
   - Gross exposure stays ≤ 1.3x NAV after applying the 2x multiplier
   - Size off the **ETF's own ATR**, not the underlying's

   If any fails, fall back to plain equity and accept being under-risked.
   An under-risked position is a smaller win; a badly chosen instrument
   is a new way to lose.

**3. Options — only when the premium genuinely fits.**
At the current $10,000 NAV the 3% premium cap is **$300**, and a
near-the-money contract on a $200+ underlying routinely costs $900+, so
this path is usually unavailable rather than unattractive. When it does
fit (debit spreads, lower-priced underlyings), the reason to prefer it is
specific: **an option's maximum loss is the premium regardless of gaps**,
whereas a stock can gap straight through its stop. That is the only thing
options offer here that the other two do not. Use them when gap risk is
the concern, not for leverage — the ETF path is simpler and has no
expiration to manage.

**SHORT mode has no choice to make.** Shorting stock is forbidden, so a
bearish signal is either a bear ETF or no trade at all.

### Worked example (2026-08-11 dry run)

| Symbol | `shares_risk` | `shares_cap` | Binding | Instrument |
|---|---|---|---|---|
| NTAP | 6 | 7 | risk budget | **plain equity** |
| BFLY | 137 | 154 | risk budget | **plain equity** |
| ACA* | 29 | 10 | **position cap** | leverage candidate |

*ACA did not pass Tier 2, but illustrates the case: its ATR is 0.62% of
price, so reaching a 0.4% risk target needs a ~42%-of-NAV position. The
cap holds it to 15%, landing at 0.14% risk — a third of target. That is
precisely the situation leverage is for.

Note the pattern: **low-volatility names are the ones that want
leverage**, because a tight stop needs a large position to carry normal
risk. High-volatility names never need it — and the ATR ≤ 4% decay gate
independently blocks them, so the two rules agree rather than conflict.

## Options overlay (defined-risk only — see gates.md for hard limits)

Used only to express a signal that already passed Tiers 1-3, when it makes
the trade more capital-efficient. Never a standalone signal source.

- Only long calls, long puts, or vertical debit spreads. Never short/naked.
  In LONG mode use calls / call debit spreads; in SHORT mode use puts /
  put debit spreads — and in SHORT mode options are the **only** allowed
  expression, since shorting stock is forbidden outright.
- Minimum 30 days to expiration, target ~45 DTE for a swing hold of up to
  10 trading days (never buy less time than you expect to need).
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
- **Tighter time-stop: 8 trading days**, versus 10 for ordinary stock
  positions. See the decay note below — time is a materially larger enemy
  here.

### Single-stock leveraged ETFs — both directions (with gates)

Most large caps have **both** a bull and a bear ETF, so the stock sleeve
can express a bearish single-name view without shorting, margin, or
options. All verified live 2026-08-10 (30-day average volumes):

| Underlying | Bull ETF | Vol | Bear ETF | Vol | Bear leverage |
|---|---|---|---|---|---|
| NVDA | NVDL | 13.5M | **NVD** | 78.8M | −2x |
| PLTR | — | — | **PLTD** | 41.2M | −1x |
| AMZN | AMZU | 3.3M | **AMZD** | 13.6M | −1x |
| MU | — | — | **MUD** | 13.3M | −1x |
| AAPL | AAPU | 2.2M | **AAPD** | 11.4M | −1x |
| MSTR | — | — | **MSTZ** | 9.0M | −2x |
| TSLA | TSLL | 76M | **TSLQ** | 5.9M | −2x |
| MSFT | MSFU | 7.7M | **MSFD** | 1.7M | −1x |
| AMD | — | — | **AMDD** | 1.07M | −1x |
| META | METU | 5.4M | **METD** | 744K | −1x |
| GOOGL | GGLL | 1.7M | **GGLS** | 368K | −1x |
| MSTR | — | — | SMST | 223K | −2x (prefer MSTZ — deeper) |
| COIN | CONL | 18.6M | **CONI** | 144K | −2x |
| NFLX | — | — | NFXS | 44K | ❌ below the 100K floor |

Verified 2026-08-10/11. **Thirteen names have a tradeable bear
expression** under the 100K-share floor (lowered from 1M on 2026-08-11 —
see `gates.md`, "Why 100K"). Only NFXS is excluded outright.

Two things to still check per trade rather than assume: that the order is
≤ 1% of that ETF's 30-day ADV, and that the spread and depth checks pass
at entry. Those are the real liquidity guards now; the volume floor
merely screens out moribund products.

**3x inverse funds (SQQQ, SPXS) are out of scope**, despite being highly
liquid (58M and 11M respectively). Daily-reset drag is `(L²−L)/2 × σ²`,
so a −3x fund carries **6× σ²** — double the −2x penalty and six times
the +2x one. At that decay rate a multi-day hold is fighting the
instrument itself. The index sleeve stays on SDS/QID (−2x).

**Note how the volatility gate interacts with this list.** MSTR and COIN
are the most decay-prone underlyings here, and `gates.md`'s ATR ≤ 4%
requirement will typically exclude them before their bear ETFs are ever
considered. That is the gate doing its job, not a gap — the names with
the most liquid inverse products are often the ones whose volatility
makes leveraged exposure worst.

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

**Bear side is consistently thinner than the bull side** — often by an
order of magnitude — which is why the floor was recalibrated to 100K
rather than kept at 1M. If a name's bear ETF is below the floor, there
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
- **5-trading-day time-stop** — shorter than the index sleeve's 8, since
  drag is several times larger.
- Requires 30-day average volume ≥ 100K shares **and** the order to be
  ≤ 1% of that ETF's 30-day ADV (see `gates.md`). Re-check before using
  any ETF not listed above — single-stock leveraged products vary enormously
  in depth, and the bear side is routinely thinner than the bull side.
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

   **Check this on the UNDERLYING, not on a leveraged ETF.** Record the
   underlying in `positions.md`'s `Underlying` column at entry (NVD →
   NVDA, SDS → SPY; for a plain stock it equals the symbol). A leveraged
   ETF's own moving averages are distorted by leverage and daily resets
   and do not describe the thesis. Stops and R-targets are the opposite —
   always tracked on the **instrument actually held**, since that is what
   was sized and filled.
3. **Time-stop — the limit depends on the sleeve** (per `gates.md`):
   **10 trading days** for ordinary stock, **8** for an index leveraged
   ETF (SSO/QLD/SDS/QID), **5** for a single-stock leveraged ETF. The
   more decay-prone the instrument, the shorter the leash. Measured from
   `Entry date` and only applies while the R-target has *not* been
   reached — once it has, the trailing stop governs and a winner is
   allowed to run past these limits.

   (An extreme RSI reading pre-R-target — >75 in a LONG, <25 in a SHORT —
   is a signal to watch more closely, not an automatic exit. The
   time-stop and trend-template rules are what actually close a stalled
   position.)
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
- **Check the `trigger` column too.** `breakout_20d` is a weaker level
  signal than `breakout_52w` and was added to close a coverage gap, not
  because it was shown to work here. Once ≥10 trades have closed on it,
  compare its win rate and average R against the other triggers. If it
  underperforms, remove it — a trigger that only adds trades is worse
  than no trigger.
- **Check the `social_*` columns once coverage accumulates.** These are
  logged and never gate anything. The hypothesis worth testing is
  **contrarian**: extreme bullish sentiment *plus* extreme message volume
  on a stock already at a 52-week high is a plausible distribution
  signature — the crowd arriving late. If a gating role is ever proposed,
  the evidence must support that direction. Wiring it as confirmation
  ("only enter when the crowd is bullish") would risk making entries
  worse exactly when it matters most. Expect ~2/3 of rows to read
  `NO_COVERAGE`; that is a property of the universe, not a bug.
- **Check the `ai_theme` column the same way.** The 1.15× tilt was added
  on a plausible thesis (AI names are high-beta and high-ADX, which suits
  momentum), not on evidence from this system. Once ≥10 trades have
  closed on each side, compare win rate and average R for `true` vs
  `false`. If AI names are not outperforming, say so and propose removing
  the tilt — a theme bet that isn't paying is just concentration risk
  with extra steps.
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

**RESOLVED 2026-08-11 — but not by this rule.** The pre-registered
threshold (≥15 candidates across ≥10 sessions with ≥90% rejected) was
**never met** — the count reached 5 names across 2 sessions. The test was
retired anyway, on a different and independent basis: **it was invented
rather than researched.** Reviewing its provenance showed the precise
form ("most recent spread strictly the widest of five") had been chosen
by the assistant for determinism, with no supporting evidence, while a
well-grounded alternative existed — O'Neil's volume confirmation, backed
by a 1995-2021 quantitative study.

Those are different claims, and the distinction is the point: the
threshold governs whether a *grounded* rule is too strict. It was never
a reason to keep an *ungrounded* one. Citing the 5 rejections as
justification would have been precisely the post-hoc rationalization this
pre-registration exists to prevent — so they were treated as what
prompted the review, not as evidence for the conclusion.

### The question that remains open

Retiring the gate did not answer whether the test had predictive value.
That is still worth knowing, so it is **computed and logged** on every
candidate as `momentum_test_would_pass`, gating nothing.

**New pre-registered rule, for reinstating it:**

- **Trigger:** ≥ 20 closed trades, **and** entries where the test would
  have passed show a materially better average R-multiple than those
  where it would have failed (≥ 0.3R difference).
- **Count distinct symbol-days, never raw CSV rows.** A name evaluated at
  the morning, midday and pre-close runs produces up to three identical
  entries, because daily-bar indicators do not change intraday — only
  price does. Confirmed 2026-08-10, when TECH was logged twice with
  byte-identical values.
- **If triggered:** reinstate it as a gate, on evidence rather than the
  intuition that produced it the first time.
- **If not:** leave it logged and non-gating. A test that does not
  separate winners from losers has no claim on the funnel.

### Daily indicators do not move intraday — plan around it

Confirmed 2026-08-10: TECH returned byte-identical EMA8, EMA21, RSI, ADX
and ATR at both the 18:42 and 19:41 runs. Daily-interval indicators are
computed from *completed* daily bars, so during a session they reflect
data through the prior close and cannot change until the day closes.

Two consequences the funnel should respect:

1. **A candidate rejected on a daily-bar-only criterion earlier today
   will be rejected again, deterministically.** The momentum test, RSI
   band, and EMA-ordering checks all fall in this category. Re-running
   them the same day cannot produce a different answer — it only burns
   wall-clock time and writes duplicate rejection rows that corrupt the
   tally above. Before advancing a candidate to Tier 2/3, check whether
   `trade_log.csv` already holds a reject row for that symbol today
   citing a daily-bar criterion; if so, skip it and do not log it again.
2. **Price-based checks *do* change intraday** and remain worth
   re-running: proximity to the 52-week high, price versus SMA(50), the
   regime check, and every exit rule on open positions. This is what the
   midday and pre-close runs are genuinely for — not re-deriving
   indicators that cannot have moved.
