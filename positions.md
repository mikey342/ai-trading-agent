# Positions — Current Virtual (Paper) Book

This is the live state of the simulated portfolio. Update it every run.
Do not touch real Robinhood positions/orders — this book is entirely
separate and simulated.

## Account summary

| Field | Value |
|---|---|
| Starting NAV | $10,000 |
| Current cash (simulated) | $7,705.29 |
| Current NAV (simulated) | $9,958.39 |
| Realized P&L (all-time) | -$42.61 |
| Today's realized P&L | -$40.50 |
| Trading halted today? | No |
| Last updated | 2026-08-13 18:45 UTC (midday run; 0 opened, 1 closed (ACAD, stop-out) — see trade_journal.md) |

## Open positions — equity and leveraged ETFs

| Symbol | Underlying | Sleeve | Direction | Counter-trend? | Qty | Entry price | Entry date | Original stop | R-target | R-target reached? | Current exit stop | Current price | Unrealized P&L | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| CVS | CVS | mean_reversion | bullish | No | 8 | $94.85 | 2026-08-12 | $90.35 | N/A (MR sleeve) | N/A | $90.35 | $94.77 | -$0.64 | Retail Trade. mr_reversal, entry rsi2=0.560 (Aug 11 session, sub-10 cohort). No volume confirmation required (MR sleeve). Stop = 1.5xATR14 (MR sleeve exempt from 3% ceiling). Exits: stop -> mr_target (RSI2>70) -> trend_break (close<SMA200) -> 5-day time-stop. SMA200 $85.2356 as of 2026-08-13 midday run -- no trend_break. RSI(2) on last completed session (Aug 12) = 44.59, not >70 -- mr_target not yet fired. 1 trading day into the 5-day time-stop clock. |
| OGN | OGN | stock | bullish | No | 109 | $13.70 | 2026-08-13 | $13.62975 | $13.8405 | No | $13.62975 | $13.715 | $0.17 | Health Technology. momentum_vol (no level test -- price missed the prior-bar Donchian-7 upper by $0.005), vol-confirmed (rel_vol 2.178). News sentiment +0.191 (Somewhat-Bullish, Sun Pharma acquisition is the dominant catalyst). Most recent earnings (2026-07-31) was an EPS miss, not a PEAD tailwind. Position capped by the 15%-of-NAV rule (shares_risk 569 >> shares_cap 109) -- under-risked at ~0.08% NAV vs the 0.4% target, plain equity per the instrument-selection rule (no single-stock leveraged ETF mapped to OGN). SMA20 (stock trend MA) $13.554 as of 2026-08-13 midday run -- no trend_break. Entered today, time-stop clock not yet started. |

`Underlying` is the symbol the **thesis** is about, and the one whose
trend MA the trend-break exit checks (SMA20 for the stock sleeve, SMA50
for the index sleeve — see `strategy.md`). For a plain stock it equals
`Symbol`. For a leveraged ETF it is the underlying stock or index
(NVD → NVDA, SDS → SPY): the ETF's own moving averages are distorted by
leverage and daily resets, so they cannot be used for the trend test.
Stops and R-targets, by contrast, are always tracked on the **instrument
actually held**, since that is what was sized and filled.

`Direction` is `bullish` or `bearish` (a bearish position is a *long*
position in an inverse ETF — never a short). `Counter-trend?` marks a
setup taken against the SPY regime; **at most one may be open at a time**,
and it is sized at half risk. Check this column before opening another.

`Sleeve` is `stock` (breakout funnel), `mean_reversion` (oversold-bounce
sleeve — **different exit rules**: no R-target flag, no EMA21 trail;
exits on RSI(2) > 70, a close below SMA(200), or a 5-day time-stop; max
2 open, LONG regime only), or `index`
(a 2x ETF: SSO/QLD long, SDS/QID inverse). Index-sleeve positions are
capped at **one at a time**, count **2x** toward gross exposure, and use
an **8-trading-day** time-stop instead of 10 — see `gates.md`.

Everything here is bought with cash. There is no margin column because
margin is disabled outright, and no short column because short selling is
forbidden — bearish exposure is a *long* position in an inverse ETF.

`R-target reached?` is a one-way flag (No → Yes, never back to No — see
`strategy.md` exit rules). `Current exit stop` = original stop until the
flag flips to Yes, then the trailing EMA21 (never lower than the original
stop).

## Open positions — options

| Symbol | Structure | Strike(s) | Expiration | Delta at entry | Entry premium | Entry date | Underlying @ entry | Underlying stop | Underlying R-target | R-target reached? | Current underlying price | Current premium | Unrealized P&L | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| _(none yet)_ | | | | | | | | | | | | | | |

`Underlying stop`/`Underlying R-target` are fixed at entry from the same
ATR-based sizing math used for the equity version of this signal (see
`strategy.md`) — never recomputed later, so exit decisions stay
consistent with what was actually decided at entry. `R-target reached?`
is a one-way flag, same semantics as the equity table.

## NAV history (append one row every run — needed for future performance metrics)

> Note: the 2026-08-10 05:15 UTC row below predates the account-size
> finalization (was $25,000, now $10,000 — see `gates.md`). Left as-is
> rather than edited, since no trades/P&L occurred under the old number;
> all NAV figures from this point forward are based on $10,000.

| Date | Run type | NAV | Cash | Realized P&L (that run) | Halted that day? |
|---|---|---|---|---|---|
| 2026-08-10 (premarket, 05:15 UTC) | premarket | $25,000 | $25,000 | $0 | No |
| 2026-08-10 (off-schedule, 05:54 UTC) | off-schedule (market closed) | $10,000 | $10,000 | $0 | No |
| 2026-08-10 (off-schedule, 09:18 UTC) | off-schedule (premarket) | $10,000 | $10,000 | $0 | No |
| 2026-08-10 (morning, 13:57 UTC) | morning | $10,000 | $10,000 | $0 | No |
| 2026-08-10 (midday, 18:42 UTC) | midday | $10,000 | $10,000 | $0 | No |
| 2026-08-10 (close, 19:41 UTC) | close | $10,000 | $10,000 | $0 | No |
| 2026-08-11 (morning, 14:01 UTC) | morning | $10,000 | $10,000 | $0 | No |
| 2026-08-11 (midday, 18:45 UTC) | midday | $10,000.00 | $7,174.85 | $0 | No |
| 2026-08-11 (close, 19:42 UTC) | close | $9,991.14 | $7,174.85 | $0 | No |
| 2026-08-12 (monitor, 14:05 UTC) | monitor | $9,997.44 | $8,670.84 | -$2.11 | No |
| 2026-08-12 (midday, 18:38 UTC) | midday | $10,000.14 | $7,912.04 | $0 | No |
| 2026-08-12 (close, 19:45 UTC) | close | $10,005.28 | $7,912.04 | $0 | No |
| 2026-08-13 (morning, 13:46 UTC) | morning | $10,007.03 | $6,418.74 | $0 | No |
| 2026-08-13 (midday, 18:45 UTC) | midday | $9,958.39 | $7,705.29 | -$40.50 | No |
