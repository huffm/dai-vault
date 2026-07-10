---
title: "V2 Day-1 Cohort Settlement -- 8/8 settled (2026-07-10)"
type: "reconciliation"
date: "2026-07-10"
status: "complete"
project: "DAI"
slice: "V2 Day-1 Settlement and Day-2 Capture"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - settlement
  - reconciliation
  - calibration
related:
  - "06 Execution/plans/v2-day1-settlement-day2-capture-slice-2026-07-10-v1.md"
  - "06 Execution/reports/v2-accelerated-capture-day1-2026-07-09-v1.md"
  - "06 Execution/reconciliations/canonical-reconciliation-residue-contract-v1.md"
  - "06 Execution/patterns/settlement-readiness-discipline-v1.md"
  - "06 Execution/reports/gate4-evidence-readout-v2-day1-2026-07-10-v1.md"
---

# v2 day-1 cohort settlement -- 8/8

Settlement of the 2026-07-09 v2 accelerated-capture day-1 cohort, blocked the prior night at
the finals gate (PARTIAL 5/3) and resumed 2026-07-10. Zero paid calls. Zero code change.

## 1. gates, in order

| gate | result |
|---|---|
| `check-settlement-finals.ps1 -Competition mlb -CheckLocalRows -RequireUnreconciled -FailOnPartial` | **READY 8/8**, exit 0 |
| `preflight-settlement.ps1 -RequireRegistry -RequireBackedDepth -RequireUnreconciled -FailOnWarnings` | exit 0 — target 8 / found 8 / ready 8 / **warnings 0 / blockers 0** |
| independent verification (`statsapi feed/live`) | 8/8 `abstractGameState=Final` AND `codedGameState=F`; 8/8 provider identity matches persisted home/away refs |
| precheck | `SingleMatch` on all 8 |

The guard's scores were treated as a readiness signal only. Each final score and each
home/away assignment was re-read from `feed/live` immediately before its write, and the
winner derived deterministically from the linescore. No score drifted between check and
write this time (a prior cohort had three games flip — the re-verify rule stays load-bearing).

## 2. writes

Eight identity `POST /api/agent-runs/reconcile` calls, all `SingleMatch`, each returning an
`evaluatedRunId` equal to the intended run. Residue complete on every write per the canonical
contract: `source=statsapi`, `sourceRef="gamePk <pk> final away <a> home <h>"`,
`notes="2026-07-09 v2 accelerated capture day-1 cohort settlement; <AWY>@<HOM> final <a>-<h>;
statsapi feed/live verified abstractGameState=Final codedGameState=F; check-settlement-finals.ps1
READY 8/8 + preflight-settlement.ps1 strict exit 0 ... on 2026-07-10"`.

## 3. per-run results

| gamePk | run | matchup | final (a-h) | winner | lean | conf | market | agree | evaluation |
|---|---|---|---|---|---|---|---|---|---|
| 823359 | 9700433e | ATL@PIT | 10-5 | away | away | 0.75 | away | yes | **correct** |
| 823277 | 9800433e | AZ@SD | 3-1 | away | home | 0.75 | home | yes | **incorrect** |
| 824816 | 9d00433e | CHC@BAL | 2-3 | home | home | 0.75 | home | yes | **correct** |
| 823683 | 9e00433e | CLE@MIN | 5-2 | away | away | 0.75 | away | yes | **correct** |
| 824251 | a100433e | ATH@DET | 1-4 | home | home | 0.75 | home | yes | **correct** |
| 823846 | a200433e | SEA@MIA | 4-8 | home | away | 0.75 | away | yes | **incorrect** |
| 823034 | a900433e | MIL@STL | 8-4 | away | away | 0.75 | away | yes | **correct** |
| 822877 | aa00433e | LAA@TEX | 6-7 | home | home | 0.75 | home | yes | **correct** |

**Cohort record: 6 correct / 2 incorrect = 0.750.** Every run carried confidence 0.75 and
was market-aligned. Both misses (823277, 823846) were market-aligned too — the market was
wrong on those games as well, so this cohort contains no evidence about DAI-vs-market edge
in either direction.

## 4. guard fields, persisted (quoted verbatim)

| gamePk | selectedPromptVersion | attributionFidelityStatus | divergenceInterpretation |
|---|---|---|---|
| 823359 | v2 | `Pass` | `MarketAligned` |
| 823277 | v2 | `Pass` | `MarketAligned` |
| 824816 | v2 | `Pass` | `MarketAligned` |
| 823683 | v2 | `Pass` | `MarketAligned` |
| 824251 | v2 | `Pass` | `MarketAligned` |
| 823846 | v2 | `Pass` | `MarketAligned` |
| 823034 | v2 | `Pass` | `MarketAligned` |
| 822877 | v2 | `UnclearMarketAttribution` | `UnclearDivergence` |

822877 reason: `both_market_directions_asserted`. This is the confirmed **classifier
ambiguity** (opponent-as-object: a single market clause names both teams, so
`DetectClaimedSides` returns `{home,away}`), not a model contradiction. It is not the
`FailMarketAttributionMismatch` hard stop. Cohort guard result: **7 Pass / 1 Unclear / 0 FAIL.**

## 5. post-write verification (persisted, not inferred)

- rows total before/after: **294 / 294** — 0 new runs
- outcomes before/after: **125 / 133** (+8); `settlementSource` present on all 133
- rows changed: **exactly 8**, and the changed gamePk set equals the target set
- duplicate outcomes per gamePk: **none**
- duplicate-active gamePks after: **0**
- residue complete (source + sourceRef + notes) on all 8: **yes**
- per-run `GET /api/agent-runs/{id}/evaluation` re-read on all 8: 6 `correct`, 2 `incorrect`

HTTP 200 was not treated as proof. Each write was confirmed by reading back the run and its
evaluation, and by diffing full `/rows` snapshots taken before and after.

## 6. read-model note (no defect)

`/rows` exposes `matchedOutcome`, which is `true` on all eight rows **including the two
incorrect ones**. This is correct: `PromptRouteCalibrationExport.cs:204` defines it as
`r.HasOutcome ? true : null` — "an outcome row exists", not "the lean was right". Correctness
lives in `resultSide` vs `leanSide` and in the evaluation table. The field name invites the
misreading; noting it here so a future readout does not re-derive a phantom defect from it.
`/rows` carries no `evalStatus` field at all.

## 7. what did not change

No prompt, model, confidence, decision, source-readiness, source-depth, ranking,
market-agreement derivation, reconciliation semantics, calibration formula, Gate 4 threshold,
schema, migration, buyer contract, or registry allowlist. Registry routing remains
default-off. `agent-service` was never started for this phase. 0 paid calls.

## 8. what this settlement licenses

Ingestion of eight v2-era settled rows into the calibration denominator, and nothing else.
It authorizes no tuning, no buyer claim, no model replacement — `conclusionsAllowed` remains
`false`. See the readout: [[gate4-evidence-readout-v2-day1-2026-07-10-v1]].
