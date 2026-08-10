# Strategy — Adaptive

This file is the agent's own playbook. Unlike `gates.md`, the agent **may
edit this file** as it learns from `trade_journal.md` — but every edit must
be a small, explainable change, committed with a rationale, and logged in
the Changelog section at the bottom. Do not rewrite this file wholesale in
one run.

## Watchlist / universe seed

Starting watchlist (large-cap, liquid — edit freely):
AAPL, MSFT, GOOGL, AMZN, NVDA, META, TSLA, JPM, V, UNH

Plus whatever `TOP_GAINERS_LOSERS` surfaces each run, filtered by the
exclusions in `gates.md` (no sub-$5 stocks).

## Signal inputs

For each candidate symbol, pull:
- **Price/technical**: `TIME_SERIES_DAILY_ADJUSTED`, `RSI`, `MACD`,
  `SMA` (20/50/200), `BBANDS`, `OBV` (volume confirmation)
- **Fundamentals**: `COMPANY_OVERVIEW` (P/E, PEG, margins), `EARNINGS`
  (recent surprise %), `EARNINGS_CALENDAR` (upcoming — avoid opening new
  positions within 3 days of an earnings date)
- **News**: `NEWS_SENTIMENT` for the symbol, last 3 days

## Entry rules (current)

Open a new long paper position only if **all** of:
1. RSI(14) between 40–65 (avoid chasing overbought, avoid catching a falling knife)
2. Price above SMA(50)
3. MACD line above signal line (bullish momentum) or a recent bullish crossover (within 3 days)
4. News sentiment score not negative (Alpha Vantage sentiment ≥ neutral)
5. No earnings report scheduled within the next 3 calendar days
6. Passes all `gates.md` checks (position size, daily loss halt, max positions, max new trades)

## Exit rules (current)

Close a paper position if **any** of:
1. Price hits the recorded stop-loss (set at entry, per `gates.md` floor)
2. Price hits the recorded take-profit (default: 2x the stop distance, i.e. 2:1 reward:risk)
3. RSI(14) crosses above 75 (take profit on extreme overbought)
4. News sentiment turns sharply negative intraday
5. Position held > 20 trading days with no progress toward target (time stop)

## Adaptation policy

Every run, after updating the journal:
- Look at the last 10 closed paper trades in `trade_journal.md`.
- If a specific rule (e.g. "RSI 40-65 entry filter") is associated with a
  losing pattern across ≥5 of the last 10 trades, propose ONE small,
  specific adjustment (e.g. narrow the RSI band, tighten the earnings
  blackout window) — not a wholesale strategy rewrite.
- Log the change below with: date, what changed, why (cite the trades),
  and what you'll watch for to know if it worked.
- Never loosen a `gates.md`-governed parameter to "fix" a losing streak —
  gates are not yours to touch.

## Changelog

- (seed) Initial strategy scaffolded by assistant setup, not yet run.
