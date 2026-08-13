---
name: spike-fade
description: On-demand technical read for a single stock the user names — is it overbought/over-extended after a spike (possible short-term fade, expressed via puts) or oversold within an uptrend (possible bounce)? Walks through qualification, extension, confirmation, stand-down conditions, instrument choice, and an exit plan before giving a read. Use when the user gives a ticker and asks whether it's overbought, oversold, extended, "due for a pullback," or worth fading after a jump/gap/earnings pop. Read-only research: never opens, sizes, or logs a trade, and is separate from the scheduled trading-agent routines.
---

# Spike / Fade Check — single-ticker, on-demand

**This is a research tool, not a routine.** It runs only when the user
names a ticker and asks. It does not read or write `positions.md` or
`trade_log.csv`, does not check `gates.md`'s position/sector/counter-trend
caps, and never places or "papers" a trade. It can look at any ticker,
not just names that clear the Tier 0 screener ($20M/day volume, $2B
market cap, ADX>25) the scheduled routines use — that screener exists to
keep an unattended scan cheap, and doesn't apply to a one-off lookup a
human is initiating directly.

The oversold/bounce side of this reuses `strategy.md`'s existing
mean-reversion sleeve (`mr_reversal`: `RSI(2) < 15`, price > SMA(200),
first up-close) — that logic is live in the trading agent, not redefined
here. **The overbought/fade side is not part of the trading agent at
all** — it lives only in this skill, standalone, and is explained in
full below rather than pointing elsewhere.

## Why the fade side is narrower than "stock jumped, short it"

This system's own framework treats post-earnings-announcement drift
(PEAD) as strong evidence: Bernard & Thomas (1989) found stocks that beat
earnings estimates keep drifting *up* for weeks, and the market
underreacts to the surprise. Live research confirms this isn't just
theoretical: earnings-driven gap-ups hold their gains ~71% of the time
within 5 days, gaps that print on confirming volume fill less than 30%
of the time within a month, and blindly fading an earnings gap is
widely described as one of the more reliable ways to lose money on a gap
trade. A very recent academic source lands on the same point from a
different angle: Jegadeesh, Luo, Subrahmanyam & Titman (*Review of
Financial Studies*, Dec 2025) model **attenuated reversals specifically
after earnings announcements** — not spikes in general, earnings
specifically.

So "it jumped on earnings → it's overbought → short it" would contradict
that evidence, not apply it. The fix used here: **only treat it as a
fade candidate once the tape has already started reversing**, rather
than anticipating that it will. That changes the claim from "this pop
will reverse" (fights PEAD) to "this pop has already cracked, and
short-horizon reversal (Jegadeesh 1990, Lehmann 1990 — the same effect
`mr_reversal` already trades on the oversold side) says that tends to
continue a few more days."

Two further, more recent findings sharpen *why* the confirmed cases
work: the reversal effect concentrates specifically in stocks with
**heavy retail order imbalance** chasing the spike (Chen/Cohen/Liang/Sun,
SSRN 2025 — the "MAX effect" enhancement only holds in the top quintile
of retail order imbalance) and is driven by **noise trading**, not
informed trading (the same RFS 2025 paper). In plain terms: this is a
retail-overreaction trade, not a "smart money knows something" trade —
which is exactly why it needs confirmation rather than anticipation, and
why a spike with no visible retail chasing behind it is a weaker
candidate even if the price/RSI numbers look identical.

