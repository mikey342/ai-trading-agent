# Run Instructions — Swing Trading Quant Agent (Paper)

You are running as a scheduled cloud agent against a fresh clone of this
repo. You have no memory of prior runs except what is written in these
files. Follow this playbook exactly, in order. `gates.md` overrides
anything here or in `strategy.md` if they ever conflict.

**Tool budget note**: Robinhood's data tools (`get_equity_quotes`,
`get_equity_fundamentals`, `get_equity_technical_indicators`,
`get_earnings_results`) are the primary data source throughout this
playbook and have shown no daily cap. Alpha Vantage is reserved for
`NEWS_SENTIMENT` only (Robinhood has no equivalent) plus `MARKET_STATUS` —
its 25-requests/day quota is shared across every session using this key,
including manual test runs, and has been fully exhausted before by
same-day manual testing. See `strategy.md`'s "Data sources" table for the
full mapping.

## 0. Orientation

1. Read `gates.md` in full. If `MODE` is not exactly `PAPER`, stop now,
   write a note to `trade_journal.md`, commit, and end the run — no
   analysis, no trading, equity or options.
2. Read `framework.md` and `strategy.md` in full.
3. Read `positions.md` — current equity positions, options positions, and
   NAV history.
4. Read the last ~10 entries of `trade_journal.md`.
5. If running the 9:30am morning slot, read `premarket_notes.md` if it's
   dated today — use it only to prioritize which candidates to look at
   first in Tier 1 (e.g., a flagged gapper is worth checking early). It
   is informational context only, never a substitute for Tier 1-3 gating
   against fresh prices — every candidate still goes through the full
   funnel regardless of what's noted there.

## 1. Market context and regime filter

This routine fires three times daily — **9:30am (market open), 2:30pm,
and 3:30pm ET** — all of them *inside* regular market hours:

- **Never before the open**, so every run sees a live regular-session
  price rather than a stale prior-day close or a thin premarket quote.
- **Never after the close**, because a run that decides to open a
  position at 4:30pm cannot actually fill it — the order would sit until
  the next morning at an unknown price, which is not what the analysis
  evaluated.
- **At the open, not an hour after it.** The analysis takes ~15 minutes,
  so the morning run naturally executes around 9:45. Delaying to avoid
  opening spreads would cost more in missed breakouts than it saves in
  execution — see `strategy.md`, "Order entry". Wide spreads are caught
  case-by-case by the 0.5% spread check at fill time.

1. Call `MARKET_STATUS` (Alpha Vantage). If it fails (quota exhausted),
   don't abort the run — fall back to treating the market as open during
   normal US trading hours (9:30am-4pm ET, weekdays) based on the current
   time, and note the fallback in the journal. If the market is closed for
   a holiday, log a one-line journal entry and end the run.
2. **Set direction.** Call `get_equity_quotes` for **SPY** and
   `get_equity_technical_indicators` (SPY, sma, period=200, output=latest).
   SPY > SMA(200) → **LONG mode**; SPY < SMA(200) → **SHORT mode**.
   **Require 3 consecutive daily closes on the far side before flipping**
   an established direction — pull `output="last:5"` on the SMA and
   compare against recent closes rather than flipping on today alone.
   Never short a stock, never use margin, in either mode.

3. **Set risk level** (independent of direction — see `strategy.md`):
   - `get_equity_technical_indicators` (SPY, sma, period=50, output=latest)
   - `get_indexes` for VIX (id `3b912aa2-88f9-4682-8ae3-e39520bdf4db`),
     then `get_index_quotes` for its current level
   - **NORMAL** (SPY > 50DMA and VIX < 20) → full rules
   - **CAUTION** (SPY < 50DMA, **or** VIX 20-25, **or** VIX ≥ 25 while SPY
     is still above its 200DMA) → max **1** new trade this run, no new
     index-sleeve entries, **counter-trend disabled**
   - **STRESSED** (VIX ≥ 25 **and** SPY below its 200DMA) → **no new
     entries at all**; still review and exit existing positions normally.
     Both conditions are required: momentum crashes cluster in high
     volatility *following declines*, not in volatility alone.
   Take the **worse** of the two readings. Record both direction and risk
   level explicitly in the journal.
   Existing positions are reviewed and exited in either mode, regardless
   of which way the regime currently points.

