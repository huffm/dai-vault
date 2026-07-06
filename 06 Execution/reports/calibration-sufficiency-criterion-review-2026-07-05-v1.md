---
title: "Calibration Sufficiency Criterion Review v1"
type: "report"
date: "2026-07-05"
status: "complete -- criterion is defective (structurally unsatisfiable); replace with discrimination-based rule; Gate 4 still correctly FALSE"
project: "DAI"
slice: "Calibration Sufficiency Criterion Review"
repos:
  dai: "unchanged (c6d4f43) -- review only"
  dai-vault: "docs-only (this report + handoff)"
tags:
  - calibration
  - gate4
  - doctrine
  - measurement
related:
  - "06 Execution/reports/gate4-coverage-sport-supply-diagnostic-2026-07-05-v1.md"
  - "02 Platform/architecture/governance/evidence-readiness-gates-v1.md"
  - "04 Products/sports-v1/confidence-calibration-rules-v1.md"
---

# Calibration Sufficiency Criterion Review v1

## 1. objective

Review, on principle, whether Gate 4's current rule -- `n >= 15` in **every exact 2dp confidence bucket** -- is the
right sufficiency criterion given DAI's observed confidence distribution. Distinguish (1) a legitimate criterion
defect, (2) insufficient evidence, (3) a confidence-distribution artifact, (4) premature desire to unlock later
work. No-spend, operator-gated; no gate change implemented here.

## 2. repo state

dai `c6d4f43` (dirty only on pre-existing csproj phantom, untouched); dai-vault `d2a2411`, 3 ahead (unpushed prior
docs), untracked `system-state-synopsis-v1.md` (pre-existing). Expected.

## 3. evidence reviewed

`services/agent-service/app/services/pooled_calibration.py` (gate logic `:45-101`); `scripts/pooled_calibration_
report.py` (CLI); `02 Platform/architecture/governance/evidence-readiness-gates-v1.md` (Gate 4 def + status);
`04 Products/sports-v1/confidence-calibration-rules-v1.md` (two-axis confidence doctrine); prior Gate-4 diagnostic;
live `/rows` (273) recomputed locally.

## 4. current Gate 4 rule

`conclusionsAllowed = slatesMet AND enrichedMarketMissingMet AND (not below_n) AND marketDisagreementMet`
(`pooled_calibration.py:96-101`). `below_n` = any confidence bucket with n<15, buckets keyed by
`f"{round(float(c),2):.2f}"` (exact 2dp, `:45-47`). Thresholds: MIN_SLATES=3, MIN_CONF_BUCKET_N=15,
MIN_MARKET_DISAGREEMENT_N=2 (`:14-16`). Origin: the code comment says it "mirrors calibration-result-review-v1's
minimum evidence bar" -- an interim mechanism, not a first-principles doctrine. Doctrine
(`confidence-calibration-rules-v1.md:64-65,88-96,114-125`) treats numeric confidence as a **provisional estimate**
that outcome reconciliation will recalibrate, and forbids confidence/posture threshold moves without
reconciled-outcome evidence (entry 12). Gate 4 IS that "enough outcome evidence" check.

## 5. current data shape (live /rows; active = ExclusionReason IS NULL, settled, directional; N=92)

Overall directional accuracy = **56/92 = 0.609**. Exact 2dp buckets:

| bucket | n | acc |
|---|---|---|
| 0.63 | 1 | 1.000 |
| 0.68 | 5 | 0.400 |
| 0.70 | 6 | 0.833 |
| 0.72 | 2 | 0.500 |
| **0.75** | **63** | **0.619** |
| **0.80** | **15** | **0.533** |

**Discrimination (the property that actually matters): NONE yet.** The only two well-sampled buckets are 0.75
(n=63, 0.619) and 0.80 (n=15, 0.533) -- the higher-confidence bucket does **not** outperform; it is mildly
inverted, though the 0.80 gap is within noise at n=15. So confidence is not yet a predictive discriminator.

## 6. candidate gate definitions (each computed against current data)

