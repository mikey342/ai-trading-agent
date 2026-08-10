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

## 1. Market context and regime filter

This routine fires three times daily (9:30am ET market open, 2:30pm ET
midday, 4:30pm ET close) — deliberately never before the open, so every
run works with a real, live regular-session price rather than a stale
prior-day close or a thin premarket quote. (The 4:30pm run uses the
official last-completed-session close, which is likewise a real,
executable price — not an approximation.)

1. Call `MARKET_STATUS` (Alpha Vantage). If it fails (quota exhausted),
   don't abort the run — fall back to treating the market as open during
   normal US trading hours (9:30am-4pm ET, weekdays) based on the current
   time, and note the fallback in the journal. If the market is closed for
   a holiday, log a one-line journal entry and end the run.
2. Call `get_equity_quotes` (Robinhood) for **SPY**, and
   `get_equity_technical_indicators` (Robinhood, SPY, type=sma, period=200,
   interval=day, output=latest) to get SPY's 200-day SMA. Check SPY price
   > SMA(200). Record the result — this gates all *new* entries this run
   (existing positions are still reviewed regardless).

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
  checkpoints).
- If an exit condition is met: simulate closing (record exit price/premium,
  compute realized P&L), update `positions.md`, and log it in
  `trade_journal.md` with the specific rule that triggered it.

## 3. New entries — only if the regime filter passed in step 1

If SPY failed the regime check, skip to step 7 (journal update) — do not
scout or open new positions this run, but say so explicitly in the journal.

Otherwise, run the funnel from `strategy.md`:

### Tier 1 — full universe
For every symbol in the `strategy.md` watchlist, plus the top 3-5 from
`TOP_GAINERS_LOSERS` (Alpha Vantage — cheap, one call for the whole list),
pull `get_equity_quotes` and `get_equity_fundamentals` (both batched —
up to 20 and 10 symbols per call respectively), then per-symbol
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
  efficiency for this specific signal — check Robinhood `get_option_chains`
  for the symbol and evaluate against the DTE/delta/liquidity gates in
  `gates.md`. If no expiration/strike meets all of them, use equity instead
  — do not loosen the options gates to force a fit.
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
7. (Leverage, if used) Within the max leverage ratio and max gross exposure?

Any failure → do not open the position. Log it under "Rejected by gates"
with the specific reason — this is the system working as intended, not a
failure to route around.

## 6. Simulate execution

For trades passing every gate: record as filled at the current
quote/premium. **Never call any real Robinhood order-placement tool —
`place_equity_order`, `place_option_order`, or any other order/exercise
tool — under any circumstance in this workflow.** This is a paper-trading
system; `MODE` in `gates.md` must be `PAPER`. Update `positions.md`
accordingly (deduct simulated cash, add the position row with stop/target
or DTE/premium as applicable).

## 7. Update positions.md and trade_journal.md

1. Recompute NAV (cash + market value of open equity + open options) and
   append one row to the `positions.md` NAV history table for this run.
2. Append one new entry to `trade_journal.md` using the template at the
   top of that file: market status, regime result, scouted symbols,
   decisions (with which tier/trigger fired), rejections, position
   reviews, and instrument choice (equity vs options) with reasoning.

## 8. Adapt strategy (small, evidence-based changes only)

Follow `strategy.md`'s Adaptation policy. Most runs should NOT change
`strategy.md`. Never touch `gates.md` or `framework.md`.

## 9. Commit and push

Commit all changed files with a message like:
`run: 2026-08-10 morning — 1 opened (equity), 0 closed, 1 rejected by gates, regime: risk-on`

Push to `main`. If this step fails or is skipped, the next run starts
blind to everything that happened today.

## Handling errors gracefully

- Alpha Vantage quota/premium error on any call: log it, skip that
  symbol/tier (don't abort the whole run), prioritize finishing the
  position-review step over scouting new ones, and note the truncation in
  the journal. `MARKET_STATUS` specifically has a time-based fallback (see
  step 1) rather than blocking the run.
- Never treat a data error as a reason to fall back to guessing a value —
  skip the candidate instead.

## Hard rules, restated

- Never call a real order-placement tool — equity or option.
- Never edit `gates.md` or `framework.md`.
- Never delete or rewrite past `trade_journal.md` entries.
- Always commit and push before ending the run.
