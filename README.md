# AI Trading Agent (Paper Trading — Swing)

An adaptive, cloud-scheduled swing-trading research agent. It scans a
watchlist, applies a tiered trend/momentum/catalyst screen, decides on
equity or defined-risk options trades, and simulates execution —
currently **paper trading only**, no real orders are ever placed.

Holding period target: ~5-15 trading days (swing, not day trading).

## Files

- `framework.md` — the methodology this is grounded in (Turtle/Donchian
  trend-following, Minervini trend template, O'Neil relative strength,
  Connors RSI pullback, PEAD earnings drift) and what's deliberately not
  validated yet (no backtest).
- `gates.md` — hard risk limits, including options and leverage limits.
  Agent reads but never edits this. **Edit this by hand** to change risk
  tolerance or go live.
- `strategy.md` — the adaptive playbook: the funnel (Tier 1/2/3 screening),
  entry/exit rules, position sizing, options/leverage overlay. The agent
  evolves this over time based on results, with a changelog of every
  change and why.
- `positions.md` — current simulated equity + options book, and NAV
  history.
- `trade_journal.md` — append-only narrative log of every run's decisions,
  including a mandatory technical rationale for every trade opened.
- `trade_log.csv` — the same decisions as machine-readable rows (opens,
  closes, and rejections, each with its full indicator snapshot) for
  computing win rate, expectancy, and R-multiples. Schema in
  `DATA_SCHEMA.md`.
- `DATA_SCHEMA.md` — column definitions for `trade_log.csv`, plus an
  honest note on why a forward paper record is not the same thing as a
  backtest.
- `tool_verification.md` — which data tools have actually been called and
  confirmed working, versus assumed. Worth reading before trusting any
  rule that depends on a data field.
- `run_instructions.md` — the step-by-step playbook the scheduled scan
  agent follows (3x/day: market open, midday, close): full scan for new
  candidates plus position review.
- `position_monitor_instructions.md` — a second, lightweight playbook for
  an hourly (market-hours-only) routine that only reviews existing open
  positions for exits — shrinks the gap between "a stop is hit" and "the
  system notices" without re-running the full scan. Robinhood-only, no
  Alpha Vantage calls, skips instantly (zero cost) when there's nothing
  open.
- `premarket_watchlist_instructions.md` — a third playbook, once daily at
  9:15am ET (15 min before open): flags notable overnight premarket
  movers and any obvious catalyst, purely as context for the 9:30am open
  run. Never opens, sizes, or decides a trade. Writes `premarket_notes.md`
  (overwritten fresh each morning).

## Status

Paper trading. Not connected to real order execution (equity or options).
Review the placeholder values in `gates.md` before relying on this for
anything. No backtest has been run yet — see `framework.md`.