## 2. Review existing positions first (always, regardless of regime)

For every row in `positions.md` (equity and options separately):
- Equity: `get_equity_quotes` for current price, and
  `get_equity_technical_indicators` (type=sma, output=latest) for the
  sleeve's trend MA — **period=20 for a stock position, period=50 for an
  index-sleeve position**; see `strategy.md`.
- If `R-target reached?` is still "No": check if current price has now hit
  the R-target — if so, flip the flag to "Yes" in `positions.md` (this
  never flips back). Whether or not it just flipped, also check if price
  has hit the original fixed stop.
- If `R-target reached?` is "Yes": pull `get_equity_technical_indicators`
  (type=ema, period=21, output=latest) and set `Current exit stop =
  max(EMA21, original stop)`. Check if price has closed at/below this
  trailing stop.
- Options: Robinhood `get_option_quotes` for current premium, and
  `get_equity_quotes` for the underlying. Compare the underlying's current
  price against the stored `Underlying stop` / `Underlying R-target` in
  `positions.md` using the same trailing-stop mechanic as equity (fixed
  stop until R-target hit, then trail EMA21 — see `strategy.md`). Also
  check days-to-expiration remaining vs. both the 21-DTE checkpoint and
  50% of DTE-at-entry.
- **`mean_reversion` positions use a different exit set** — no trailing
  EMA21 stop and no 2R target. Exit on: the fixed stop, **`mr_target`
  (RSI(2) > 70)**, a close **below SMA(200)**, or a **5-trading-day**
  time-stop. Precedence: stop → mr_target → trend_break → time_stop.
- Check against the full exit rules in `strategy.md` (current exit stop
  per the above, trend-template break on the sleeve's MA, time-stop, options DTE
  checkpoints). **Time-stops differ by sleeve** — 10 trading days for an
  ordinary stock, **8** for an index leveraged ETF, **5** for a
  single-stock leveraged ETF (decay scales with volatility; see
  `gates.md`). These apply **only while `R-target reached?` is "No"**;
  never time-stop a position that has already reached its R-target.
- If an exit condition is met: simulate closing (record exit price/premium,
  compute realized P&L), update `positions.md`, and log it in
  `trade_journal.md` with the specific rule that triggered it.

## 3. New entries — direction depends on the mode set in step 1

**Test every candidate against BOTH the long and short templates** — the
regime does not decide which direction to look for, only how strictly a
passing setup is treated. A stock cannot pass both; most pass neither.

For each pass, classify it:
- **with-trend** — bullish while SPY > SMA(200), or bearish while below
- **counter-trend** — bearish while SPY > SMA(200), or bullish while below

**With-trend setups** follow the normal rules. Express a bullish pass as
plain equity or that name's **bull** ETF; a bearish pass by buying that
name's **bear** ETF.

**Counter-trend setups** must additionally clear *all five* gates in
`strategy.md` / `gates.md`: at most 1 counter-trend position open,
**ADX(14) > 30 measured by a direct `get_equity_technical_indicators`
call** (type=adx, period=14, interval=day, output=latest) — **not** the
Tier 0 scan's ADX column, a more extreme 52-week-range reading (≤0.25
bearish / ≥0.75 bullish), **no degraded data — `news_sentiment` must not
be `UNAVAILABLE`** — and half size (**0.2%** NAV risk vs 0.4%
with-trend). If any fails, drop the setup and log it as a counter-trend
rejection. Never downgrade a failed counter-trend setup into a normal
trade.

The ADX source is load-bearing: the scan computes it with `session="all"`
while a direct call uses regular session, and the two disagree by 7-12
points in both directions (FTNT: scan 33.80 vs direct 26.05 — passes this
gate on one measure, fails on the other). Scan ADX is for screening and
ranking only; every gate re-measures.

For any bear ETF, confirm via `get_equity_fundamentals` both that it
clears the 100K-share liquidity floor (and that the order is ≤ 1% of its
30-day ADV) and **what its actual leverage is**
(several are −1x, not −2x — never infer it from the bull-side ticker).

**Index sleeve — regime-aligned only.** Consider **one** index position:
SSO/QLD if SPY > SMA(200) and the index passes the long template, or
SDS/QID if SPY < SMA(200) and it passes the inverted one. The index
sleeve is never counter-trend.

