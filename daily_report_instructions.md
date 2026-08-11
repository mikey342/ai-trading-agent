# Run Instructions — End-of-Day Report (Reporting Only)

You are a scheduled cloud agent running **once daily at 4:15pm ET**,
after the market close, against a fresh clone of this repo. Your job is
to summarize what the system did today. **You take no trading action of
any kind** — no scouting, no funnel, no positions opened or closed, no
exits evaluated. Those belong to the scan and monitor routines.

You write exactly one file: `daily_report.md`, overwritten fresh each
day.

## 1. Gather today's activity

Read, and pull out only entries dated **today**:

- `trade_journal.md` — every run's narrative entry from today
- `trade_log.csv` — every row with today's date (opens, closes,
  rejections)
- `positions.md` — the current book, plus today's NAV history rows
- `premarket_notes.md` — if dated today, the morning's gap/catalyst read

If there was no activity at all today (no runs logged, market holiday),
say so in one line and stop — do not manufacture content.

## 2. Write daily_report.md

Overwrite the file with this structure. Keep it factual and short;
this is a record, not an essay.

```
# Daily Report — YYYY-MM-DD

## Summary
One paragraph: regime (long/short mode), how many runs executed, how many
positions opened/closed, and the day's realized and unrealized P&L.

## Account
| Field | Open of day | Close of day | Change |
NAV, cash, open position count, whether the daily-loss halt tripped.

## Positions opened today
Per position: symbol, sleeve, direction, counter-trend?, qty, fill price,
stop, R-target, and the one-sentence thesis from the journal. If none,
say "None" and give the most common reason candidates were rejected.

## Positions closed today
Per position: symbol, exit rule that fired, exit price, realized P&L,
R-multiple, and how many days it was held.

## Open book
Current positions with entry, current price, unrealized P&L, distance to
stop, and distance to R-target.

## Rejections worth noting
From trade_log.csv rows with action=reject. Group by which check failed.
This section matters: a filter that keeps declining setups that would
have worked is invisible unless rejections are read.

## Data quality
Any degraded inputs today — Alpha Vantage quota exhaustion, unavailable
news sentiment, failed calls, market-closed fallbacks. Be specific; this
is how systematic data problems get noticed.

## Open-question tracking (include when relevant)
`strategy.md` carries a pre-registered decision rule on the breakout
momentum test. Each day, count the cumulative `action=reject` rows in
`trade_log.csv` where `trigger=breakout` and the notes cite the momentum
test, alongside the total breakout candidates that reached Tier 3. Report
both as a running tally (e.g. "momentum test: 4 of 4 breakout candidates
rejected, across 1 session"). State plainly whether the pre-registered
threshold — ≥15 candidates over ≥10 sessions with ≥90% rejected — has
been met. Do **not** propose changing the rule before it is met; that
threshold exists precisely to stop a small sample from driving a change.

## Watch items
Anything a human should look at: a position near its stop, a counter-trend
trade in progress, a rule that behaved unexpectedly, a strategy.md
adaptation made today.
```

Close every report with this line verbatim, so no reader mistakes the
figures for a real account:

> Paper trading — simulated book, no real orders placed. P&L charges the
> bid-ask spread but omits fill uncertainty, slippage, options fees, ETF
> expense ratios, and dividends; expect live results to run modestly
> worse. See `DATA_SCHEMA.md`.

## 3. Commit and push

Commit as `report: YYYY-MM-DD daily summary` and push to `main`.

## Hard rules

- Never call any order-placement tool.
- Never open, close, or size a position.
- Never edit `gates.md`, `framework.md`, `strategy.md`, `positions.md`,
  `trade_log.csv`, or `trade_journal.md` — this routine is read-only
  everywhere except `daily_report.md`.
- Never invent activity. An empty day is a valid, useful report.
