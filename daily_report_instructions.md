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

## 3. Publish the report as an Artifact

After writing `daily_report.md`, publish it as a shareable page — but
treat this as **best-effort**: it must never block or fail the run.

1. Read `artifact_url.txt`. If it contains a URL, that is the existing
   report page.
2. Call the `Artifact` tool with `file_path` set to `daily_report.md`,
   and — **critically** — pass that stored URL as the `url` parameter.
   Without it, every run mints a brand-new link, because each scheduled
   run is a separate session that didn't publish the original. Passing
   `url` updates the same page in place so the human keeps one stable
   bookmark.
   - `description`: a one-line summary of the day (e.g. "2 positions
     opened, 1 closed, NAV +0.4%").
   - `favicon`: keep it **stable** at `📊` across every run. A changing
     favicon reads as a different page.
3. If the call succeeds and returns a URL different from the stored one,
   overwrite `artifact_url.txt` with the new URL so tomorrow's run
   updates the right page.
4. If `artifact_url.txt` is missing or empty, publish without `url` and
   write the returned URL into that file.

**If the `Artifact` tool is unavailable in this environment**, or the
call errors for any reason: do not retry in a loop and do not fail the
run. Add one line to the report's Watch items noting that publishing was
unavailable, then continue to the commit step. The committed
`daily_report.md` is the source of truth; the artifact is a convenience
layer on top of it.

## 4. Commit and push

Commit as `report: YYYY-MM-DD daily summary` and push to `main` —
including `artifact_url.txt` if it changed.

## Hard rules

- Never call any order-placement tool.
- Never open, close, or size a position.
- Never edit `gates.md`, `framework.md`, `strategy.md`, `positions.md`,
  `trade_log.csv`, or `trade_journal.md` — this routine is read-only
  everywhere except `daily_report.md` and `artifact_url.txt`.
- Never invent activity. An empty day is a valid, useful report.
