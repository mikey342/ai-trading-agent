# Tool Verification Record

What has actually been called and confirmed working, versus assumed. Kept
because two separate structural bugs in this system came from *assuming*
a tool worked: Alpha Vantage's `MACD` and `TIME_SERIES_DAILY_ADJUSTED`
both turned out permanently premium-gated, and each would have silently
blocked every trade forever.

Last verified: **2026-08-10**.

## Robinhood — verified working

| Tool | Verified | Notes |
|---|---|---|
| `get_equity_quotes` | ✅ | Live price + official prior-session close. Batches ≤20 symbols. |
| `get_equity_fundamentals` | ✅ | P/E, P/B, market cap, 52wk high/low + dates, sector, float. Batches ≤10. **No PEG field** — value ranking uses raw P/E cross-sectionally. |
| `get_equity_technical_indicators` (sma) | ✅ | JPM SMA50 = $333.00 — cross-checked identical to Alpha Vantage's value. |
| `get_equity_technical_indicators` (rsi) | ✅ | `output:"last:5"` works — returns exactly 5 points, no huge dump. |
| `get_equity_technical_indicators` (ema) | ✅ | JPM EMA21 last-5 confirmed rising. |
| `get_equity_technical_indicators` (atr) | ✅ | JPM ATR14 = 7.164. **Load-bearing — all position sizing depends on it.** |
| `get_earnings_results` | ✅ | 8 trailing quarters actual-vs-estimate EPS + next report date. Covers both the earnings blackout and the PEAD surprise-history read. |
| `get_option_chains` | ✅ | Returns chain id + all expiration dates. |
| `get_option_instruments` | ✅ | **Required middle step** — chains do not contain contracts. Filter by expiration/type/strike. |
| `get_option_quotes` | ✅ | Takes instrument UUIDs. Returns delta, open_interest, bid/ask, mark, full Greeks, IV. **Confirms every options gate in `gates.md` is actually enforceable.** |
| `get_equity_price_book` | ✅ | Level 2 depth, up to 4 symbols. **Now used in the entry liquidity check** — spread width alone is not enough. Verified 2026-08-10 (market closed): JPM best bid $356.10 for **1 share** vs a $367.39 ask (3.1% effective spread), while SSO showed 679/779 shares at $0.02. A quote-only check would have passed JPM. |
| `get_earnings_calendar` | ✅ | Market-wide, up to a 31-day window, optional `high_market_cap` filter. **Now used for the Tier 1 earnings blackout** — one call replaces per-finalist `get_earnings_results` checks and filters candidates *before* expensive per-symbol work. |
| `get_scanner_filter_specs` | ✅ | Full filter catalog — fundamental, price/volume, technical, and options-flow groups. |
| `create_scan` / `run_scan` | ✅ | Real market-wide screener. Created `de1b1994-b5db-472a-9b79-c052f1215193`; returned **266 live matches**, with ADX/RSI/volume/market-cap/price per row. **Replaced the hardcoded watchlist as the candidate source** — see `strategy.md` Tier 0. |

### Options flow is three steps, not two
`get_option_chains` (get chain + expirations) → `get_option_instruments`
(get contract UUIDs for an expiration/strike/type) → `get_option_quotes`
(get delta/OI/spread for those UUIDs). Skipping the middle step doesn't
work — chains carry no contracts.

### Verified options gate enforceability (JPM $360C, 2026-09-18)
delta 0.4987 (gate 0.40-0.60 ✅), open_interest 3,944 (gate ≥500 ✅),
bid/ask 9.25/10.15 on 9.70 mark = 9.3% spread (gate ≤10% — passes, but
narrowly). **Caveat:** that spread was measured while the market was
closed, when spreads are widest; live-session spreads should be tighter.
Don't loosen the gate based on a closed-market reading.

### Fill realism
`get_option_quotes` returns `high_fill_rate_buy_price` /
`high_fill_rate_sell_price` alongside `mark_price`. Simulating fills at
mark is systematically optimistic versus what a real order gets. Paper
fills should use the fill-rate prices — see `strategy.md`.

## Alpha Vantage — mixed

