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

## Not yet verified

- `place_equity_order` / `place_option_order` — **deliberately never
  called.** `MODE: PAPER` forbids it. Do not "test" these.
- Robinhood `get_equity_historicals` — not used by the live funnel, but
  it's the path to a real backtest (up to 20 years of OHLCV bars).
- Alpha Vantage deep-dive tools (`INSTITUTIONAL_HOLDINGS`,
  `INSIDER_TRANSACTIONS`, `EARNINGS_CALL_TRANSCRIPT`) — reserved for
  manual deep dives, not the automated funnel.
