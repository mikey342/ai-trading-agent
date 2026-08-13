---
name: spike-fade
description: On-demand technical read for a single stock the user names — is it overbought/over-extended after a spike (possible short-term fade, expressed via puts) or oversold within an uptrend (possible bounce)? Use when the user gives a ticker and asks whether it's overbought, oversold, extended, "due for a pullback," or worth fading after a jump/gap/earnings pop. Read-only research: never opens, sizes, or logs a trade, and is separate from the scheduled trading-agent routines.
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
underreacts to the surprise. Separately, live research (2026-08-13)
found: earnings-driven gap-ups hold their gains ~71% of the time within 5
days, and gaps that print on confirming volume fill less than 30% of the
time within a month. Multiple independent trading-education sources
describe blindly fading an earnings gap as one of the more reliable ways
to lose money on a gap trade.

So "it jumped on earnings → it's overbought → short it" would contradict
that evidence, not apply it. The fix used here: **only treat it as a fade
candidate once the tape has already started reversing** — a first
down-close off the spike — rather than anticipating that it will. That
changes the claim from "this pop will reverse" (fights PEAD) to "this
pop has already cracked, and short-horizon reversal (Jegadeesh 1990,
Lehmann 1990 — extreme short-horizon winners underperform over the
following 1-4 weeks, the same effect `mr_reversal` already trades on the
oversold side) says that tends to continue a few more days." The
overbought threshold itself (`RSI(2) ≥ 90`) is the ConnorsRSI convention
(Connors & Alvarez) — a practitioner heuristic, not a peer-reviewed
result, same evidence tier as the `<15` oversold threshold already in
use. The exhaustion-gap/blow-off-top framing (volume climax, price
stretched several ATRs from its short moving average) is classic
technical-analysis pattern recognition — weaker still, used here only as
descriptive context, never as a gate on its own.

**No sub-daily version of this exists on purpose.** This trading agent
holds 3-10 days and runs three fixed times a day; nothing here is
designed for intraday timeframes (see `framework.md`). A same-day
"lost VWAP by 11am" fade is a day-trading rule and doesn't fit this
system's data or cadence — don't add one.

## Steps

1. **Get oriented.** Pull a live quote with official prior close
   (`get_equity_quotes`), fundamentals for sector/52-week range
   (`get_equity_fundamentals`), and check for a recent or upcoming
   earnings event (`get_earnings_results` for trailing surprise history,
   `get_earnings_calendar` for an upcoming date). If nothing else
   identifies a catalyst, ask the user what triggered the move, or note
   plainly that no catalyst was found.

2. **Quantify the move.**
   - Last completed session's return vs. the one before, and the
     3-session cumulative return.
   - Relative volume on the spike session: last completed session's
     volume ÷ 30-day average, from `get_equity_fundamentals` — never a
     scan column (this skill doesn't use the scan, but if a value ever
     comes from `run_scan` output, note the `session="all"` vs.
     regular-session discrepancy documented in `tool_verification.md`
     before trusting it).

3. **Measure extension, both directions, from one data pull.**
   Use `get_equity_technical_indicators`:
   - `RSI(2)` and `RSI(14)` (`output: "last:5"` or similar — never the
     unbounded Alpha Vantage endpoint, see `tool_verification.md`).
   - `ATR14`, `SMA20`, `SMA50`, `SMA200`, `EMA8`, `EMA21`.
   - `ADX14` (direct call — the scan's `session="all"` ADX has been
     measured up to 12 points off the direct regular-session value).
   - `MACD` via Robinhood's own `type=macd` (Alpha Vantage's `MACD` is
     permanently premium-gated on this account — never call it).
   - Compute ATR-stretch: `(price - SMA20) / ATR14`. A stock several ATRs
     above its own 20-day average is stretched by construction.
   - 52-week range position: `(price - low52) / (high52 - low52)`, from
     the fundamentals call already made in step 1.
   - Optional, best-effort only: Alpha Vantage `BBANDS` for %B. This tool
     has **not** been verified working in this repo's own
     `tool_verification.md` — try it, but if it errors or the quota is
     exhausted, drop it from the report rather than blocking on it. Never
     treat an unverified indicator as load-bearing.