| # | definition | current verdict | protects against | fails to protect | gaming risk | complexity | tuning? | buyer? | Stage-3? |
|---|---|---|---|---|---|---|---|---|---|
| A | exact 2dp, every bucket n>=15 | **FAIL** | over-reading a thin bucket | **structurally unsatisfiable** (low buckets never fill) + tests population not discrimination | low | none (current) | no | no | no |
| B | active-bucket (only buckets >= X% must hit 15) | 5% FAIL / **10% PASS** | thin-bucket over-read | ignores discrimination -> can pass while inverted | **HIGH** (threshold is a dial) | low | no | no | no |
| C | confidence bands (low<0.70 / 0.70-0.74 / 0.75-0.79 / >=0.80) | **FAIL** (low n=6, med n=8) | value-scatter artifact | low/med bands still thin; no discrimination test | med | low | no | no | no |
| D | route-specific (registry vs legacy) | **FAIL** both (registry n=25, legacy n=66) | mixing regimes | tiny corpora fail; no discrimination | high (more corpora, more sparse) | med | no | no | no |
| E | directional-only (exclude no-decision) | no-op | -- | already the current behavior (lean-null excluded from accuracy) | n/a | none | -- | -- | -- |
| F | **hybrid: total sample + active-region n + discrimination(no-inversion) + disagreement split + coverage + exclusion-correct** | **FAIL** (inversion + 2-2 disagreement) | over-reading thin data AND non-discriminating confidence | more moving parts to specify | **LOW** (fails on the merits today) | med-high | when true | when true | when true |

Key results: A is unsatisfiable; B passes only by dialing the active-share threshold (gaming risk); C and D still
fail; only **F actually tests the right thing** (does confidence discriminate?) and correctly returns FAIL today.

## 7. ExclusionReason filter finding

`pooled_calibration.py` filters only `outcomeStatus` (`_is_reconciled`, `:23-24`), NOT `ExclusionReason`; the CLI
passes raw `/rows` (which surfaces `exclusionReason` per row but does not drop excluded rows). So excluded runs CAN
enter the pooled buckets/denominator -- a mismatch with the binding denominator doctrine (valid = settled AND
ExclusionReason IS NULL). **Magnitude today: negligible** -- only **2 excluded directional-settled rows, both at
0.75** (0.75 would go 63->65, still >=15; no bucket crosses a threshold; the verdict is unchanged). Classification:
**latent correctness bug (doctrine mismatch), currently harmless to the verdict.** Recommendation: **fix it in the
same implementation slice that changes the criterion** (not a separate slice; not before -- it changes no current
result). Minimal fix: filter `not exclusionReason` alongside `outcomeStatus` in `_is_reconciled` (or in the CLI
before pooling).

## 8. gate purpose review (Gate 4 need not be one boolean)

| consequence | current evidence sufficient? | gate? |
|---|---|---|
| prompt/model tuning | NO (no discrimination; would fit noise) | **BLOCK** |
| buyer performance claims | NO (confidence uncalibrated) | **BLOCK** |
| model replacement | NO (no baseline to prove improvement) | **BLOCK** |
| ProbeRefresh Stage-3 artifact mutation (confidence/posture/lean) | NO (entry 4/12 unmet) | **BLOCK** |
| internal diagnostics / packaging | yes (read-only) | **ALLOW** |
| continued manual paid capture/validation | yes (capture is how evidence is gathered) | **ALLOW** (blocking it is circular) |

Gate 4 today is a single `conclusionsAllowed` boolean, but its real consequences already differ (capture +
diagnostics continue while tuning/claims are blocked). The review recommends making that separation **explicit**:
Gate 4 blocks tuning / buyer-claims / model-replacement / Stage-3-mutation, and explicitly permits capture +
diagnostics.

## 9. recommendation (principal-engineer decision)

**Replace the exact-2dp-bucket sufficiency with a discrimination-based hybrid criterion (candidate F), fix the
ExclusionReason filter in the same slice, and make the by-purpose gate split explicit -- implemented in a later
operator-approved slice. Net effect today: Gate 4 stays FALSE.**

