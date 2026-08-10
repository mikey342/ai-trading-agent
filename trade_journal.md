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

---

### 2026-08-10 18:42 UTC — midday run

**gates.md MODE check:** `PAPER` — confirmed, run proceeds under normal paper-trading rules.

**Market status:** `MARKET_STATUS` (Alpha Vantage) failed again — shared 25/day quota exhausted (same recurring pattern as every prior entry today). Time-based fallback: 14:38 ET Monday, inside 9:30am-4pm ET → market open. Corroborated via Robinhood `get_equity_quotes` (SPY): live regular-session trade at 2026-08-10T18:38 UTC. This run fired close to the 2:30pm ET slot (a few minutes late) — the midday run.

**Account:** NAV $10,000, cash $10,000, halted: no (no open positions).

**Position review:** `positions.md` shows zero open equity/options positions — nothing to review.

**Regime filter:** SPY $772.39 > 200DMA $702.97 → **LONG mode / risk-on** (unchanged from this morning).

**Scouted (Tier 0):** `run_scan` (scan_id de1b1994...) returned **79 total matches** this run (much broader than this morning's thin 7-name universe — market conditions shifted intraday). Top 15 by ADX(14) desc / relative options volume tiebreak:

| Rank | Symbol | ADX(14) | Rel. options vol |
|---|---|---|---|
| 1 | TECH | 59.71 | 0.61 |
| 2 | EAT | 45.42 | 0.59 |
| 3 | PARR | 41.62 | 0.53 |
| 4 | BFLY | 39.15 | 2.39 |
| 5 | TXG | 39.15 | 3.87 |
| 6 | GFS | 37.52 | 0.93 |
| 7 | DFTX | 37.23 | 1.17 |
| 8 | JD | 36.91 | 1.12 |
| 9 | NTNX | 35.64 | 1.27 |
| 10 | PRU | 35.41 | 0.53 |
| 11 | AAON | 34.19 | 2.67 |
| 12 | CXW | 33.83 | 3.65 |
| 13 | GGG | 33.60 | 13.91 |
| 14 | DINO | 33.60 | 0.88 |
| 15 | FSLR | 32.96 | 1.30 |

**Earnings blackout (Tier 1, one `get_earnings_calendar` days=7 call):** AAON reports today (2026-08-10, am) — dropped. EAT reports 2026-08-12 (within 3 days) — dropped. JD reports 2026-08-13 (within 3 days) — dropped. 12 candidates proceed: TECH, PARR, BFLY, TXG, GFS, DFTX, NTNX, PRU, CXW, GGG, DINO, FSLR.

**Tier 1 — trend template, both directions, all 12:**
- **TECH** ($72.115, SMA50 $64.7845, SMA200 $59.9093, 52wk $43.195–$72.39 [new high made today]): price>SMA50>SMA200 ✓; within 25% of high ✓; ≥25% above low ✓; pct_52w_range = 0.9906 ≥0.6 ✓. **PASS LONG (bullish).**
- **PARR** ($72.28, SMA50 $64.8326, SMA200 $52.2711, 52wk $26.92–$87.03): stack ✓; pct_52w_range=0.7547 ✓. **PASS LONG.**
- **BFLY** ($9.785, SMA50 $6.918, SMA200 $4.5113, 52wk $1.32–$10.05 [new high today]): stack ✓; pct_52w_range=0.9696 ✓. **PASS LONG.**
- **TXG** ($58.61, SMA50 $38.9354, SMA200 $24.2510, 52wk $11.16–$59.00 [new high today]): stack ✓; pct_52w_range=0.9918 ✓. **PASS LONG.**
- **GFS** ($51.7276, SMA50 $69.2058, SMA200 $51.9567): fails LONG (price<SMA50). Fails SHORT too (SMA50>SMA200, descending stack broken). **FAIL both.**
- **DFTX** ($43.66, SMA50 $37.4734, SMA200 $21.8935, 52wk $8.70–$49.70): stack ✓; pct_52w_range=0.8527 ✓. **PASS LONG.**
- **NTNX** ($64.62, SMA50 $53.5186, SMA200 $49.0692, 52wk $34.01–$82.42): stack ✓; pct_52w_range=0.6323 ✓ (barely). **PASS LONG.**
- **PRU** ($121.46, SMA50 $112.4738, SMA200 $106.2511, 52wk $91.89–$127.72): stack ✓; pct_52w_range=0.8253 ✓. **PASS LONG.**
- **CXW** ($33.41, SMA50 $28.8746, SMA200 $21.5236, 52wk $15.735–$34.86 [new high today]): stack ✓; pct_52w_range=0.9242 ✓. **PASS LONG.**
- **GGG** ($83.225, SMA50 $76.2254, SMA200 $82.4367): price>SMA50 but SMA50<SMA200 → fails LONG stack. Fails SHORT (price not < SMA50). **FAIL both.**
- **DINO** ($85.13, SMA50 $77.4414, SMA200 $61.4016, 52wk $42.82–$94.22): stack ✓; pct_52w_range=0.8231 ✓. **PASS LONG.**
- **FSLR** ($238.315, SMA50 $243.5538, SMA200 $234.43): price<SMA50 → fails LONG. SMA50>SMA200 → fails SHORT stack. **FAIL both.**

**Tier 1 survivors (9, all bullish/with-trend since SPY regime is LONG):** TECH, PARR, BFLY, TXG, DFTX, NTNX, PRU, CXW, DINO.

**Value/momentum composite ranking (0.6×rel-strength + 0.4×value-rank, top 8 advance):** DINO (0.894), CXW (0.805), PARR (0.803), PRU (0.795), TECH (0.744), TXG (0.695), BFLY (0.632), NTNX (0.579), DFTX (0.512) — **DFTX cut** (negative P/E ranked worst on the value leg). Top 8 to Tier 2: DINO, CXW, PARR, PRU, TECH, TXG, BFLY, NTNX.

**Tier 2 — entry trigger, top 8:**
- **DINO:** price $85.13 vs high52 $94.22 → 9.6% off high, breakout fails. RSI(14)=44.60 (in 35-45 band) but EMA21 last-5 declining (85.62→84.99) → pullback trend-not-flat-to-rising fails. **DROP.**
- **CXW:** price $33.41 vs high52 $34.86 → 4.2% off, breakout fails. RSI=65.28, outside 35-45. **DROP.**
- **PARR:** price $72.28 vs high52 $87.03 → 17.0% off, breakout fails. RSI=39.50 (in band) but EMA21 last-5 declining (75.54→74.24). **DROP.**
- **PRU:** price $121.46 vs high52 $127.72 → 4.9% off, breakout fails. RSI=59.76, outside 35-45. **DROP.**
- **TECH:** price $72.115 within 2% of today's new high $72.39 ✓; EMA8 $72.034 > EMA21 $70.909 ✓. **Breakout trigger fires — PASS.**
- **TXG:** price $58.61 within 2% of today's new high $59.00 ✓; EMA8 $48.003 > EMA21 $45.580 ✓. **Breakout trigger fires — PASS.**
- **BFLY:** price $9.785 vs today's new high $10.05 → 2.6% off, just outside the 2% breakout band. RSI=61.60, outside 35-45. **DROP.**
- **NTNX:** price $64.62 vs high52 $82.42 → 21.6% off, breakout fails. RSI=71.89, outside 35-45. **DROP.**

**Tier 2 survivors: TECH, TXG** (both breakout triggers, both well under the Tier 3 cap of 3).

**Tier 3 — momentum confirmation (|EMA8−EMA21| must be strictly widest at the most recent of 5 sessions):**
- **TECH:** spread Aug3→Aug7: 1.5696, 1.4470, 1.3192, 1.1891, **1.1243** (most recent). Most recent is the *narrowest*, not the widest — decelerating breakout. **FAILS. Dropped.**
- **TXG:** spread Aug3→Aug7: 2.6499, 2.5307, 2.3427, 1.9182, **2.4237** (most recent). Most recent is not greater than all four priors (Aug3's 2.6499 is higher) — choppy, not accelerating. **FAILS. Dropped.**

Both logged to `trade_log.csv` as Tier 3 rejects (trade_id `TECH-20260810-1`, `TXG-20260810-1`) with full indicator snapshots. `news_sentiment` recorded `UNAVAILABLE` for both — Alpha Vantage quota was already confirmed exhausted this run via the `MARKET_STATUS` failure, so the Tier 3 news call was not attempted (would fail identically); this is a non-gating step for a candidate already dropped on momentum, consistent with `strategy.md`'s note that ATR/RSI/EMA are the load-bearing Tier 3 data, not sentiment.

**Index sleeve (SSO, since SPY is in LONG mode):** SPY trend template: $772.39 > SMA50 $747.19 > SMA200 $702.97 ✓; within 25% of high52 ($776.85) ✓. **Fails rule 3** — at least 25% above low52 ($629.28 × 1.25 = $786.60) — price doesn't clear it. Same structural issue as this morning: SPY's 52-week range is currently only ~19-23% top-to-bottom (low set 2026-03-30), so this rule is unsatisfiable at the current range regardless of trend strength. **No SSO trade.** Not a new adaptation trigger — same observation logged this morning, still just one data point.

**Decisions:** No new positions opened (equity, options, or index sleeve). No positions closed (none open).

**Rejected by gates:** None reached gates.md-level sizing/execution checks. TECH and TXG were dropped at Tier 3 (momentum confirmation) — see `trade_log.csv`. DINO, CXW, PARR, PRU, BFLY, NTNX were dropped at Tier 2 (no entry trigger fired); GFS, GGG, FSLR at Tier 1 (trend template); DFTX at the Tier 1→2 value/momentum cut. Per `DATA_SCHEMA.md`, only Tier 3/gate-level rejections are logged to the CSV.

**Strategy adaptation this run:** None. Zero closed trades on record; the adaptation policy requires reviewing closed-trade history, which doesn't exist yet.

---

### 2026-08-10 19:41 UTC — close run (3:30pm ET slot)

**gates.md MODE check:** `PAPER` — confirmed, run proceeds under normal paper-trading rules.

**Market status:** `MARKET_STATUS` (Alpha Vantage) failed again — shared 25/day quota exhausted (same recurring pattern as every prior entry today). Time-based fallback: 15:37 ET Monday, inside 9:30am-4pm ET → market open. Corroborated via Robinhood `get_equity_quotes` (SPY): live regular-session trade at 2026-08-10T19:37:53Z. This run landed close to the 3:30pm ET slot — the close run, third and last of the day.

**Account:** NAV $10,000, cash $10,000, halted: no (no open positions).

**Position review:** `positions.md` shows zero open equity/options positions — nothing to review.

**Regime filter:** SPY $772.97 > 200DMA $702.97 → **LONG mode / risk-on** (unchanged all day).

**Scouted (Tier 0):** `run_scan` (scan_id de1b1994...) returned **96 total matches** this run. Top 15 by ADX(14) desc / relative options volume tiebreak:

| Rank | Symbol | ADX(14) | Rel. options vol |
|---|---|---|---|
| 1 | TECH | 59.71 | 0.61 |
| 2 | EAT | 45.42 | 0.67 |
| 3 | LEGN | 43.17 | 0.52 |
| 4 | BMY | 41.78 | 0.57 |
| 5 | PARR | 41.62 | 0.58 |
| 6 | BFLY | 39.15 | 2.50 |
| 7 | TXG | 39.15 | 4.05 |
| 8 | ADP | 38.58 | 0.51 |
| 9 | PBF | 37.95 | 0.53 |
| 10 | GFS | 37.52 | 1.00 |
| 11 | DFTX | 37.23 | 2.27 |
| 12 | JD | 36.91 | 1.34 |
| 13 | IMAX | 36.59 | 0.76 |
| 14 | NTNX | 35.64 | 1.29 |
| 15 | ICE | 35.64 | 0.62 |

**Earnings blackout (Tier 1, one `get_earnings_calendar` days=7 call):** LEGN reports 2026-08-11 (tomorrow) — dropped. EAT reports 2026-08-12 (2 days) — dropped. JD reports 2026-08-13 (3 days) — dropped. 12 candidates proceed: TECH, BMY, PARR, BFLY, TXG, ADP, PBF, GFS, DFTX, IMAX, NTNX, ICE.

**Tier 1 — trend template, both directions, all 12:**
- **TECH** ($72.165, SMA50 $64.7845, SMA200 $59.9093, 52wk $43.195–$72.39 [new high today]): stack ✓; pct_52w_range=0.9923 ✓. **PASS LONG.**
- **BMY** ($65.075, SMA50 $58.7366, SMA200 $56.0958, 52wk $42.52–$68.10): stack ✓; pct_52w_range=0.8817 ✓. **PASS LONG.**
- **PARR** ($72.02, SMA50 $64.8326, SMA200 $52.2711, 52wk $26.92–$87.03): stack ✓; pct_52w_range=0.7503 ✓. **PASS LONG.**
- **BFLY** ($9.74, SMA50 $6.918, SMA200 $4.5113, 52wk $1.32–$10.05 [new high today]): stack ✓; pct_52w_range=0.9645 ✓. **PASS LONG.**
- **TXG** ($59.555, SMA50 $38.9354, SMA200 $24.2510, 52wk $11.16–$59.775 [new high today]): stack ✓; pct_52w_range=0.9955 ✓. **PASS LONG.**
- **ADP** ($272.01, SMA50 $240.5449, SMA200 $235.5970, 52wk $188.16–$310.08): stack ✓; pct_52w_range=0.6877 ✓. **PASS LONG.**
- **PBF** ($66.13, SMA50 $51.503, SMA200 $40.6261, 52wk $21.46–$74.74): stack ✓; pct_52w_range=0.8384 ✓. **PASS LONG.**
- **GFS** ($51.11, SMA50 $69.2058, SMA200 $51.9567): price<SMA50 → fails LONG stack. SMA50>SMA200 → fails SHORT stack too (not a descending stack). **FAIL both.**
- **DFTX** ($44.16, SMA50 $37.4734, SMA200 $21.8935, 52wk $8.70–$49.70): stack ✓; pct_52w_range=0.8649 ✓. **PASS LONG.**
- **IMAX** ($50.775, SMA50 $42.2032, SMA200 $37.8884, 52wk $24.20–$51.60): stack ✓; pct_52w_range=0.9699 ✓. **PASS LONG.**
- **NTNX** ($64.57, SMA50 $53.5186, SMA200 $49.0692, 52wk $34.01–$82.42): stack ✓; pct_52w_range=0.6313 ✓ (barely). **PASS LONG.**
- **ICE** ($150.61, SMA50 $140.1366, SMA200 $154.5897): price>SMA50 but SMA50<SMA200 → fails LONG stack (52wk range only ~44% top-to-bottom, and pct_52w_range=0.4266 fails the 0.6 threshold anyway). SHORT: price not < SMA50 → fails. **FAIL both.**

**Tier 1 survivors (10, all bullish/with-trend since SPY regime is LONG):** TECH, BMY, PARR, BFLY, TXG, ADP, PBF, DFTX, IMAX, NTNX.

**Value/momentum composite ranking (0.6×rel-strength + 0.4×value-rank; negative-PE names ranked below all positive-PE names, ordered least-negative-first; top 8 advance):** PBF (0.903), BMY (0.840), PARR (0.806), IMAX (0.760), TECH (0.729), ADP (0.679), BFLY (0.623), DFTX (0.608), NTNX (0.601), TXG (0.597) — **NTNX and TXG cut** (bottom two). Top 8 to Tier 2: PBF, BMY, PARR, IMAX, TECH, ADP, BFLY, DFTX.

**Tier 2 — entry trigger, top 8** (indicator series still dated through Fri 2026-08-07 close — no new daily bar has rolled yet today):
- **PBF:** price $66.13 vs high52 $74.74 → 11.5% off, breakout fails. RSI=53.51, outside 35-45. **DROP.**
- **BMY:** price $65.075 vs high52 $68.10 → 4.4% off, breakout fails. RSI=63.25, outside 35-45. **DROP.**
- **PARR:** price $72.02 vs high52 $87.03 → 17.2% off, breakout fails. RSI=39.50 (in band) but EMA21 last-5 declining (75.54→74.24), not flat-to-rising. **DROP.**
- **IMAX:** price $50.775 within 1.60% of high52 $51.60 ✓; EMA8 $48.663 > EMA21 $45.598 ✓. **Breakout trigger fires — PASS.**
- **TECH:** price $72.165 within 0.31% of today's new high $72.39 ✓; EMA8 $72.034 > EMA21 $70.909 ✓. **Breakout trigger fires — PASS.**
- **ADP:** price $272.01 vs high52 $310.08 → 12.3% off, breakout fails. RSI=65.84, outside 35-45. **DROP.**
- **BFLY:** price $9.74 vs today's new high $10.05 → 3.08% off, just outside the 2% breakout band. RSI=61.60, outside 35-45. **DROP.**
- **DFTX:** price $44.16 vs high52 $49.70 → 11.2% off, breakout fails. RSI=61.88, outside 35-45. **DROP.**

**Tier 2 survivors: IMAX, TECH** (both breakout triggers, under the Tier 3 cap of 3).

**Tier 3 — momentum confirmation (|EMA8−EMA21| must be strictly widest at the most recent of 5 sessions):**
- **TECH:** spread Aug3→Aug7: 1.5696, 1.4470, 1.3192, 1.1891, **1.1243** (most recent) — narrowest of the 5, decelerating. **FAILS.** (Identical indicator readings and identical outcome to this morning's midday reject, since no new daily bar has closed since Friday.)
- **IMAX:** spread Aug3→Aug7: 3.2059, 3.5001, 3.5547, 3.4033, **3.0648** (most recent) — peaked Aug 5, now the narrowest of the 5, rolling over. **FAILS.**

Both logged to `trade_log.csv` as Tier 3 rejects (`TECH-20260810-2`, `IMAX-20260810-1`) with full indicator snapshots. `news_sentiment` recorded `UNAVAILABLE` for both — Alpha Vantage quota already confirmed exhausted this run, so the Tier 3 news call was not attempted (would fail identically); non-gating for a candidate already dropped on momentum.

**Index sleeve (SSO, since SPY is in LONG mode):** SPY trend template: $772.97 > SMA50 $747.19 > SMA200 $702.97 ✓; within 25% of high52 ($776.85) ✓. **Fails rule 3** — at least 25% above low52 ($629.28 × 1.25 = $786.60) — price doesn't clear it. Same structural issue as every run today: SPY's 52-week range (low set 2026-03-30) makes this rule unsatisfiable at the current range regardless of trend strength. **No SSO trade.** Third occurrence today — still just one day's worth of data points, not yet a pattern across ≥5 of the last 10 *closed* trades per the adaptation policy's threshold, so no strategy change proposed. Worth flagging for the human's attention if this persists once real trades start closing: a structural SPY range condition that blocks the index sleeve on every single run this system has ever made is worth a deliberate look, distinct from the self-adaptation policy's evidence bar.

**Decisions:** No new positions opened (equity, options, or index sleeve). No positions closed (none open).

**Rejected by gates:** None reached gates.md-level sizing/execution checks. TECH and IMAX were dropped at Tier 3 (momentum confirmation) — see `trade_log.csv`. PBF, BMY, PARR, ADP, BFLY, DFTX were dropped at Tier 2 (no entry trigger fired); GFS, ICE at Tier 1 (trend template); NTNX, TXG at the Tier 1→2 value/momentum cut.

**Strategy adaptation this run:** None. Zero closed trades on record; the adaptation policy requires reviewing closed-trade history, which doesn't exist yet.

**Day summary (all three runs today):** Zero trades opened across morning, midday, and close runs. The funnel worked as designed — real candidates reached Tier 2/3 each run (TSLA counter-trend at Tier 2 this morning; TECH/TXG then IMAX/TECH again at Tier 3 midday/close) but none cleared the final bar. No gate was loosened to force a fit. First actual trading day still pending a candidate that clears every tier.