| Tool | Status | Notes |
|---|---|---|
| `COMPANY_OVERVIEW` | ✅ works | No longer used — Robinhood covers it. |
| `GLOBAL_QUOTE` | ✅ works | No longer used — Robinhood covers it. |
| `TIME_SERIES_DAILY_ADJUSTED` | ❌ **premium-gated** | Never call. |
| `MACD` | ❌ **premium-gated** | Never call. Robinhood's `type=macd` works instead. |
| `RSI`, `EMA`, `ATR`, weekly/monthly series | ⚠️ works but unusable | Dumps full multi-year history (60-100k+ chars) to a file, no compact option. Robinhood's equivalents with `output` are strictly better. |
| `NEWS_SENTIMENT` | ⚠️ works, large | Use `limit` 5-10. **The only capability Robinhood has no equivalent for.** |
| `MARKET_STATUS` | ⚠️ quota-limited | Has a time-based fallback in `run_instructions.md`. |
| `TOP_GAINERS_LOSERS` | ⚠️ quota-limited | Untested successfully — every attempt so far hit the quota wall. |

### The quota is the real constraint
**25 requests/day, shared across every session using this key** — manual
tests and scheduled runs draw from the same pool. Confirmed exhausted on
2026-08-10 at both 05:15 and 08:30 UTC. Any run hitting an exhausted
quota loses news sentiment and top-gainers scouting; see the graceful
degradation rules in `strategy.md` and `run_instructions.md` — those
paths must never hard-block trading, or a free-tier quota limit silently
becomes a permanent no-trade bug.

### Leveraged ETFs verified (2026-08-10)
All four quote and return fundamentals normally through the standard
equity tools — no special handling needed. Their own descriptions confirm
the exposure, including the load-bearing word *daily*:

| Symbol | Description (verbatim from `get_equity_fundamentals`) | Price | 30d avg vol |
|---|---|---|---|
| SSO | "2x **daily** leveraged exposure" to the S&P 500 | $71.68 | 3.0M |
| SDS | "2x inverse exposure" to the S&P 500 | $53.35 | 2.7M |
| QLD | "2x leveraged exposure" to the Nasdaq-100 | $92.23 | 4.7M |
| QID | "2x inverse exposure" to the Nasdaq-100 | $13.94 | 16.1M |

Spreads are tight (SSO bid/ask $71.90/$71.91). Note `pe_ratio` and
`pb_ratio` are **null** for all of them — the value tiebreaker in
`strategy.md` cannot be computed for ETFs, which is fine because the
index sleeve is signaled off the underlying index's trend, not off the
cross-sectional value ranking.

Decay evidence captured the same day, useful for judging holding period:
SSO ran +47% off its March low ($48.63 → $71.68) while SDS fell −34%
($80.50 → $53.35) and printed its 52-week low three days prior. Daily
compounding rewards sustained trends and punishes chop — hence the
tighter 10-day time-stop in `gates.md`.

### Single-stock leveraged ETFs verified (2026-08-10)
All quote and return fundamentals through the standard equity tools.
Volumes are 30-day averages:

| Underlying | 2x ETF | Volume | 52wk high → current | Drawdown |
|---|---|---|---|---|
| TSLA | TSLL | 76M | $23.74 → $7.80 | −67% |
| COIN | CONL | 18.6M | $52.40 → $4.07 | **−92%** |
| NVDA | NVDL | 13.5M | $43.27 → $35.71 | −17% |
| MSFT | MSFU | 7.7M | $58.89 → $38.21 | −35% |
| META | METU | 5.4M | $51.20 → $21.00 | −59% |
| AMZN | AMZU | 3.3M | $47.14 → $41.88 | −11% |
| AAPL | AAPU | 2.2M | $48.89 → $39.56 | −19% |
| GOOGL | GGLL | 1.7M | $153.00 → $109.81 | −28% |

CONL's −92% is the load-bearing observation: Coinbase itself fell nowhere
near that. Decay on a 2x daily-reset fund scales with σ², so a
high-volatility underlying compounds losses through chop far faster than
an index does. This is why `gates.md` imposes an ATR ≤ 4% volatility gate
and a 7-day time-stop on the single-stock variant.

### ⚠ Scan indicators ≠ direct indicator calls (different session bounds)

**Verified 2026-08-11.** The screener's generated expressions carry
`session="all"` — e.g. `adx(candlePeriod="1d", candleCount=14,
session="all")` — so its indicators include extended-hours bars.
`get_equity_technical_indicators` defaults to `bounds="regular"`. Same
symbol, same period, same day, materially different values:

| Symbol | Scan ADX (`session="all"`) | Direct ADX (regular) |
|---|---|---|
| TECH | 59.71 | 72.03 |
| FTNT | 33.80 | 26.05 |

