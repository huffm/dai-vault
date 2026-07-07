---
title: "PATTERN: Gate-4 Evidence Readout Template v1"
type: "pattern"
date: "2026-07-06"
status: "ACTIVE"
project: "DAI"
related:
  - "02 Platform/architecture/governance/evidence-readiness-gates-v1.md"
  - "06 Execution/patterns/cohort-selection-and-run-discipline-v1.md"
---

# gate-4 evidence readout template v1

purpose: one comparable page per settlement cohort. fill this in during the pooled-calibration
re-read phase of every settlement slice so gate-4 movement is reported in the same format
every time. this template is a read-only consumer of gate logic; it never changes it.

data discipline (binding):
- source rows: GET /api/agent-runs/prompt-route-calibration/rows (camelCase fields below).
- valid denominator = settled AND exclusionReason IS NULL. no exceptions.
- never take denominators from /metrics: that endpoint intentionally does not filter
  exclusionReason (continuity); binding counts come from /rows only.
- gate authority: discrimination_hybrid_v1 in agent-service pooled_calibration.py
  (the discriminationHybrid block + conclusionsAllowed). the legacy exact-2dp `gates`
  block is informational only and never gates conclusions.

## 1. cohort header

- date settled: <yyyy-mm-dd>
- cohort: <name + link to its frozen slate doc>
- runs settled this cohort: <n>
- source provider: <sourceProvider>
- data regime: <observedDataRegime; note selectedDataRegime if registry-routed>
- backed_depth divergence present: <yes/no; divergent gamePks>

## 2. before/after evidence table

| measure (field / threshold)                                  | before | after | delta |
| ------------------------------------------------------------ | ------ | ----- | ----- |
| valid settled n (outcomeStatus set, exclusionReason null)     |        |       |       |
| directional n (leanSide home/away)                            |        |       |       |
| settled rows with marketAgreement = false                     |        |       |       |
| market disagreement n (marketDisagreementN / >= 10)           |        |       |       |
| market coverage (marketCoverage / >= 0.60)                    |        |       |       |
| populated confidence regions (populatedRegionCount / >= 2)    |        |       |       |
| region accuracy lt_0.70 (n)                                   |        |       |       |
| region accuracy 0.70_0.74 (n)                                 |        |       |       |
| region accuracy 0.75_0.79 (n)                                 |        |       |       |
| region accuracy gte_0.80 (n)                                  |        |       |       |
| discrimination status + top-bottom delta                      |        |       |       |
| failingReasons (full list)                                    |        |       |       |
| conclusionsAllowed                                            |        |       |       |

legacy exact-2dp bucket accuracies (byConfidenceBucket) may be appended for continuity;
they are informational and never a verdict input.

## 3. per-run outcome list

| agentRunId | gamePk | matchup (away@home) | dai lean | market side | agree | final | evaluation | divergence | provenance |
| ---------- | ------ | ------------------- | -------- | ----------- | ----- | ----- | ---------- | ---------- | ---------- |

- final = outcomeStatus (resultSide derived); evaluation from GET /api/agent-runs/{id}/evaluation.
- divergence = marketAgreement false at decision time; flag these rows -- they are the only
  candidate edge-over-market signal and do not imply demonstrated edge by themselves.
- provenance = settlementSource / settlementSourceRef / settlementNotes; all three must be
  non-null on every settled row (reconciliation residue contract). a null here is a defect,
  not a formatting gap.

## 4. gate-4 verdict

gate 4: FALSE unless the live criterion says otherwise; a TRUE here requires merit
verification, not celebration.

record verbatim from the live payload: conclusionsAllowed = <true/false>;
failingReasons = <list>. if conclusionsAllowed is true, verify every sub-gate value
against its threshold in section 2 before the word true appears anywhere else.

## 5. what this readout does not license

- no tuning
- no threshold edits
- no model replacement
- no buyer-facing accuracy, edge, or performance claims
- no gate edits
- no registry default-on change
