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
- `trade_journal.md` — append-only log of every run's decisions.
- `run_instructions.md` — the step-by-step playbook the twice-daily
  scheduled agent follows: full scan for new candidates plus position
  review.
- `position_monitor_instructions.md` — a second, lightweight playbook for
  an hourly (market-hours-only) routine that only reviews existing open
  positions for exits — shrinks the gap between "a stop is hit" and "the
  system notices" without re-running the full scan. Robinhood-only, no
  Alpha Vantage calls, skips instantly (zero cost) when there's nothing
  open.

## Status

Paper trading. Not connected to real order execution (equity or options).
Review the placeholder values in `gates.md` before relying on this for
anything. No backtest has been run yet — see `framework.md`.
