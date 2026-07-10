---
title: "Gate-4 Evidence Readout -- V2 day-1 cohort (2026-07-10)"
type: "evidence-report"
date: "2026-07-10"
status: "complete"
project: "DAI"
slice: "V2 Day-1 Settlement and Day-2 Capture"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - calibration
  - gate-4
  - settlement
related:
  - "06 Execution/reconciliations/v2-day1-cohort-settlement-2026-07-10-v1.md"
  - "06 Execution/reports/gate4-evidence-sufficiency-projection-2026-07-07-v1.md"
  - "06 Execution/reports/deliberate-divergence-settlement-and-n7-checkpoint-2026-07-08-v1.md"
  - "06 Execution/reports/diagnostic-readout-failure-taxonomy-2026-07-07-v1.md"
  - "06 Execution/plans/v2-day1-settlement-day2-capture-slice-2026-07-10-v1.md"
---

# gate-4 evidence readout -- v2 day-1 cohort

Third filled readout. Recomputed from live `/rows` (294 rows) via
`scripts/pooled_calibration_report.py` immediately after the 8 day-1 writes. Criterion:
`discrimination_hybrid_v1`. Language follows the taxonomy section 7.6 rules: *market-opposed
rows*, taxonomy class per row, *candidate edge* reserved for `CountsAsCandidateEdge`
(deliberate) rows only.

## 1. canonical live fields (quoted verbatim, before any interpretation)

```
counts:            reconciled 130 | directional 112 | noDecision 18 | settledSlates 15 | excludedRowCount 39
criterion:         discrimination_hybrid_v1
validDirectionalN: 112

confidenceRegions:
  lt_0.70     n=6   correct=3   incorrect=3   accuracy=0.5     populated=false
  0.70_0.74   n=8   correct=6   incorrect=2   accuracy=0.75    populated=false
  0.75_0.79   n=81  correct=49  incorrect=32  accuracy=0.6049  populated=true
  gte_0.80    n=17  correct=8   incorrect=9   accuracy=0.4706  populated=true
populatedRegionCount: 2

discrimination:
  status     = "inverted"
  topBand    = gte_0.80    (accuracy 0.4706, n 17)
  bottomBand = 0.75_0.79   (accuracy 0.6049, n 81)
  delta      = -0.1343

marketDisagreementN:        7
marketDisagreementReadable: false
marketCoverage:             0.6518
marketCoverageMet:          true

marketAgreement:
  withMarketSide 73
  agreement    n=66  correct=42  incorrect=24  accuracy=0.6364
  disagreement n=7   correct=2   incorrect=5   accuracy=0.2857

blocks:  [prompt_model_tuning, buyer_performance_claims, model_replacement, probe_refresh_stage3_mutation]
allows:  [internal_diagnostics, manual_paid_capture]

failingReasons:     ["discrimination_inverted", "insufficient_market_disagreement"]
conclusionsAllowed: false
```

Day-1 slate, as recorded by the pooled report: `2026-07-09 -> n=8, correct=6, incorrect=2,
accuracy=0.75`.

## 2. before / after

| field | before (07-08 n=7 checkpoint) | after (this readout) | delta |
|---|---|---|---|
| reconciled | 122 | 130 | +8 |
| directional | 104 | 112 | +8 |
| settledSlates | 14 | 15 | +1 |
| 0.75_0.79 | n=73, acc 0.5890 | n=81, acc 0.6049 | +8 rows, +0.0159 |
| gte_0.80 | n=17, acc 0.4706 | n=17, acc 0.4706 | unchanged |
| discrimination delta | -0.1184 | **-0.1343** | deeper by 0.0159 |
| marketCoverage | 0.625 | 0.6518 | +0.0268 |
| marketDisagreementN | 7 | **7** | **unchanged** |
| conclusionsAllowed | false | false | unchanged |

## 3. day-1 contribution to the market-opposed ledger: zero (verified)

All eight day-1 runs were market-aligned (`marketAgreement=true`), so the market-opposed
ledger is untouched at **n=7 (2 correct / 5 incorrect)**. This was verified from persisted
rows, not assumed from the capture report: the settled, active, `marketAgreement=false` set
is exactly `[824500, 824743, 823281, 823529, 824662, 823036, 824820]` — the same seven as
before the writes.

**Deliberate-divergence ledger (`CountsAsCandidateEdge`): 1 active settled row (823281,
incorrect) = 0/1. Unchanged.** No candidate edge signal exists in this cohort or any other.
The remaining six market-opposed rows are `AccidentalDivergence` per the taxonomy and do not
earn edge language in either direction.

## 4. hardened-regime (v2) attribution, against the frozen baseline

Frozen v1-era baseline: **Pass 72 / FAIL 10 / Unclear 203** (285 rows).
Live corpus now (294 rows): **Pass 80 / FAIL 10 / Unclear 204**.

The eight day-1 rows are all `selectedPromptVersion=v2` and resolve **7 Pass / 1 Unclear / 0
FAIL**. Corpus FAIL count is **unchanged at 10** — no v2-era run has produced a new
attribution mismatch. The single Unclear (822877, `both_market_directions_asserted`) is the
confirmed classifier opponent-as-object ambiguity, not a model contradiction.

Per the standing measurement rule, v2 attribution rates are **not pooled** with v1 rates.
Direction only at this n: nothing has worsened; 8 rows cannot establish that the hardening
improved anything. A proper baseline measurement is the 07-11 wrap deliverable.

## 5. gate-4 verdict

`conclusionsAllowed = false`, failing on `[discrimination_inverted,
insufficient_market_disagreement]`. `insufficient_market_coverage` remains resolved
(0.6518 >= 0.60).

The day-1 cohort added settled evidence to the 0.75-0.79 confidence band but did not increase
the market-opposed sample. The higher-confidence band remained weaker than the band beneath
it, so the discrimination inversion persisted and deepened, from -0.1184 to -0.1343. **This is
an honest calibration result, not an operational regression.**

The mechanism is arithmetic, not causal: eight correct-leaning 0.75 rows raised the bottom
band's accuracy (0.5890 -> 0.6049) while the top band gained no rows at all, widening the gap
the criterion measures. Nothing about the system's behavior changed; the measurement got more
precise about a disorder that was already there.

Two constraints from the 07-07 projection continue to hold, and this readout is consistent
with both:

- **Discrimination is not volume-purchasable.** Capturing more 0.75-band rows cannot fix an
  inversion whose top band is the one that is starved and weak. Only `gte_0.80` rows that
  land correct can move it; if the true 0.80 rate really is more than 0.05 below the 0.75
  rate, the gate is failing correctly and should never turn true.
- **The disagreement sub-gate needs +3 market-opposed rows** (7 -> 10), and only
  divergence-targeted capture produces them. Day-1's yield was 0/8.

Noted fragility, unchanged and still unexercised: the inversion could also clear by *bottom-band
decay* (enough incorrect 0.75 rows dropping the bar) rather than by top-band improvement. Any
future flip to `conclusionsAllowed=true` must be merit-verified against which band actually moved.

## 6. what this readout does not license

No tuning, no prompt or model change, no confidence recalibration, no buyer-facing performance
claim, no model replacement, no ProbeRefresh Stage-3 mutation. Those remain blocked by the
criterion's own `blocks` list. It licenses exactly what `allows` names: internal diagnostics
and further manual, approval-gated paid capture.

No result in this readout is causal. An 8-row cohort at a single confidence value is not
evidence about prompt quality, model quality, or edge.
