---
title: "Pre-Settlement Hardening Plan v1"
type: "plan"
date: "2026-07-01"
status: "complete"
project: "DAI"
slice: "Pre-Settlement Hardening v1"
repos:
  dai: "code+docs"
  dai-vault: "docs-only"
tags:
  - calibration
  - reconciliation
  - provenance
  - observability
related:
  - "04 Products/sports-v1/calibration/calibration-result-review-v1.md"
  - "06 Execution/plans/live-calibration-cohort-planning-v1.md"
---

# Pre-Settlement Hardening Plan v1 -- Plan

## purpose

Make Outcome Reconciliation Follow-up v7 **safer** and the post-v7 pooled reassessment **more informative**,
using measurement infrastructure only. This slice adds read-only tooling, additive read-only export fields, tests,
and docs. **It changes no decision behavior** -- no prompt/allowlist/confidence/advertised-strength/source/model
change, no DB migration, no reconciliation writes, no paid calls, no game runs.

## chosen order (from the survey, ranked by leverage x safety)

1. **This plan doc** (docs).
2. **Reconciliation collision pre-check helper** (read-only) -- de-risks v7's MultipleMatches cases.
3. **Pooled multi-slate reassessment tool** (read-only) -- makes the post-v7 review near-one-command.
4. **Market signal in the calibration export** (additive, read-only) -- closes the named "market-disagreement
   n>1" evidence gap.
5. **analyzerConfidence/confidence registry-path diagnostic** (read-only) -- resolves the review's supported
   observation; diagnosis only.
6. **Optional insurance** -- cold build + targeted suite.

## grounding confirmed in code (pre-implementation)

- **Collision:** `OutcomeReconciliationMatcher.Match` is a pure classifier over active candidates
  (`ExclusionReason == null`); `OutcomeReconciliationService.MatchAsync` loads candidates read-only. A precheck
  reuses this shape without the write path.
- **Market source (clean):** `MarketSnapshotBatch` already **persists** `ConsensusSide`,
  `MedianHomeImpliedProbability`, `MedianAwayImpliedProbability`, `BookCount`, linked to the run via
  `LinkedAgentRunId` and keyed by `ExternalGameId`. No re-derivation, no migration -- the exporter can left-join
  and surface these additively (mirroring the `fallbackDetail` additive pattern).
- **Confidence formula (uniform):** `SportsEvaluator.Evaluate` computes
  `calibrated = Clamp(analyzerConfidence * Dampening, Min, Max)` where `Dampening` is a function of the
  **grounded-signal ratio** (`groundedCount / maxGrounded`). This is a single evaluator path for every sports run
  -- there is no registry-specific branch. Preliminary read: `analyzerConfidence == confidence` is expected when a
  run is **fully grounded** (dampening 1.0, value within clamp), which is exactly the enriched MLB case. The
  diagnostic will confirm and document this.

## approval boundaries

- Additive read-only export fields and read-only tooling: **in scope, this slice.**
- Any change to the confidence formula, advertised-strength thresholds, prompts, allowlist, source strategy, or
  reconciliation write path: **out of scope** -- requires a separate approved slice.
- If the confidence diagnostic reveals a genuine defect: **stop at the finding**, document it, do not fix here.
- Any paid capture: **behind the hard approval gate** in `live-calibration-cohort-planning-v1`.

## explicit non-actions

No paid calls; no game runs; no reconciliation writes; no prompt/allowlist/confidence/advertised-strength/source/
buyer/model changes; no DB migration/schema mutation. No decision-behavior change of any kind.

## acceptance criteria

- **Collision pre-check:** given a game identity, lists active runs, flags MultipleMatches, surfaces the likely
  backlog (unreconciled active) run, recommends per-run vs identity; read-only; unit-tested.
- **Pooled tool:** groups/summarizes calibration rows by route/provenance, confidence bucket, home/away lean,
  slate (date), and market agreement (when present); reports per-bucket n; refuses tuning conclusions below sample
  thresholds; read-only; unit-tested.
- **Market export:** `/rows` exposes `marketConsensusSide` + median implied probs + a `marketAgreement` flag from
  persisted `MarketSnapshotBatch`; additive; existing consumers unaffected; no migration; tested.
- **Confidence diagnostic:** documents whether the registry path bypasses the shrink (expected: no -- it is a
  uniform grounding-ratio dampening); characterization test asserts current behavior; no behavior change.
- **Verification:** dai builds 0/0; targeted suites green; `/metrics` + `/rows` cross-check unchanged; YAML valid;
  all non-actions confirmed.

## next slice

Outcome Reconciliation Follow-up v7 once the 07-02 games are Final -> pooled reassessment (using the new tool +
market field) across >= 3 settled slates -> paid cohort only behind the explicit approval gate.