Why this is principled, not gaming:
- The exact-bucket rule is a **legitimate criterion defect**: it is *structurally unsatisfiable* -- the low-
  confidence buckets (0.63/0.68/0.70/0.72) never reach 15 because DAI's leans cluster at 0.75/0.80, so the gate
  would report FALSE **forever**, even if 0.80 later clearly beat 0.75 (real discrimination). A gate that cannot
  turn true on genuine evidence is broken and must be replaced.
- The replacement is chosen because it tests the **right thing** (does confidence discriminate accuracy in
  populated regions?), and it **still returns FALSE on today's data** (mild inversion; 2-2 disagreement). A change
  that leaves the verdict unchanged cannot be goalpost-moving.
- What remains BLOCKED after the change: tuning, buyer performance claims, model replacement, ProbeRefresh Stage-3.
  What becomes ALLOWED: nothing new today -- the change unbreaks the gate so that *future* accrued evidence
  (discrimination + a readable disagreement split) can actually move it.
- What must be implemented later (approved slice): the hybrid criterion in `pooled_calibration.py` (active-region
  sufficiency + no-severe-inversion discrimination + disagreement-split + coverage threshold + exclusion filter),
  with tests proving it returns FALSE on the current corpus and TRUE only on a constructed discriminating corpus.
- Validation that proves it is safe: `/metrics` byte-identical (no runtime decision surface touched); pooled
  summary returns `conclusionsAllowed=False` on today's rows; a synthetic discriminating-corpus test returns True;
  exclusion-filter unit test drops the 2 excluded rows.

This is (3) a confidence-distribution artifact interacting with (1) a real criterion defect -- NOT (2) merely
insufficient evidence (evidence is thin too, but the gate is *additionally* broken) and NOT (4) premature unlock
(the recommended criterion unlocks nothing today).

## 10. ranked next implementation options (Calib-correctness / Risk / Measurement / Readiness / Cost / Revenue)

| # | option | C | Risk | M | Rdy | Cost | Rev | total |
|---|---|---|---|---|---|---|---|---|
| 1 | **Gate-4 discrimination-based criterion + exclusion-filter fix (combined, TDD)** | 5 | 5 | 4 | 4 | 5 | 1 | **24** |
| 2 | Backed-Depth Divergence Capture v2 (paid, morning) -- grows the disagreement split the new gate needs | 3 | 4 | 5 | 4 | 4 | 3 | 23 |
| 3 | ExclusionReason filter fix alone | 4 | 4 | 2 | 5 | 5 | 1 | 21 |
| 4 | Gate confidence-band criterion (C) alone | 3 | 3 | 2 | 4 | 5 | 1 | 18 |
| 5 | Gate active-bucket criterion (B) alone | 2 | 2 | 2 | 4 | 5 | 1 | 16 (gaming risk) |
| 6 | Buyer trust-surface packaging | 2 | 4 | 2 | 4 | 5 | 5 | 22 (premature until edge) |
| 7 | WNBA feasibility/enablement | 2 | 3 | 2 | 4 | 4 | 3 | 18 (factory, not gate) |

## 11. deferred

- Standalone exclusion-filter slice (fold into option 1 instead).
- Band-only / active-bucket-only criteria (both miss discrimination; superseded by the hybrid).
- WNBA, buyer packaging, more Interrogate -- per prior audits.

## 12. validation performed

Read-only. 273 `/rows` recomputed locally; each candidate gate + the discrimination check + exclusion impact
computed from the same data; gate logic + doctrine read from source (file:line). No mutating endpoints, no DB
writes, no services started, no paid calls, no new AgentRuns, no gate change implemented.

## 13. what did not change

No runtime code, prompts, prompt registry recipes, routing, confidence logic, calibration rules/gate,
buyer copy, schema/migrations, reconciliation. No new AgentRuns, no DB writes, no paid calls, no services started.
dai unchanged at `c6d4f43`. Only artifacts: this report + handoff.