4. **Check the load-bearing thing: has it already turned?** This is what
   separates this analysis from just noticing a stock is extended:
   - **Overbought case:** did the most recently completed session close
     *down* from the one before? (First down-close — see "why the fade
     side is narrower" above.)
   - **Oversold case:** did the current price close *up* from the most
     recently completed session? (`mr_reversal`'s first up-close.)
   - If neither has happened yet, say so explicitly and do not soften it
     into a signal — see the read categories below. Anticipating a
     reversal that the tape hasn't shown yet is exactly the
     PEAD-contradicting mistake described above.

5. **Sentiment, logged not gating.** `NEWS_SENTIMENT` (Alpha Vantage,
   quota-limited — use sparingly, `limit: 5`). Report the score and
   whether it reads as a durable beat (still-positive, still-drifting
   story) vs. an already-fading one. Don't invent a hard threshold here;
   this repo's own `tool_verification.md` explicitly declines to gate on
   sentiment direction without a closed-trade record.

## Report format

Answer directly in the chat, using this shape:

```
## <TICKER> — Spike/Fade Check (<date>)

**The move:** last session X%, 3-session Y%, relative volume Zx.
**Catalyst:** <earnings beat/miss/inline | news | none identified>, <N> days ago.

**Extension:**
RSI(2) X · RSI(14) X · ATR-stretch from SMA20: X.Xx ATR
52-week range position: X% · ADX(14): X · MACD: <rising/falling/crossing>
[Bollinger %B: X (unverified tool, best-effort)]

**Already turning?** <most recent session was an up-close / down-close vs. prior>

**Read:** one of —
- OVERBOUGHT — FADE CANDIDATE: spike confirmed, RSI(2) ≥ 90, and a first
  down-close is already in.
- OVERBOUGHT — TOO EARLY: extended, RSI(2) high, but no down-close yet.
  PEAD says the average case here keeps drifting — don't anticipate it.
- OVERSOLD — BOUNCE CANDIDATE: RSI(2) < 15, price > SMA200, first
  up-close present — matches `mr_reversal`, the trading agent's live
  long-side signal.
- OVERSOLD — FALLING KNIFE: RSI(2) low but price < SMA200. Buying this
  is the failure mode this system deliberately excludes, not a signal —
  see `strategy.md`'s "the 200-day condition is load-bearing" note.
- NEUTRAL / NO SIGNAL.

**If expressing this as a trade** (informational only — nothing is
opened, sized, or logged): a fade is a long put (equity shorting is
forbidden system-wide, not just in the scheduled routines); a bounce is
equity or a call. Reference points only, not enforced here: 30+ DTE,
delta 0.40-0.60 absolute, open interest ≥500, spread ≤10% of mid,
underlying stop ~1.5x ATR14 against entry.

**Evidence caveat:** overbought reads are weaker-evidenced than oversold
reads (practitioner convention + pattern recognition vs. `mr_reversal`'s
established basis) and only apply once a down-close is already in —
repeat that reminder whenever the read is OVERBOUGHT — FADE CANDIDATE.
```

## What this skill deliberately does not do

- Does not open, size, or place any position — paper or real.
- Does not write to `positions.md`, `trade_log.csv`, or
  `trade_journal.md`. If the user wants a lookup kept for the record,
  ask before writing anywhere.
- Does not check `gates.md`'s position/sector/counter-trend caps — there
  is nothing being opened for those caps to apply to.
- Does not run on a schedule and is not part of `run_instructions.md`,
  `position_monitor_instructions.md`, or the premarket/daily-report
  routines, and is not itself a trigger in `strategy.md` or `gates.md`.
  Those stay exactly as they were before this skill existed.
