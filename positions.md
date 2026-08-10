# Positions — Current Virtual (Paper) Book

This is the live state of the simulated portfolio. Update it every run.
Do not touch real Robinhood positions/orders — this book is entirely
separate and simulated.

## Account summary

| Field | Value |
|---|---|
| Starting NAV | $10,000 |
| Current cash (simulated) | $10,000 |
| Current NAV (simulated) | $10,000 |
| Realized P&L (all-time) | $0 |
| Today's realized P&L | $0 |
| Trading halted today? | No |
| Last updated | 2026-08-10 (account size finalized to $10,000, see gates.md; zero trades/P&L to date so no historical inconsistency) |

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

> Note: the 2026-08-10 05:15 UTC row below predates the account-size
> finalization (was $25,000, now $10,000 — see `gates.md`). Left as-is
> rather than edited, since no trades/P&L occurred under the old number;
> all NAV figures from this point forward are based on $10,000.

| Date | Run type | NAV | Cash | Realized P&L (that run) | Halted that day? |
|---|---|---|---|---|---|
| 2026-08-10 (premarket, 05:15 UTC) | premarket | $25,000 | $25,000 | $0 | No |