Never short a stock and never use margin, in any circumstance.

### Tier 0 — universe (one call)
Call `run_scan` with `scan_id: de1b1994-b5db-472a-9b79-c052f1215193`
("Swing Agent - Trend Candidates"). This returns live market-wide results
already filtered on market cap > $2B, price > $5, 30d avg volume > 250K,
ADX(14) > 25, and RSI(14) 25-75 — with all those values included per row,
so no extra calls are needed to read them. Results are sorted **ADX(14)
descending**. Note the total match count in the journal, and note that
the response caps at 200 rows (~396 matched on 2026-08-11).

A `relative options volume > 0.5` filter was removed on 2026-08-11: it
compared today's *partial* options volume against a *full-day* average,
so it rejected nearly the entire universe during the morning run (4
matches at 09:57 vs 51 at 12:57 the same day). See `strategy.md`. **If a
morning scan ever again returns a single-digit match count while later
runs return dozens, suspect a partial-intraday-accumulation filter before
concluding the market is thin.**

**Then apply the dollar-volume screen client-side** (free — the data is
already in the response): compute `Last × Average volume` and drop
anything below **$20M/day**. Share volume alone is a poor liquidity
measure across price levels; see `strategy.md`.

Rank client-side (free) by `Average directional index (14)`
descending. **Multiply
the score by 1.15 for any symbol listed in `ai_theme.md`** — a tilt, not
a filter; non-AI names still displace AI names when they rank higher.
Take the **top 15** into Tier 1. This is the entire candidate universe for the run
— there is no hardcoded watchlist anymore. If `run_scan` fails, fall back
to screening a small diversified set of liquid large-caps and flag it in
the journal as degraded operation.

### Tier 1 — trend template (top 15 from Tier 0)
First, **one** `get_earnings_calendar` call (days=7) — drop any candidate
reporting within 3 calendar days before spending anything else on it.
Then pull `get_equity_quotes` (one batched call for all 15) and
`get_equity_fundamentals` (2 batched calls), then per-symbol
`get_equity_technical_indicators` (sma, period=20, period=50 and
period=200, output=latest). Apply the Tier 1 trend-template checklist **in both
directions** (long and inverted). Drop anything that passes neither.
The **proximity-to-52-week-extreme criterion was removed 2026-08-11**
(both directions) — the SMA stack and the upper/lower-40%-of-range test
still apply. (This note originally also cited `breakout_52w`'s stricter
2% bar; that trigger was removed 2026-08-12 — **no 52-week level test
remains anywhere in the funnel.**) See `strategy.md`.

**Before advancing anything to Tier 2:** check `trade_log.csv` for reject
rows dated today. If a candidate was already rejected today on a
daily-bar criterion (**volume confirmation**, an RSI band, EMA ordering),
skip it — daily indicators are computed from completed bars and cannot
change intraday, so re-testing is guaranteed to reach the same answer. Do
not log a duplicate rejection row; duplicates corrupt the pre-registered
tallies in `strategy.md`. Price-based checks are exempt and should still
re-run, since price does move.

### Tier 2 — shortlist
For Tier 1 survivors (capped at top 8 by the **relative-strength proxy**
alone — the value leg was removed 2026-08-11 as horizon-inappropriate for
a ≤10-day hold; see `strategy.md`), call `get_equity_technical_indicators` for RSI
(period=14, output="last:5"), EMA (period=8 and 21, output="last:5"), and
**donchian_channels (period=7, output="last:3")** — shortened from 20 on
2026-08-12 to match the ≤10-day hold. Three triggers now qualify —
**`breakout_7d`**, **`momentum_vol`** (EMA direction + volume, **no level
requirement at all**), and `pullback` (all mirrored in SHORT mode as
`breakdown_7d`, `momentum_vol`, `rally_to_resistance`). For
`breakout_7d`, compare today's price against the **prior bar's** upper
channel, not today's — using today's own channel is circular and always
passes. Every `breakout_7d` also satisfies `momentum_vol`; when both
fire, label it **`breakout_7d`**, the more specific signal. The `output`
parameter trims the response server-side — no file/jq extraction needed.

