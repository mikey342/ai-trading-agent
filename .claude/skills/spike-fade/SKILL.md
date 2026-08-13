---
name: spike-fade
description: On-demand technical read for a single stock the user names — is it overbought/over-extended after a spike (possible short-term fade, via puts or a bear ETF) or oversold/over-extended after a plunge (possible bounce, via calls or equity)? Walks through qualification, extension, confirmation, stand-down conditions, instrument choice, and an exit plan for both directions before giving a read. Use when the user gives a ticker and asks whether it's overbought, oversold, extended, "due for a pullback," or worth fading/buying after a jump, gap, crash, or earnings move. Read-only research: never opens, sizes, or logs a trade, and is separate from the scheduled trading-agent routines.
---

# Spike / Fade Check — single-ticker, on-demand, both directions

**This is a research tool, not a routine.** It runs only when the user
names a ticker and asks. It does not read or write `positions.md` or
`trade_log.csv`, does not check `gates.md`'s position/sector/counter-trend
caps, and never places or "papers" a trade. It can look at any ticker,
not just names that clear the Tier 0 screener ($20M/day volume, $2B
market cap, ADX>25) the scheduled routines use — that screener exists to
keep an unattended scan cheap, and doesn't apply to a one-off lookup a
human is initiating directly.

This skill checks **both directions** with equal rigor: a stock that
spiked up (possible overbought fade) and a stock that plunged (possible
oversold bounce). The long/bounce side's core numeric trigger
(`RSI(2) < 15`, price > SMA(200), first up-close) matches `strategy.md`'s
existing **live** `mr_reversal` signal — that part is not redefined here,
just wrapped in the same qualification funnel built for the short side.
**The overbought/fade side has no live counterpart in the trading agent
at all** — it exists only in this skill. Neither side opens, sizes, or
logs anything; both are pure analysis.

## Why both directions require confirmation, not anticipation

This system's own framework treats post-earnings-announcement drift
(PEAD) as strong evidence, and it cuts both ways:

- **Beats drift up.** Bernard & Thomas (1989): stocks that beat earnings
  keep drifting up for weeks; the market underreacts to the surprise.
  Live research confirms this operationally — earnings-driven gap-ups
  hold their gains ~71% of the time within 5 days, and volume-confirmed
  gaps rarely fill within a month.
- **Misses drift down — and can drift down *harder*.** The same
  underreaction mechanism runs on negative surprises: research finds
  investors underreact to negative surprises specifically **in stocks
  already far from their 52-week high**, producing a *stronger*
  subsequent downward drift than for stocks closer to their highs. That
  matters here because the stocks most likely to look like "great
  oversold bounce candidates" (already beaten down, far from their
  highs) are exactly the population where this stronger continuation
  effect applies. A very recent paper (Jegadeesh, Luo, Subrahmanyam &
  Titman, *Review of Financial Studies*, Dec 2025) frames this
  symmetrically: their model predicts **attenuated reversals specifically
  around earnings announcements**, on both sides, not just the upside.

So neither "it popped, short it" nor "it crashed, buy it" applies the
evidence — both would fight PEAD. The fix, symmetric on both sides:
**only treat either as a candidate once the tape has already started
reversing**, not in anticipation. That's short-horizon reversal
(Jegadeesh 1990, Lehmann 1990) applied to a confirmed crack, not a
predicted one.

Two more recent findings sharpen *why the confirmed cases work*, and
they're explicitly two-sided in the source research: the SSRN 2025 "MAX
effect" paper studied stocks that were extreme movers **and** past
1-week winners *or* losers, finding the reversal enhancement concentrates
in stocks with heavy **retail order imbalance** — and that it's a
**noise-trading** effect (RFS 2025), not an informed-money signal. In
plain terms: both sides of this are a retail-overreaction trade, not a
"smart money knows something" trade. That's exactly why both need
confirmation rather than anticipation.

