# Premarket Notes — 2026-08-13

Universe: `run_scan` (scan_id de1b1994-b5db-472a-9b79-c052f1215193), 396 total matches,
first 200 result rows returned — this routine covered exactly what the scan handed back.
Note: Stocktwits (`get_symbol_pulse`) was unavailable this session — the connector still
requires an OAuth authorization that can't run in this headless scheduled context (same as
2026-08-12). Catalyst checks below went straight from earnings to NEWS_SENTIMENT where
earnings didn't explain the move.

## Notable movers (|gap| >= 2%)

- **HPQ**: +7.43%, prior close $29.28 -> premarket $31.46 — catalyst: no earnings today
  (last reported 2026-05-27, next 2026-08-26). NEWS_SENTIMENT shows same-day coverage
  ("Dell Technologies (DELL), HP Inc. (HPQ) Strength May be Tied to Lenovo Results",
  Somewhat-Bullish, 2026-08-13 07:19) plus a related NTAP-feed headline calling out an
  "AI server boom" reaction to Lenovo/SMCI earnings lifting PC/server hardware names
  broadly. Reads as sector-wide spillover, not an HPQ-specific event — largest mover of
  the morning, worth a sanity check against a second source given the size (7%+).
- **AMBP** (Ardagh Metal Packaging): +5.24%, prior close $5.06 -> premarket $5.33 —
  catalyst: same-day NEWS_SENTIMENT headline "Potential sale of Ardagh Metal Packaging
  (AMBP) explored by 76% owner" (2026-08-13 09:43, Neutral-labeled but M&A-relevant) plus
  an SEC Schedule 13D amendment filing by Ardagh Holdings same day. Clear M&A-driven
  catalyst. No earnings today (last reported 2026-07-23).
- **FIVE** (Five Below): +3.75%, prior close $238.15 -> premarket $247.07 — catalyst: no
  obvious catalyst found. No earnings today (next 2026-08-26). NEWS_SENTIMENT shows only
  general Somewhat-Bullish coverage (analyst notes, store-opening/product pieces) dated
  8/9–8/12, nothing dated today and nothing that explains a move of this size.
- **DFTX** (Definium Therapeutics): +3.30%, prior close $42.79 -> premarket $44.20 —
  catalyst: positive Phase 3 "Voyage" topline data for DT120 (GAD/anxiety), reported
  2026-08-12 — cut HAM-A score by 5.4 pts vs. placebo, called "unprecedented efficacy."
  Today's move looks like continued follow-through on that news rather than a fresh
  overnight catalyst. No earnings today (last reported 2026-08-06).
- **MANH** (Manhattan Associates): +3.05%, prior close $192.63 -> premarket $198.50 —
  catalyst: no obvious catalyst found. No earnings today (last reported 2026-07-28, strong
  beat). NEWS_SENTIMENT feed this week is dominated by CEO insider-selling coverage
  (~$593k, mixed Neutral/Somewhat-Bearish/Somewhat-Bullish depending on outlet) — doesn't
  explain an upside gap, and nothing dated today.
- **NTAP** (NetApp): +2.64%, prior close $202.05 -> premarket $207.39 — catalyst: same-day
  NEWS_SENTIMENT headline "HCLTech expands NetApp partnership to launch consumption-based
  storage offering for enterprise AI" (2026-08-13 06:09, Bullish), plus spillover from the
  same Lenovo/SMCI-driven "AI server boom" sentiment lifting hardware/storage names this
  morning (see HPQ above). Partially offset by an EVP insider sale reported yesterday
  (Somewhat-Bearish). No earnings today (next 2026-09-02).
- **VRDN** (Viridian Therapeutics): -2.14%, prior close $22.88 -> premarket $22.39 —
  catalyst: continued fallout from post-earnings analyst estimate cuts — multiple
  NEWS_SENTIMENT items 8/9–8/13 note consensus revenue forecasts being cut (~22%) since
  its last report (2026-08-06), with "consensus forecasts have become a little darker"
  coverage recurring through the week. No earnings today.

## Everything else

Flat / no notable premarket movement (193): GBTG, TECH, PLSE, CDNA, OGN, PAY, FBP, ATAI,
LIND, ACAD, AMLX, CIB, AWI, WTW, PYPL, GRMN, LXP, KLAC, OFG, SNEX, SAIA, BXMT, PAG, LAD,
CORT, BFLY, AVTR, BMY, ALLE, PAYO, HQY, MYRG, KNSA, BAX, MAN, CHRW, TXG, MEDP, ATHM, ELVN,
LNC, SEIC, SCI, SCHW, CNK, ORA, ROL, HNI, ROKU, MMM, TAL, NEO, BZ, ATR, HOMB, TNET, PTGX,
DLB, ADP, RTX, DGX, ACA, RUSHA, APAM, ITRI, BHF, ELF, LEGN, CRUS, BLKB, ITW, MTG, IMAX, J,
PRU, THC, AMGN, ICUI, QLYS, TX, NAMS, RGEN, URGN, DVA, ICE, JLL, MRSH, SF, EDU, GGG, ASH,
CRWD, EXPE, ULTA, DD, FBK, VNT, AWR, IQV, ANF, PCTY, DPZ, SBCF, CGON, GEF, RXO, AMP, RAPP,
ADT, PBF, GEO, AAON, TIMB, VERA, GHRS, MSA, GFL, EMR, RDY, POWL, FCF, IONS, WEX, PARR, NI,
LTH, TXRH, BAC, GFS, MSFT, MGNI, AIZ, SHAK, AXTA, CTVA, NMIH, BPOP, JD, MLI, BHC, PGEN, AXS,
RDN, ALRM, CVS, SLG, QTWO, GPN, EMBJ, FTNT, SSNC, TWST, RGA, RSI, MA, RNG, BRO, ALHC, RVMD,
NN, NATL, SAP, FA, ROST, INSM, RS, SSB, BWXT, UNIT, CTSH, VIV, EXLS, WSFS, MOD, PANW, WSO,
OWL, CC, CCK, WERN, HURN, TOST, RITM, MIRM, CXW, PCOR, JXN, MTRN, EIX, NXPI, TRI, ABT, MPC
