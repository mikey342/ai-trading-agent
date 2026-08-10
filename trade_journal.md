# Trade Journal

Append-only log. Never delete or rewrite past entries — if a past decision
turned out wrong, say so in a new entry, don't erase the record.

Each run should add one entry using the template below.

---

## Template

### YYYY-MM-DD HH:MM UTC — [morning|midday|close] run

**Market status:** (open/closed, per `MARKET_STATUS`)
**Account:** NAV $X, cash $X, halted: yes/no

**Scouted:** (symbols considered this run and why, e.g. from `TOP_GAINERS_LOSERS`)

**Decisions:**
- SYMBOL — action (opened/closed/held/skipped) — qty — price — rationale
  (indicator values, news summary, which strategy.md rule fired)
- ...

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
