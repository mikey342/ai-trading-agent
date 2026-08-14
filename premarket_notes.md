# Premarket Notes — 2026-08-14

Universe: `run_scan` (scan_id de1b1994-b5db-472a-9b79-c052f1215193), 397 total matches,
first 200 result rows returned — this routine covered exactly what the scan handed back.

Note: Stocktwits (`get_symbol_pulse`) was unavailable this session — the connector still
requires an OAuth authorization that can't run in this headless scheduled context. Catalyst
checks below went earnings -> NEWS_SENTIMENT, skipping the Stocktwits step entirely.
Unusually high notable-mover count today (35 of 200, vs. a handful on recent days) —
23 down / 12 up, not a one-sided bloc, so this doesn't read as a single market-wide gap
event; treating each as its own name below. Given the count, NEWS_SENTIMENT (25/day quota,
shared with the 9:30am scan) was only run for the 5 largest-magnitude movers to leave quota
for the scan run; the other 30 are flagged "not checked" rather than "no catalyst" — that's
an honest gap in coverage, not a finding of no news.

None of the 35 notable movers below reported earnings today (checked via
`get_earnings_results` for all 35; nearest reports were 2026-08-10 to 2026-08-11 or later
this month — no same-day reports).

## Notable movers (|gap| >= 2%)

