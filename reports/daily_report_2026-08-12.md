# Daily Report — 2026-08-12

## Summary

Long regime all day (SPY held well above SMA200 throughout). Three runs executed: a monitor run (14:05 UTC), a midday scan (18:38 UTC), and a close scan (19:45 UTC) — no 9:30am ET scan ran today, only the hourly position monitor covered that slot (see Watch items). One position closed (PAYO, Sleeve A, `trend_break`, realized -$2.11 / -0.123R), one position opened (CVS, Sleeve B, `mr_reversal`). Realized P&L today: **-$2.11**. Unrealized P&L on the open book at close: **+$7.39** (ACAD +$5.63, CVS +$1.76). NAV rose $9,991.14 → $10,005.28 (+$14.14, +0.14%) despite the small realized loss, driven by unrealized gains on both open names.

> **Two sleeves run over one book.** Sleeve A (breakout, `sleeve=stock`/`index`) and Sleeve B (mean reversion, `sleeve=mean_reversion`) are separate strategies with opposite payoff structures and are reported separately throughout — no blended win rate or average R.

## Account

| Field | Open of day | Close of day | Change |
|---|---|---|---|
| NAV | $9,991.14 | $10,005.28 | +$14.14 (+0.14%) |
| Cash | $7,174.85 | $7,912.04 | +$737.19 |
| Open positions | 2 (ACAD, PAYO) | 2 (ACAD, CVS) | PAYO closed, CVS opened |
| Daily-loss halt tripped? | No | No | — |

## Positions opened today

**Sleeve A (breakout):** None. Best Tier-1 survivors (CDNA, OGN, FBP, ATAI, LIND, AMLX, and counter-trend candidate BXMT) all failed Tier 2 — no level trigger, and volume confirmation (`relative_volume` ≥1.4) failed on every name in both the midday and close runs. One name, **CIB**, did trigger (`momentum_vol`) and pass Tier 3, but was rejected at execution when the bid/ask spread widened to 0.706% of mid (over the 0.5% liquidity cap) — see Rejections below.

**Sleeve B (mean reversion):**

| Symbol | Trigger | Direction | Qty | Fill | Stop | Entry RSI(2) | Thesis |
|---|---|---|---|---|---|---|---|
| CVS | `mr_reversal` | bullish | 8 | $94.85 | $90.35 | 0.560 (sub-10 cohort) | Deeply oversold RSI(2) print (0.56) on 2026-08-11 inside an intact long-term uptrend (price > SMA200), confirmed first up-close, non-negative/positive-today news sentiment. First fill in this sleeve since it was added 2026-08-12. |

## Positions closed today

**Sleeve A (breakout):**

| Symbol | Exit rule | Exit price | Realized P&L | R-multiple | Days held |
|---|---|---|---|---|---|
| PAYO | `trend_break` (close < SMA20) | $7.09 (bid) | -$2.11 | -0.123R | 1 |

PAYO's exit was mechanically caused by the 2026-08-11 rule change (stock-sleeve trend MA moved from SMA50 to SMA20, applied retroactively to open positions) — the 2026-08-11 close of $7.09 sat below the new SMA20 of $7.12. PAYO remained above the SMA50 ($6.80) it was originally entered under; original stop ($7.019) was not breached.

**Sleeve B (mean reversion):** No closed trades today (no positions were open in this sleeve prior to today).

## Open book

| Symbol | Sleeve | Entry | Current | Unrealized P&L | Stop | Distance to stop | R-target / MR-target | Distance |
|---|---|---|---|---|---|---|---|---|
| ACAD | stock | $29.49 (2026-08-11) | $29.615 | +$5.63 | $28.6053 | -3.4% | $31.2594 | +5.6% to target |
| CVS | mean_reversion | $94.85 (2026-08-12) | $95.07 | +$1.76 | $90.35 | -4.9% | N/A (RSI(2)>70 mr_target) | RSI(2) not yet measurable — entered today, no completed session since |

Neither position is near its stop; neither has a trend-break or time-stop condition pending (ACAD day 2 of 10, CVS time-stop clock not yet started).

## Rejections worth noting

**Sleeve A (breakout) — logged to trade_log.csv:**
- **CIB** — cleared Tier 1 (trend template) and Tier 2 (`momentum_vol` trigger, rel_volume 2.600) and Tier 3 (news sentiment +0.154). Rejected at **execution**: fresh quote at fill time showed a 0.706%-of-mid spread, over the 0.5% liquidity cap — plausible still-settling post-earnings volatility in a Colombian ADR. This is the spread/chase-protection gate doing its documented job, not a funnel miscalibration.

