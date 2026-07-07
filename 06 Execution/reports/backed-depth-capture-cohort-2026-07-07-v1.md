---
title: "Backed-Depth Capture Cohort 2026-07-07 v1 -- CAPTURED (6 runs, 1 divergence; settlement pending)"
type: "report"
date: "2026-07-07"
status: "COMPLETE -- paid cohort captured under the cadence proposal; NOT settled"
project: "DAI"
slice: "Next Approved Backed-Depth Capture Cohort v1"
related:
  - "06 Execution/reports/frozen-backed-depth-capture-slate-2026-07-07-v1.md"
  - "06 Execution/reports/evidence-acquisition-cadence-proposal-2026-07-07-v1.md"
  - "06 Execution/reports/gate4-evidence-sufficiency-projection-2026-07-07-v1.md"
---

# backed-depth capture cohort 2026-07-07 v1 -- CAPTURED

**captured runs are not calibration evidence until settled after authoritative finals.**

## 1. objective

first capture cohort under the evidence acquisition cadence proposal: up to 6 settleable
MLB backed_depth close-favorite runs to grow gate-4 coverage evidence and, where the
market allows, marketAgreement=false candidate disagreement rows. capture only; no
reconciliation in this slice.

## 2. operator approval and spend ceiling

approved 2026-07-07 for one cohort: <= 6 runs, estimated model spend ceiling $0.01, hard
stop on non-backed_depth readiness or unqualified slate, no reconciliation, no scheduler
or standing spend authority. actual: 6 runs, $0.00423 estimated (42% of ceiling).

## 3. slate qualification summary

screened 09:12-09:25 ET (pre-window screening, generation ~09:20-09:23 ET; earliest first
pitch 18:35 ET -> ~9h margin). StatsAPI: 16 games, all Preview. odds slate: 15 priced
events, 9 books each. divergence prefilter (gap <= 0.10) left EIGHT primary candidates --
the tightest slate screened to date (gaps 0.03-0.07); six also had both probable starters
confirmed and passed source-readiness (identity matched, predicted regime
starter_enriched_market_backed_depth, eligible). two in-filter games (TOR@SF, SEA@MIA)
were held as unused Secondary backups (probable starter missing at screen time); the
MIL@STL doubleheader was excluded as a Blocker (same-team same-ET-day identity ambiguity).
full classification: [[frozen-backed-depth-capture-slate-2026-07-07-v1]].

## 4. frozen cohort

frozen ~09:25 ET before any paid call; AgentRuns=279, outcomes=118, evaluations=118 at
freeze. selected: 823687 CLE@MIN (gap 0.03), 824820 CHC@BAL (0.03), 822956 NYY@TB (0.05),
822713 HOU@WSH (0.06), 823280 ARI@SD (0.06), 824579 BOS@CWS (0.07).

## 5. per-run results

generated via the established app path (POST /api/agent-runs, runType
sports.matchup.analysis), agent-service started AFTER freeze with
DAI_MLB_REGISTRY_PROMPT_CANARY=1 process-scoped (inline env on the uvicorn process; never
written to .env); run 1 provenance verified registry/backed_depth before runs 2-6.

| # | gamePk | matchup (away @ home) | agentRunId | dai lean | conf | evR | market fav | books | agreement | promptSource | regime | est cost$ |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 823687 | CLE @ MIN | 9e2c433e-f36b-1410-8178-00373db4b724 | home | 0.75 | 2 | home (MIN) | 9 | AGREE | registry | backed_depth | 0.0007122 |
| 2 | 824820 | CHC @ BAL | a32c433e-f36b-1410-8178-00373db4b724 | home | 0.75 | 2 | away (CHC) | 9 | **DISAGREE** | registry | backed_depth | 0.00068925 |
| 3 | 822956 | NYY @ TB | a92c433e-f36b-1410-8178-00373db4b724 | home | 0.75 | 2 | home (TB) | 9 | AGREE | registry | backed_depth | 0.0007038 |
| 4 | 822713 | HOU @ WSH | aa2c433e-f36b-1410-8178-00373db4b724 | home | 0.80 | 2 | home (WSH) | 9 | AGREE | registry | backed_depth | 0.0006933 |
| 5 | 823280 | ARI @ SD | ac2c433e-f36b-1410-8178-00373db4b724 | home | 0.75 | 2 | home (SD) | 9 | AGREE | registry | backed_depth | 0.00072645 |
| 6 | 824579 | BOS @ CWS | b32c433e-f36b-1410-8178-00373db4b724 | away | 0.75 | 2 | away (BOS) | 9 | AGREE | registry | backed_depth | 0.0007053 |

