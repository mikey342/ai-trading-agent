# AI Trading Agent (Paper Trading — Swing)

An adaptive, cloud-scheduled swing-trading research agent. It screens the
US equity market three times a day, decides on equity, 2x-ETF or
defined-risk options trades, and simulates execution — currently **paper
trading only**, no real orders are ever placed.

**Two independent strategies run over one $10,000 book:**

| | Sleeve A — breakout | Sleeve B — mean reversion |
|---|---|---|
| Buys | strength breaking to a 7-day high on volume | sharp 2-day weakness inside a long-term uptrend |
| Entry | `breakout_7d` / `momentum_vol` + volume ≥1.4x | `RSI(2) < 15`, price > SMA(200), first up-close |
| Exit | trailing EMA21 after 2R; SMA(20) break | `RSI(2) > 70`; SMA(200) break |
| Max hold | 10 trading days (suspended once 2R hit) | 5 trading days, never suspended |
| Expects | ~35-45% win rate, carried by a fat tail | high win rate, capped wins |

They have **opposite payoff structures**, so their statistics must never
be pooled — see the `sleeve` column in `DATA_SCHEMA.md`. Holding period is
capped at **10 trading days** (2 calendar weeks) for anything unresolved.

**Separate from the trading agent entirely:** `.claude/skills/spike-fade/`
is an on-demand Claude Code skill — give it any ticker in chat and it
checks whether it's overbought after a spike (possible fade, via puts)
or oversold within an uptrend (possible bounce), for research only. It
never opens, sizes, or logs a trade, and doesn't touch `positions.md`,
`trade_log.csv`, or `gates.md` — entirely outside the scheduled routines
above.

There is **no watchlist**. A saved market-wide screener defines the
universe on every run.

Bearish exposure is taken by *buying* 2x inverse ETFs with cash. **Short
selling and margin are forbidden outright.**

## Files

- `framework.md` — the methodology this is grounded in (Donchian channel
  breakouts, Minervini-style trend template, O'Neil volume confirmation,
  Connors & Alvarez RSI(2) mean reversion, Jegadeesh/Lehmann short-term
  reversal, PEAD earnings drift) **and an explicit evidence audit** of
  which rules carry published support and which are conventions. Note the
  George & Hwang 52-week-high trigger was **removed 2026-08-12**, which
  weakened Sleeve A's entry evidence — recorded there rather than glossed.
  Nothing here has been backtested.
- `gates.md` — hard risk limits, including options and leverage limits.
  Agent reads but never edits this. **Edit this by hand** to change risk
  tolerance or go live.
- `strategy.md` — the adaptive playbook: both sleeves' funnels (Tier 0-3),
  entry/exit rules, position sizing, instrument selection, options and
  leverage overlays. Carries a changelog of every rule change and why,
  plus **pre-registered criteria** for the experiments currently running
  (`momentum_vol`, and the `RSI(2) < 15` threshold) — fixed in advance so
  results cannot be rationalised after the fact.
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
  movers and any obvious catalyst, purely as context for the 9:30am morning
  run. Never opens, sizes, or decides a trade. Writes `premarket_notes.md`
  (overwritten fresh each morning).
- `daily_report_instructions.md` — a fourth playbook, once daily at
  4:15pm ET after the close: summarizes the day's runs, fills, exits,
  rejections, and data-quality problems into a dated
  `reports/daily_report_YYYY-MM-DD.md`, publishes it as its own
  permanent page, and indexes the link in `reports/INDEX.md`
  (overwritten each day). Read-only everywhere else; takes no trading
  action.

## Schedule

| Time (ET) | Routine | Role |
|---|---|---|
| 9:15am | Premarket watchlist | Gap/catalyst context. Informational only. |
| 9:30am · 2:30pm · 3:30pm | Scan | Full funnel, both directions. Can open positions and adapt `strategy.md`. |
| 10am–4pm, hourly | Position monitor | Exits only. Costs nothing when the book is empty. |
| 4:15pm | Daily report | Writes a dated report to `reports/`, publishes it, indexes the link. No trading action. |

**Reading the reports:** `reports/INDEX.md` is the map — one row per
trading day, newest first, each linking to that day's permanent page.

All trading runs sit inside regular market hours — never pre-open (stale
or thin quotes), never post-close (a decision that could not be filled
until the next session).

## Status

**Paper trading only.** `MODE: PAPER` in `gates.md` is a hard rule every
routine checks first; no real order-placement tool is ever called. Short
selling and margin are both disabled outright — bearish exposure is a
*long* position in an inverse ETF.

Risk parameters are finalized (not placeholders): $10,000 paper account,
15% max position, 0.4% risk per trade, −2% daily loss halt, 6 max
positions, 1 max counter-trend.

**Not yet validated:** no backtest of this synthesis exists — see
`framework.md` for an honest audit of which parts rest on peer-reviewed
evidence and which are practitioner heuristics.
