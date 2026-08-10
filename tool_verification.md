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
