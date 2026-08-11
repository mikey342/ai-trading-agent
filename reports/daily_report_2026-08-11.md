# Daily Report — 2026-08-11

## Summary

Regime stayed LONG/risk-on all day (SPY comfortably above its 200DMA, VIX
<20 → NORMAL risk level). Three scheduled runs executed (morning 14:01
UTC, midday 18:45 UTC, close 19:42 UTC) plus a partial data-quality
recovery from yesterday's full-day Alpha Vantage quota exhaustion — all
three runs today had live `MARKET_STATUS` and full data access. The
midday run produced this system's **first-ever fills**: two new
with-trend long positions, ACAD and PAYO, both breakout_52w setups
confirmed on volume. No positions were closed. Realized P&L today: $0.
Unrealized P&L on the two new positions at the close run: **-$8.86**
(ACAD -$6.75, PAYO -$2.11) — both essentially flat, well inside normal
noise and nowhere near either position's stop.

## Account

| Field | Open of day | Close of day | Change |
|---|---|---|---|
| NAV | $10,000.00 | $9,991.14 | -$8.86 (-0.09%) |
| Cash | $10,000.00 | $7,174.85 | -$2,825.15 (deployed into ACAD + PAYO) |
| Open position count | 0 | 2 | +2 |
| Daily-loss halt tripped? | No | No | — |

## Positions opened today

- **ACAD** — stock sleeve, bullish, with-trend, 45 shares @ $29.49, stop
  $28.6053 (3.00% away), R-target $31.2594 (2R). Thesis: fresh 52-week-high
  breakout on confirmed above-average volume (rel_vol 1.565), backed by
  the cheapest positive P/E (11.66) and the strongest recent earnings beat
  (5 of last 7 quarters, incl. a +350% surprise on 2026-08-04) in the
  run's candidate set — the cleanest with-trend long the funnel has
  produced yet.
- **PAYO** — stock sleeve, bullish, with-trend, 211 shares @ $7.10, stop
  $7.0190 (1.14% away), R-target $7.2620 (2R). Under-risked at the 15%
  position cap (actual risk ~0.17% of NAV vs. the 0.4% target — no
  leveraged ETF exists for this name). Thesis: cleared every mechanical
  gate (volume-confirmed breakout, rel_vol 1.513) but weaker on every soft
  signal than ACAD — decelerating momentum, a recent EPS miss (swung to a
  loss on 2026-08-06), and news sentiment that is technically neutral
  (-0.002) but skewed bearish in its most recent articles. Taken because
  the gates are the gates; flagged in the journal as the name to watch
  first for an early exit signal.

## Positions closed today

None — no positions were open before the midday run's fills.

## Open book

| Symbol | Entry | Current (close run) | Unrealized P&L | Stop | Dist. to stop | R-target | Dist. to R-target |
|---|---|---|---|---|---|---|---|
| ACAD | $29.49 | $29.34 | -$6.75 | $28.6053 | -2.5% | $31.2594 | +6.6% |
| PAYO | $7.10 | $7.09 | -$2.11 | $7.0190 | -1.0% | $7.2620 | +2.4% |

Neither position has hit its stop or R-target; no trend-break or
time-stop conditions apply (both entered today).

## Rejections worth noting

- **Counter-trend ADX gate (TSLA, morning run):** failed at 29.9037,
  just under the 30 threshold. This is the second day in a row TSLA has
  landed right at this boundary — 30.56 (pass) on 2026-08-10, 29.90
  (fail) today. Worth watching: a setup that keeps landing within ~2% of
  a hard threshold either way is a candidate for review once there's more
  data, not yet actionable.
