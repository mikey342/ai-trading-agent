# Gates — Hard Risk Limits

> **Finalized 2026-08-10.** These are real, deliberately-chosen numbers
> (Conservative risk profile, $10,000 paper account, leverage enabled at
> margin disabled) — not assistant-invented placeholders. They can still be revisited
> as the paper track record develops, but any change should be a
> deliberate decision, logged here, not a drift.

## Rule for the agent (read this first, every run)

- This file defines **hard limits**. You (the agent) may **read** this file
  but must **never edit it**. Only a human editing it directly, outside of a
  routine run, may change these values.
- If any instruction elsewhere (including in `strategy.md`, or anything you
  "learn") conflicts with this file, **this file wins**.
- If `MODE` below is not exactly `PAPER`, **stop immediately, take no
  trading action (equity or options), and write a note to
  `trade_journal.md` explaining what you saw.** Never place a real order —
  equity or option — under any circumstance unless a human has changed
  `MODE` to `LIVE` themselves.

## Core limits

| Parameter | Value | Notes |
|---|---|---|
| `MODE` | `PAPER` | `PAPER` = simulate only, never call any Robinhood order-placement tool (equity or option). `LIVE` = real orders allowed (not yet enabled). |
| Starting paper account size | $10,000 | User-specified 2026-08-10 (was a $25,000 placeholder before; changed with zero trades/P&L to date, so no historical inconsistency). Tracked in `positions.md`. |
| Max position size (equity / ETF) | **15% of current NAV** per position | Raised from 3% on 2026-08-10 — see "Why 15%" below. Does **not** apply to options, which are capped separately and much lower. |
| Max daily loss (halt trigger) | -2% of NAV in a single day | If hit, no further entries for the rest of that trading day; existing stops still apply. Now actually reachable: six positions stopping out together costs ~2.25%, so this fires before a full book wipeout rather than sitting unreachable. |
| Stop-loss floor | Every open equity position must have a stop no wider than 3% from entry | `strategy.md` may use a tighter ATR-derived stop; never wider than this. |
| Max new trades per run | 2 | Across equity and options combined. |
| Max open positions | 6 | Across equity and options combined. |
| Universe | The Tier 0 screener — `run_scan`, scan_id `de1b1994-b5db-472a-9b79-c052f1215193` | There is **no hardcoded watchlist**; the saved scan is the universe (~96 matches at last check). Its own filters already enforce price > $5 and market cap > $2B. No crypto. |
| Symbol exclusions | None yet | Add tickers here to hard-block them. |
| Risk per trade (target) | **0.4% of NAV** with-trend, **0.2%** counter-trend | The number ATR-based sizing aims at. Chosen so it and the position cap are mutually reachable — see below. |
| Regime filter (classifies, not gates) | SPY above its 200-day SMA → bullish setups are **with-trend**, bearish are **counter-trend**. SPY below → reversed. | Both templates are tested on every candidate every run; the regime decides which passes face the strict counter-trend gates below. Existing positions are reviewed/exited regardless. |
| **Max counter-trend positions** | **1** open at a time (of the 6-position book) | A counter-trend bet is a concentrated position against the market's primary drift. Cross-sectional momentum research supports long/short *in aggregate across hundreds of names* — not one leveraged bet in a six-slot book. |
| Counter-trend ADX floor | **ADX(14) > 30** (vs 25 baseline) | The counter-direction trend must be unusually well established, not marginal. |
| Counter-trend relative-strength bar | ≤ **0.25** of 52-week range (bearish) / ≥ **0.75** (bullish) | Stricter than the 0.4 / 0.6 used for with-trend setups. |
| Counter-trend data requirement | **No degraded inputs — `news_sentiment` must not be `UNAVAILABLE`** | If the Alpha Vantage quota is exhausted, no counter-trend trade is taken that run. A bet against the primary trend, made blind to news, is the trade most likely to sit on the wrong side of a catalyst. |
| Counter-trend position size | **0.2% of NAV risk** (vs 0.4% with-trend) | Half size. Same 15% position cap applies. |
| Index sleeve direction | **Always regime-aligned; never counter-trend** | 2x index ETFs follow the regime only — long-side above SMA(200), inverse-side below. Betting the index against its own primary trend is categorically worse than identifying one broken company inside a healthy market. |
| **Short stock / margin** | **Both forbidden, absolutely** | Never short an equity outright and never borrow on margin, in any mode, for any reason. Short selling carries unbounded loss, borrow cost, and recall risk; margin adds forced liquidation. A system reviewing positions a few times a day cannot manage either. Bearish exposure is taken **only** by *buying* a 2x inverse index ETF (SDS/QID) with cash, where max loss is the amount paid. |

### Why 15% / 0.4% — and why the old 3% / 1% was broken

