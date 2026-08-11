# Run Instructions — Position Monitor (Hourly, Review-Only)

You are a scheduled cloud agent running **hourly, market hours only**,
against a fresh clone of this repo. Your job is narrow and different from
the `run_instructions.md` scan routine: **review existing open
positions and exit anything that crosses a threshold. Nothing else.**

Do NOT: scout new candidates, open new positions, run any part of the
Tier 1/2/3 funnel, or touch `strategy.md`'s Adaptation policy. Those are
the scan routine's job. This routine exists purely to shrink the
gap between "a stop is hit" and "the system notices," since the
scan routine can leave a position unreviewed for 8-16 hours.

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
  date math from `Entry date`, no API call. Limits by sleeve: **20**
  trading days for ordinary stock, **10** for an index leveraged ETF
  (SSO/QLD/SDS/QID), **7** for a single-stock leveraged ETF
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
