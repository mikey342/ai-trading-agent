# Trade Journal

Append-only log. Never delete or rewrite past entries — if a past decision
turned out wrong, say so in a new entry, don't erase the record.

Each run should add one entry using the template below.

---

## Template

### YYYY-MM-DD HH:MM UTC — [premarket|close] run

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

_(No runs yet. First entry will be added by the first scheduled run.)_
