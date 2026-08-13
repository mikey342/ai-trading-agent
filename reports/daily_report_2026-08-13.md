# Daily Report — 2026-08-13

## Summary

Regime: LONG / NORMAL all day (SPY held above SMA200 and SMA50, VIX ~14.5-14.7). Four runs executed: morning (13:46 UTC), midday (18:45 UTC), close (19:42 UTC), plus a 00:30 UTC operator note (not a run — see Data quality). 1 position opened (OGN, Sleeve A), 1 position closed (ACAD, Sleeve A, stop-out). Today's realized P&L: **-$40.50**. Unrealized P&L on the open book: **-$2.12** (CVS -$2.28, OGN +$0.16). NAV moved $10,005.28 → $9,956.76 (**-0.49%**) across the day.

> **Two sleeves run over one book.** Sleeve A (breakout, `sleeve=stock`/`index`) and Sleeve B (mean reversion, `sleeve=mean_reversion`) are separate strategies with opposite payoff structures and are reported separately throughout — no blended win rate or average R.

## Account

| Field | Open of day | Close of day | Change |
|---|---|---|---|
| NAV | $10,005.28 | $9,956.76 | -$48.52 (-0.49%) |
| Cash | $7,912.04 | $7,705.29 | -$206.75 |
| Open positions | 2 (ACAD, CVS) | 2 (CVS, OGN) | — |
| Daily-loss halt tripped? | No | No | — |

## Positions opened today

**Sleeve A (breakout) — 1 opened:**

| Symbol | Trigger | Direction | Counter-trend? | Qty | Fill | Stop | R-target |
|---|---|---|---|---|---|---|---|
| OGN | `momentum_vol` | bullish | No | 109 | $13.70 | $13.62975 | $13.8405 |

Thesis (morning run): low-volatility (0.34% ATR) name at a fresh 52-week high on the Sun Pharma acquisition catalyst (shareholder-approved), volume-confirmed (rel_vol 2.178) though it missed its own 7-day Donchian upper by $0.005 (third consecutive session doing so). News sentiment +0.191 (Somewhat-Bullish); most recent earnings (Jul 31) was an EPS miss, noted but non-gating for this sleeve. Position capped by the 15%-of-NAV rule — under-risked at ~0.08% NAV vs the 0.4% target (no leveraged-ETF equivalent exists for OGN).

**Sleeve B (mean reversion) — None opened.** Most common rejection reason across all three runs: **Tier1-MR fail** — RSI(2) qualifiers (ROL, AXS, WSO, all <15 on the last completed session) all sit below their SMA200 (broken trend, not an oversold dip in an uptrend). AXS was re-verified with a fresh quote at both midday and close; still fails. 0 candidates have reached Tier 2-MR on any run since 2026-08-11.

## Positions closed today

**Sleeve A (breakout) — 1 closed:**

| Symbol | Exit rule | Exit price | Realized P&L | R-multiple | Days held |
|---|---|---|---|---|---|
| ACAD | `stop` | $28.59 | -$40.50 | -1.017R | 2 |

Entered 2026-08-11 at $29.49; R-target ($31.2594) was never reached, so this was a plain fixed-stop exit. No company-specific catalyst identified — broad intraday pullback in the name.

**Sleeve B (mean reversion) — No closed trades yet.** CVS remains open (entered 2026-08-12, 2 of 5 trading days into its time-stop clock).

## Open book

| Symbol | Sleeve | Entry | Current | Unrealized P&L | Stop | Dist. to stop | R-target | Dist. to R-target |
|---|---|---|---|---|---|---|---|---|
| CVS | mean_reversion | $94.85 | $94.565 | -$2.28 | $90.35 | $4.215 (4.46%) | N/A (MR sleeve) | N/A |
| OGN | stock | $13.70 | $13.7151 | +$0.16 | $13.62975 | $0.0854 (0.62%) | $13.8405 | $0.1254 (0.91%) |

Neither position is near its stop. CVS: RSI(2) on Aug 12 = 44.58 (not >70, `mr_target` not fired); SMA200 $85.2356 (no `trend_break`). OGN: SMA20 $13.554 (no `trend_break`); time-stop clock not yet started (entered today).

## Rejections worth noting