**`breakout_7d` and `momentum_vol` both require volume confirmation:
`relative_volume ≥ 1.4`** — the last **COMPLETED** session's volume ÷ the
30-day average, both already available from the Tier 1
`get_equity_fundamentals` call. Two traps:

- **Never use a partial day.** During a live run today's volume is only
  partial, so comparing it to a full-day average understates every name
  and would reject everything. Use the last completed session —
  consistent with every other daily indicator in this funnel.
- **Compute it; do not read the scan's `Relative volume` column**, which
  uses `session="all"` (the same session-bounds problem as ADX).

Basis: O'Neil's 40%-above-average rule, supported by O'Neil Global
Advisors' 1995-2021 study finding volume-confirmed breakouts
significantly outperform. This **replaced** the old Tier 3 momentum test.

**Record the MOST SPECIFIC trigger that fires** — `breakout_7d` →
`momentum_vol`. A candidate is labelled `momentum_vol` only when **no
level test passed**; that label isolates the population with no level
requirement, which is the entire point of the experiment. Populate
`pct_from_52w_high` on **every** row — opens, rejects, both sleeves — so
the comparison can be run as a continuous variable. See `strategy.md`,
"The `momentum_vol` experiment", including its **pre-registered** review
criterion: no conclusions before 10 closed `momentum_vol` trades.

### Tier 3 — finalists (cap 3 per gates.md)
For Tier 2 survivors, pull `NEWS_SENTIMENT` (Alpha Vantage, limit 5-10),
`get_earnings_results` (Robinhood, per symbol),
`get_equity_technical_indicators` (type=atr, period=14, output=latest),
and `get_sentiment` (Stocktwits, log-only).

**Also compute the EMA8/EMA21 spread test and log it to
`momentum_test_would_pass` — but do NOT gate on it.** It was retired on
2026-08-11 because it was invented rather than researched; volume
confirmation at Tier 2 replaced it. Logging preserves the ability to
check later whether it actually predicted anything.

Drop anything failing the remaining checks. `MACD` (Alpha Vantage) is
permanently premium-gated and must never be called.

### Tier 0-3 MR — the mean-reversion sleeve (second pass, same scan)

Run this **after** the breakout funnel above, using the **same `run_scan`
response** — no second scan call. See `strategy.md`, "Mean-reversion
sleeve," for the full rationale.

**Skip this entire pass if SPY < SMA(200)** (the direction reading from
step 1). The sleeve is LONG-mode only; in SHORT mode it is off, not
merely restricted.

Also skip if `positions.md` already holds **2** `mean_reversion`
positions (`gates.md` cap).

**Order the calls this way — it matters for cost:**

1. From scan rows clearing the $20M/day dollar-volume screen, take the
   **10 lowest `RSI`** (the scan's RSI(14)). Free.
2. `get_equity_technical_indicators` (type=rsi, **period=2**,
   interval=day, output="last:2") for those 10. Keep only names whose
   **last completed session** printed **RSI(2) < 10**. Usually leaves
   2-4 names.
3. Only now pull `get_equity_quotes` and SMA(200) for the survivors.
   Running step 3 first triples the call count for no benefit.

**Tier 1-MR:** price **> SMA(200)**, and not reporting earnings within 3
days (reuse the same `get_earnings_calendar` result already pulled — do
not call it twice). Do **not** apply the SMA(20) test, the 0.6
range-position floor, or the ADX rank — those belong to the breakout
template and are mutually exclusive with a pullback by construction.

**Tier 2-MR — trigger `mr_reversal`:** the oversold session's RSI(2) < 10
**and** the current price is above that session's close (the first
up-close). **No volume confirmation** — that is a breakout concept and
oversold bounces often come on declining volume.

**Tier 3-MR:** `NEWS_SENTIMENT` must not be negative **and must not be
`UNAVAILABLE`**. If the quota is exhausted, take no mean-reversion trade
this run — same standard as counter-trend, because news is what separates
temporary selling from a real problem.

**Sizing is unchanged** — 0.4% NAV risk, ATR stop with the 3% floor, 15%
cap, same instrument-selection procedure in step 4.

**Log with `sleeve=mean_reversion`, `trigger=mr_reversal`, and populate
the `rsi2` column.** These rows must stay separable from breakout rows —
the two sleeves have opposite payoff structures and pooling them produces
statistics that describe neither.