**No sub-daily version of this exists on purpose.** This trading agent
holds 3-10 days and runs three fixed times a day; nothing here is
designed for intraday timeframes (see `framework.md`). A same-day
"lost VWAP by 11am" fade is a day-trading rule and doesn't fit this
system's data or cadence — don't add one. It's also worth being honest
that the academic edge is front-loaded (Quantpedia: "the mean-reversion
edge is heavily concentrated in the first trading day after entry") — a
multi-day swing hold, which is what this system uses, captures a slower,
diluted slice of the effect, not its purest form.

## The funnel — six questions, in order

Work through these in order. A fail at **A** or **D** means "not a
candidate" — stop there, don't keep scoring the rest for a stock that
already doesn't qualify.

### A. Does the setup even qualify?

All of:
- **A real spike**: last completed session's return ≥8%, or ≥15%
  cumulative over 3 sessions.
- **Volume-real**: relative volume ≥1.4x 30-day average on the spike
  session (O'Neil's "this move is real" bar — never read from a scan
  column, see the tool-call notes below). ≥3-5x reads as a genuine
  climax rather than just a confirmed move.
- **An identifiable catalyst** within ~3 calendar days — earnings or
  news. No catalyst, no trade: this isn't the pattern, it's generic
  momentum, a different setup.
- **Retail-chasing signature, logged as a qualifier**: check Stocktwits
  message volume around the spike (`get_message_volume` /
  `get_symbol_pulse`) for a surge versus its own baseline. This is the
  closest available proxy for the "heavy retail order imbalance" the
  2025 SSRN research ties the reversal enhancement to. Coverage is
  sparse on mid-caps (per `tool_verification.md`) — if there's no
  coverage, say `NO_COVERAGE` and treat this qualifier as unavailable,
  don't treat silence as a negative signal.
- **Turnover / range-position check, not range-position alone**: compute
  a turnover proxy — `(volume × price) / market_cap` — from fields
  already fetched, and combine it with 52-week range position. A stock
  that is **both** high-turnover **and** near its 52-week high
  historically shows momentum continuation, not reversal (ScienceDirect,
  2024) — that combination is a strike against the setup, not neutral
  color. High turnover with a **lower** range position still reverses
  well.
- **Earnings catalysts start with a lower prior than news catalysts** —
  not disqualifying, but the report should say so rather than treating
  all catalysts as equal.

### B. Is it actually overbought? (extension)

- `RSI(2) ≥ 90` (ConnorsRSI convention)
- `RSI(14)` elevated, for context
- **ATR-stretch**: `(price − SMA20) / ATR14` — several ATRs above its
  own short-term average
- 52-week range position — **read together with turnover from A**, not
  in isolation
- Bollinger %B > 1, best-effort only (see tool-call notes)
- ADX: still climbing hard → the existing trend has real structural
  strength, less ripe yet. Flattening or rolling over on a marginal new
  high → corroborates exhaustion rather than just a pause.

### C. Is it actually turning? (confirmation — the part that matters most)

Use the **four pillars**: one is required, plus at least two of the
remaining three present at the same time.

1. **Pattern/structural break — required.** Minimum bar: the most
   recently completed session (or current price vs. it) is a down-close.
   Stronger: an actual level break — below the prior session's low,
   below the spike's opening print, or below a short support average
   (EMA8) — not just a marginally lower close.
2. **Volume** — the down day on *above-average* volume (distribution),
   checked separately from the spike-day volume check in A. A red day
   on light volume is weaker evidence than one on real volume.
3. **Exhaustion candlestick** — a shooting star, bearish engulfing, or
   doji at the highs.
4. **Divergence** — RSI/MACD printing a lower high while price prints a
   higher high. **Weight this one lightest of the four.** It's
   specifically flagged as unreliable on parabolic, catalyst-driven
   moves — exactly this setup — so treat it as the weakest possible leg
   of the 2-of-3, never sufficient alone.

Report which pillars are present, not just a yes/no. Optionally note
whether a second consecutive down day / lower low is also in (extra
confirmation, at the cost of a later entry), and whether the more
conservative entry (wait for a retest of the broken level) vs. the more
aggressive one (act on the initial break) applies.

### D. When to stand down

- No down-close yet, however extended per B → too early. Don't
  anticipate it — this is what keeps the setup from fighting PEAD.
- Choppy, range-bound broader tape → RSI/MACD readings are unreliable in
  chop; say so rather than forcing a read.
- High turnover **and** near the 52-week high, per A, with no crack yet
  → weaker odds even with every box in B checked.
- Fresh beat still drifting, sentiment still strongly positive, no
  retail-chasing signature found → the textbook PEAD case. Stand down.
- `NEWS_SENTIMENT` unavailable (Alpha Vantage quota exhausted) → no read
  possible on that leg; say so, don't guess — same standard this repo
  already applies to counter-trend trades.
- Divergence present but nothing else from C → not enough on its own,
  especially here.

### E. Instrument choice, given what confirms

- **Put** — structurally benefits from IV crush having mostly already
  happened by the time C can even fire (confirmation requires a
  *completed* session after the catalyst) — cheaper premium, less vega
  risk than buying into the event itself. Reference gates, informational
  only: 30+ DTE, delta 0.40-0.60 absolute, open interest ≥500, spread
  ≤10% of mid.
- **Single-stock bear ETF** (e.g. a −2x product) — no IV-crush cushion;
  waiting purely costs entry price. Must verify actual leverage per
  instrument (some "bear" ETFs are −1x, not −2x — never infer from the
  ticker) and check the underlying's volatility against decay
  suitability.
- **Size to confirmation strength**: pattern break alone, nothing else
  from C → probe size or skip. Pattern + 2 pillars → the stronger
  candidate for full size. A scaled entry (small probe at B-level
  extension, add to full size only at C-level confirmation) is the
  professional middle ground between waiting for everything and jumping
  in early — mention it as an option, not something this stateless
  skill tracks or manages.

### F. If wrong, when to exit

Informational reference only — this skill doesn't hold or track
positions, so nothing here is monitored automatically. Recompute by
asking again.

- **Thesis invalidation (the real stop)**: the underlying reclaims and
  closes back above the level that confirmed the turn in C — the broken
  support level, or the pre-down-close spike high.
- **Stop distance**: an ATR-derived stop (~1.5x ATR14) rather than a
  tight percent stop — this setup selects elevated-ATR names by
  construction, same reasoning this repo already applies to its
  long-side mean-reversion stop.
- **Thesis played out (target)**: `RSI(2)` back down to a low/neutral
  reading (e.g. < 30) — mirrors `mr_reversal`'s `RSI(2) > 70` exit on
  the long side.
- **Time cap**: the academic edge is short-horizon and front-loaded
  (concentrated in the first day or two, largely decayed by 1-4 weeks) —
  a ~5 trading day cap is generous relative to where the research says
  the edge actually lives, not conservative.
- **Options-specific**: the standard 21-DTE checkpoint (gamma risk), and
  a checkpoint at 50% of entry DTE elapsed with no progress — same
  discipline this repo already uses elsewhere for options.

## Tool calls, in order (minimize calls, never trust the wrong source)

1. **Orient**: `get_equity_quotes` (live price + official prior close),
   `get_equity_fundamentals` (sector, 52-week range, market cap, 30-day
   avg volume), `get_earnings_results` (trailing surprise history) and
   `get_earnings_calendar` (upcoming date). If nothing identifies a
   catalyst, ask the user, or say plainly that none was found.
2. **Quantify the move (A)**: last-session and 3-session returns;
   relative volume = last completed session's volume ÷ 30-day average,
   from `get_equity_fundamentals` — **never** a `run_scan` column, whose
   `session="all"` indicators have been measured up to 12 points off the
   direct regular-session value (`tool_verification.md`).
3. **Retail-chasing check (A)**: Stocktwits `get_message_volume` /
   `get_symbol_pulse` around the spike date. Use `score` and `label`,
   **never `bullish_pct`** — it's a known trap on thin names (reads as
   false consensus). `NO_COVERAGE` is a valid, common answer.
