# Trade Journal

Append-only log. Never delete or rewrite past entries — if a past decision
turned out wrong, say so in a new entry, don't erase the record.

Each run should add one entry using the template below.

---

## Template

### YYYY-MM-DD HH:MM UTC — [morning|midday|pre-close] run

**Market status:** (open/closed, per `MARKET_STATUS`)
**Account:** NAV $X, cash $X, halted: yes/no

**Scouted:** (Tier 0 scan total match count, and the top-15 taken into
Tier 1 with their ADX / relative options volume)
**Direction mode:** LONG or SHORT (SPY vs its SMA(200), with values)

**Decisions:**

For every position **opened**, record all of the following — this is
mandatory, not optional, and the same values must also go into
`trade_log.csv` (see `DATA_SCHEMA.md`):

- SYMBOL — opened — instrument (equity/option) — qty — fill price
  - **Trigger:** breakout or pullback (which `strategy.md` rule fired)
  - **Trend:** price $X vs SMA50 $Y vs SMA200 $Z (the stack that
    qualified it)
  - **Momentum:** RSI(14) = X, EMA8 = Y, EMA21 = Z, spread widening?
  - **Relative strength:** pct_52w_range = X (and composite rank)
  - **Valuation:** P/E = X (rank within this run's candidate set)
  - **Catalyst:** news sentiment = X (or `UNAVAILABLE` — never guessed),
    earnings in X days, recent EPS surprise history
  - **Risk:** ATR14 = X → stop $Y (Z% away), R-target $W, position sized
    to risk $V (0.4% of NAV with-trend, 0.2% counter-trend), position
    $P (cap: 15% of NAV)
  - **Direction:** bullish/bearish, and with-trend or counter-trend
  - **If options:** strike, expiration, delta, IV, DTE, and why an option
    beat plain equity for this specific signal
  - **One-sentence thesis:** why this trade, in plain language

For every position **closed**: which exit rule fired, exit price,
realized P&L, and R-multiple.

For every candidate **rejected**: which specific check failed and the
value that failed it.

**Rejected by gates:** (any candidate trade that failed a `gates.md` check,
and which check)

**Position review:** (any open positions checked against stop/target/time-stop)

**Strategy adaptation this run:** (none, or a link to the `strategy.md`
changelog entry made this run)

---

### 2026-08-10 (manual test run, ~03:20 UTC) — premarket run

**Market status:** Closed (Monday premarket, not a holiday)
**Account:** NAV $25,000, cash $25,000, halted: no (first-ever run, no
history)

**Regime filter:** SPY $773.26 > 200DMA $700.13 → PASS (risk-on). New
entries permitted.

**Budget constraint hit:** Alpha Vantage free-tier daily cap (25
requests/day) meant the full ~19-symbol watchlist could not be screened.
Narrowed Tier 1 to 5 diversified candidates: AAPL, JPM, UNH, XOM (one per
major sector from the seed watchlist) + PLTR (opportunistic, from
most-actively-traded — top gainers/losers were dominated by sub-$5
penny stocks/warrants, excluded by `gates.md`).

**Scouted:** AAPL, JPM, UNH, XOM, PLTR

**Tier 1 results:**
- AAPL — PASS (price $313.33 > 50DMA $309.79 > 200DMA $279.41, within 9%
  of 52wk high, RS proxy 0.74)
- JPM — PASS (price $357.52 > 50DMA $333.00 > 200DMA $313.24, 1.5% below
  52wk high, RS proxy 0.94 — highest-ranked)
- UNH — FAIL (price $407.08 below 50DMA $412.80, trend stack broken)
- XOM — PASS (price $153.04 > 50DMA $146.48 > 200DMA $140.20, RS proxy 0.70)
- PLTR — FAIL (50DMA $132.60 below 200DMA $152.28, trend stack broken;
  GLOBAL_QUOTE call skipped to conserve budget once this was clear)

**Tier 1 survivors, ranked by value/momentum tiebreaker:** JPM (0.94),
AAPL (0.74), XOM (0.70) — all 3 well under the top-8 cap, so all could
have advanced; JPM vetted first given remaining budget (~11 calls left)
and highest rank.

**Tier 2 (JPM only, budget-limited):** EMA8 $355.01 > EMA21 $348.65, price
within 1.5% of 52wk high → breakout trigger fires. PASS.

**Tier 3 (JPM):** Blocked structurally — `MACD` returned a premium-only
endpoint error (confirmed, not a transient rate-limit). At the time of
this run, Tier 3 still required MACD confirmation, so JPM could not be
confirmed and no trade was opened. **This has since been fixed**: Tier 3
now uses the EMA8/EMA21 spread (already fetched in Tier 2) instead of
MACD, so this specific blocker should not recur.

**Decisions:** No new positions opened. Correct outcome under the rules
in force at the time — a candidate without complete Tier 3 confirmation
is dropped, never force-traded.

**Rejected by gates:** None reached the gating step (JPM was dropped at
Tier 3 confirmation, before gates.md checks apply).

**Position review:** None — first-ever run, no open positions.

**Infrastructure note (not a trading decision):** This run's results
could not be committed/pushed — the cloud session's GitHub access
returned "GitHub access is not enabled for this session. An org admin
must connect the Claude GitHub App for this organization." This journal
entry was manually backfilled from the run's output after the fact, since
the run's own local commit was lost with the ephemeral session. Until
this is fixed, no scheduled run can persist state on its own.

**Strategy adaptation this run:** None from the routine itself (blocked
before that step). Two adaptations made manually afterward, in response
to this run's findings: (1) removed the `MACD` dependency from Tier 3
(structurally premium-gated, not fixable by retrying), (2) this journal
entry backfilled to preserve the analysis.

---

_(Further runs will be appended below by the scheduled routine, once
GitHub push access is fixed.)_

---

### 2026-08-10 05:15 UTC — premarket run

**Market status:** Unknown — `MARKET_STATUS` call itself failed (see below).
Could not confirm via `MARKET_STATUS`.

**Account:** NAV $25,000, cash $25,000, halted: no (no open positions, no
prior trades)

**gates.md MODE check:** `PAPER` — confirmed, run proceeds under normal
paper-trading rules.

**Position review:** No open equity or options positions in `positions.md`
— nothing to review this run regardless of data availability.

**Data availability:** Total Alpha Vantage blackout. `MARKET_STATUS`,
`GLOBAL_QUOTE` (SPY), and `COMPANY_OVERVIEW` (SPY) all returned the same
error: `"our standard API rate limit is 25 requests per day"`. Retried
`MARKET_STATUS` once after a 5-second pause with an identical result,
confirming this is the persistent daily quota (already exhausted,
likely by the earlier 2026-08-10 ~03:20 UTC manual test run's ~17+ calls
against the same key), not a transient per-second burst limit. No Alpha
Vantage call of any kind is possible for the remainder of today's quota
window.

**Regime filter:** Could not be evaluated — SPY quote/200DMA unavailable.

**Scouted:** None. With the regime filter itself unconfirmable and zero
budget for Tier 1 `COMPANY_OVERVIEW`/`GLOBAL_QUOTE` calls, no candidate
screening was possible. Per `run_instructions.md`'s error-handling section
("never fall back to guessing a value — skip the candidate instead") and
`gates.md`'s operational notes, no scouting was attempted rather than
proceeding on stale or fabricated data.

**Decisions:** No new positions opened. No positions closed (none open).

**Rejected by gates:** N/A — no candidates reached the gating step; the
blocker was data availability, not a gate.

**Strategy adaptation this run:** None. Adaptation policy requires
reviewing the last 10 closed trades in `trade_journal.md`; there are zero
closed trades on record to date, so there is nothing to evaluate.

**Infrastructure note:** This is a recurring risk for this system on the
free Alpha Vantage tier — a single manual/test run earlier in the same
UTC day can exhaust the shared 25-request daily budget before the
scheduled run executes, leaving the scheduled run with zero usable
budget. Worth flagging to the human: either space out manual runs from
scheduled ones, or upgrade the Alpha Vantage plan if same-day double
runs become routine. Not something this agent can fix by editing
`gates.md` or `strategy.md`.

---

### 2026-08-10 05:54 UTC — off-schedule run (market closed)

**gates.md MODE check:** `PAPER` — confirmed, run proceeds under normal
paper-trading rules.

**Market status:** Closed. `MARKET_STATUS` (Alpha Vantage) failed —
`"our standard API rate limit is 25 requests per day"`, same shared-quota
exhaustion flagged in the prior entry and in `gates.md`'s operational
notes. Fell back to the time-based check per `run_instructions.md` step
1: current time 05:54 UTC = 01:54 ET Monday, well outside the 9:30am-4pm
ET regular session, so market is closed. Corroborated independently via
Robinhood `get_equity_quotes` (SPY): `venue_last_trade_time` (regular
session) is `2026-08-07T19:59:59Z` — Friday's close, three days stale —
while only `last_non_reg_trade_price`/extended-hours activity is recent.
No live regular-session price exists right now.

**Scheduling note:** This run fired at 01:54 ET, which does not match any
of the three documented run times in `run_instructions.md`
(9:30am/2:30pm/4:30pm ET). Flagging for the human to check the schedule
config — this looks like an off-schedule or misconfigured trigger, not
one of the intended three daily runs. No action taken by this agent
beyond logging it, since `run_instructions.md`/`gates.md` don't authorize
editing scheduling infrastructure.

**Regime filter:** Not evaluated — no new entries are being considered
this run regardless (market closed, no live session price), so this was
skipped rather than spending an unusable data call.

**Position review:** `positions.md` shows zero open equity or options
positions — nothing to review.

**Scouted:** None. Consistent with `run_instructions.md`'s design
rationale for the 9:30/2:30/4:30 ET schedule ("every run works with a
real, live regular-session price rather than a stale prior-day close or a
thin premarket quote") — scouting against Friday's stale close or a thin
overnight quote would violate that rationale, so no candidates were
pulled.

**Decisions:** No new positions opened. No positions closed (none open).

**Rejected by gates:** N/A — no candidates reached the gating step; the
blocker was market/session availability, not a gate.

**Strategy adaptation this run:** None. Adaptation policy requires
reviewing the last 10 closed trades; there are still zero closed trades
on record.

---

### 2026-08-10 09:18 UTC — off-schedule run (premarket)

**gates.md MODE check:** `PAPER` — confirmed, run proceeds under normal
paper-trading rules (though no trading action is taken this run — see
below).

**Market status:** `MARKET_STATUS` (Alpha Vantage) failed again —
`"our standard API rate limit is 25 requests per day"`, same shared-quota
exhaustion pattern noted in every prior entry. Fell back to the
time-based check per `run_instructions.md` step 1: system clock read
2026-08-10T09:18 UTC, confirmed via `date -u`. UTC-4 (EDT) makes that
**05:18 ET**, more than four hours before the 9:30am ET open. Corroborated
independently via Robinhood `get_equity_quotes` (SPY): the regular-session
`venue_last_trade_time` is still `2026-08-07T19:59:59Z` (last Friday's
close); only `last_non_reg_trade_price` (premarket) is recent. No live
regular-session price exists right now.

**Scheduling note (now the third occurrence):** This is the third
consecutive scheduled/off-schedule run to fire outside the three
documented windows (9:30am/2:30pm/3:30pm ET) — after the 05:15 UTC and
05:54 UTC runs earlier today, all premarket. Across all runs logged in
this journal to date, **zero have fired during an actual regular-session
window**, meaning this system has never yet had the opportunity to
evaluate or open a live trade. This is a scheduling/trigger configuration
issue outside what `run_instructions.md`/`gates.md` authorize this agent
to fix (no permission to edit scheduling infrastructure) — flagging for
the human explicitly, including via a direct notification alongside this
run, since three-for-three premarket misfires is a pattern, not a fluke.

**Regime filter (informational only, not acted on):** Before recognizing
the time-window issue, `run_scan` (Tier 0, scan_id
`de1b1994-b5db-472a-9b79-c052f1215193`) and a SPY quote/200-SMA check were
already in flight and are logged here for continuity, but **no candidate
funnel or trade was run against them** — a premarket price is not a valid
basis for a simulated fill per `strategy.md`'s order-entry rules. SPY
last regular-session close $773.26 (2026-08-07), 200-day SMA $702.97 →
SPY comfortably above SMA(200), consistent with the risk-on regime noted
in prior entries. Tier 0 scan returned 92 total matches (`total_items`),
similar order of magnitude to the 96 seen previously. None of this was
used to screen, rank, or open anything.

**Position review:** `positions.md` shows zero open equity or options
positions — nothing to review regardless of session status.

**Scouted:** None — funnel intentionally not run; see above.

**Decisions:** No new positions opened. No positions closed (none open).

**Rejected by gates:** N/A — no candidates reached the gating step; the
blocker was market/session availability, not a gate.

**Strategy adaptation this run:** None. Adaptation policy requires
reviewing the last 10 closed trades; there are still zero closed trades
on record.

---

### 2026-08-10 13:57 UTC — morning run

**gates.md MODE check:** `PAPER` — confirmed, run proceeds under normal paper-trading rules.

**Market status:** `MARKET_STATUS` (Alpha Vantage) failed again — shared 25/day quota exhausted (same recurring pattern noted in every prior entry). Time-based fallback: 09:54 ET Monday, a regular trading weekday, inside 9:30am-4pm ET → market open. Corroborated independently via Robinhood `get_equity_quotes` (SPY): `venue_last_trade_time` 2026-08-10T13:55 UTC, live regular-session trades today. This is the first run of the day to actually land inside a regular-session window (the prior three runs today were all premarket misfires, per earlier entries).

**Account:** NAV $10,000, cash $10,000, halted: no (no open positions).

**Position review:** `positions.md` shows zero open equity/options positions — nothing to review.

**Regime filter:** SPY $773.20 > 200DMA $702.97 → **LONG mode / risk-on**. New with-trend entries are bullish stock/ETF or bull index ETF; bearish stock passes count as counter-trend.

**Scouted (Tier 0):** `run_scan` (scan_id de1b1994...) returned only **7 total matches** today (well below the ~92-96 typically seen — a genuinely thin trending universe today, not a call failure). All 7 taken into Tier 1 (below the top-15 cap), ranked by ADX(14) desc / relative options volume tiebreak:

| Rank | Symbol | ADX(14) | Rel. options vol |
|---|---|---|---|
| 1 | ANF | 30.66 | 0.81 |
| 2 | TSLA | 30.56 | 0.64 |
| 3 | MP | 29.12 | 0.55 |
| 4 | RKLB | 28.02 | 0.53 |
| 5 | VIPS | 27.45 | 0.67 |
| 6 | BRK.B | 27.18 | 0.52 |
| 7 | COO | 25.05 | 4.70 |

**Earnings blackout (Tier 1, one `get_earnings_calendar` days=7 call):** RKLB reports today (2026-08-10, PM) — dropped, within the 3-day blackout window. 6 candidates proceed: ANF, TSLA, MP, VIPS, BRK.B, COO.

**Tier 1 — trend template, both directions, all 6:**
- **ANF** ($114.70, SMA50 $91.45, SMA200 $91.80): SMA50 < SMA200 → fails LONG stack (price>SMA50>SMA200 requires SMA50>SMA200). price ($114.70) not < SMA50 → fails SHORT. **FAIL both.**
- **TSLA** ($327.90, SMA50 $380.44, SMA200 $408.80, 52wk $297.38-$498.83): fails LONG (price<SMA50). SHORT: price<SMA50<SMA200 ✓ (327.90<380.44<408.80); within 25% of low52 ✓ (327.90 ≤ 371.73); ≥25% below high52 ✓ (327.90 ≤ 374.12); relative-weakness proxy = (327.90-297.38)/(498.83-297.38) = **0.152** ≤0.4 ✓. **PASS SHORT (bearish).**
- **MP** ($54.62, SMA50 $53.10, SMA200 $58.22): SMA50<SMA200 → fails LONG stack. price ($54.62) not < SMA50 ($53.10) → fails SHORT. **FAIL both.**
- **VIPS** ($15.49, SMA50 $14.13, SMA200 $16.36): SMA50<SMA200 → fails LONG stack. price not < SMA50 → fails SHORT. **FAIL both.**
- **BRK.B** ($536.75, SMA50 $495.07, SMA200 $491.03, 52wk $462.55-$537.49): LONG stack holds (536.75>495.07>491.03) and within 25% of high, but **fails rule 3** — "at least 25% above low52" requires price ≥ $578.19 (462.55×1.25); price is $536.75. BRK.B's 52-week range is only ~16% top-to-bottom, so this rule is structurally unsatisfiable for it right now regardless of how close to the high it trades. SHORT: price not < SMA50 → fails. **FAIL both.**
- **COO** ($75.44, SMA50 $69.28, SMA200 $73.15): SMA50<SMA200 → fails LONG stack. price not < SMA50 → fails SHORT. **FAIL both.**

**Tier 1 survivor: TSLA only**, bearish/SHORT-mode pass. Since SPY regime is LONG (risk-on), this is **counter-trend**.

**Counter-trend gate check (TSLA, all 5 required):**
1. Max 1 counter-trend position open: 0 currently open → OK.
2. ADX(14) > 30: TSLA ADX = 30.56 → **PASS** (barely).
3. Extreme 52wk-range reading, bearish needs ≤0.25: computed 0.152 → **PASS**.
4. No degraded data (news_sentiment must not be UNAVAILABLE) — not reached; gated at Tier 3.
5. Half size: 0.2% NAV risk (gates.md is authoritative over strategy.md's stale "0.5%" reference in its counter-trend section) — to be applied only if TSLA reaches sizing.

**Tier 2 — entry trigger (TSLA, SHORT mode):**
- RSI(14) last 5 sessions: 36.67, 39.05, 37.39, 36.80, **41.26** (most recent).
- EMA8 last 5: 321.64, 322.91, 322.61, 321.92, **323.40**.
- EMA21 last 5: 349.34, 347.34, 344.99, 342.68, **341.40**.
- **Breakdown trigger**: price within 2% of low52 ($297.38 × 1.02 = $303.33) — price $327.90 is **not** within 2% (it's ~10.3% above the low). **FAIL.**
- **Rally-to-resistance trigger**: RSI(14) between 55-65 — most recent RSI is 41.26, well outside the band. **FAIL.**

**TSLA dropped at Tier 2 — no entry trigger fired.** The trend/counter-trend gates were satisfied, but neither SHORT-mode timing trigger did; per `run_instructions.md` this is dropped, not force-fit, and never downgraded. (Not logged to `trade_log.csv` — DATA_SCHEMA.md scopes required reject rows to Tier 3/gate-level rejections, and TSLA didn't reach Tier 3.)

**Index sleeve (SSO, since SPY is in LONG mode):** SPY trend template: price $773.20 > SMA50 $747.19 > SMA200 $702.97 ✓; within 25% of high52 ($776.85) ✓. **Fails rule 3** — at least 25% above low52 ($629.28 × 1.25 = $786.60) — price $773.20 doesn't clear it. Same structural cause as BRK.B: SPY's current 52-week range is only ~23.4% top-to-bottom (low $629.28 on 2026-03-30, high $776.85 on 2026-08-05, just 3 sessions old), so this specific rule cannot be satisfied by any price inside that range. **No SSO trade.** Flagging as an observation, not a strategy change — the adaptation policy requires a pattern across ≥5 of the last 10 *closed* trades before touching a named parameter, and there are zero closed trades to date. Worth re-checking if this rule keeps blocking otherwise-strong setups once there's a real track record.

**Decisions:** No new positions opened (equity, options, or index sleeve). No positions closed (none open).

**Rejected by gates:** None reached gates.md-level sizing/execution checks — TSLA was dropped at Tier 2 (entry trigger), everything else at Tier 1 (trend template) or the Tier 1 earnings blackout (RKLB).

**Strategy adaptation this run:** None. Zero closed trades on record; adaptation policy requires reviewing closed-trade history.