The gap runs in **both** directions, so it is not a constant offset that
could be calibrated away. **FTNT clears the counter-trend `ADX > 30` gate
on the scan value and fails it on the direct value.**

Rule now recorded in `gates.md`, `strategy.md` and `run_instructions.md`:
scan indicators are fine for **screening and ranking** (relative, coarse)
but must **never satisfy a gate** — gates re-measure with a direct call.
Assume the same session-bounds caveat applies to every other scan column
(RSI, volume, relative volume), not just ADX.

### ⚠⚠ Indicators are silently computed over **interpolated (fake) bars**

**Found 2026-08-13 on NBIS — the most dangerous data defect recorded here
so far, because it produces plausible-looking numbers rather than an
error.**

The daily bar feed backing `get_equity_technical_indicators` can lag the
quote/fundamentals feeds by a full session. When it does, it does **not**
return short — it emits a **synthetic gap-fill bar** that repeats the
prior close as OHLC with zero volume:

```
Aug 11: open 187.50, close 193.23, high 195.60, low 187.31, vol 22,528,729
Aug 12: open 193.23, close 193.23, high 193.23, low 193.23, vol 0, interpolated: true
```

NBIS actually closed Aug 12 at **$259.20** (+34%, 63.5M shares, on
earnings). `get_equity_quotes` and `get_equity_fundamentals` both had the
real session. The bar feed had none of it.

**Every indicator then computed perfectly — over fabricated input:**

| Indicator | Reported | Why it's wrong |
|---|---|---|
| `ATR(14)` | 21.749 | Flat bar → true range 0 → `23.4223 × 13/14 = 21.749` exactly. ATR **fell** on a $43-range day. |
| `RSI(2)` | 60.597 | Zero change halves both avg gain and avg loss → ratio unchanged → **byte-identical to the prior day**. |
| `RSI(14)` | 47.549 | Same mechanism, also byte-identical. |
| `EMA8` | 194.123 | Back-solving the EMA recurrence gives an implied close of **exactly $193.23** — the interpolated value. |

**The trap: `get_equity_technical_indicators` does not expose the
`interpolated` flag.** Only `get_equity_historicals` does. The indicator
response looks completely normal.

**Rule: before trusting any indicator value, call
`get_equity_historicals` for the same symbol/interval and confirm the
most recent bar is not `interpolated: true`.** If it is, treat every
indicator for that session as **UNAVAILABLE** — never as a real reading.

**Why this is load-bearing, not cosmetic.** `gates.md` sizes every
position off `ATR(14)`, and the mean-reversion sleeve enters on
`RSI(2) < 15` and exits on `RSI(2) > 70`. On an interpolated bar, ATR
decays toward zero (each flat bar multiplies it by 13/14), which would
**shrink stops and inflate position sizes** — and a frozen RSI(2) can
hold a stale entry or exit signal indefinitely. Both failure modes are
silent and would look like normal operation in the logs.

Note the mechanism also explains why a *quiet* symbol can look stable:
repeated interpolated bars produce smoothly decaying ATR and perfectly
flat RSI, which reads as "low volatility" rather than "no data."

### ⚠ Hourly bars have a *second, independent* corruption — not the same bug as the daily interpolation above

**Found 2026-08-14, also on NBIS, while trying to work around the daily
interpolation issue.** The instinct "daily bars are broken, try hourly"
does not work — hourly bars carry their own defect, and it is not the
interpolation pattern (no `interpolated` flag, real-looking nonzero
volume). Cross-checking a hearsay-hourly bar against the same window's
verified 5-minute bars found it silently wrong:

| | Hourly bar (`14:00 UTC` bucket) | True (summed from 5-minute bars) |
|---|---|---|
| Open | $267.15 | $268.90 |
| High | $267.49 | **$275.96** |
| Low | $266.71 | $264.11 |
| Volume | 22,074 | **3,812,141** (≈173x higher) |

