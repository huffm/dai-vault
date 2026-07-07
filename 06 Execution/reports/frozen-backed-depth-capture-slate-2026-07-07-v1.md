---
title: "Backed-Depth Capture Cohort Candidate Slate 2026-07-07 v1 (FROZEN)"
type: "report"
date: "2026-07-07"
status: "FROZEN -- slate frozen before any paid generation"
project: "DAI"
slice: "Next Approved Backed-Depth Capture Cohort v1"
related:
  - "06 Execution/reports/evidence-acquisition-cadence-proposal-2026-07-07-v1.md"
  - "06 Execution/patterns/frozen-cohort-slate-template-v1.md"
  - "06 Execution/patterns/cohort-selection-and-run-discipline-v1.md"
---

# backed-depth capture cohort candidate slate 2026-07-07 v1 (FROZEN)

## freeze statement

Frozen at 2026-07-07 ~09:25 ET (~13:25 UTC). No paid model call was made, and
agent-service was not started for generation, before this freeze. Screening used only
read-only sources: StatsAPI schedule (free), one the-odds-api h2h slate read, and six
GET /api/agent-runs/source-readiness screens (no model, no db write).
AgentRuns count at freeze = 279. Outcomes = 118, Evaluations = 118.
After this freeze: no game is added because early outputs agree with market, no generated
game is removed because it is inconvenient, and all failures are recorded explicitly.

## measurement objective

grow gate-4 evidence under the cadence proposal: settleable backed_depth rows that are
market-covered (coverage sub-gate) with close-favorite games where marketAgreement=false
divergence is plausible (disagreement sub-gate). candidate edge signal capture only.

## operator approval

approved 2026-07-07 for ONE MLB backed_depth capture cohort: up to 6 runs, estimated model
spend ceiling $0.01, hard stop if source-readiness cannot reach backed_depth or slate does
not qualify, no reconciliation in this slice, no scheduler or standing spend authority.

## caps (binding)

- MAX_PAID_RUNS = 6
- ESTIMATED_MODEL_SPEND_CEILING_USD = 0.01
- MAX_MODEL_CALLS_PER_RUN = 1
- MODEL_EXPECTED = gpt-4o-mini
- SETTLEMENT_IN_THIS_SLICE = false

## run window

cadence window 10:00-13:00 ET; screened 09:12-09:25 ET (parity with the validated 07-06
capture, screened 08:37 ET). earliest selected first pitch 18:35 ET -> ~9h margin.
(the 14:15 ET doubleheader game 1 was excluded, so the margin is uniform and wide.)

## sport / competition

MLB, competition code `mlb`, GAME_DATE = 2026-07-07. sourceProvider expected on every run:
`mlb_statsapi`, ExternalGameId = StatsAPI gamePk.

## candidate universe

StatsAPI schedule 2026-07-07 (sportId=1, hydrate=probablePitcher): 16 games, ALL
abstractGameState=Preview at screen time, incl. one MIL@STL doubleheader (823062 game 1
14:15 ET, 823035 game 2 19:45 ET). the-odds-api returned 15 priced h2h events in the ET
date window (game 2 of the doubleheader not separately priced at screen time).

## screening timestamp(s)

- statsapi schedule read: 2026-07-07 ~09:14 ET (free)
- the-odds-api h2h slate read: ~09:16 ET, 1 unit; quota after read = 386 remaining
- source-readiness screens: ~09:20 ET, 6 candidates x 1 unit each
- registry canary state at screen time: absent from `.env`, unset in shell (default-off)

## screening method

1. schedule -> pre-game only (16/16 Preview) + probable-starter presence.
2. one odds slate read -> per-book de-vig -> median implied probs -> favP + gap;
   divergence prefilter gap <= 0.10 primary, 0.10-0.15 secondary, > 0.15 exclude.
3. source-readiness per surviving candidate -> identity / predicted regime / eligibility.

## candidate classification (all games considered)

