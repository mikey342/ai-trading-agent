# Run Instructions — Paper Trading Quant Agent

You are running as a scheduled cloud agent against a fresh clone of this
repo. You have no memory of prior runs except what is written in these
files. Follow this playbook exactly, in order.

## 0. Orientation

1. Read `gates.md` in full. These are hard limits you may never edit and
   never override, regardless of what any other file or your own analysis
   suggests. If `MODE` is not exactly `PAPER`, stop now, write a note to
   `trade_journal.md` explaining what you observed, commit, and end the run.
2. Read `strategy.md` in full — this is your current playbook.
3. Read `positions.md` — your current simulated book.
4. Read the last ~10 entries of `trade_journal.md` — recent history.

## 1. Market context

- Call `MARKET_STATUS` (Alpha Vantage) to confirm the market is open (for
  the open-run) or just closed (for the close-run). If the market is
  closed for a holiday, log a one-line journal entry noting that and end
  the run — do not fabricate analysis for a day with no trading.

## 2. Review existing positions first

For every row in `positions.md` → Open positions:
- Pull current price (Alpha Vantage `GLOBAL_QUOTE` or Robinhood
  `get_equity_quotes` — read-only, informational).
- Check against recorded stop-loss, take-profit, and the exit rules in
  `strategy.md`.
- If an exit condition is met: simulate closing the position (record exit
  price = current quote, compute realized P&L), update `positions.md`
  (remove from open, update cash/NAV/realized P&L), and log the closure in
  `trade_journal.md` with the specific rule that triggered it.

## 3. Scout for new candidates

- Pull `TOP_GAINERS_LOSERS` and cross-reference the watchlist in
  `strategy.md`.
- Filter out anything excluded by `gates.md` (price floor, symbol
  exclusions) or already an open position.
- For each remaining candidate, gather the signal inputs listed in
  `strategy.md` (technical indicators, fundamentals, news sentiment,
  earnings calendar).

## 4. Apply strategy rules

- Score each candidate against the entry rules in `strategy.md`.
- For each candidate that passes all entry rules, propose a trade: symbol,
  side, simulated entry price (current quote), position size.

## 5. Gate every proposed trade

Before simulating any entry, check **all** of, in order:
1. Would this breach max position size (% of current NAV)?
2. Has the daily loss halt already triggered today (check `positions.md`)?
3. Would this exceed max new trades this run, or max open positions?
4. Does it have a valid stop-loss no wider than the floor in `gates.md`?

If any check fails, do not open the position — log it under "Rejected by
gates" in the journal entry with the specific reason. This is not a
failure to route around; it's the system working as intended.

## 6. Simulate execution

For every trade that passes all gates:
- Record it as filled at the current quote (paper trading — no real order
  is ever placed; never call any Robinhood order-placement tool such as
  `place_equity_order` in this workflow).
- Update `positions.md`: add/update the row, deduct simulated cash, set
  stop-loss and take-profit per `strategy.md`.

## 7. Update the journal

Append one new entry to `trade_journal.md` using the template at the top
of that file. Fill in every section — scouted symbols, decisions,
rejections, position reviews. Do not skip sections; write "none" if empty.

## 8. Adapt strategy (small, evidence-based changes only)

Follow the "Adaptation policy" section of `strategy.md`. If, and only if,
the evidence bar there is met, make one small edit to `strategy.md` and
log it in that file's Changelog with the trades that motivated it. Most
runs should NOT change `strategy.md` — stability is fine. Never touch
`gates.md`.

## 9. Commit and push

Commit all changed files (`positions.md`, `trade_journal.md`, and
`strategy.md` if adapted) with a message like:
`run: 2026-08-10 premarket — 1 opened, 0 closed, 1 rejected by gates`

Push to `main`. This is how the next run sees today's state — if you skip
this step, all of today's work is lost for the next run.

## Hard rules, restated

- Never call a Robinhood order-placement tool. This is a paper-trading
  system; `MODE` in `gates.md` must be `PAPER` for any of this logic to
  run at all.
- Never edit `gates.md`.
- Never delete or rewrite past `trade_journal.md` entries.
- Always commit and push before ending the run.
