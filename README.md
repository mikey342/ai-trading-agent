# AI Trading Agent (Paper Trading)

An adaptive, cloud-scheduled quant research agent. It scouts stocks,
reads indicators/fundamentals/news, decides on trades, and simulates
execution — currently **paper trading only**, no real orders are placed.

## Files

- `gates.md` — hard risk limits. Agent reads but never edits this.
  **Edit this by hand** to change risk tolerance or go live.
- `strategy.md` — the adaptive playbook (indicators, entry/exit rules,
  watchlist). The agent evolves this over time based on results, with a
  changelog of every change and why.
- `positions.md` — current simulated portfolio state.
- `trade_journal.md` — append-only log of every run's decisions.
- `run_instructions.md` — the step-by-step playbook the scheduled agent
  follows every run.

## Status

Paper trading. Not connected to real order execution. Review the
placeholder values in `gates.md` before relying on this for anything.
