---
title: "Reconciliation Planning Readout v1 (as of 2026-07-11 cadence close)"
type: "evidence-report"
date: "2026-07-13"
status: "complete"
project: "DAI"
slice: "WI-0010 Planner Evidence Fidelity v1.1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - planning
  - reconciliation
  - calibration
related:
  - "06 Execution/reports/gate4-evidence-readout-v2-day2-2026-07-11-v1.md"
  - "06 Execution/reports/hardened-regime-baseline-measurement-2026-07-11-v1.md"
  - "06 Execution/reconciliations/v2-day1-cohort-settlement-2026-07-10-v1.md"
  - "06 Execution/reconciliations/v2-day2-cohort-settlement-2026-07-11-v1.md"
---

# reconciliation planning readout v1

Machine-readable planning evidence for the snapshot tooling (WI-0010). Every value below is
**copied verbatim from the cited authoritative closeout records -- never computed here**. This
sidecar is updated manually at each cadence close (same convention as the timeline
authorization block); tooling only reads it. If this file's `source_date` is older than the
newest reconciliation record, consumers must treat it as stale and warn.

```yaml
planning-readout:
  schema_version: "1.0"
  as_of: "2026-07-13"
  source_date: "2026-07-11"
  source_type: cadence-closeout
  sources:
    - "06 Execution/reports/gate4-evidence-readout-v2-day2-2026-07-11-v1.md"
    - "06 Execution/reports/hardened-regime-baseline-measurement-2026-07-11-v1.md"
    - "06 Execution/reconciliations/v2-day1-cohort-settlement-2026-07-10-v1.md"
    - "06 Execution/reconciliations/v2-day2-cohort-settlement-2026-07-11-v1.md"
  cadence_status: ended
  operating_posture: no-spend
  captured: 16
  settled: 15
  excluded: 1
  correct: 9
  incorrect: 6
  gate_conclusions_allowed: false
  gate_failing_reasons:
    - discrimination_inverted
    - insufficient_market_disagreement
  gate_coverage: 0.6723
  gate_coverage_met: true
  gate_discrimination_delta: -0.1486
  gate_discrimination_status: inverted
  gate_high_confidence_sample_size: 18
  gate_high_confidence_accuracy: 0.4444
  gate_valid_directional_n: 119
  market_opposed_sample_size: 8
  market_opposed_correct: 3
  market_opposed_incorrect: 5
  market_opposed_readable: false
  attribution_pass: 14
  attribution_fail: 0
  attribution_unclear: 1
  evidence_confidence: high
```

## provenance notes

- captured/settled/excluded: hardened-regime baseline ("All 16 captured rows"; "823357
  excluded (postponed non-event) -> 15 active v2 rows, all settled").
- correct/incorrect: day-1 settlement ("6 correct / 2 incorrect") + day-2 settlement
  (7 settled: 3 correct / 4 incorrect) = 9 / 6.
- gate block: gate4 evidence readout v2-day2 (criterion `discrimination_hybrid_v1`,
  `conclusionsAllowed: false`, failing reasons, delta -0.1486, coverage 0.6723 met,
  gte_0.80 n=18 acc=0.4444, validDirectionalN=119, marketDisagreement n=8 readable=false,
  disagreement record 3/5).
- attribution: hardened-regime baseline ("Pass 14 | FAIL 0 | Unclear 1").
- cadence/posture: current-slice 2026-07-11 wrap (authorization ENDS) + timeline
  authorization block.
