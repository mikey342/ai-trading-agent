# Premarket Notes — 2026-08-12

Universe: `run_scan` (scan_id de1b1994-b5db-472a-9b79-c052f1215193), 200 symbols returned.
Note: Stocktwits (`get_symbol_pulse`) was unavailable this session — the connector requires
an OAuth authorization that can't run in this headless scheduled context. Catalyst checks
below went straight from earnings to NEWS_SENTIMENT where earnings didn't explain the move.

## Notable movers (|gap| >= 2%)

- **NOK**: +10.17%, prior close $9.44 -> premarket $10.40 — catalyst: no obvious catalyst
  found. Checked NEWS_SENTIMENT (last 5 items, 8/8–8/11): coverage is mixed
  (bullish Tue, bearish Mon, neutral/somewhat-bullish before that) with nothing resembling
  a move of this size. Given Nokia's size and analyst coverage, this reads as a likely data
  quirk (e.g. a stale/thin premarket print) rather than a confirmed 10% gap — worth a sanity
  check against a second quote source before treating it as real.
- **DFTX**: +17.01%, prior close $40.91 -> premarket $47.87 — catalyst: no obvious catalyst
  found. No earnings this morning; Stocktwits unavailable; NEWS_SENTIMENT call hit the Alpha
  Vantage rate limit (concurrent-request burst) and returned no data — not rechecked, to
  preserve quota for the 9:30 scan. Large enough move to flag as highest priority for the
  scan run to verify independently.
- **EAT** (Brinker Intl.): +4.71%, prior close $221.38 -> premarket $231.80 — catalyst:
  reported Q4 earnings this morning (2026-08-12, AM), EPS $3.07 actual vs. $3.08 est.
  (essentially in-line). Move size relative to a flat EPS print suggests guidance/commentary
  drove the reaction; earnings release itself is the clear catalyst.
- **GFS** (GlobalFoundries): +4.80%, prior close $50.82 -> premarket $53.26 — catalyst: no
  obvious catalyst found. Last reported earnings 8/5 (beat), not today. NEWS_SENTIMENT call
  hit the Alpha Vantage rate limit — not rechecked, to preserve quota.
- **MOD** (Modine Manufacturing): +3.02%, prior close $198.32 -> premarket $204.30 —
  catalyst: no earnings today (last reported 7/29), but NEWS_SENTIMENT shows sustained
  bullish coverage through 8/9–8/11 of Modine's Q1 beat and "bold 2027 sales guidance,"
  plus continued institutional-stake writeups — plausible continuation of that storyline
  rather than a fresh overnight catalyst.
- **POWL** (Powell Industries): +3.18%, prior close $208.38 -> premarket $215.01 —
  catalyst: no obvious catalyst found. Last reported earnings 8/3, not today. NEWS_SENTIMENT
  call hit the Alpha Vantage rate limit — not rechecked, to preserve quota.
- **IMNM** (Immunome): -2.74%, prior close $26.25 -> premarket $25.53 — catalyst: reported
  Q2 earnings yesterday after the close (2026-08-11, PM), EPS -$0.65 actual vs. -$0.51 est.
  (wider loss than expected) — explains the gap down.
- **WHD** (Cactus Inc): -2.82%, prior close $72.04 -> premarket $70.01 — catalyst:
  NEWS_SENTIMENT shows COO Steven Bender sold ~$1.69M of WHD stock (reported 8/11,
  Somewhat-Bearish/Bearish coverage) plus other insider-transaction writeups this week —
  insider selling is the likely drag.
- **PCTY** (Paylocity): -2.69%, prior close $145.05 -> premarket $141.15 — catalyst: no
  obvious catalyst found. Last reported earnings 8/4 (beat), not today. NEWS_SENTIMENT call
  hit the Alpha Vantage rate limit — not rechecked, to preserve quota.
- **ADPT** (Adaptive Biotech): +2.21%, prior close $25.39 -> premarket $25.95 — catalyst
  check not run (gap close to the 2% threshold; Stocktwits unavailable, Alpha Vantage quota
  preserved for the scan run). No earnings today (last reported 7/29).
- **AXTA** (Axalta Coating): +2.06%, prior close $37.28 -> premarket $38.05 — catalyst
  check not run (same reasoning as ADPT). No earnings today (last reported 7/28).
- **LTH** (Life Time Group): +2.30%, prior close $43.81 -> premarket $44.82 — catalyst
  check not run (same reasoning). No earnings today (last reported 7/30).
- **PGEN** (Precigen): +2.04%, prior close $6.86 -> premarket $7.00 — catalyst check not
  run (same reasoning). No earnings today (last reported 8/4).

## Everything else

Flat / no notable premarket movement (187): GBTG, TECH, CDNA, OGN, FBP, PAY, ATAI, LIND,
ACAD, OFG, PAYO, PYPL, AWI, AMLX, BXMT, SAIA, CIB, LXP, LAD, PAG, GRMN, CORT, LNC, BMY, AVTR,
MAN, HQY, BFLY, MYRG, BAX, ATHM, KNSA, ORA, ELVN, ALLE, MEDP, PGNY, CHRW, SCI, TXG, CNK, FIVE,
TNET, TAL, SEIC, BZ, RUSHA, HOMB, MMM, ATR, LEGN, NEO, SCHW, ROL, ROKU, ADP, IDCC, DGX, ACA,
NTNX, DLB, APAM, EDU, BLKB, ITW, IMAX, PTGX, MANH, MTG, ITRI, ICUI, AMBP, ELF, CRUS, NAMS,
ADT, PRU, GEO, PARR, GGG, JD, MRSH, NTAP, THC, AWR, JLL, VNT, ICE, DPZ, LPLA, BHF, AMGN, RXO,
FBK, J, GEF, AIZ, IQV, URGN, PBF, RDN, ULTA, IONS, SF, FCF, RDY, TX, SBCF, MSA, ANF, RGEN,
WEX, CGON, ASH, VRDN, AMP, MLI, ALRM, AAON, EXPE, BDC, FA, GFL, MA, VERA, VSXY, BPOP, QTWO,
ATRC, NMIH, BHC, TXRH, BWXT, CTVA, BRO, ROST, BAC, FTNT, RAPP, WSFS, UHAL.B, AXS, TRV, MGNI,
SSNC, GPN, SLG, RGA, NATL, NI, THG, EMBJ, ALL, RVMD, WERN, EMR, RSI, CTSH, CVS, HURN, TOST,
CCK, CXW, ALHC, SHAK, CTAS, EIX, CRWD, SSB, NN, STGW, HPQ, CC, LSTR, UNIT, GOLF, TU, EFXT,
AMX, EXLS, TWST, RS, SKWD, FSLR, BABA, MTRN, CWT

Note: `run_scan` reported 396 total matches but returned the first 200 result rows — this
routine covered exactly what the scan handed back.