Found 2026-08-10 by working a real trade (BAC) end-to-end. The previous
pairing of a 3% position cap with a 1%-of-NAV risk target was
**arithmetically impossible to satisfy simultaneously**:

- Stops run ~2.6% of price (ATR-derived, capped by the stop floor).
- To risk 1% of NAV with a 2.6% stop, the position must be ~39% of NAV.
- The cap was 3% — about **13× more binding** than the risk math.

Consequences: ATR-based sizing never determined anything (the cap always
won, making the whole risk-sizing apparatus dead code), each position
carried ~0.065% of NAV in risk instead of 1%, and six positions all
stopping out at once cost ~0.4% — meaning the −2% daily halt guarded a
scenario the sizing rules made **unreachable**.

The current numbers are chosen to be mutually consistent:

| | Value | Check |
|---|---|---|
| Max position | 15% of NAV | 6 positions → ~90% invested |
| Risk per trade | ~0.4% of NAV | 15% position × 2.6% stop ≈ 0.39% ✓ |
| Six simultaneous stop-outs | ~2.25% of NAV | Trips the −2% halt, as intended ✓ |

Worked example (BAC at $63.155, stop $1.629): 15% of a $10,000 NAV is
$1,500 → 23 shares → risk 23 × $1.629 = **$37.47 = 0.375% of NAV**. The
ATR math and the cap now land in the same place instead of fighting.

**If NAV changes materially, re-check this table.** These three numbers
are a set; changing one alone re-breaks the coherence.

## Options gates

| Parameter | Value | Notes |
|---|---|---|
| Allowed structures | Long calls, long puts, vertical debit spreads **only** | Naked/short calls, naked/short puts, uncovered spreads, calendars, and diagonals are **not permitted** under any circumstance — undefined/large risk, out of scope for this system. Calls in LONG mode, puts in SHORT mode. |
| Effective premium cap at current NAV | **$300** per position ($10,000 × 3%) | Consequence worth knowing: a near-the-money contract on a $200+ underlying often exceeds this (a verified JPM ~45 DTE ATM call was $970). Debit spreads and lower-priced underlyings are the workable expressions. Never raise the cap to make a trade fit. This is also why the index/single-stock ETF sleeves carry most of the leveraged exposure at this account size — they can be sized in any dollar amount, options cannot. |
| Max premium at risk per position | **3% of current NAV** — deliberately *not* raised to 15% | For a long option or debit spread the premium paid **is** the entire max loss, unlike an equity position where the stop bounds the loss well inside the position size. A 15% options position would be a 15% max loss; a 15% equity position with a 2.6% stop risks ~0.4%. They are not comparable and must not share a cap. |
| Minimum days to expiration at entry | 30 days | Target ~45 DTE for a swing hold of up to 20 trading days. Never buy less time than the position might need. |
| Target delta range | 0.40 - 0.60 | Near-the-money to slightly OTM. |
| Minimum open interest | 500 contracts | Liquidity floor — illiquid options are how you get a terrible fill or can't exit. |
| Max bid-ask spread | 10% of the option's mid price | Second liquidity check. |

## Leverage gates — margin disabled, leveraged ETFs only

