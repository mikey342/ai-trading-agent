# Run Instructions — Swing Trading Quant Agent (Paper)

You are running as a scheduled cloud agent against a fresh clone of this
repo. You have no memory of prior runs except what is written in these
files. Follow this playbook exactly, in order. `gates.md` overrides
anything here or in `strategy.md` if they ever conflict.

## 0. Orientation

1. Read `gates.md` in full. If `MODE` is not exactly `PAPER`, stop now,
   write a note to `trade_journal.md`, commit, and end the run — no
   analysis, no trading, equity or options.
2. Read `framework.md` and `strategy.md` in full.
3. Read `positions.md` — current equity positions, options positions, and
   NAV history.
4. Read the last ~10 entries of `trade_journal.md`.

## 1. Market context and regime filter

1. Call `MARKET_STATUS`. If the market is closed for a holiday, log a
   one-line journal entry and end the run.
2. Call `GLOBAL_QUOTE` and `COMPANY_OVERVIEW` for **SPY**. Check SPY price
   > `200DayMovingAverage`. Record the result — this gates all *new*
   entries this run (existing positions are still reviewed regardless).

## 2. Review existing positions first (always, regardless of regime)

For every row in `positions.md` (equity and options separately):
- Equity: `GLOBAL_QUOTE` for current price, and `COMPANY_OVERVIEW` (cheap)
  for the current `50DayMovingAverage`.
- If `R-target reached?` is still "No": check if current price has now hit
  the R-target — if so, flip the flag to "Yes" in `positions.md` (this
  never flips back). Whether or not it just flipped, also check if price
  has hit the original fixed stop.
- If `R-target reached?` is "Yes": pull `EMA` (21, daily), extract the
  last 5 values (same jq/tail rule as step 4 below), and set
  `Current exit stop = max(EMA21, original stop)`. Check if price has
  closed at/below this trailing stop.
- Options: Robinhood `get_option_quotes` for current premium, and check
  days-to-expiration remaining vs. both the 21-DTE checkpoint and 50% of
  DTE-at-entry.
- Check against the full exit rules in `strategy.md` (current exit stop
  per the above, 50DMA trend-template break, time-stop, options DTE
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
`TOP_GAINERS_LOSERS`, pull `COMPANY_OVERVIEW` + `GLOBAL_QUOTE`. Apply the
Tier 1 trend-template checklist. Drop anything that fails.

### Tier 2 — shortlist
For Tier 1 survivors, call `RSI` (14, daily) and `EMA` (8 and 21, daily).
**Extraction rule for these and all indicator/time-series tools in this
workflow:** the result will be saved to a file because it's too large to
return inline. Do **not** read the whole file. Use `jq` (or `tail`) to pull
just the last 5 data points, e.g. roughly:
`jq '.data | to_entries | .[-5:]' <saved_file>` (adjust the path expression
to the actual JSON shape returned — inspect the file's top-level keys with
`jq keys` first if unsure, rather than dumping the whole thing).
Apply the Tier 2 breakout/pullback trigger logic. Drop anything that fails.

### Tier 3 — finalists (cap 3 per gates.md)
For Tier 2 survivors: check the EMA8/EMA21 spread from the values already
extracted in Tier 2 (no new call — `MACD` is permanently premium-gated on
this plan and must never be called; see `gates.md`). Then pull
`NEWS_SENTIMENT` (limit param 5-10), `EARNINGS_CALENDAR` (per symbol), and
`ATR` (14, daily) — same last-5-values extraction rule for `ATR`. Apply
the Tier 3 confirmation checklist. Drop anything that fails.

## 4. Decide instrument: equity, options, or skip

For each finalist that survives Tier 3:
- Default to equity.
- Consider an options expression only if it clearly improves capital
  efficiency for this specific signal — check Robinhood `get_option_chains`
  for the symbol and evaluate against the DTE/delta/liquidity gates in
  `gates.md`. If no expiration/strike meets all of them, use equity instead
  — do not loosen the options gates to force a fit.

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
`run: 2026-08-10 premarket — 1 opened (equity), 0 closed, 1 rejected by gates, regime: risk-on`

Push to `main`. If this step fails or is skipped, the next run starts
blind to everything that happened today.

## Handling errors gracefully

- Premium/rate-limit error on any call: log it, skip that symbol/tier
  (don't abort the whole run), prioritize finishing the position-review
  step over scouting new ones, and note the truncation in the journal.
- Never treat a data error as a reason to fall back to guessing a value —
  skip the candidate instead.

## Hard rules, restated

- Never call a real order-placement tool — equity or option.
- Never edit `gates.md` or `framework.md`.
- Never delete or rewrite past `trade_journal.md` entries.
- Never read a full RSI/EMA/MACD/ATR/time-series file into context —
  extract only what's needed via `jq`/`tail`.
- Always commit and push before ending the run.
