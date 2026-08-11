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
  `get_equity_technical_indicators` (type=sma, period=50, output=latest)
  for the current SMA(50).
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
- Check against the full exit rules in `strategy.md` (current exit stop
  per the above, SMA(50) trend-template break, time-stop, options DTE
  checkpoints). **Time-stops differ by sleeve** — 20 trading days for an
  ordinary stock, **10** for an index leveraged ETF, **7** for a
  single-stock leveraged ETF (decay scales with volatility; see
  `gates.md`).
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
ADX(14) > 25, RSI(14) 25-75, and relative options volume > 0.5 — with all
those values included per row, so no extra calls are needed to read them.
Note the total match count in the journal (~116 at last check).

**Then apply the dollar-volume screen client-side** (free — the data is
already in the response): compute `Last × Average volume` and drop
anything below **$20M/day**. Share volume alone is a poor liquidity
measure across price levels; see `strategy.md`.

Rank client-side (free) primarily by `Average directional index (14)`
descending, using `Relative options volume` as a tiebreaker. **Multiply
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
`get_equity_technical_indicators` (sma, period=50 and period=200,
output=latest). Apply the Tier 1 trend-template checklist **in both
directions** (long and inverted). Drop anything that passes neither.

**Before advancing anything to Tier 2:** check `trade_log.csv` for reject
rows dated today. If a candidate was already rejected today on a
daily-bar criterion (**volume confirmation**, an RSI band, EMA ordering),
skip it — daily indicators are computed from completed bars and cannot
change intraday, so re-testing is guaranteed to reach the same answer. Do
not log a duplicate rejection row; duplicates corrupt the pre-registered
tallies in `strategy.md`. Price-based checks are exempt and should still
re-run, since price does move.

### Tier 2 — shortlist
For Tier 1 survivors (capped at top 8 by the value/momentum tiebreaker —
see `strategy.md`), call `get_equity_technical_indicators` for RSI
(period=14, output="last:5"), EMA (period=8 and 21, output="last:5"), and
**donchian_channels (period=20, output="last:3")**. Three triggers now
qualify — `breakout_52w`, `breakout_20d`, `pullback` (mirrored in SHORT
mode). For `breakout_20d`, compare today's price against the **prior
bar's** upper channel, not today's. When both breakout triggers fire,
label it `breakout_52w` — the stronger signal. The `output` parameter
trims the response server-side — no file/jq extraction needed.

**Both breakout triggers also require volume confirmation:
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

## 4. Decide instrument: equity, options, or skip

For each finalist that survives Tier 3:
- Default to equity.
- Consider an options expression only if it clearly improves capital
  efficiency for this specific signal. The lookup is **three steps**
  (verified — see `tool_verification.md`): `get_option_chains`
  (chain + expirations) → `get_option_instruments` (contract UUIDs for
  the chosen expiration/type/strike) → `get_option_quotes` (delta,
  open_interest, bid/ask, Greeks). Evaluate against the DTE/delta/
  liquidity gates in `gates.md`. If no expiration/strike meets all of
  them, use equity instead — do not loosen the options gates to force a
  fit.
- If choosing options: compute the underlying stop/R-target using the same
  ATR-based math as equity (see `strategy.md`), and record them in
  `positions.md`'s `Underlying stop`/`Underlying R-target` columns at
  entry — this is what future runs will check against, not a
  recalculation.

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
