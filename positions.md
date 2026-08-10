# Positions — Current Virtual (Paper) Book

This is the live state of the simulated portfolio. Update it every run.
Do not touch real Robinhood positions/orders — this book is entirely
separate and simulated.

## Account summary

| Field | Value |
|---|---|
| Starting NAV | $25,000 |
| Current cash (simulated) | $25,000 |
| Current NAV (simulated) | $25,000 |
| Realized P&L (all-time) | $0 |
| Today's realized P&L | $0 |
| Trading halted today? | No |
| Last updated | 2026-08-10, scheduled premarket run (data-blocked, see `trade_journal.md`) |

## Open positions — equity

| Symbol | Qty | Entry price | Entry date | Original stop | R-target | R-target reached? | Current exit stop | Current price | Unrealized P&L | Leverage used? | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|
| _(none yet)_ | | | | | | | | | | | |

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

| Date | Run type | NAV | Cash | Realized P&L (that run) | Halted that day? |
|---|---|---|---|---|---|
| 2026-08-10 | premarket (manual test) | $25,000 | $25,000 | $0 | No |
| 2026-08-10 | premarket (scheduled, data-blocked) | $25,000 | $25,000 | $0 | No |