- **VERA** (Vera Therapeutics): +7.11%, prior close $28.89 -> premarket $30.95 — catalyst:
  checked via NEWS_SENTIMENT. No earnings today (last reported 2026-08-10 pm, big EPS miss:
  -1.52 actual vs -1.59 est. — but that's 4 days stale). Recent feed is dominated by CEO
  Marshall Fordyce insider stock sales (~$680k-$2M, dated 2026-08-13, routine 10b5-1 filings)
  — doesn't explain an upside gap this size. **No obvious catalyst found.**
- **TWST** (Twist Bioscience): -4.97%, prior close $125.29 -> premarket $119.06 — catalyst:
  not checked (NEWS_SENTIMENT hit Alpha Vantage per-second rate limit; not retried to
  preserve quota). No earnings today (last reported 2026-08-03). **Not checked.**
- **AMLX** (Amylyx Pharmaceuticals): -4.87%, prior close $22.48 -> premarket $21.385 —
  catalyst: checked via NEWS_SENTIMENT. No earnings today (last reported 2026-08-06 pm).
  Feed shows a repeated skeptical piece ("Is Amylyx Quietly Rewriting Its R&D Risk-Reward
  Profile As Losses Deepen Without Revenue?", recurring 2026-08-10 to 08-11) plus a
  "Moderate Buy" analyst consensus note (2026-08-13) — mixed, nothing dated today.
  **No obvious same-day catalyst found**, though the skeptical coverage may be weighing
  on sentiment.
- **CC** (Chemours): +4.13%, prior close $15.26 -> premarket $15.89 — catalyst: not checked
  (Alpha Vantage rate limit). No earnings today (last reported 2026-08-04 pm, in-line EPS).
  **Not checked.**
- **BFLY** (Butterfly Network): -3.89%, prior close $8.86 -> premarket $8.515 — catalyst:
  not checked (Alpha Vantage rate limit). No earnings today (last reported 2026-07-30).
  **Not checked.**
- **LFST** (LifeStance Health): +3.76%, prior close $12.22 -> premarket $12.68 — no earnings
  today (last reported 2026-08-06). **Not checked** (quota reserved for scan run).
- **PCTY** (Paylocity): -3.68%, prior close $152.71 -> premarket $147.095 — no earnings
  today (last reported 2026-08-04 pm, beat). **Not checked.**
- **CDNA** (CareDx): -3.50%, prior close $47.38 -> premarket $45.72 — no earnings today
  (last reported 2026-07-30 pm, beat). **Not checked.**
- **POWL** (Powell Industries): +3.46%, prior close $204.04 -> premarket $211.105 — no
  earnings today (last reported 2026-08-03 pm, slight miss). **Not checked.**
- **TRI** (Thomson Reuters): -3.35%, prior close $106.07 -> premarket $102.52 — no earnings
  today (last reported 2026-08-05 am, beat). **Not checked.**
- **PGEN** (Precigen): -3.24%, prior close $6.49 -> premarket $6.28 — no earnings today
  (last reported 2026-08-04 pm, beat). **Not checked.**
- **ALHC** (Alignment Healthcare): +3.16%, prior close $13.46 -> premarket $13.885 — no
  earnings today (last reported 2026-07-30 pm, beat). **Not checked.**
- **GFS** (GlobalFoundries): +3.15%, prior close $52.69 -> premarket $54.35 — no earnings
  today (last reported 2026-08-05 am, beat). **Not checked.**
- **ONC** (BeOne Medicines): -3.04%, prior close $356.01 -> premarket $345.17 — no earnings
  today (last reported 2026-08-05 am, big beat) — move is a pullback despite the beat.
  **Not checked.**
- **RSI** (Rush Street Interactive): +3.01%, prior close $24.61 -> premarket $25.35 — no
  earnings today (last reported 2026-07-29 pm, beat). **Not checked.**
- **CLDX** (Celldex Therapeutics): -3.00%, prior close $42.61 -> premarket $41.33 — no
  earnings today (last reported 2026-08-06 pm, beat). **Not checked.**
- **DFTX** (Definium Therapeutics): -2.98%, prior close $43.90 -> premarket $42.59 — no
  earnings today (last reported 2026-08-06 pm, miss). **Not checked.**
- **GPN** (Global Payments): -2.88%, prior close $94.83 -> premarket $92.10 — no earnings
  today (last reported 2026-08-05 am, roughly in-line). **Not checked.**
- **INSM** (Insmed): -2.83%, prior close $126.33 -> premarket $122.75 — no earnings today
  (last reported 2026-08-06 am, beat). **Not checked.**
- **TNET** (TriNet): -2.70%, prior close $70.11 -> premarket $68.22 — no earnings today
  (last reported 2026-07-30 am, big beat) — pullback despite beat. **Not checked.**
- **NN** (NN Inc): -2.61%, prior close $19.51 -> premarket $19.00 — no earnings today (last
  reported 2026-08-11 pm, miss — 3 days stale). **Not checked.**
- **MYRG** (MYR Group): +2.48%, prior close $327.28 -> premarket $335.40 — no earnings
  today (last reported 2026-07-29 pm, beat). **Not checked.**
- **QTWO** (Q2 Holdings): -2.45%, prior close $66.22 -> premarket $64.595 — no earnings
  today (last reported 2026-07-29 pm, roughly in-line). **Not checked.**
- **TXG** (10x Genomics): -2.39%, prior close $58.15 -> premarket $56.76 — no earnings
  today (last reported 2026-08-06 pm, miss). **Not checked.**
- **FTNT** (Fortinet): -2.32%, prior close $165.44 -> premarket $161.60 — no earnings today
  (last reported 2026-07-29 pm, beat). **Not checked.**
- **WHD** (Cactus Inc): +2.30%, prior close $72.04 -> premarket $73.70 — no earnings today
  (last reported 2026-07-29 pm, beat). **Not checked.**
- **KLAC** (KLA Corp): -2.29%, prior close $209.37 -> premarket $204.575 — no earnings
  today (last reported 2026-07-28 pm, beat). **Not checked.**
- **HPQ** (HP Inc): -2.28%, prior close $31.30 -> premarket $30.585 — no earnings today
  (report scheduled 2026-08-26, not yet happened). **Not checked.**
- **EMBJ**: +2.22%, prior close $74.22 -> premarket $75.87 — no earnings today (last
  reported 2026-08-10 am, big beat — 4 days stale). **Not checked.**
- **PANW** (Palo Alto Networks): -2.21%, prior close $396.00 -> premarket $387.255 — no
  earnings today (report scheduled 2026-09-01, not yet happened). **Not checked.**
- **CRWD** (CrowdStrike): -2.20%, prior close $225.53 -> premarket $220.56 — no earnings
  today (report scheduled 2026-08-26, not yet happened). **Not checked.**
- **TDW** (Tidewater): +2.18%, prior close $91.52 -> premarket $93.515 — no earnings today
  (last reported 2026-08-03 pm, beat). **Not checked.**
- **ZETA** (Zeta Global): -2.13%, prior close $29.62 -> premarket $28.99 — no earnings
  today (last reported 2026-08-04 pm, miss). **Not checked.**
- **NHI** (National Health Investors): +2.03%, prior close $71.27 -> premarket $72.72 — no
  earnings today (last reported 2026-08-10 pm, roughly in-line). **Not checked.**
- **NEO** (NeoGenomics): -2.01%, prior close $16.39 -> premarket $16.06 — no earnings today
  (last reported 2026-07-28 pm, beat). **Not checked.** Borderline, smallest gap on the list.

## Everything else

Flat / no notable premarket movement (165 of 200): AAON, ABT, ACA, ACAD, ADP, ADT, AIZ,
ALLE, ALRM, AMBP, AMGN, AMP, AMX, ANF, APAM, ASH, ATAI, ATHM, ATKR, ATR, ATRC, AVTR, AVY,
AWI, AWR, AXS, AXTA, BAC, BAX, BDC, BDX, BHC, BHF, BKNG, BLKB, BMY, BPOP, BRO, BWIN, BXMT,
BZ, CAKE, CCK, CGON, CHRW, CIB, CNK, CORT, CRUS, CTSH, CTVA, CVS, DD, DGX, DNOW, DPZ, DVA,
EAT, EDU, ELF, ELVN, EMR, EQH, EXLS, EXPE, FBK, FBP, FCF, FIVE, FLS, GBTG, GEF, GEO, GFL,
GGG, GOLF, HNI, HOMB, HQY, HURN, ICE, ICUI, IDCC, IMAX, IMNM, IONS, IQV, ITRI, ITW, J, JD,
JLL, JXN, KNSA, LAD, LEGN, LIND, LNC, LPLA, LTH, LXP, MA, MAN, MANH, MEDP, MGNI, MIRM, MLI,
MMM, MRSH, MSA, MSFT, MTG, NAMS, NI, NMIH, NWL, OFG, ORA, PAG, PARR, PAY, PAYO, PBF, PRU,
PTC, PTGX, PYPL, RAPP, RDN, RDY, RGA, RGEN, RITM, ROKU (closest to notable at +1.98%), ROL,
RS, RTX, RUSHA, RVMD, RXO, SAIA, SAP, SBCF, SCHW, SCI, SF, SHAK, SLG, SNEX, SSB, SSNC, STGW,
TAL, TECH, THC, TX, TXRH, ULTA, URGN, VNT, VRDN, WSO, WTW, YUMC