- **Tier 2 volume confirmation — the dominant rejection reason today.**
  Across the midday and close runs, 6-8 candidates per run (TECH, CDNA,
  OGN, FBP, ATAI, LXP, and — newly tested at close — LIND, CRWD) cleared
  the price/EMA breakout condition but failed the ≥1.4 relative-volume
  bar (readings 0.17-0.996×). The journal frames this as the gate "doing
  real work" (rejecting weak-participation breakouts), but it is worth
  tracking whether this is a market-wide low-volume day or a
  systematically tight threshold, per the note below.
- **Index sleeve (SSO):** failed Tier 2 volume confirmation on **all
  three runs today** (relative_volume 0.798, unchanged all day since it's
  the same completed Aug-10 session bar) — the 3rd/4th/5th consecutive
  run this gate has blocked SSO since the trend-template fix. It has
  never once cleared Tier 2. Flagged again in the journal as worth a
  deliberate look at whether the 1.4 threshold (calibrated for
  single-stock breakouts) fits a structurally less-bursty index ETF.
- **Tier 1 trend template:** RKLB (morning, broken SMA50/200 stack);
  KLAC, PAY, PYPL, SAIA (midday and close, broken/wrong-order stack,
  re-confirmed on live price at close without re-pulling fundamentals).
- **Earnings blackout:** EAT dropped both midday and close (reports
  2026-08-12).
- **Tier 2, no trigger (morning):** PBF, DK — both >11% off their 52-week
  highs, RSI outside the 35-45 pullback band.

## Data quality

- Alpha Vantage was fully available on all three runs today (`MARKET_STATUS`
  succeeded each time) — a recovery from yesterday's full-day 25-request
  quota exhaustion that blocked most of 2026-08-10's runs.
- `Stocktwits get_sentiment` remains unavailable — the connector is not
  authenticated this session. Logged as `NO_COVERAGE` (not a data gap on
  Stocktwits' end) for both ACAD and PAYO at Tier 3.
- No other degraded inputs, failed calls, or market-closed fallbacks
  today.

## Open-question tracking

**Momentum-test reinstatement** (`momentum_test_would_pass`, logged
non-gating since 2026-08-11): 0 trades have closed to date — both of
today's fills (ACAD, PAYO) remain open. The pre-registered reinstatement
threshold (≥20 closed trades, and a ≥0.3R average advantage for the
passing group) is **not met** and cannot be evaluated until trades start
closing. No reinstatement proposed.

**Momentum-test retirement provenance question** (≥15 candidates across
≥10 sessions with ≥90% rejected): per `strategy.md`, this was already
**resolved on 2026-08-11** — the threshold was never met (5 names across
2 sessions when the test was retired), and the retirement was made on
provenance (the test was invented, not researched) rather than on hitting
this bar. No longer an open tracking item; noted here for completeness
only.

## Watch items

- **PAYO** is the position most likely to need an early exit call —
  decelerating momentum, a recent earnings miss, and bearish-leaning
  recent news, despite clearing every mechanical gate. Worth a closer
  look at tomorrow's position review.
- **Index sleeve (SSO) volume gate**, now 5-for-5 failing since the
  trend-template fix — a systematic, not one-off, pattern worth a
  deliberate review of whether the 1.4 relative-volume threshold fits an
  index ETF.
- **TSLA counter-trend ADX gate** landed within ~2% of the 30 threshold
  on both 2026-08-10 (pass) and 2026-08-11 (fail) — a boundary case worth
  a second look once more samples exist.
- **Judgment call, not yet codified:** the close run excluded ACAD/PAYO
  (already open) from new-entry Tier 1-3 ranking rather than re-running
  them, since this system has no pyramiding mechanism. Reasonable, but
  `run_instructions.md`/`strategy.md` don't explicitly say so yet — worth
  making explicit.
- No trend-break exits, time-stops, or counter-trend positions in
  progress today.

---

> Paper trading — simulated book, no real orders placed. P&L charges the
> bid-ask spread but omits fill uncertainty, slippage, options fees, ETF
> expense ratios, and dividends; expect live results to run modestly
> worse. See `DATA_SCHEMA.md`.
