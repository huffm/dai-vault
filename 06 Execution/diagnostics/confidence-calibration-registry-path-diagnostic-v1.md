---
title: "Confidence Calibration Registry-Path Diagnostic v1"
type: "diagnostic"
date: "2026-07-01"
status: "complete"
project: "DAI"
slice: "Pre-Settlement Hardening v1"
repos:
  dai: "tests-only"
  dai-vault: "docs-only"
tags:
  - calibration
  - provenance
  - diagnostic
  - observability
related:
  - "04 Products/sports-v1/calibration/calibration-result-review-v1.md"
  - "06 Execution/plans/pre-settlement-hardening-plan-v1.md"
---

# Confidence Calibration Registry-Path Diagnostic v1 -- Evidence Report

## question

Calibration Result Review v1 flagged a supported observation: reconciled `/rows` showed
`analyzerConfidence == confidence` on the provenance-bearing (registry/enriched) rows, while some legacy rows
showed roughly a 0.9x shrink. The review asked whether the calibration shrink is **bypassed on the registry
path** (a possible defect) or whether the equality is expected. This diagnostic answers that -- read-only, no
behavior change.

## finding (definitive): not a defect, not a registry-path bypass

The confidence calibration is a **single, uniform grounding-ratio dampening** in `SportsEvaluator.Evaluate`
(`platform/dotnet/DevCore.Api/AgentRuns/SportsEvaluator.cs`). It runs for **every** sports analysis, regardless
of which prompt (registry-authoritative or live/legacy) fed the model. **There is no registry-specific branch.**

The formula:

```
calibrated = Clamp(analyzerConfidence * Dampening, MinConfidence, MaxConfidence)
```

with the tier resolved from the grounded-signal ratio (`ResolveConfidenceCalibration(groundedCount, maxGrounded)`):

| grounding | Dampening | clamp | effect |
|---|---|---|---|
| fully grounded (`groundedCount == maxGrounded`) | **1.00** | [0.35, 0.85] | pass-through -> `calibrated == analyzerConfidence` for analyzer in [0.35, 0.85] |
| partial (`0 < groundedCount < maxGrounded`) | **0.90** | [0.35, 0.75] | ~0.9x shrink |
| priors-only (`groundedCount == 0`) | 0.75 | [0.30, 0.60] | heavier shrink + tight ceiling |

## why the observed data matches exactly

- **Enriched MLB registry rows** are **fully grounded** (`evidenceRichness == 2 == MLB max grounded signals`).
  Fully grounded -> Dampening 1.00 -> `calibrated == analyzerConfidence` for analyzer values 0.70-0.80 (all
  within [0.35, 0.85]). Hence `aConf == conf` on every enriched row. **Correct, by design.**
- The legacy shrink rows are the **partial-grounding** case (Dampening 0.90). The observed legacy values are
  exactly `analyzerConfidence * 0.90`: `0.75 -> 0.675`, `0.80 -> 0.72`, `0.70 -> 0.63`. These are competitions/runs
  with less than full grounding (e.g. NBA 1-of-2), **not** "the legacy path."

So the review's "registry bypasses shrink / legacy ~0.9x" over-generalized: the true driver is **grounding
completeness (`evidenceRichness` relative to the competition max), not the prompt path.** A fully-grounded live
run would also show `aConf == conf`; a partially-grounded registry run would also shrink.

## verification (read-only, no behavior change)

- Source-verified: `SportsEvaluator.Evaluate` + `ResolveConfidenceCalibration` (single path; `EvaluatorOutput`
  stores both `AggregateConfidence` and `AnalyzerConfidence`).
- The existing `SportsEvaluatorTests` already characterize every tier
  (`fully_grounded_..._passes_confidence_through` = 0.75x1.00=0.75; `partially_grounded_nba_..._0_90_dampening`
  = 0.80x0.90=0.72; priors-only clamp).
- Added 2 characterization tests pinning the diagnosis directly:
  `fully_grounded_run_stores_analyzer_confidence_equal_to_calibrated` (aConf == conf) and
  `partially_grounded_run_shrinks_calibrated_below_analyzer_confidence` (conf < aConf).

## conclusion + non-action

**No defect. No fix. No behavior change.** The `aConf == conf` on enriched rows is the correct pass-through of a
fully-grounded run. The only follow-on is interpretive, already captured in the review: confidence remains an
internal coherence score, not an empirical win probability, until outcomes demonstrate discrimination. This
diagnostic changes no calibration behavior (tests-only in `dai`).

## note for pooled reassessment

When reading calibration by confidence bucket, remember the calibrated `confidence` is the analyzer value only for
fully-grounded runs and a dampened value otherwise -- so bucket comparisons should be made within a grounding
tier (or on `analyzerConfidence`) to avoid mixing pass-through and dampened values. The pooled reassessment tool
reports per-bucket n so this stays visible.
