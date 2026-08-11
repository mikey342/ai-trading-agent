# Run Instructions — End-of-Day Report (Reporting Only)

You are a scheduled cloud agent running **once daily at 4:15pm ET**,
after the market close, against a fresh clone of this repo. Your job is
to summarize what the system did today. **You take no trading action of
any kind** — no scouting, no funnel, no positions opened or closed, no
exits evaluated. Those belong to the scan and monitor routines.

You write one **dated** report per trading day into the `reports/`
directory, plus a row in that directory's index. Nothing is ever
overwritten — each day's report is permanent.

## 1. Gather today's activity

Read, and pull out only entries dated **today**:

- `trade_journal.md` — every run's narrative entry from today
- `trade_log.csv` — every row with today's date (opens, closes,
  rejections)
- `positions.md` — the current book, plus today's NAV history rows
- `premarket_notes.md` — if dated today, the morning's gap/catalyst read

If there was no activity at all today (no runs logged, market holiday),
say so in one line and stop — do not manufacture content.

## 2. Write reports/daily_report_YYYY-MM-DD.md

Create a **new** file named for today's date in US/Eastern — e.g.
`reports/daily_report_2026-08-11.md`. Use the trading day's date, not the
UTC date: the 4:15pm ET slot is the same calendar day in both zones, but
never assume that, and never overwrite a previous day's file. The
filename becomes the published page's title, which is why it carries the
date.

Keep it factual and short; this is a record, not an essay.

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
test — **deduplicated by (symbol, date)**, never raw rows. A name
evaluated at more than one run in a day produces identical rejections,
because daily-bar indicators don't move intraday; counting rows would
reach the threshold on a fraction of the intended evidence. Report as a
running tally (e.g. "momentum test: 3 of 3 distinct breakout candidates
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

## 3. Publish it as its own Artifact, then index it

Each day gets a **permanent page of its own** — never update a previous
day's page. Publishing is still **best-effort** and must never block or
fail the run.

1. Call the `Artifact` tool with `file_path` set to today's file
   (`reports/daily_report_YYYY-MM-DD.md`). **Do not pass a `url`
   parameter** — omitting it is what mints a fresh page. Passing one
   would overwrite a previous day's report, destroying the archive.
   - `description`: a one-line summary of the day (e.g. "2 positions
     opened, 1 closed, NAV +0.4%").
   - `favicon`: keep it **stable** at `📊` across every run, so all
     reports are recognizable as the same series.
2. Take the returned URL and **prepend a row to the table in
   `reports/INDEX.md`** (newest first): the date, a markdown link titled
   `daily_report_YYYY-MM-DD`, and a short headline for the day. Without
   this row the page is effectively unfindable — the index is the only
   map to the archive.
3. Never edit or remove existing rows in `reports/INDEX.md`.

**If the `Artifact` tool is unavailable** or the call errors: do not
retry in a loop and do not fail the run. Add one line to the report's
Watch items noting that publishing was unavailable, add the index row
with `(not published)` in place of the link, and continue to the commit
step. The committed markdown is the source of truth; the artifact is a
convenience layer on top of it.

## 4. Commit and push

Commit as `report: YYYY-MM-DD daily summary` and push to `main` —
including the new `reports/daily_report_YYYY-MM-DD.md` and the updated
`reports/INDEX.md`.

## Hard rules

- Never call any order-placement tool.
- Never open, close, or size a position.
- Never edit `gates.md`, `framework.md`, `strategy.md`, `positions.md`,
  `trade_log.csv`, or `trade_journal.md` — this routine may only create
  today's `reports/daily_report_YYYY-MM-DD.md` and prepend a row to
  `reports/INDEX.md`.
- Never overwrite a previous day's report file or republish over a
  previous day's page. The archive is permanent.
- Never invent activity. An empty day is a valid, useful report.