4. **Extension (B)**: `get_equity_technical_indicators` for `RSI(2)`,
   `RSI(14)`, `ATR14`, `SMA20`, `SMA50`, `SMA200`, `EMA8`, `EMA21`,
   `ADX14` (direct call, not a scan column) and `MACD` via Robinhood's
   own `type=macd` (Alpha Vantage's `MACD` is permanently premium-gated
   on this account — never call it). Compute ATR-stretch and the
   turnover proxy from fields already fetched — no new calls. Bollinger
   `%B` via Alpha Vantage `BBANDS` is optional, best-effort only — this
   tool has **not** been verified working in `tool_verification.md`; if
   it errors or the quota is exhausted, drop it rather than blocking.
5. **Confirmation (C)**: check the four pillars against the last
   completed session (and current price vs. it).
6. **Sentiment, logged not gating**: `NEWS_SENTIMENT` (Alpha Vantage,
   quota-limited — `limit: 5`). Report the score and whether it reads as
   a durable, still-positive story or an already-fading one. Don't
   invent a hard directional threshold — this repo's own
   `tool_verification.md` explicitly declines to gate on sentiment
   direction without a closed-trade record.

## Report format

```
## <TICKER> — Spike/Fade Check (<date>)

**A. Qualifies?** spike X% (Y% 3-session) · rel. volume Zx · catalyst:
<earnings beat/miss/inline | news | none> <N>d ago · retail chasing:
<Stocktwits volume surge | none | NO_COVERAGE> · turnover proxy X% ·
range position X% → <QUALIFIES / DOES NOT QUALIFY: reason>

**B. Overbought?** RSI(2) X · RSI(14) X · ATR-stretch X.Xx ·
ADX X (<climbing/flat/rolling over>) · [%B: X, best-effort]

**C. Turning?** Pillars present: <list — e.g. "pattern break, volume;
missing: candlestick, divergence">. <N>/4, pattern break <present/absent>
(required)

**D. Stand-down check:** <none triggered | reason to stand down>

**Read:** one of —
- OVERBOUGHT — FADE CANDIDATE (pattern break + ≥2 pillars, A qualifies, no D triggers)
- OVERBOUGHT — WEAK CONFIRMATION (pattern break only, <2 other pillars — probe size or skip)
- OVERBOUGHT — TOO EARLY (B extended, no pattern break yet — don't anticipate)
- OVERSOLD — BOUNCE CANDIDATE (RSI(2) < 15, price > SMA200, first up-close — matches `mr_reversal`)
- OVERSOLD — FALLING KNIFE (RSI(2) low, price < SMA200 — excluded by design, see `strategy.md`)
- NEUTRAL / NO SIGNAL
- DOES NOT QUALIFY (failed A — state which condition)

**E. If expressing this as a trade** (informational only — nothing
opened, sized, or logged): put vs. bear ETF trade-off, and probe-vs-full
sizing given the pillar count above.

**F. Exit plan if entered** (informational, not tracked): invalidation
level, ATR stop distance, RSI(2) target, time cap, options checkpoints.

**Evidence caveat:** repeat whenever the read is OVERBOUGHT — FADE
CANDIDATE or WEAK CONFIRMATION: this is a retail-overreaction, noise-
trading effect (not a "smart money" signal), weaker specifically on
earnings catalysts than on other news, and its academic edge is front-
loaded — a multi-day hold captures a diluted slice of it.
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
  routines, and is not itself a trigger in `strategy.md` or `gates.md`.
  Those stay exactly as they were before this skill existed.