The hourly bar understated volume by two orders of magnitude and missed
the session's actual high entirely — the real $275.96 print (visible
correctly in 5-minute bars) never surfaces in the hourly series at all.
An `RSI(14)` computed on the hourly series (attempted the same day, to
check for bearish divergence against the prior day's peak) therefore
**inherits this corruption** and cannot be trusted either, even though
nothing about the response looks suspicious on its own (no repeated
values, no impossible direction — the failure mode that made the daily
bug easy to catch by eye is absent here).

**Only 5-minute (and finer) bars have been verified clean so far.**
Daily and hourly are both compromised, via two different mechanisms.
**Before trusting any non-5-minute aggregation, spot-check it against
the 5-minute bars for the same window** — do not assume a bar's
plausibility from its shape alone; this one looked completely normal
and was still wrong by 173x on volume.

### ⚠ Robinhood's own consumer-app chart disagrees with the API's RSI(14) — same symbol, same day, ~30 points apart

**Found 2026-08-14, user cross-check on NBIS.** The user read `RSI(14)`
directly off the Robinhood app chart for Aug 12 and got **90.19**. The
same day, via `get_equity_technical_indicators` (this tool), the value
was **61.96** — after the daily-interpolation bug (above) had already
cleared, so the API reading was on real data, not a fabricated bar.

**First hypothesis (session bounds, like the ADX case above) — tested
and disproven.** Re-ran the same call with `bounds="extended"`: it
returned **the exact same value**, 61.963657798400945, byte-identical
to the `bounds="regular"` call. So unlike ADX, this indicator's API
value does not vary with the `bounds` parameter at all for this symbol —
the chart/API gap is caused by something else.

**More likely explanation, also not yet confirmed**: RSI has more than
one standard formulation. The classic **Wilder's RSI** (exponential/
Wilder smoothing of gains and losses — what this API almost certainly
computes) and **Cutler's RSI** (a simple moving average of gains and
losses) are both commonly called "RSI(14)" but can diverge meaningfully,
especially in the days right after a large price move, before Wilder's
smoothing has "settled" from its seed value. Consumer charting products
don't always disclose which variant they use. Not confirmed which
variant Robinhood's app chart uses — flagging as the leading candidate
since the bounds explanation is now ruled out.

**Practical conclusion, unchanged**: the size of the gap (30 points,
enough to flip a reading from "neutral" to "near-overbought") means any
single-source RSI reading should be treated with caution until
cross-checked. Do not assume "it's all Robinhood data" implies
agreement between the consumer app and this API — it demonstrably
doesn't, and the cause is still open.

