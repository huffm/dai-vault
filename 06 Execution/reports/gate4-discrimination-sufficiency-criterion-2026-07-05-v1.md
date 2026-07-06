---
title: "Gate-4 Discrimination-Based Sufficiency Criterion v1"
type: "report"
date: "2026-07-05"
status: "complete -- discrimination_hybrid criterion + exclusion filter shipped; Gate 4 still FALSE on live data (on the merits)"
project: "DAI"
slice: "Gate-4 Discrimination-Based Sufficiency Criterion"
repos:
  dai: "code+tests (pooled_calibration.py + test_pooled_calibration.py)"
  dai-vault: "docs-only (this report + handoff)"
tags:
  - calibration
  - gate4
  - measurement
  - tdd
related:
  - "06 Execution/reports/calibration-sufficiency-criterion-review-2026-07-05-v1.md"
  - "06 Execution/reports/gate4-coverage-sport-supply-diagnostic-2026-07-05-v1.md"
  - "02 Platform/architecture/governance/evidence-readiness-gates-v1.md"
---

# Gate-4 Discrimination-Based Sufficiency Criterion v1

## 1. objective

Replace the structurally defective exact-2dp confidence-bucket Gate-4 rule with a discrimination-based hybrid
sufficiency criterion, and fix the latent ExclusionReason filtering gap. Approval-gated
(`GATE4_CRITERION_CHANGE_APPROVED=true`), TDD, conservative. The new criterion MUST return
`conclusionsAllowed=False` on current live data -- and it does, on the merits.

## 2. why the exact-bucket rule was defective

`conclusionsAllowed` required `not below_n` = no exact 2dp confidence bucket with n<15. DAI's confidence clusters at
0.75/0.80, so the low buckets (0.63/0.68/0.70/0.72) never reach 15 -> the gate would report FALSE **forever**, even
once 0.80 clearly beat 0.75 (real discrimination). A gate that can never turn true on genuine evidence is broken. It
also tested the wrong thing (bucket population, not whether confidence is informative). See the criterion review.

## 3. new criterion definition (`discrimination_hybrid_v1`, `pooled_calibration.py`)

`conclusionsAllowed = (failingReasons == [])`, where all of these must hold on the exclusion-filtered directional
active-settled rows:

1. **slatesMet** (>=3 settled slates) -- preserved.
2. **enrichedMarketMissingMet** (>=1 enriched_market_missing directional row) -- preserved.
3. **minimum directional sample** >= 40.
4. **>=2 populated confidence regions** -- regions are bands (`lt_0.70 / 0.70_0.74 / 0.75_0.79 / gte_0.80`);
   populated = n>=15.
5. **non-inverted discrimination** -- no adjacent higher region may fall more than 0.05 below a lower region
   (severe inversion); `positive` when the top populated region strictly beats the bottom.
6. **readable market disagreement** >= 10 (2-2 is noise).
7. **market coverage** >= 0.60 of directional rows carry a decision-time market baseline.

Bands (not exact values) so clustered/legacy-dampened confidence pools into comparable regions instead of
scattering into sparse never-fillable exact buckets.

## 4. thresholds selected (documented constants, not magic numbers)

`MIN_DIRECTIONAL_TOTAL=40, MIN_REGION_N=15, MIN_POPULATED_REGIONS=2, INVERSION_TOLERANCE=0.05,
MIN_MARKET_DISAGREEMENT_READABLE_N=10, MIN_MARKET_COVERAGE=0.60`. Conservative and chosen so the current live corpus
fails on the merits (inversion + thin disagreement + thin coverage), NOT to force a pass. Legacy thresholds
(`MIN_SLATES=3, MIN_CONF_BUCKET_N=15, MIN_MARKET_DISAGREEMENT_N=2`) retained for the informational `gates` block.

## 5. ExclusionReason filter fix

`pooled_calibration_summary` now drops rows with a non-empty `exclusionReason` up front (binding denominator: valid
= settled AND ExclusionReason IS NULL) and reports `counts.excludedRowCount`. Excluded runs no longer enter
denominators, buckets, regions, disagreement, coverage, or the gates. On live data this removed **16 excluded rows**
(previously silently pooled); the gate verdict was unchanged (they did not flip a threshold), but the read is now
correct.

## 6. tests added/updated (`test_pooled_calibration.py`; agent-service 436 passed)

Preserved 5 structural tests. Replaced the old single-bucket "thresholds met" test. Added 10 criterion tests:
constructed discriminating corpus passes; current-live-shape (inverted 0.75>0.80) fails; accuracy-inversion fails;
single-populated-region fails; low-disagreement fails; low-coverage fails; excluded rows do not count
(excludedRowCount + denominator unchanged); no-decision rows do not distort directional; slate+emm gates
preserved; result is diagnostic-rich on failure. TDD: 9 red -> implement -> 15 green; full suite 436 passed.

## 7-8. current live-data result -- FALSE, on the merits

Run against live `/rows` (273): `conclusionsAllowed=False`, `excludedRowCount=16`, `validDirectionalN=92`.
Populated regions: `0.75_0.79` (acc 0.619, n=63), `gte_0.80` (acc 0.533, n=15) -> **inverted, delta -0.086**.
`marketDisagreementN=4` (readable=false), `marketCoverage=0.565` (met=false). `slatesMet=true`,
`enrichedMarketMissingMet=true`. **failingReasons = [discrimination_inverted, insufficient_market_disagreement,
insufficient_market_coverage].** The primary, principled reason is that confidence is not informative (the
higher-confidence region does not outperform). This proves the change is not goalpost-moving: the new gate returns
FALSE for honest reasons while the old gate returned FALSE for a structural artifact.

## 9-10. what remains blocked / allowed

Blocked (reported in `discriminationHybrid.blocks`): prompt/model tuning, buyer performance claims, model
replacement, ProbeRefresh Stage-3 artifact mutation. Allowed (`.allows`): internal diagnostics, continued manual
paid capture. This is reporting only -- no enforcement wiring changed; consumers already differ (capture +
diagnostics continue while tuning/claims are gated).

## 11. validation proof

- TDD: 9 new tests red pre-impl -> 15 green post-impl; full agent-service suite **436 passed**.
- Live: `conclusionsAllowed=False` with principled failingReasons (above); excluded 16 rows dropped.
- `/metrics` (.NET endpoint) unaffected: totalRows 273 / reconciled 94 / matched 57 / matchRate 0.6064 --
  `pooled_calibration.py` is an offline CLI-only tool with no runtime/endpoint consumer, so /metrics is
  structurally untouched (verified identical).
- No DB writes, no AgentRuns, no paid calls, no services started.

## 12. runtime safety proof

Changed files: `services/agent-service/app/services/pooled_calibration.py` (offline diagnostic) +
`tests/test_pooled_calibration.py`. NOT changed: confidence generation (`SportsEvaluator`), prompts, prompt
registry recipes, routing, buyer copy, schema/migrations, ProbeRefresh, artifact generation, the .NET /metrics
endpoint. `pooled_calibration.py` has no importer other than its CLI + test (grep-verified). dai diff = these 2
files + the pre-existing DevCore.Data.csproj phantom (untouched).

## 13. repo state

dai `c6d4f43` -> (this slice's code commit); dai-vault `f4b6953` -> (this slice's docs commit). Both to be
committed, not pushed. csproj phantom left uncommitted; synopsis left untracked.
