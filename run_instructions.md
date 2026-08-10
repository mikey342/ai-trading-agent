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
5. If running the 10:30am morning slot, read `premarket_notes.md` if it's
   dated today — use it only to prioritize which candidates to look at
   first in Tier 1 (e.g., a flagged gapper is worth checking early). It
   is informational context only, never a substitute for Tier 1-3 gating
   against fresh prices — every candidate still goes through the full
   funnel regardless of what's noted there.

## 1. Market context and regime filter

This routine fires three times daily — **10:30am, 2:30pm, and 3:30pm ET**
— all of them *inside* regular market hours. That placement is
deliberate on both ends:

- **Never before the open**, so every run sees a live regular-session
  price rather than a stale prior-day close or a thin premarket quote.
- **Never after the close**, because a run that decides to open a
  position at 4:30pm cannot actually fill it — the order would sit until
  the next morning at an unknown price, which is not what the analysis
  evaluated.
- **10:30am rather than the 9:30am bell**: the first ~30 minutes carry
  the day's widest spreads and sharpest reversals. Paying that spread
  buys nothing on a 5-15 day horizon (see `strategy.md`, "Order entry").

1. Call `MARKET_STATUS` (Alpha Vantage). If it fails (quota exhausted),
   don't abort the run — fall back to treating the market as open during
   normal US trading hours (9:30am-4pm ET, weekdays) based on the current
   time, and note the fallback in the journal. If the market is closed for
   a holiday, log a one-line journal entry and end the run.
2. Call `get_equity_quotes` (Robinhood) for **SPY**, and
   `get_equity_technical_indicators` (Robinhood, SPY, type=sma, period=200,
   interval=day, output=latest) to get SPY's 200-day SMA. This sets the
   **direction mode for the whole run**: SPY > SMA(200) → **LONG mode**
   (stock sleeve, plus optionally a 2x long ETF); SPY < SMA(200) →
   **SHORT mode** (2x inverse index ETF only — never short a stock, never
   use margin, see `gates.md`). Record the mode explicitly in the journal.
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

### If SHORT mode (SPY below its SMA(200))
**Skip the entire stock-selection funnel** (Tiers 0-3 below). The only
bearish expression is the index sleeve: check whether SPY (or QQQ) passes
the *inverted* trend template and a short trigger fires per `strategy.md`;
if so, buy **SDS** (or QID) — a 2x inverse ETF, bought with cash. Never
short a stock and never use margin. Then continue to step 5 (gates).
Say explicitly in the journal that the stock funnel was skipped because
of SHORT mode, so a quiet run isn't mistaken for a broken one.

### If LONG mode
Run the stock-selection funnel below. Additionally, consider **one**
index-sleeve position (SSO or QLD) if SPY/QQQ itself passes the long
trend template and a trigger fires — subject to the max-1-index-position
rule and the 2x exposure accounting in `gates.md`.

### Tier 0 — universe (one call)
Call `run_scan` with `scan_id: de1b1994-b5db-472a-9b79-c052f1215193`
("Swing Agent - Trend Candidates"). This returns live market-wide results
already filtered on market cap > $2B, price > $5, 30d avg volume > 1M,
ADX(14) > 25, RSI(14) 25-75, and relative options volume > 0.5 — with all
those values included per row, so no extra calls are needed to read them.
Note the total match count in the journal (~96 at last check).

Rank client-side (free) primarily by `Average directional index (14)`
descending, using `Relative options volume` as a tiebreaker, and take the
**top 15** into Tier 1. This is the entire candidate universe for the run
— there is no hardcoded watchlist anymore. If `run_scan` fails, fall back
to screening a small diversified set of liquid large-caps and flag it in
the journal as degraded operation.

### Tier 1 — trend template (top 15 from Tier 0)
Pull `get_equity_quotes` (one batched call for all 15) and
`get_equity_fundamentals` (2 batched calls), then per-symbol
`get_equity_technical_indicators` (sma, period=50 and period=200,
output=latest). Apply the Tier 1 trend-template checklist. Drop anything
that fails.

### Tier 2 — shortlist
For Tier 1 survivors (capped at top 8 by the value/momentum tiebreaker —
see `strategy.md`), call `get_equity_technical_indicators` for RSI
(period=14, output="last:5") and EMA (period=8 and 21, output="last:5").
The `output` parameter trims the response server-side — no file/jq
extraction needed. Apply the Tier 2 breakout/pullback trigger logic. Drop
anything that fails.

### Tier 3 — finalists (cap 3 per gates.md)
For Tier 2 survivors: check the EMA8/EMA21 spread from the values already
pulled in Tier 2 (no new call). Then pull `NEWS_SENTIMENT` (Alpha Vantage,
limit param 5-10), `get_earnings_results` (Robinhood, per symbol), and
`get_equity_technical_indicators` (Robinhood, type=atr, period=14,
output=latest). Apply the Tier 3 confirmation checklist. Drop anything
that fails. `MACD` (Alpha Vantage) is permanently premium-gated on this
plan and must never be called — the EMA-spread check already covers this.

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
4. Would this exceed max new trades this run, or max open positions
   (equity + options combined)?
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
2. Apply the **spread sanity check**: if bid-ask exceeds 0.5% of mid for
   an equity/ETF, skip and log it.
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
  - `TOP_GAINERS_LOSERS` → skip opportunistic candidates, still screen the
    `strategy.md` watchlist normally.
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