### Scanner filters available but not yet used
The screener exposes an **options-flow filter group** that nothing in the
current strategy touches: `FILTER_TYPE_RELATIVE_OPTIONS_VOLUME` (unusual
options activity vs. that name's own baseline),
`FILTER_TYPE_TOTAL_CALL_VOLUME` / `TOTAL_PUT_VOLUME`,
`FILTER_TYPE_IMPLIED_VOLATILITY`, `FILTER_TYPE_OPEN_INTEREST_VOLUME`.
Unusual options volume is a well-known early-positioning signal built
entirely from public exchange data, and it's free here. Also unused:
`FILTER_TYPE_PEG` (the scanner has PEG even though
`get_equity_fundamentals` does not), `FILTER_TYPE_GAP` (premarket gaps —
directly useful to the premarket routine), `FILTER_TYPE_EARNINGS_DATE`
(could enforce the earnings blackout at screen time instead of per
candidate), and `FILTER_TYPE_SECTOR` (sector-diversification caps).

## Not yet verified

- `place_equity_order` / `place_option_order` — **deliberately never
  called.** `MODE: PAPER` forbids it. Do not "test" these.
- Robinhood `get_equity_historicals` — not used by the live funnel, but
  it's the path to a real backtest (up to 20 years of OHLCV bars).
- Alpha Vantage deep-dive tools (`INSTITUTIONAL_HOLDINGS`,
  `INSIDER_TRANSACTIONS`, `EARNINGS_CALL_TRANSCRIPT`) — reserved for
  manual deep dives, not the automated funnel.

## Macro data — available, deliberately not wired in

**Verified 2026-08-11:** Alpha Vantage's macro series work and are **not**
premium-gated — `TREASURY_YIELD` returned clean monthly data back to 1953.
Also present: `CPI`, `FEDERAL_FUNDS_RATE`, `UNEMPLOYMENT`, `REAL_GDP`,
`INFLATION`, `NONFARM_PAYROLL`, `RETAIL_SALES`. **This closes the "we need
FRED" gap with no new connector.**

It is still unused, for reasons worth recording so it doesn't get "fixed"
later:

- **Horizon mismatch.** Monthly series against a 5-15 day holding period.
  Yield-curve inversion is a quarters-long recession signal; it says
  nothing about the next two weeks.
- **Already covered, better.** The regime layer (SPY vs 200DMA/50DMA plus
  VIX) measures market stress directly and daily. Macro would be a slower,
  noisier proxy for something already measured well.
- **The one plausible use is unevidenced.** Daily 10-year yield spikes
  plausibly pressure high-multiple growth names — much of `ai_theme.md` —
  but any threshold would be invented, with *less* justification than the
  VIX levels, which were at least fitted to an observed distribution.

If a macro rule is ever added, pre-register it with a falsifiable trigger
like every other change here.

## Stocktwits — connected 2026-08-11, scoped narrowly

Connector added and verified live. Tools: `get_sentiment`,
`get_sentiment_history`, `get_message_volume`, `get_symbol_pulse`,
`get_trending_symbols`, `get_symbol_messages`. Read-only.

**Coverage does not match this system's universe.** Measured on real
candidates:

| Symbol | Message volume ("now") | Sentiment | Verdict |
|---|---|---|---|
| NVDA | 93,317 | score 61, 78.9% bull, 658K watchers | Rich |
| IMAX | 1,635 (1.7% of NVDA) | score 47 NEUTRAL — despite "100% bullish" | Marginal |
| **TECH** | — | **empty data array** | **None** |

TECH was the **#1-ranked candidate** on 2026-08-10 and has no Stocktwits
data at all. This is structural: the Tier 0 screener selects for ADX > 25
plus a $2B+ market cap, which surfaces mid-cap names in strong trends,
while retail chatter concentrates in mega-caps and meme stocks. Roughly a
third of a typical top-15 has meaningful coverage.

**The `bullish_pct` field is a trap on thin names.** IMAX returned
`bullish_pct: 100, bearish_pct: 0` — which reads as overwhelming
consensus but is computed from a handful of tagged messages. The
canonical `score` was 47 (neutral). The tool's own documentation calls
`bullish_pct` a "retained legacy tagged-message metric." **Use `score`
and `label`; never gate on `bullish_pct`.**

### Where it is used

1. **Premarket routine — catalyst identification.** This is the good fit.
   That routine already hunts for *why* a name gapped, currently via
   quota-limited Alpha Vantage news. Stocktwits posts answer that
   directly, and a gapping stock is far likelier to have chatter than an
   arbitrary screener hit. Also saves Alpha Vantage quota.
2. **Tier 3 — logged, never gating.** Record `score`, `label`, and
   message volume into `trade_log.csv` where coverage exists; write
   `NO_COVERAGE` otherwise. This measures whether the signal predicts
   anything before it is allowed to influence anything.

### Where it is deliberately not used

- **Not a gate, in either direction.** Coverage is too sparse and too
  correlated with "is this a retail favorite," which is not a quality we
  are trying to select for.
- **Not `get_trending_symbols` as a universe source.** That would pull
  the funnel toward meme names the market-cap, liquidity, and ADX filters
  deliberately exclude.
- **The hypothesis worth testing is contrarian.** Extreme bullish
  sentiment plus extreme message volume on a stock already at a 52-week
  high is a plausible distribution signature — the crowd arriving late.
  If sentiment ever earns a gating role, that is the direction the
  evidence would have to support. Wiring it as confirmation ("only buy
  when the crowd is bullish") would risk making entries *worse* at
  precisely the moments that matter.

## Reddit — still unavailable, assessed and deprioritized

The one capability neither connector provides. Three routes, all with real
obstacles:

1. **A claude.ai connector** is the only route that reaches the scheduled
   routines, since they resolve data sources by `connector_uuid`. Needs
   checking in claude.ai settings; existence unknown.
2. **A locally-configured MCP server does not work.** It exists only in an
   interactive session; cloud routines see only what is attached via
   `mcp_connections`. Wiring one locally would make sentiment queryable by
   hand while leaving every routine blind — a failure mode that looks like
   success.
3. **Direct HTTP from `Bash`** is possible in principle but blocked on
   credentials: Reddit has required OAuth since 2023 and blocks
   unauthenticated cloud IPs, StockTwits requires auth, and there is no
   clean secret-holding mechanism for routines. Committing keys to the
   repo is not an acceptable workaround.

**Deprioritized on merit, not only access.** Social sentiment is the
noisiest input in this stack, frequently contrarian, concentrated in meme
names the market-cap and ADX filters already exclude, and pitched at a
far shorter horizon than a 5-15 day swing.

**The binding constraint is the Alpha Vantage quota, not the absence of
social data.** The more reliable news sentiment already exists and went
dark on every Tier 3 candidate on 2026-08-10 — which also silently
disables counter-trend trading, since those require undegraded inputs.
Upgrading that plan buys more than either new source would.
