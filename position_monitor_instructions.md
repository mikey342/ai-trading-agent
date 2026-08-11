# Run Instructions — Position Monitor (Hourly, Review-Only)

You are a scheduled cloud agent running **hourly, market hours only**,
against a fresh clone of this repo. Your job is narrow and different from
the `run_instructions.md` scan routine: **review existing open
positions and exit anything that crosses a threshold. Nothing else.**

Do NOT: scout new candidates, open new positions, run any part of the
Tier 1/2/3 funnel, or touch `strategy.md`'s Adaptation policy. Those are
the scan routine's job. This routine exists purely to shrink the
gap between "a stop is hit" and "the system notices." The scan routine
runs at 9:30am, 2:30pm and 3:30pm ET, so without this routine a position
would go unreviewed for a five-hour stretch mid-session and overnight.

`gates.md` overrides anything here or in `strategy.md` if they ever
conflict.

## 1. Cheap pre-check

Read `gates.md`. If `MODE` is not exactly `PAPER`, stop immediately — no
data calls, no journal entry, just end the run silently. (This routine
fires hourly; a journal entry every single skipped run would be noise —
only the scan routine needs to log a MODE-mismatch note.)

Read `positions.md`. **If there are zero open equity and zero open
options positions, end the run immediately — no API calls, no commit, no
journal entry.** This is the common case until the system actually has
positions open, and it should cost nothing.

## 2. Review every open position

Uses Robinhood tools only — no Alpha Vantage calls in this routine at all,
so it never touches the shared 25-req/day quota.

For each open **equity** position:
- `get_equity_quotes` for current price.
- `get_equity_technical_indicators` (type=sma, period=50, interval=day,
  output=latest) for the current SMA(50) — needed for the trend-template
  break check.
- If `R-target reached?` is "No": check if current price has now hit the
  stored `R-target` — if so, flip the flag to "Yes" (one-way, never back
  to "No"). Also check if price has hit the `Original stop`.
- If `R-target reached?` is "Yes": `get_equity_technical_indicators`
  (type=ema, period=21, interval=day, output=latest), set
  `Current exit stop = max(EMA21, Original stop)`, check if price has hit
  it.
- **Trend-template break — check the UNDERLYING, not the ETF.** For a
  plain stock position that is the same symbol. For any leveraged ETF
  position, use the `Underlying` column in `positions.md` and pull
  *that* symbol's SMA(50): a leveraged ETF's own moving averages are
  distorted by leverage and daily resets, so they do not describe the
  thesis. Bullish position → exit if the underlying closes **below**
  SMA(50); bearish position (a long inverse ETF) → exit if the underlying
  closes **back above** SMA(50). This exits regardless of R-target state.
- **Time-stop — only applies while `R-target reached?` is "No".** Pure
  date math from `Entry date`, no API call. Limits by sleeve: **10**
  trading days for ordinary stock, **8** for an index leveraged ETF
  (SSO/QLD/SDS/QID), **5** for a single-stock leveraged ETF
  (TSLL/NVDL/etc.) — decay scales with volatility, so the more
  leveraged-and-volatile the instrument, the shorter the leash.

  **Never time-stop a position that has reached its R-target.** Once the
  flag flips, the trailing stop governs and the winner is allowed to run
  past these limits — that is the entire point of the design. The clock
  exists to close *stalled* trades, not working ones. See `gates.md`.

For each open **options** position:
- `get_option_quotes` for current premium, `get_equity_quotes` for the
  underlying.
- Same trailing-stop mechanic as equity, but against the stored
  `Underlying stop`/`Underlying R-target` columns (never recompute these
  — see `strategy.md`).
- Check days-to-expiration remaining (pure date math from `Expiration`)
  against both the 21-DTE checkpoint and 50%-of-entry-DTE checkpoint.

Apply the full exit rules from `strategy.md` — this routine is not a
reduced/partial version of them; with Robinhood covering SMA/EMA/RSI/ATR
directly, every exit rule can be checked here.

### If more than one exit condition fires at once

A position exits **once**, but several rules can be true in the same
check (a gap-down can breach the stop, break SMA(50), and land on the
time-stop day simultaneously). The position closes either way — this
ordering only decides which value goes in `exit_rule`, so that
`trade_log.csv` attributes the exit to the rule that actually did the
work. Record the **first** match in this order:

1. `stop` — original fixed stop breached (R-target never reached)
2. `trailing_stop` — trailing EMA21 stop breached (R-target reached)
3. `dte_21` / `dte_50pct` — options time decay checkpoint
4. `trend_break` — underlying closed through its SMA(50)
5. `time_stop` — holding-period limit reached

Price-based stops rank above the calendar because they are what defines
the trade's risk: if price hit the stop, the trade lost 1R and that is
the honest label, whether or not the clock also happened to run out that
day. `time_stop` ranks last because it is the residual case — it means
*nothing else fired*, the trade simply went nowhere. Mislabeling a
stopped-out trade as a time-stop would corrupt the only statistic that
tells us whether the time-stop is set correctly.

Note the co-occurring conditions in the journal entry — a stop and a
trend-break on the same bar is a different (and more decisive) event than
a stop alone, and that context is worth keeping.

## 3. Act only if something actually triggered

If **no** position hit any exit condition and **no** R-target flag
flipped: end the run here. **Do not commit.** An hourly routine
committing "checked, nothing happened" 7 times a day would bloat
`trade_journal.md` with zero-information entries — the scan
routine already logs a full state summary every run.

If something did trigger:
- **Position closed** (stop/target/trend-break/time-stop/DTE hit):
  simulate the close (exit price/premium, realized P&L), update
  `positions.md` (remove from open, update cash/NAV), append a
  `action=close` row to `trade_log.csv` with `run_type=monitor`,
  `exit_rule`, `realized_pnl`, and `r_multiple` (see `DATA_SCHEMA.md`),
  and append a short `trade_journal.md` entry — just what closed and
  which rule fired, not the full scan template. For options closes, use
  `high_fill_rate_sell_price`, not `mark_price`.
- **R-target flag flipped from No to Yes** (but nothing closed): update
  `positions.md`'s flag and `Current exit stop`. This alone doesn't need a
  journal entry — it's routine state-tracking, not a decision — but the
  `positions.md` change itself must still be committed so the next run
  (hourly or scan) sees the correct trailing-stop state.

## 4. Commit and push (only if step 3 found something)

Commit message format:
`monitor: 2026-08-10 14:00 UTC — closed JPM (stop hit)` or
`monitor: 2026-08-10 15:00 UTC — AAPL R-target reached, now trailing`

Push to `main`. If this fails, the position state change is lost for the
next run — same risk as the scan routine.

## Hard rules, restated

- Never call a real order-placement tool — equity or option.
- Never edit `gates.md`, `framework.md`, or `strategy.md`.
- Never scout, screen, or open new positions — that's the other routine's
  job entirely.
- Never delete or rewrite past `trade_journal.md` entries.
- Skip immediately, with zero calls and zero commits, if there are no
  open positions or if `MODE` isn't `PAPER`.