| gamePk | matchup (away @ home) | start ET | pre-game | probables | books | fav | favP | gap | identity | predicted regime | class | decision | reason |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 823687 | CLE @ MIN | 19:40 | yes | both | 9 | home | 0.52 | 0.03 | matched | backed_depth | Primary | **SELECT** | close favorite, clean starters, eligible |
| 824820 | CHC @ BAL | 18:35 | yes | both | 9 | away | 0.52 | 0.03 | matched | backed_depth | Primary | **SELECT** | close favorite, clean starters, eligible |
| 822956 | NYY @ TB | 18:40 | yes | both | 9 | home | 0.52 | 0.05 | matched | backed_depth | Primary | **SELECT** | close favorite, clean starters, eligible |
| 822713 | HOU @ WSH | 18:45 | yes | both | 9 | home | 0.53 | 0.06 | matched | backed_depth | Primary | **SELECT** | close favorite, clean starters, eligible |
| 823280 | ARI @ SD | 21:40 | yes | both | 9 | home | 0.53 | 0.06 | matched | backed_depth | Primary | **SELECT** | close favorite, clean starters, eligible |
| 824579 | BOS @ CWS | 19:40 | yes | both | 9 | away | 0.53 | 0.07 | matched | backed_depth | Primary | **SELECT** | close favorite, clean starters, eligible |
| 823203 | TOR @ SF | 21:45 | yes | away SP missing | 9 | away | 0.52 | 0.03 | not screened | -- | Secondary | backup only | in-filter gap but probable starter missing at screen time (regime risk); 6 clean primaries available |
| 823847 | SEA @ MIA | 18:40 | yes | away SP missing | 9 | home | 0.52 | 0.04 | not screened | -- | Secondary | backup only | same starter-missing risk |
| 823607 | KC @ NYM | 19:10 | yes | home SP missing | 9 | home | 0.58 | 0.15 | not screened | -- | Exclude | no | gap at secondary boundary AND starter missing |
| 823361 | ATL @ PIT | 18:40 | yes | both | 9 | home | 0.60 | 0.20 | not screened | -- | Exclude | no | gap > 0.15 (clear favorite) |
| 822881 | LAA @ TEX | 20:05 | yes | both | 9 | home | 0.60 | 0.20 | not screened | -- | Exclude | no | gap > 0.15 |
| 824495 | PHI @ CIN | 19:10 | yes | away SP missing | 9 | away | 0.61 | 0.22 | not screened | -- | Exclude | no | gap > 0.15 |
| 824254 | ATH @ DET | 18:40 | yes | both | 9 | home | 0.62 | 0.25 | not screened | -- | Exclude | no | gap > 0.15 (overwhelming favorite) |
| 823062 | MIL @ STL game 1 | 14:15 | yes | home SP missing | 8 | away | 0.63 | 0.27 | not screened | -- | Blocker | no | doubleheader: two same-team events same ET day -> name-based odds/identity matching ambiguous; also gap > 0.15 |
| 823035 | MIL @ STL game 2 | 19:45 | yes | none listed | -- | -- | -- | not priced | not screened | -- | Blocker | no | doubleheader identity ambiguity; no probables; not separately priced |
| 823929 | COL @ LAD | 22:10 | yes | both | 9 | home | 0.70 | 0.41 | not screened | -- | Exclude | no | gap > 0.15 (overwhelming favorite) |

## selected cohort (6)

823687 CLE@MIN, 824820 CHC@BAL, 822956 NYY@TB, 822713 HOU@WSH, 823280 ARI@SD,
824579 BOS@CWS. all six: gap 0.03-0.07 (tightest six-pack of close favorites yet
screened), 9 books, identity matched, predicted regime
starter_enriched_market_backed_depth, generation-eligible, 0 existing active runs per
gamePk. each contributes a market-covered settled row once settled; close gaps maximize
the chance of marketAgreement=false capture. NOTE: NYY@TB / HOU@WSH / ARI@SD are the same
series as the settled 07-06 cohort but are DIFFERENT games (new gamePks) -- no duplicate
identity.

## exclusions

- gap > 0.15 (overwhelming/clear favorites, low divergence information): 823361, 822881,
  824495, 824254, 823929.
- doubleheader identity ambiguity (never generate against a Blocker): 823062, 823035.
- starter-missing at screen time (regime risk; enough clean primaries existed): 823203,
  823847 (Secondary backups, unused), 823607 (also boundary gap).

## stop conditions (verbatim from the slice prompt)

stop immediately if: paid-call count would exceed 6; estimated model spend would exceed
$0.01; source regime drops below backed_depth unexpectedly; registry/canary state becomes
unsafe; generation fails in a way that risks duplicate or invalid runs.

## pre-generation baselines (for post-slice validation)

- AgentRuns = 279
- AgentRunOutcomes = 118
- AgentRunEvaluations = 118
- existing active runs for selected gamePks = 0 for each of the six
- the-odds-api quota remaining ~380 after screening (386 after slate read - 6 readiness)
- registry canary: absent from `.env`, shell unset (verified)
- estimated cohort cost: 6 x ~$0.00071 = ~$0.0043 (< $0.01 ceiling)

## generation results (filled AFTER capture, never edited before)

| # | gamePk | agentRunId | lean | conf | market favorite | agreement | promptSource | regime | cost$ |
|---|---|---|---|---|---|---|---|---|---|
| (to be filled by the capture report) |

## settlement plan

settlement is a separate later slice: after ALL six games are Final (last first pitch
21:45 ET -> finals ~01:00 ET 07-08), run preflight-settlement.ps1 strict, then identity
POST /reconcile per the residue contract, then the filled gate-4 evidence readout.
