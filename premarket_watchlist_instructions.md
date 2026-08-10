# Run Instructions — Premarket Watchlist (Informational Only)

You are a scheduled cloud agent running **once daily, ~9:15am ET** (15
minutes before market open), against a fresh clone of this repo. Your job
is narrow: surface what moved overnight so the 9:30am morning run has
context. **You never open, size, or simulate a trade. You never touch
`positions.md`, `trade_journal.md`, or NAV. You write exactly one file:
`premarket_notes.md`.**

This exists because a stock that gapped hard on overnight news is a
completely different situation at the open than one that's flat — this
routine's only purpose is making that visible before the real analysis
runs, not deciding anything itself.

## 1. Gather the watchlist

Read the seed watchlist from `strategy.md`, plus pull `TOP_GAINERS_LOSERS`
(Alpha Vantage — cheap, one call) for opportunistic names.

## 2. Check premarket movement

For every symbol, call `get_equity_quotes` (Robinhood, batched — up to 20
symbols per call). This returns both the current (premarket) price and
the official last-completed-session close in one call. Compute:

`gap_pct = (current_price - prior_close) / prior_close`

Flag a symbol as **notable** if `|gap_pct| >= 2%`.

## 3. Light catalyst check — notable movers only

To keep this cheap, only do this for symbols flagged notable in step 2
(not the whole universe):
- `get_earnings_results` (Robinhood): did this symbol report earnings
  this morning (report date = today)? If so, note actual vs. estimate.
- `NEWS_SENTIMENT` (Alpha Vantage, limit=5, this symbol only): a quick
  read on whether there's an obvious news catalyst. Use sparingly — this
  is the one Alpha Vantage call in this routine, keep it to notable
  movers only, not the full universe.

If no clear catalyst is found, say so plainly ("gapping, no obvious
catalyst found") rather than guessing one.

## 4. Write premarket_notes.md

Overwrite the file (this is daily prep, not a permanent record — no
lasting value once the trading day is over, unlike `trade_journal.md`
which must never be overwritten). Format:

```
# Premarket Notes — YYYY-MM-DD

## Notable movers
- SYMBOL: gap_pct%, prior close $X -> premarket $Y — catalyst: ... (or
  "no obvious catalyst found")

## Everything else
Flat / no notable premarket movement: SYMBOL, SYMBOL, ...
```

## 5. Commit and push

Commit message: `premarket: 2026-08-10 — 2 notable movers (JPM +2.3%, XOM -2.1%)`
(or `0 notable movers` if nothing flagged). Push to `main`.

## Hard rules

- Never call any order-placement tool, real or otherwise — this routine
  has no trading logic at all, but state it anyway.
- Never edit `gates.md`, `framework.md`, `strategy.md`, `positions.md`,
  or `trade_journal.md`.
- Never treat anything in this file as a trading decision — the open run
  always re-checks real prices and runs the full Tier 1-3 gating
  regardless of what's noted here.