**No sub-daily version of either side exists on purpose.** This trading
agent holds 3-10 days and runs three fixed times a day; nothing here is
designed for intraday timeframes (see `framework.md`). It's also worth
being honest that the academic edge is front-loaded on both sides
(Quantpedia: "the mean-reversion edge is heavily concentrated in the
first trading day after entry") — this system's multi-day swing hold
captures a slower, diluted slice of the effect, not its purest form.

## The funnel, run for both directions

Work through six questions, in order, for whichever direction the recent
move points at. A fail at **A** or **D** means "not a candidate" — stop
there rather than keep scoring the rest.

---

### SHORT SIDE — overbought spike → fade

#### A. Does the setup even qualify?

All of:
- **A real spike**: last completed session's return ≥8%, or ≥15%
  cumulative over 3 sessions.
- **Volume-real**: relative volume ≥1.4x 30-day average on the spike
  session (O'Neil's "this move is real" bar). ≥3-5x reads as a genuine
  climax rather than just a confirmed move.
- **An identifiable catalyst** within ~3 calendar days — earnings or
  news. No catalyst, no trade: this isn't the pattern, it's generic
  momentum, a different setup.
- **Retail-chasing signature, logged as a qualifier**: Stocktwits message
  volume around the spike, for a surge vs. its own baseline — the
  closest available proxy for the "heavy retail order imbalance" the
  2025 SSRN research ties the reversal enhancement to. Sparse coverage
  on mid-caps is expected; `NO_COVERAGE` is a valid answer, not a
  negative signal.
- **Turnover / range-position check, not range-position alone**: a
  stock that is **both** high-turnover **and** near its 52-week high
  historically shows momentum continuation, not reversal (ScienceDirect,
  2024) — a strike against the setup, not neutral color. High turnover
  with a **lower** range position still reverses well.
- **Earnings catalysts start with a lower prior than news catalysts** —
  not disqualifying, but say so rather than treating all catalysts as
  equal (see the PEAD section above).

#### B. Is it actually overbought? (extension)

- `RSI(2) ≥ 90` (ConnorsRSI convention)
- `RSI(14)` elevated, for context
- **ATR-stretch**: `(price − SMA20) / ATR14`
- 52-week range position — read together with turnover from A
- Bollinger %B > 1, best-effort only
- ADX: still climbing hard → less ripe yet. Flattening/rolling over on a
  marginal new high → corroborates exhaustion.

#### C. Is it actually turning? (confirmation — the part that matters most)

The **four pillars**: one required, plus at least two of the remaining
three at the same time.

1. **Pattern/structural break — required.** Minimum: a down-close.
   Stronger: a level break — below the prior session's low, below the
   spike's opening print, or below EMA8.
2. **Volume** — the down day on *above-average* volume (distribution).
3. **Exhaustion candlestick** — shooting star, bearish engulfing, or
   doji at the highs.
4. **Divergence** — RSI/MACD lower high vs. price higher high. **Weight
   lightest of the four** — flagged as specifically unreliable on
   parabolic, catalyst-driven moves, exactly this setup.

Report which pillars are present, not just a count. Note whether a
second consecutive down day is also in, and whether a conservative entry
(wait for a retest of the broken level) vs. an aggressive one (act on
the initial break) applies.

#### D. When to stand down

- No down-close yet, however extended → too early. Don't anticipate it.
- Choppy, range-bound tape → RSI/MACD unreliable in chop.
- High turnover **and** near the 52-week high, no crack yet → weaker
  odds even with every box in B checked.
- Fresh beat still drifting, sentiment still strongly positive, no
  retail-chasing signature → the textbook PEAD case. Stand down.
- `NEWS_SENTIMENT` unavailable → no read on that leg; say so.
- Divergence present but nothing else → not enough alone.

#### E. Instrument choice, given what confirms

- **Put** — benefits structurally from IV crush having mostly already
  happened by the time C can fire (confirmation requires a *completed*
  session after the catalyst). Reference gates, informational only:
  30+ DTE, delta 0.40-0.60 absolute, OI ≥500, spread ≤10% of mid.
- **Single-stock bear ETF** (e.g. a −2x product) — no IV-crush cushion;
  waiting purely costs entry price. Verify actual leverage per
  instrument (some "bear" ETFs are −1x, not −2x).
- **No plain-equity option** — equity shorting is forbidden system-wide.
  This is a real asymmetry vs. the long side below, not an oversight.
- Size to confirmation strength: pattern break alone → probe or skip.
  Pattern + 2 pillars → stronger candidate for full size.

#### F. If wrong, when to exit

Informational only — nothing is tracked between invocations.

- **Thesis invalidation**: underlying reclaims and closes back above the
  level that confirmed the turn.
- **Stop distance**: ~1.5x ATR14, not a tight percent stop.
- **Target**: `RSI(2)` back down to < 30 (mirrors `mr_reversal`'s
  `RSI(2) > 70` exit).
- **Time cap**: ~5 trading days — generous relative to where the
  academic edge actually lives (front-loaded, decayed by 1-4 weeks).
- **Options-specific**: 21-DTE checkpoint, and 50%-of-entry-DTE-elapsed
  checkpoint.

---

### LONG SIDE — oversold plunge → bounce

#### A. Does the setup even qualify?

**Two admissible paths, not one:**

- **Path 1 — `mr_reversal` (live, already in `strategy.md`)**: a plain
  short-term pullback within an intact uptrend. `RSI(2) < 15` on the
  last completed session, price > SMA(200). No catalyst or spike
  magnitude required — this is the system's existing, evidenced rule.
- **Path 2 — plunge-exhaustion bounce (new, mirrors the short side)**:
  same structure as the short side's qualification, sign-flipped:
  - A real plunge: last completed session ≤ −8%, or ≤ −15% cumulative
    over 3 sessions.
  - Volume-real: relative volume ≥1.4x on the plunge session; ≥3-5x for
    a genuine capitulation read.
  - An identifiable catalyst within ~3 days — an earnings miss or
    negative news.
  - Retail panic-selling signature, logged: a surge in Stocktwits
    message volume / a shift toward bearish `label` around the plunge.
    `NO_COVERAGE` is a valid answer.
  - **⚠ The downside-PEAD caution belongs here, explicitly**: if the
    catalyst is an earnings miss and the stock is already far from its
    52-week high, the underreaction research above says continued
    drift **down** is the stronger prior, not a bounce. Path 2 should be
    treated with *more* suspicion than Path 1, not less, precisely
    because it's the catalyst-driven, PEAD-adjacent case.
  - **Short-interest / days-to-cover, if available** — a mechanism with
    **no mirror on the short side**: high short interest (>20-30% of
    float) raises bounce potential through short-covering dynamics, not
    just mean reversion. This repo's current toolset has no confirmed
    short-interest data source — log it only if some tool surfaces it,
    never block on its absence.
  - No confirmed research equivalent of the short side's
    turnover-near-52-week-**low** relationship was found — say so rather
    than inventing a mirrored rule that doesn't have a citation behind
    it.

#### B. Is it actually oversold? (extension)

- `RSI(2) < 15` — note this is **already looser than the ConnorsRSI
  convention** (Connors published `< 5` / `< 10`); this repo's live
  `mr_reversal` runs it at 15 as a standing, pre-registered experiment
  (`strategy.md`). This is a pre-existing asymmetry with the short
  side's strict `≥ 90`, not something introduced here.
- `RSI(14)` — needed for the dead-cat-bounce check in C, not just
  context.
- **ATR-stretch below SMA20**: `(SMA20 − price) / ATR14`
- 52-week range position — near the bottom
- Bollinger %B < 0, best-effort only
- ADX: still climbing hard on the downside → less ripe yet. Flattening
  or rolling over → corroborates exhaustion.

#### C. Is it actually turning? (confirmation, mirrored four pillars)

1. **Pattern/structural break — required.** Minimum: a first up-close
   (this is `mr_reversal`'s live trigger). Stronger: a level reclaim —
   above the prior session's high, above the plunge's opening print, or
   above EMA8.
2. **Volume** — the up day on *above-average* volume (accumulation).
   **This is the load-bearing dead-cat-bounce check**: a real reversal
   tends to show rising volume; a dead-cat bounce typically shows up on
   *low* volume that fails to sustain.
3. **Exhaustion candlestick** — hammer, bullish engulfing, or doji at
   the lows.
4. **Divergence** — bullish RSI/MACD divergence (price lower low,
   indicator higher low). Some practitioner sources rate bullish
   divergence as *more* reliable specifically when RSI is simultaneously
   oversold than the equivalent overbought-divergence case — a small
   asymmetric edge in this direction's favor — but the same
   parabolic/catalyst-driven-move caution still applies to an
   earnings-miss crash. Weight it second-lightest, not the primary leg.

**Dead-cat-bounce disqualifiers, specific to this side** — treat any of
these as reasons to downgrade the read even if the four-pillar count
looks adequate:
- The bounce retraces only ~10-40% of the drop and stalls.
- `RSI(14)` climbs but stays **below the 50 midline** — a bounce that
  can't reclaim momentum's own halfway point is the textbook dead-cat
  signature, not a real reversal.
- Price fails to clear the level it broke down through (old support,
  now resistance) and rolls back below the pre-bounce low.

#### D. When to stand down

- No up-close yet, however oversold → too early.
- Choppy, range-bound tape → same caution as the short side.
- **Fresh earnings miss, sentiment still very negative, stock already
  far from its 52-week high (low PTH)** → per the citation above, this
  is the case with the *strongest* continued-downward-drift prior.
  Stand down hardest here — this is downside PEAD's textbook case.
- Bounce shows the dead-cat signature from C (low volume, RSI(14) stuck
  below 50, shallow retrace, rejected at old support) → likely not a
  real reversal.
- `NEWS_SENTIMENT` unavailable → no read on that leg; say so.
- Divergence present but nothing else → not enough alone.

#### E. Instrument choice, given what confirms

- **Call** — same IV-crush-cushion logic as the short side's put: most
  of the crush has typically already happened by the time C can fire.
  Reference gates, informational only: 30+ DTE, delta 0.40-0.60, OI
  ≥500, spread ≤10% of mid.
- **Plain equity — a real option here, unlike the short side.** Going
  long has no forbidden-mechanism constraint the way shorting stock
  does, so equity is a legitimate, simpler instrument choice this side
  gets that the short side structurally cannot. Note this asymmetry
  explicitly rather than defaulting to options out of habit.
- **If the read leans on short-interest/squeeze dynamics** (see A):
  treat it as a *faster, shorter* opportunity, not a bigger one.
  Research on squeezes is explicit that they "reverse quickly once
  covering ends" and that late entries carry the most risk, since
  there's no fundamental support left once the squeeze itself is over —
  this argues for tighter time expectations, not larger size.
- Size to confirmation strength, same as the short side: pattern break
  alone → probe or skip; pattern + 2 pillars → stronger candidate for
  full size.

#### F. If wrong, when to exit

Informational only — nothing is tracked between invocations.

- **Thesis invalidation**: underlying breaks back below the level that
  confirmed the turn — the reclaimed level, or the pre-up-close low.
- **Stop distance**: ~1.5x ATR14 below entry — mirrors this repo's live
  `mr_reversal` stop exemption and its reasoning exactly (elevated-ATR
  names by construction).
- **Target**: `RSI(2) > 70` — this **is** `mr_reversal`'s actual live
  exit rule, not just a mirrored analogy.
- **Time cap**: ~5 trading days — again matches `mr_reversal`'s live
  time-stop exactly. If the read leaned on short-interest/squeeze
  dynamics, treat this cap as tighter still.
- **Options-specific**: 21-DTE checkpoint, 50%-of-entry-DTE-elapsed
  checkpoint.

---

## What doesn't actually mirror between the two sides

Worth stating plainly rather than pretending this is a clean sign-flip:

- **Equity is a valid instrument long, never short** — shorting stock is
  forbidden system-wide; going long isn't restricted the same way.
- **Short-covering/squeeze dynamics exist only on the bounce side** —
  there's no equivalent mechanism pushing an overbought stock down
  faster the way covering pushes an oversold one up.
- **The 52-week-high+turnover research (ScienceDirect 2024) has no
  confirmed 52-week-low equivalent** in what's been gathered — this is a
  real evidence gap on the long side, not filled by assumption.
- **`mr_reversal`'s live `RSI(2) < 15` is already looser than convention**
  (Connors published `<5`/`<10`), while the short side's `RSI(2) ≥ 90`
  follows the stricter convention directly — a pre-existing asymmetry in
  the live trading agent, not introduced by this skill.
- **This repo's own `framework.md` notes stocks drift upward structurally
  over time** (the equity risk premium) — a background tailwind for the
  long side that has no long-side mirror on the short side.
- **Downside PEAD is the sharper caution.** Negative-surprise
  underreaction is *stronger* for stocks already far from their highs —
  exactly the stocks that otherwise look like the best oversold-bounce
  candidates. The short side's equivalent caution (fresh beats drift up)
  doesn't have as pointed a "the best-looking candidates are the
  riskiest" twist.

## Tool calls, in order (minimize calls, never trust the wrong source)

1. **Orient**: `get_equity_quotes` (live price + official prior close),
   `get_equity_fundamentals` (sector, 52-week range, market cap, 30-day
   avg volume), `get_earnings_results` (trailing surprise history) and
   `get_earnings_calendar` (upcoming date). If nothing identifies a
   catalyst, ask the user, or say plainly that none was found.
2. **Quantify the move (A, both sides)**: last-session and 3-session
   returns; relative volume = last completed session's volume ÷ 30-day
   average, from `get_equity_fundamentals` — **never** a `run_scan`
   column, whose `session="all"` indicators have been measured up to 12
   points off the direct regular-session value (`tool_verification.md`).
3. **Retail-chasing/panic-selling check (A, both sides)**: Stocktwits
   `get_message_volume` / `get_symbol_pulse` around the move date. Use
   `score` and `label`, **never `bullish_pct`** — a known trap on thin
   names. `NO_COVERAGE` is a valid, common answer.
4. **Extension (B, both sides)**: `get_equity_technical_indicators` for
   `RSI(2)`, `RSI(14)`, `ATR14`, `SMA20`, `SMA50`, `SMA200`, `EMA8`,
   `EMA21`, `ADX14` (direct call, not a scan column) and `MACD` via
   Robinhood's own `type=macd` (Alpha Vantage's `MACD` is permanently
   premium-gated on this account — never call it). Compute ATR-stretch
   and the turnover proxy (`(volume × price) / market_cap`) from fields
   already fetched — no new calls. Bollinger `%B` via Alpha Vantage
   `BBANDS` is optional, best-effort only — **not** verified working in
   `tool_verification.md`; if it errors or the quota is exhausted, drop
   it rather than blocking.
5. **Confirmation (C, both sides)**: check the four pillars against the
   last completed session (and current price vs. it), for whichever
   direction the move points.
6. **Sentiment, logged not gating**: `NEWS_SENTIMENT` (Alpha Vantage,
   quota-limited — `limit: 5`). Report the score and whether it reads as
   a durable, still-moving story or an already-fading one. Don't invent
   a hard directional threshold — `tool_verification.md` explicitly
   declines to gate on sentiment direction without a closed-trade
   record.

## Report format

```
## <TICKER> — Spike/Fade Check (<date>)

**Direction:** <recent move points overbought/spike | oversold/plunge | neither>

**A. Qualifies?** move X% (Y% 3-session) · rel. volume Zx · catalyst:
<earnings beat/miss/inline | news | none> <N>d ago · retail signature:
<Stocktwits surge, bullish/bearish | none | NO_COVERAGE> · turnover
proxy X% · range position X% · [short interest: X%, if available]
→ <QUALIFIES (path: mr_reversal / plunge-exhaustion / spike-fade) /
DOES NOT QUALIFY: reason>

**B. Extension:** RSI(2) X · RSI(14) X · ATR-stretch X.Xx · ADX X
(<climbing/flat/rolling over>) · [%B: X, best-effort]

**C. Turning?** Pillars present: <list, e.g. "pattern break, volume;
missing: candlestick, divergence">. Pattern break <present/absent>
(required). [Dead-cat-bounce check, long side only: RSI(14) vs. 50,
retrace %, old-support reclaim]

**D. Stand-down check:** <none triggered | reason, incl. downside/
upside PEAD caution if relevant>

**Read:** one of —
- OVERBOUGHT — FADE CANDIDATE / OVERBOUGHT — WEAK CONFIRMATION /
  OVERBOUGHT — TOO EARLY
- OVERSOLD — BOUNCE CANDIDATE / OVERSOLD — WEAK CONFIRMATION (dead-cat
  risk) / OVERSOLD — TOO EARLY
- OVERSOLD — FALLING KNIFE (price < SMA200, excluded by design)
- NEUTRAL / NO SIGNAL
- DOES NOT QUALIFY (state which A condition failed)

**E. If expressing this as a trade** (informational only — nothing
opened, sized, or logged): put/bear-ETF (short) or call/equity (long)
trade-off, and probe-vs-full sizing given the pillar count above.

**F. Exit plan if entered** (informational, not tracked): invalidation
level, ATR stop distance, RSI(2) target, time cap, options checkpoints.

**Evidence caveat:** repeat whenever the read is a FADE or BOUNCE
CANDIDATE: this is a retail-overreaction, noise-trading effect (not a
"smart money" signal), weaker specifically on earnings catalysts than
other news — sharper on the downside, where the best-looking candidates
carry the strongest continuation risk — and its academic edge is
front-loaded, so a multi-day hold captures a diluted slice of it.
```

## What this skill deliberately does not do

- Does not open, size, or place any position — paper or real.
- Does not write to `positions.md`, `trade_log.csv`, or
  `trade_journal.md`. If the user wants a lookup kept for the record,
  ask before writing anywhere.
- Does not check `gates.md`'s position/sector/counter-trend caps — there
  is nothing being opened for those caps to apply to.
- Does not track section F over time — there is no state between
  invocations. "When to exit" is guidance to re-check next time, not a
  monitored stop.
- Does not run on a schedule and is not part of `run_instructions.md`,
  `position_monitor_instructions.md`, or the premarket/daily-report
  routines, and is not itself a trigger in `strategy.md` or `gates.md`
  (except where it explicitly points at the live `mr_reversal` rule).
  Those stay exactly as they were before this skill existed.