**Sleeve A — 2 logged gate rejections, both the same name/gate:** **CIB** fired `momentum_vol` on both the midday and close runs (Tier 1-3 all passed each time: trend template, EMA8>EMA21, `relative_volume` ~1.40, positive news sentiment) but was rejected at execution by the **spread gate** (bid/ask spread > 0.5% of mid) both times — 0.627% at midday, 0.684% at close. This is now the third consecutive same-day/same-session instance (2026-08-12 close, 2026-08-13 midday, 2026-08-13 close) of CIB failing this exact gate, all shortly after its Aug 10 earnings report. Still n=3 of one name — not evidence toward any adaptation threshold, but worth a human read if the pattern persists into CIB's next earnings cycle.

**Sleeve B — 0 gate-level rejections logged** (Tier1-MR declines aren't gate rejections and aren't written to `trade_log.csv` per schema scope). Narratively: ROL, AXS, WSO qualified on RSI(2) but all failed Tier1-MR (price below SMA200) on every run today.

## Data quality

- **Stocktwits (`get_symbol_pulse`) unavailable all day** — the connector requires an OAuth authorization that cannot run in this headless scheduled context. Logged as `NO_COVERAGE` for social_score/social_label/social_msg_volume on every candidate evaluated today (OGN, CIB, and the premarket movers list), same as every prior session.
- **Two large tool-result payloads read in place, not copied**, per the fix documented in the 00:30 UTC operator note (`run_scan` ~102K chars on the morning run; `NEWS_SENTIMENT` for OGN also overflowed) — no data loss, just noting the pattern continues.
- No Alpha Vantage quota exhaustion, failed calls, or market-closed fallbacks today — the market was open 09:30-16:15 ET and all scheduled runs completed normally.
- **Operator note (00:30 UTC, not a run):** documents why the 2026-08-12 9:30am ET scan hung (an oversized `get_earnings_calendar` payload triggered a scratchpad-copy permission prompt with no human available to answer it) and the same-day fix (narrower `get_earnings_calendar` call + a standing rule to read oversized results in place). Both fixes were verified working on the 2026-08-12 18:38 and 19:45 UTC runs. The note also flags a correction to the 2026-08-12 report's experiment-3 tally (see below).

## Pre-registered experiment tracking

**1. `momentum_vol` vs `breakout_7d` (level-test removal, admitted 2026-08-12):** **0 of the required 10 closed `momentum_vol` trades.** OGN (opened today) is the only `momentum_vol` position ever taken, and it is still open. No comparison possible yet.

**2. RSI(2) < 15 threshold (raised from 10 on 2026-08-12):** **0 closed mean-reversion trades.** CVS (entry rsi2 = 0.560, sub-10 cohort) is the only mean-reversion position ever taken, and it is still open. No cohort split possible yet.

**3. Momentum-test reinstatement (`momentum_test_would_pass`, logged non-gating):** **1 closed trade counted, of the required 20.** Per the 00:30 UTC operator note's correction, PAYO's 2026-08-12 close is **excluded** from this tally — its `trend_break` exit was caused by the same-day SMA50→SMA20 trend-MA config change applied retroactively to an open position, not by a market event. The one countable close is **ACAD** (`momentum_test_would_pass=true`, R = -1.017), a `breakout_52w`-era trade that predates the momentum_vol/breakout_7d rewrite. `true` bucket: n=1, mean R = -1.017R. `false` bucket: n=0. Far short of the ≥20-closed-trades threshold — no comparison or reinstatement decision is warranted.

## Watch items

- **CIB spread gate:** 3 consecutive same-day/session rejections on the same 0.5%-of-mid liquidity cap, all post-earnings. Not yet actionable (n=3, one name) but worth revisiting if it recurs around CIB's next earnings cycle.
- **OGN and FBP have each missed their own 7-day Donchian breakout level by a small margin on multiple consecutive sessions** (OGN by $0.005 three sessions running) while other Tier 2 legs pass — flagged in the journal as a possible sign the channel width is near the edge of what current volume conditions can confirm. Not a rule change, just visibility.
- **Both open positions (CVS, OGN) are comfortably away from their stops** — nothing urgent on the book tonight.
- **Index sleeve (SSO):** the level test (Donchian-7 breakout) cleared for the first time in over a week on both the midday and close runs (SPY made fresh 52-week highs), but `relative_volume` (0.69) still fails the 1.4 confirmation threshold — now an 8th-9th consecutive run blocked at this one gate despite the price condition finally clearing.

---

Paper trading — simulated book, no real orders placed. P&L charges the bid-ask spread but omits fill uncertainty, slippage, options fees, ETF expense ratios, and dividends; expect live results to run modestly worse. See `DATA_SCHEMA.md`.
