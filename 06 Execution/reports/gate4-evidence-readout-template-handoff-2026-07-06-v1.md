---
title: "HANDOFF: Gate-4 Evidence Readout Template v1 (2026-07-06)"
type: "handoff"
date: "2026-07-06"
status: "COMPLETE -- template shipped"
project: "DAI"
slice: "Gate-4 Evidence Readout Template v1"
related:
  - "06 Execution/patterns/gate4-evidence-readout-template-v1.md"
  - "06 Execution/reports/pre-settlement-qa-script-and-cohort-discipline-handoff-2026-07-06-v1.md"
---

# HANDOFF: Gate-4 Evidence Readout Template v1 (2026-07-06)

## 1. objective

docs-only, no-spend: create a one-page reusable evidence readout template so every
settlement cohort reports gate-4 movement in the same comparable format, sourced from the
real /rows shape and the discrimination_hybrid_v1 criterion.

## 2. outcome

COMPLETE. `06 Execution/patterns/gate4-evidence-readout-template-v1.md` shipped: cohort
header, before/after evidence table keyed to actual payload fields and thresholds, per-run
outcome list with residue-contract provenance check, fixed gate-4 verdict language, fixed
does-not-license list. one page, lowercase ascii.

## 3. repo state before / after

- dai: `d79c38f` + preflight script commit, 1 ahead, csproj phantom only. UNCHANGED.
- dai-vault before: `002b72f` (context pack), 1 ahead, 2 untracked (manifest json,
  synopsis). after: +1 docs commit (template + this handoff), 2 ahead, same untracked.

## 4. services used

none started. a read-only GET probe of :5007 /rows was attempted and refused (services
stopped since the 17:08 handoff), so shapes were read from source instead -- which is
authoritative for field names anyway. no db access, no statsapi, no agent-service.

## 5. work performed

read pooled_calibration.py (discrimination_hybrid_v1 thresholds + summary shape) ->
read AgentRunsController.cs /metrics + /rows endpoints (confirmed /metrics does NOT
filter exclusionReason; binding denominator must come from /rows) -> read
PromptRouteCalibrationExport.cs (exact camelCase row fields incl. marketAgreement,
settlementSource/sourceRef/notes, observedDataRegime, exclusionReason) -> read the
pre-settlement-qa handoff (cohort field conventions: gamePk, agree/disagree) -> wrote the
template -> this handoff -> docs-only vault commit.

## 6. files changed

dai-vault only (committed):
- `06 Execution/patterns/gate4-evidence-readout-template-v1.md` (new)
- `06 Execution/reports/gate4-evidence-readout-template-handoff-2026-07-06-v1.md` (new)

## 7. db writes / side effects

0 db writes. no services started; the single live probe was a refused GET. no dai changes.

## 8. paid calls / cost

0 paid model calls, $0.00.

## 9. validation proof

- every field named in the template exists verbatim in PromptRouteCalibrationRow json
  property names (agentRunId, leanSide, marketConsensusSide, marketAgreement,
  outcomeStatus, exclusionReason, settlementSource/settlementSourceRef/settlementNotes,
  observedDataRegime) or in the pooled summary (marketDisagreementN, marketCoverage,
  populatedRegionCount, confidenceRegions bands lt_0.70/0.70_0.74/0.75_0.79/gte_0.80,
  failingReasons, conclusionsAllowed).
- thresholds in the table match pooled_calibration.py constants (>=10 disagreement,
  >=0.60 coverage, >=2 populated regions at n>=15).
- fixed verdict + does-not-license wording matches the slice spec verbatim, including
  "no registry default-on change".
- pooled_calibration.py, gate criteria, buyer surfaces: untouched (docs-only diff).

## 10. what did not change

pooled_calibration.py, gate thresholds, any dai code, prompts, routing, registry flags,
buyer copy, schema, the 07-06 cohort (still 6/6 unreconciled), .env. nothing pushed.

## 11. open issues

- template is unexercised: its first real instantiation should be the 07-06 divergence
  cohort settlement (finals ~00:30 ET 07-07).
- /metrics exclusion-filter gap remains deferred (documented in the template's data
  discipline block so readers do not trip on it).
- dai 1 ahead + dai-vault 2 ahead, unpushed (push on approval).

## 12. recommended next slice

Backed-Depth Divergence Settlement / Reconciliation v1 (prompt already exists in the
pre-settlement-qa handoff section 13), with its phase-4 pooled-calibration re-read
reported AS the first filled instance of this template.

## 13. suggested next prompt

use the settlement prompt from
`pre-settlement-qa-script-and-cohort-discipline-handoff-2026-07-06-v1.md` section 13
verbatim, with one addition to phase 4: "report the before/after read as the first filled
instance of 06 Execution/patterns/gate4-evidence-readout-template-v1.md and save it to
06 Execution/reports/."
