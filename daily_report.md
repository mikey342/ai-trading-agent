# Daily Report — 2026-08-10

## Summary
Regime was LONG/risk-on all day (SPY above its 50- and 200-day SMAs throughout). Three scheduled scan runs executed (morning 13:57 UTC, midday 18:42 UTC, close 19:41 UTC), plus three earlier off-schedule/premarket runs before the market opened. Zero positions opened, zero positions closed. Realized P&L: $0. Unrealized P&L: $0 (no open book). Candidates reached Tier 2/3 in every scheduled run (TSLA counter-trend at Tier 2 in the morning; TECH/TXG at Tier 3 midday; TECH/IMAX at Tier 3 at close) but none cleared the final bar — no gate was loosened to force a fit.

## Account
| Field | Open of day | Close of day | Change |
|---|---|---|---|
| NAV | $10,000 | $10,000 | $0 |
| Cash | $10,000 | $10,000 | $0 |
| Open position count | 0 | 0 | 0 |
| Daily-loss halt tripped? | No | No | — |

## Positions opened today
None. Most common rejection reason: candidates that passed Tier 1 (trend template) and Tier 2 (breakout trigger) failed Tier 3 momentum confirmation — the EMA8−EMA21 spread was not strictly widest at the most recent session (decelerating or rolling-over breakouts). TSLA passed Tier 1 as a counter-trend short but never got an entry trigger at Tier 2.

## Positions closed today
None — no positions were open.

## Open book
Empty — no open equity, options, or index-sleeve positions.

## Rejections worth noting
- **Tier 3 momentum confirmation (4 logged rejects, all bullish breakouts):** TECH (rejected twice, midday and close — same indicator readings both times since no new daily bar rolled between runs), TXG, and IMAX. All four passed Tier 1 trend template and Tier 2 breakout trigger, then failed because the 5-session EMA8−EMA21 spread was not monotonically widening into the most recent session — i.e. decelerating or rolling-over breakouts. This is the only rejection reason that produced logged CSV rows today; worth watching over time since it's currently a 100% kill rate on Tier 2 survivors.
- **SSO index sleeve rule-3 block (3 occurrences, not logged to CSV — pre-Tier-1 observation):** every run today, SPY passed the trend-template stack and the "within 25% of 52wk high" test but failed "at least 25% above 52wk low" — SPY's current 52-week range is only ~19–23% top-to-bottom (low set 2026-03-30), so that specific rule is structurally unsatisfiable at the current range regardless of trend strength. Not yet a strategy-adaptation trigger (policy requires a pattern across ≥5 of the last 10 *closed* trades, and there are zero closed trades), but flagged in the journal each time it recurred. Worth a deliberate look once there's a real track record, since it has blocked the index sleeve on literally every run the system has made.
- **TSLA counter-trend, Tier 2 (morning run, not logged to CSV per schema — didn't reach Tier 3):** passed the trend template as a bearish/counter-trend setup and cleared all 5 counter-trend gate checks (ADX 30.56, 52wk-range 0.152), but neither the breakdown trigger nor the rally-to-resistance trigger fired at Tier 2, so it was dropped rather than force-fit.
- Tier 1 earnings-blackout drops: RKLB (morning), AAON/EAT/JD (midday), LEGN/EAT/JD (close) — routine, not a filter-calibration concern.

## Data quality
- **Alpha Vantage `MARKET_STATUS` failed on every run today** (shared 25/day quota exhausted) — same recurring pattern noted in every journal entry. Market-open/closed state was determined via a time-based fallback each time, cross-checked against Robinhood `get_equity_quotes` live trade timestamps.
- **`news_sentiment` recorded UNAVAILABLE for all 4 Tier 3 candidates** today (TECH x2, TXG, IMAX) — quota exhaustion confirmed via the MARKET_STATUS failure, so the Tier 3 news call was skipped rather than retried (would fail identically). Non-gating in each case since all four were already dropped on momentum grounds.
- Premarket notes (2026-08-10) show the scan universe was thin pre-open (only COO), with news unavailable for COO's premarket move as well — same quota issue.
- No other degraded inputs identified; equity price/indicator data (SMA/EMA/RSI/ADX/ATR) came through normally via Robinhood on all runs.

## Watch items
- Tier 3 momentum confirmation has rejected every single Tier 2 survivor today (4 for 4) — not yet a pattern per the ≥5-of-10-closed-trades adaptation bar (zero closed trades exist), but worth tracking as real trades start closing.
- SSO index-sleeve rule-3 (25%-above-low52) has blocked every run since the system went live, purely due to SPY's currently compressed 52-week range — a structural condition, not a signal problem. No strategy.md change made; flagged for human attention if it persists.
- Alpha Vantage's shared 25/day quota is being exhausted early in the day on every run — the system is currently relying entirely on Robinhood for market-status and news is going dark for every candidate that reaches Tier 3. If this quota constraint doesn't change, it's worth confirming the fallback path is acceptable long-term.
- No positions open, no counter-trend trade in progress, no strategy.md adaptation made today.

> Paper trading — simulated book, no real orders placed. P&L charges the
> bid-ask spread but omits fill uncertainty, slippage, options fees, ETF
> expense ratios, and dividends; expect live results to run modestly
> worse. See `DATA_SCHEMA.md`.