## 4. Decide instrument: equity, leveraged ETF, or options

**This is mechanical — do not treat it as a judgement call.** See
`strategy.md`, "Instrument selection," for the reasoning.

Compute both share counts (you need them for sizing anyway):

```
shares_risk = floor(risk_budget / stop_distance)   # 0.4% NAV, 0.2% counter-trend
shares_cap  = floor(0.15 x NAV / price)            # the 15% position cap
```

**If `shares_risk <= shares_cap` → PLAIN EQUITY.** The risk budget binds,
so the position already carries its intended risk. Leverage would
overshoot the target and add decay for nothing. This is the default and
the common case.

**If `shares_risk > shares_cap` → the cap binds and the position is
under-risked. A leveraged ETF may be used**, but only if every one holds:
- a mapped ETF exists for the name, clears the 100K-share floor, and the
  order is ≤ 1% of its 30-day ADV
- the underlying's **ATR ≤ 4% of price** (decay gate)
- its leverage is **verified per instrument** (several bear ETFs are −1x)
- gross exposure stays ≤ 1.3x NAV after the 2x multiplier
- sizing uses the **ETF's own ATR**, not the underlying's

If any fails, use plain equity and accept the under-risked position. An
under-risked position is a smaller win; a wrong instrument is a new way
to lose.

**Options — only when the premium actually fits** the 3%-of-NAV cap
($300 at current NAV), which on a $200+ underlying it usually will not.
The reason to choose them is **gap protection**: max loss is the premium
regardless of how far price gaps, whereas a stock can gap through its
stop. Not for leverage — the ETF path is simpler with no expiration to
manage. Lookup is **three steps** (see `tool_verification.md`):
`get_option_chains` → `get_option_instruments` → `get_option_quotes`.
Evaluate against the DTE/delta/liquidity gates in `gates.md`; if nothing
fits, use equity rather than loosening a gate.
- If choosing options: compute the underlying stop/R-target using the same
  ATR-based math as equity, and record them in `positions.md`'s
  `Underlying stop`/`Underlying R-target` columns at entry — that is what
  future runs check against, never a recalculation.

**In SHORT mode there is no choice:** shorting stock is forbidden, so a
bearish signal is a bear ETF or no trade.

## 5. Gate every proposed trade

Before simulating any entry, in order:
1. Position sizing per `strategy.md` (ATR-based stop, risk-per-trade sizing
   for equity; premium cap for options).