**Sleeve A (breakout) — funnel declines, not CSV-logged (Tier 1/2 non-gate declines, narrated in trade_journal.md):**
- **Tier 1 (trend template) failures** across both runs: PAY, PYPL, AWI, SAIA, KLAC — mixed/broken SMA stacks (SMA50 below SMA200 while price sits above SMA20, or vice versa).
- **Tier 2 (volume confirmation) failures** — the binding constraint this run: CDNA, OGN, FBP, ATAI, LIND, AMLX all passed Tier 1 but failed on `relative_volume` < 1.4 in both the midday and close runs (values ranged 0.19–1.10). OGN came closest to a level trigger (`breakout_7d` missed by $0.005) but would still have failed volume confirmation.
- **Counter-trend candidate BXMT** passed the structural counter-trend checks (ADX 46.91 > 30, relative-weakness well inside the stricter bar) both runs but never reached a trigger — no breakdown through the Donchian lower channel, and volume confirmation failed (0.944).
- **Index sleeve (SSO)** failed Tier 2 volume confirmation in both runs (rel_volume 0.755) — now 6+ consecutive runs failing on this same gate. Standing observation, not yet acted on (no defined adaptation trigger for this specific gate).
- **Earnings blackout**: TECH and EAT dropped both runs (both reported this morning, inside the 3-day window; note both reports had already resolved by the midday run — the gate doesn't distinguish already-reported from upcoming and isn't overridden on discretion).

**Sleeve B (mean reversion) — funnel declines, not CSV-logged:**
- **Tier 1-MR (price > SMA200) failures**: GOLF, AMX, BHF — all genuinely below their 200-day trend (falling-knife exclusion working as intended), consistent across both runs.
- **Tier 2-MR (`mr_reversal` up-close) failure**: PGNY qualified on RSI(2) and passed Tier 1-MR both runs, but never printed an up-close versus its oversold session's close — trigger did not fire either run.

## Data quality

- `run_scan` returned 200 of 396 total matches again today (response cap unchanged, consistent with every run this week) — the funnel only ever sees the top 200 by scan order.
- Premarket: `NEWS_SENTIMENT` calls for DFTX, GFS, POWL, and PCTY hit the Alpha Vantage rate limit (concurrent-request burst) and returned no data; not rechecked, to preserve quota for the 9:30 scan slot.
- Stocktwits/social sentiment unavailable both times it was needed today (CIB, CVS) — connector not authenticated this session, logged `NO_COVERAGE`.
- CIB's `get_earnings_results` showed `actual` still `null` in Robinhood's feed for the 2026-08-10 report despite price action and bullish coverage indicating a beat already happened — a data-lag artifact, not treated as a blackout conflict.
- No 9:30am ET scan ran today — only the hourly position monitor covered that slot (see Watch items).

## Pre-registered experiment tracking

**1. `momentum_vol` vs `breakout_7d`.** 0 closed `momentum_vol` trades so far (CIB triggered `momentum_vol` today but was rejected at execution before opening, so it does not count). Threshold is ≥10 closed trades — not close to being met. No comparison possible yet.

**2. RSI(2) < 15 threshold (raised from 10 on 2026-08-12).** 0 closed mean-reversion trades (CVS opened today with entry RSI(2)=0.560, sub-10 cohort, but is still open). Threshold is ≥10 closed trades with ≥4 in each cohort — not met. No cohort comparison possible yet, and per instructions, CVS's status as an open in-progress position is not cited as support either way.

**3. Momentum test reinstatement (retired EMA-spread test, logged non-gating as `momentum_test_would_pass`).** 1 closed trade all-time (PAYO, deduplicated by symbol/date), with `momentum_test_would_pass=false` at entry and r_multiple -0.123R. Reinstatement threshold is ≥20 closed trades and a ≥0.3R advantage for the passing group — nowhere near met (1 of 20). No split-average is meaningful at n=1; not proposing any change.

## Watch items

- **No scan ran in the 9:30am ET slot today** — the 14:05 UTC run was logged as a "monitor run" (position review + PAYO's `trend_break` exit) rather than a full scan, unlike the "morning run" scans on 2026-08-10 and 2026-08-11. Worth confirming this was intentional scheduling rather than a missed run.
- **Index sleeve (SSO) volume-confirmation gate has now failed 6+ consecutive runs.** Still a standing observation without a defined adaptation trigger — flagging again in case a human wants to review the 1.4 threshold or the sleeve's participation.
- **CVS (Sleeve B)** is the first mean-reversion fill since the sleeve was added today — nothing actionable yet, but it's the first live data point toward the RSI(2) threshold experiment above.
- **CIB** cleared every analytical gate and was only stopped by execution-time spread widening — a clean illustration of the chase-protection/liquidity checks working, not a funnel issue.

---

Paper trading — simulated book, no real orders placed. P&L charges the bid-ask spread but omits fill uncertainty, slippage, options fees, ETF expense ratios, and dividends; expect live results to run modestly worse. See `DATA_SCHEMA.md`.