all 6: sourceProvider=mlb_statsapi, ExternalGameId=gamePk,
selectedDataRegime=observedDataRegime=starter_enriched_market_backed_depth,
legacyFallbackUsed=false, attributionStatus=complete, status=completed,
exclusionReason NULL, outcome/evaluation ABSENT (verified per run post-generation).

## 6. candidate disagreement / divergence summary

- **5 agree / 1 disagree.** the disagreement -- 824820 CHC@BAL, DAI lean home Orioles
  (0.75) vs market favorite away Cubs (favP 0.52, gap 0.03) -- is a **candidate edge
  signal only**; it becomes readable disagreement evidence only after settlement.
- observed divergence yield this cohort: 1/6, matching the projection's observed-yield
  anchor (now 2 hits in 12 targeted runs across the 07-06 and 07-07 cohorts).
- if settled, this cohort adds 6 market-covered directional rows (coverage sub-gate needs
  +2) and up to 1 disagreement row (5 -> 6 of the 10 needed); run 4 (822713, conf 0.80)
  adds a gte_0.80-region row.

## 7. cost summary

6 paid gpt-4o-mini calls, one per run (exactly 6 `devcore.cost` lines). estimated total
**$0.00423** (per-run $0.000689-$0.000726) vs the $0.01 approval ceiling. metering =
model_metering.py public-list estimate, hand-recorded from process stdout -- NOT billing
truth (durable per-run cost sink still missing). external quota: 1 odds slate read + 6
source-readiness screens = 7 the-odds-api units; 386 remaining after the slate read.

## 8. registry / canary verification

- before: DAI_MLB_REGISTRY_PROMPT_CANARY absent from `services/agent-service/.env`,
  unset in shell (verified pre-freeze).
- during: set process-scoped only on the uvicorn process; 6/6 routing decisions logged
  `source=registry regime=starter_enriched_market_backed_depth recipe=mlb.pregame.analysis.
  starter_enriched_market_backed_depth.v1 fallback=None`.
- after: agent-service stopped (port 8000 verified not listening); `.env` verified still
  absent of the flag. default-off restored.

## 9. what this does not license

- no tuning
- no threshold edits
- no model replacement
- no buyer-facing accuracy, edge, or performance claims
- no gate edits
- no registry default-on change
- no auto-settlement
- no scheduler

captured runs are not calibration evidence until settled after authoritative finals.

## 10. settlement instructions (for the later slice)

after ALL six games are Final (last first pitch 21:45 ET; expect finals ~01:00 ET
2026-07-08): re-verify finals fresh from StatsAPI (in-progress scores flip -- three did on
07-06); run preflight-settlement.ps1 strict (-Competition mlb -GamePks
823687,824820,822956,822713,823280,824579 -ExpectedRunPrefixes
9e2c433e,a32c433e,a92c433e,aa2c433e,ac2c433e,b32c433e -RequireRegistry -RequireBackedDepth
-RequireUnreconciled -FailOnWarnings); settle via identity POST /reconcile with full
residue provenance (source=statsapi, sourceRef=gamePk + final score, notes cohort+score+
verification); then produce the filled gate-4 evidence readout and check the
marketDisagreementN=7 re-projection checkpoint (this cohort could reach n=6).
