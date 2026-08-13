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

The indicator logic and evidence standing below are the same as the
`mr_spike_fade` (short/fade) and `mr_reversal` (long/bounce) signals
documented in this repo's `strategy.md`, and the reasoning for *why* those
thresholds are what they are — including the PEAD-reconciliation that
keeps a fade from just fighting post-earnings drift — lives in
`framework.md` item 6 and the mean-reversion sections of `strategy.md`.
**Read those before changing anything here; don't re-derive the reasoning
from scratch.** This skill applies that logic on demand; it doesn't
redefine it.

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
     *down* from the one before? (`mr_spike_fade`'s first down-close.)
   - **Oversold case:** did the current price close *up* from the most
     recently completed session? (`mr_reversal`'s first up-close.)
   - If neither has happened yet, say so explicitly and do not soften it
     into a signal — see the read categories below. Anticipating a
     reversal that the tape hasn't shown yet is exactly the PEAD-
     contradicting mistake `framework.md` item 6 warns about.

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
  down-close is already in — matches `mr_spike_fade`.
- OVERBOUGHT — TOO EARLY: extended, RSI(2) high, but no down-close yet.
  PEAD says the average case here keeps drifting; this is not yet the
  narrower pattern that clears `mr_spike_fade`. Don't anticipate it.
- OVERSOLD — BOUNCE CANDIDATE: RSI(2) < 15, price > SMA200, first
  up-close present — matches `mr_reversal`.
- OVERSOLD — FALLING KNIFE: RSI(2) low but price < SMA200. Deliberately
  excluded — see `strategy.md`'s "the 200-day condition is load-bearing"
  note. Buying this is the failure mode, not the opportunity.
- NEUTRAL / NO SIGNAL.

**If expressing this as a trade** (informational only — nothing is
opened, sized, or logged): a fade is a long put (equity shorting is
forbidden system-wide, not just in the scheduled routines); a bounce is
equity or a call. Reference points only, not enforced here: 30+ DTE,
delta 0.40-0.60 absolute, open interest ≥500, spread ≤10% of mid,
underlying stop ~1.5x ATR14 against entry.

**Evidence standing:** point back to `framework.md` item 6 (Tier A/B/C
breakdown) rather than restating it. If the read is OVERBOUGHT — FADE
CANDIDATE, repeat the one-line reminder: this only clears the bar because
the tape already shows a down-close, not because a jump happened.
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
  routines. Those stay exactly as they are.