2. Would this breach max position size / max premium at risk?
3. Has the daily loss halt already triggered today (check `positions.md`
   NAV history for today's row, if one exists from an earlier run today)?
4. Would this exceed max new trades this run (**1 if risk level is
   CAUTION**, 2 if NORMAL, **0 if STRESSED**), or max open positions
   (all sleeves combined)? If counter-trend, would it exceed the
   **1 counter-trend position** limit — and is counter-trend even
   permitted at the current risk level?
4b. **Sector concentration:** would this make a 4th position in the same
   sector? Max 3 of 6 per sector (`sector` from `get_equity_fundamentals`).
   Correlated positions don't stop out independently, and the risk math
   assumes they roughly do.
5. Does the stop-loss respect the `gates.md` floor?
6. (Options only) Does it pass DTE / delta / OI / spread gates?
7. (Leveraged ETF only) Is this the *only* index-sleeve position (never a
   long and an inverse ETF at once)? Does gross exposure stay within
   limits **after applying the 2x multiplier** to this position? Is it
   sized off the **ETF's own** ATR rather than the index's?
8. Confirm no margin is involved and nothing is being sold short — both
   are disabled outright in `gates.md`.

Any failure → do not open the position. Log it under "Rejected by gates"
with the specific reason — this is the system working as intended, not a
failure to route around.

## 6. Simulate execution

Before recording any fill: re-fetch a **fresh** quote for the symbol —
never reuse a price read earlier in the funnel (Tier 1-3 can take several
minutes across the full universe). Then, in order:

1. Apply `strategy.md`'s **chase-protection** check against this fresh
   price; if it fails, drop the trade and log it — don't force it through
   at a worse price than what was actually evaluated.
2. Apply the **two-part liquidity check**: (a) if bid-ask exceeds 0.5% of
   mid for an equity/ETF, skip and log it; (b) call
   `get_equity_price_book` and confirm the resting size at the best
   ask/bid is at least the order quantity — a tight-looking top-of-book
   price backed by almost no size means the order walks the book. Skip
   and log if depth is insufficient.
3. Record the fill as a **marketable limit order**, priced per
   `strategy.md`'s "Order entry" section — **buys at the `ask_price`,
   sells at the `bid_price`** (options: `high_fill_rate_buy_price` /
   `high_fill_rate_sell_price`). Never simulate a fill at the midpoint or
   at `last_trade_price`; both flatter the record versus a real fill.
   Never simulate a market order.

**Never call any real Robinhood order-placement tool —
`place_equity_order`, `place_option_order`, or any other order/exercise
tool — under any circumstance in this workflow.** This is a paper-trading
system; `MODE` in `gates.md` must be `PAPER`. Update `positions.md`
accordingly (deduct simulated cash, add the position row with stop/target
or DTE/premium as applicable).

**Always populate the `Underlying` column.** It is the symbol the thesis
is about, and the one whose trend MA the trend-break exit checks. For a
plain stock it equals `Symbol`; for a leveraged ETF it is the underlying
(NVD → NVDA, SDS → SPY). Leave it blank and the hourly monitor cannot
evaluate the trend-break exit correctly — it would test the ETF's own
moving average, which leverage and daily resets have distorted.

## 7. Update positions.md, trade_log.csv, and trade_journal.md

1. Recompute NAV (cash + market value of open equity + open options) and
   append one row to the `positions.md` NAV history table for this run.
2. **Append a row to `trade_log.csv` for every position opened, every
   position closed, and every candidate rejected at Tier 3 or at the
   gates** — see `DATA_SCHEMA.md` for the column definitions. Include the
   full indicator snapshot that justified the decision (ATR/RSI/EMA/SMA/
   52wk-range/PE/sentiment/regime). Rejections matter as much as fills:
   without them there's no way to tell whether a filter is well-calibrated
   or just declining good trades. Never fabricate a value — write
   `UNAVAILABLE` if a data source failed.
3. Append one new entry to `trade_journal.md` using the template at the
   top of that file, including the **mandatory per-trade rationale
   fields** (trigger, trend, momentum, relative strength, valuation,
   catalyst, risk sizing, and a one-sentence thesis). The journal is the
   human-readable "why"; the CSV is the queryable "what" — both get
   written, and they must agree.

## 8. Adapt strategy (small, evidence-based changes only)

Follow `strategy.md`'s Adaptation policy. Most runs should NOT change
`strategy.md`. Never touch `gates.md` or `framework.md`.

## 9. Commit and push

Commit all changed files with a message like:
`run: 2026-08-10 morning — 1 opened (equity), 0 closed, 1 rejected by gates, regime: risk-on`

Push to `main`. If this step fails or is skipped, the next run starts
blind to everything that happened today.

## Handling errors gracefully

- Alpha Vantage quota/premium error on any call: log it, don't abort the
  whole run, prioritize finishing the position-review step over scouting
  new ones, and note the degradation in the journal. Specifically:
  - `MARKET_STATUS` → time-based fallback (see step 1).
  - `run_scan` (Robinhood, no AV quota involved) → if it fails, fall back
    to screening a small diversified set of liquid large-caps and flag
    the run as degraded in the journal.
  - `NEWS_SENTIMENT` → **do not drop the candidate.** Record
    `news_sentiment: UNAVAILABLE` and proceed if every other check passes
    (see `strategy.md`'s graceful-degradation rule). A hard block here
    would turn a free-tier quota limit into a permanent no-trade bug.
- Never fabricate a value to fill a gap. "Skip the candidate" applies to
  a *Robinhood* data failure (trend/momentum/ATR data is load-bearing and
  has no substitute); the Alpha Vantage news check has an explicit
  logged-degradation path instead, because it is a secondary safety
  filter rather than a core signal.

## Hard rules, restated

- Never call a real order-placement tool — equity or option.
- Never edit `gates.md` or `framework.md`.
- Never delete or rewrite past `trade_journal.md` entries.
- Always commit and push before ending the run.
