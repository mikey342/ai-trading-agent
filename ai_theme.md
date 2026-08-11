# AI Theme List

Used **only to tilt Tier 0 ranking**, never as a filter. Names here get a
1.15× multiplier on their ranking score, so AI exposure rises without the
universe narrowing. A name **not** on this list can still rank top-15
purely on merit — nothing here excludes anything.

**Why tilt rather than filter.** The scan already surfaces AI names when
they trend. Filtering to AI only would trade that adaptiveness for
concentration risk — six correlated names gap through their stops
together, making one bet sized as if it were six.

**Failure mode is deliberately safe.** If this list goes stale, the tilt
weakens and nothing breaks — unlike the old hardcoded watchlist, which
*was* the universe and silently capped what could ever be found.

**Review quarterly.** Membership is a judgment call about what counts as
"AI-related," and the answer drifts as the theme evolves.

## Parsing rules — read before editing

Tickers live **inside the fenced block below, one per line**, and nothing
else in this file is parsed. This format was chosen after a wrapped
comma-separated list silently dropped roughly half the tickers: lines
that ended in a comma failed to match, so real AI names (GFS among them)
were scored as non-AI. One ticker per line has no such failure mode.

Blank lines and `#` comments inside the block are ignored.

```tickers
# semiconductors & chips
NVDA
AMD
AVGO
MU
TSM
ASML
AMAT
LRCX
KLAC
MRVL
ARM
QCOM
INTC
GFS
ON
MPWR
ALAB
CRDO
SNPS
CDNS
TER
ENTG
AXTI
RMBS
KLIC
UCTT
ACLS

# AI software & applications
PLTR
MSFT
GOOGL
META
CRM
NOW
SNOW
AI
PATH
DDOG
MDB
ESTC
TEAM
ADBE
INOD
BBAI

# networking & photonics
ANET
CIEN
LITE
COHR
AAOI
POET
VIAV
FN
APH
GLW
NTAP

# data center, compute & infrastructure
SMCI
DELL
VRT
EQIX
DLR
APLD
WULF
IREN
CORZ
NBIS
CRWV
HPE
PSTG
WDC
STX

# power & energy for AI
CEG
VST
TLN
NRG
OKLO
SMR
GEV
BWXT
LEU
PWR

# cybersecurity (AI-adjacent)
PANW
CRWD
FTNT
S
NET
ZS
OKTA
CYBR

# quantum & frontier compute
IONQ
RGTI
QBTS
QUBT

# critical materials & supply chain
MP
USAR
ALB
```