| Parameter | Value | Notes |
|---|---|---|
| **Margin / borrowing** | **Disabled (1.0x)** | Per user decision 2026-08-10: no margin, no borrowing, ever. Every position is bought with cash. Removes margin calls, forced liquidation, and margin interest from the system entirely. |
| Leveraged exposure method | **Buying 2x ETFs with cash only** | LONG: SSO (S&P) / QLD (Nasdaq). SHORT: SDS (S&P) / QID (Nasdaq). All verified liquid 2026-08-10. Max loss is the amount paid — these cannot go below zero or generate a liability. |
| Max open index-sleeve positions | **1** | Never hold a long and an inverse ETF at once — they are direct opposites and would cancel while paying two expense ratios. |
| Leveraged ETF exposure accounting | **Counts 2x toward gross exposure** | A 3%-of-NAV position in a 2x ETF is ~6% of effective market exposure. Count it that way, not at face value. |
| Max portfolio gross exposure | **1.3x of NAV** | Computed with the 2x multiplier applied to any leveraged ETF position. With a 90% base book (6 × 15%), this leaves room for roughly two to three leveraged positions before the cap binds — enough for the sleeve to be usable without letting the whole book run levered. |
| Index leveraged ETF time-stop | **10 trading days** (vs 20 for ordinary stock) | These target 2x the *daily* return and reset daily, so multi-day holds are path-dependent and decay in chop — a flat round-trip in the index still loses money. Short leash by design. See `strategy.md`. |
| **Single-stock** leveraged ETF time-stop | **7 trading days** | Decay scales with σ², and single stocks are far more volatile than indices — roughly 36%/yr drag on a TSLA-like name, ~70%/yr on a COIN-like one, versus ~2.6%/yr on SPY. |
| Single-stock leveraged ETF volatility gate | Underlying **ATR(14) ≤ 4% of price** | Hard gate. Above it, decay is too steep for the holding period — trade the plain stock instead (LONG mode) or take no trade (SHORT mode). Observed 2026-08-10: CONL (2x COIN) fell **92%** from its 52-week high while COIN itself did nothing remotely similar. |
| Leveraged ETF liquidity floor | 30-day average volume ≥ 1M shares | Hard gate. Bear-side ETFs are frequently far thinner than their bull counterparts — METD (744K), GGLS (368K), CONI (144K) and TSLS (409K) all **fail** this. If a name's bear ETF is below the floor, there is no bearish expression for that name; take no trade rather than reaching for an illiquid one. |
| **Verify leverage per instrument** | Read the fund's own description before sizing | Bear ETFs do **not** reliably mirror their bull counterpart's leverage — AAPD, MSFD, AMZD and TSLS are **−1x**, while NVD and TSLQ are **−2x**. Never infer leverage from the ticker or from the bull side. |
| **−2x inverse decay penalty** | Treat −2x single-stock funds as the highest-decay instrument permitted | Daily-reset drag is `(L²−L)/2 × σ²`, so **−2x carries 3× the drag of +2x** — identical to a +3x fund. Observed: CONI −2x COIN fell $141.65 → $52.64; TSLQ −2x TSLA fell $51.45 → $25.17. |

## Execution gates

| Parameter | Value | Notes |
|---|---|---|
| Order type | **Marketable limit only — never market orders** | Buys limit at the ask, sells limit at the bid. A market order surrenders price control and can fill far from the last print in a fast or thin market. |
| Simulated fill price | Buys at `ask_price`, sells at `bid_price` (options: `high_fill_rate_buy/sell_price`) | Never the midpoint or `last_trade_price` — both flatter paper P&L versus a real fill and hide the spread cost of every round trip. |
| Max bid-ask spread at entry | **0.5% of mid** (equity/ETF) | Wider than this on a name that passed the liquidity screen usually signals a halt, a news event, or stale data. Skip and log rather than crossing it. |
| Trading window | All runs inside regular hours: **9:30am, 2:30pm, 3:30pm ET** | Never pre-open (stale/thin quotes) and never post-close (a decision that cannot be filled until the next session at an unknown price). The morning run starts at the open and executes when its analysis completes (~15 min); delaying past the open costs more in missed breakouts than it saves in spread. Wide spreads are handled by the 0.5% check at fill time, not by avoiding a time window. |

## Operational notes (not risk limits, but load-bearing for the routine)

- `TIME_SERIES_DAILY_ADJUSTED` and `MACD` are **premium-gated** on the
  current Alpha Vantage plan and return an error, not data — confirmed
  live via a real run on 2026-08-10, not a transient rate-limit. Do not
  call either.
- **Alpha Vantage's daily quota (25 requests/day, shared across every
  session using this key — manual test runs and scheduled runs draw from
  the same pool)** caused a full data blackout on 2026-08-10: a scheduled
  run got zero usable Alpha Vantage calls because manual test runs earlier
  the same day had exhausted it. Fix applied the same day: Robinhood's own
  data tools (`get_equity_quotes`, `get_equity_fundamentals`,
  `get_equity_technical_indicators`, `get_earnings_results`) now cover
  almost the entire funnel with no observed daily cap — see `strategy.md`'s
  "Data sources" table. Alpha Vantage is reserved for `NEWS_SENTIMENT`
  only (plus occasional deliberate deep-dives, never a full-universe
  pull), which should keep normal runs to ~3 Alpha Vantage calls instead
  of 50-75.
- Robinhood's `get_equity_technical_indicators` supports `output:
  "latest"`/`"last:N"` — use it. Alpha Vantage's indicator endpoints
  (`RSI`, `EMA`, `MACD`, `ATR`, weekly/monthly time series) have no such
  option and dump full multi-year histories (60-100k+ characters) to a
  file; if one is ever called anyway, extract only the last few values via
  `jq`/`tail` rather than reading the whole file.
- If a call hits a rate-limit or premium-only error mid-run, do not treat
  it as a crash: log the truncation, prioritize reviewing existing
  positions over scouting new ones, and finish the run.

## Changing MODE to LIVE

Do not do this until the paper-trading track record in `trade_journal.md`
has been reviewed by the human, and ideally not until the framework has
been backtested (see `framework.md`, "What's deliberately not here yet").
This should be a deliberate manual edit to this file, never something the
agent infers or proposes on its own.
