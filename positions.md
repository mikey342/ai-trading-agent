# Positions — Current Virtual (Paper) Book

This is the live state of the simulated portfolio. Update it every run.
Do not touch real Robinhood positions/orders — this book is entirely
separate and simulated.

## Account summary

| Field | Value |
|---|---|
| Starting NAV | $10,000 |
| Current cash (simulated) | $7,174.85 |
| Current NAV (simulated) | $9,991.14 |
| Realized P&L (all-time) | $0 |
| Today's realized P&L | $0 |
| Trading halted today? | No |
| Last updated | 2026-08-11 19:42 UTC (close run; no new trades, no exits — see trade_journal.md) |

## Open positions — equity and leveraged ETFs

| Symbol | Underlying | Sleeve | Direction | Counter-trend? | Qty | Entry price | Entry date | Original stop | R-target | R-target reached? | Current exit stop | Current price | Unrealized P&L | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| ACAD | ACAD | stock | bullish | No | 45 | $29.49 | 2026-08-11 | $28.6053 | $31.2594 | No | $28.6053 | $29.34 | -$6.75 | Health Technology. breakout_52w, vol-confirmed (rel_vol 1.565). PEAD tailwind (beat 5/7 qtrs, incl. most recent). |
| PAYO | PAYO | stock | bullish | No | 211 | $7.10 | 2026-08-11 | $7.0190 | $7.2620 | No | $7.0190 | $7.09 | -$2.11 | Commercial Services. breakout_52w, vol-confirmed (rel_vol 1.513). Under-risked (cap-bound, no leveraged ETF available). Soft-negative PEAD (missed 6/7 qtrs, incl. most recent) and mixed/slightly-negative recent news — watch closely. |

`Underlying` is the symbol the **thesis** is about, and the one whose
SMA(50) the trend-break exit checks. For a plain stock it equals
`Symbol`. For a leveraged ETF it is the underlying stock or index
(NVD → NVDA, SDS → SPY): the ETF's own moving averages are distorted by
leverage and daily resets, so they cannot be used for the trend test.
Stops and R-targets, by contrast, are always tracked on the **instrument
actually held**, since that is what was sized and filled.

`Direction` is `bullish` or `bearish` (a bearish position is a *long*
position in an inverse ETF — never a short). `Counter-trend?` marks a
setup taken against the SPY regime; **at most one may be open at a time**,
and it is sized at half risk. Check this column before opening another.

`Sleeve` is `stock` (individual name from the Tier 0-3 funnel) or `index`
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
