---
title: "Gate-4 Evidence-Sufficiency Projection v1 (2026-07-07)"
type: "report"
date: "2026-07-07"
status: "COMPLETE -- planning projection, read-only, no-spend"
project: "DAI"
slice: "Gate-4 Evidence-Sufficiency Projection v1"
related:
  - "06 Execution/reports/gate4-evidence-readout-backed-depth-divergence-2026-07-06-v1.md"
  - "06 Execution/reports/backed-depth-divergence-settlement-handoff-2026-07-07-v1.md"
  - "06 Execution/patterns/gate4-evidence-readout-template-v1.md"
  - "06 Execution/reports/backed-depth-divergence-capture-2026-07-06-v2.md"
---

# gate-4 evidence-sufficiency projection v1

## 1. purpose

quantify what additional settled evidence gate 4 (discrimination_hybrid_v1) needs to become
reachable, per failing sub-gate. **this is a planning projection, not evidence.** it does not
move the gate, does not license tuning, and contains nothing buyer-facing. every scenario
below is an estimate whose inputs are small-n and explicitly caveated.

## 2. current gate-4 status

recorded verbatim from the live payload (fresh /rows read, 2026-07-07):

- conclusionsAllowed = false
- failingReasons = ["discrimination_inverted", "insufficient_market_disagreement",
  "insufficient_market_coverage"]

## 3. binding data source

- binding row source: GET /api/agent-runs/prompt-route-calibration/rows.
- valid denominator: settled rows with exclusionReason IS NULL (the same filter
  pooled_calibration.py applies up front).
- /metrics is NOT used as a denominator anywhere in this report (it intentionally does not
  filter exclusionReason).
- gate authority: `discrimination_hybrid_v1` in
  `dai/services/agent-service/app/services/pooled_calibration.py` (read, not edited).

## 4. current evidence snapshot (fresh live read, 2026-07-07)

cross-checked against the filled readout
[[gate4-evidence-readout-backed-depth-divergence-2026-07-06-v1]]: **identical on every
measure; no discrepancy.**

- rows total: 279; excluded rows: 16
- valid settled n: 116; no-decision: 18; settled slates: 12
- valid directional n: 98 (threshold 40: passing)
- market-covered directional n: 58 (marketCoverage 0.5918; threshold >= 0.60: FAILING)
- uncovered directional n: 40 -- all legacy rows without a persisted decision-time market
  baseline; backfill was reviewed and REJECTED as unbackfillable (2026-07-05 diagnostic),
  so this drag is permanent and can only be diluted by new covered rows
- marketAgreement=false settled (marketDisagreementN): 5 (threshold >= 10: FAILING);
  current split 2 correct / 3 incorrect -- candidate edge signal only, unreadable at n=5
- populatedRegionCount: 2 (threshold >= 2: passing); populated = 0.75_0.79 (n=68,
  acc 0.5882) and gte_0.80 (n=16, acc 0.5000); unpopulated = lt_0.70 (n=6, acc 0.5),
  0.70_0.74 (n=8, acc 0.75)
- discrimination: status inverted; adjacent-pair rule (higher populated band must not sit
  more than 0.05 below the band beneath it); gte_0.80 at 0.5000 vs required
  >= 0.5882 - 0.05 = 0.5382; delta (top - bottom) = -0.0882
- structure of the 5 disagreement rows (raw /rows extract): ALL are dai-lean home vs
  market-consensus away, confidence 0.70-0.75 (four in the 0.75_0.79 band, one in
  0.70_0.74, none at gte_0.80)

## 5. sub-gate gap table

thresholds and comparators from pooled_calibration.py source (all comparators are >=, so
exact-threshold values pass).

| sub-gate | current | threshold | status | absolute gap | future rows that help | future rows that do NOT help | notes / fragility |
| --- | --- | --- | --- | --- | --- | --- | --- |
| insufficient_market_disagreement (marketDisagreementN) | 5 | >= 10 | FAIL | **+5 settled divergence rows** | settled, exclusion-null, directional rows whose decision-time marketConsensusSide != leanSide | agreement rows, no-decision rows, uncovered rows, excluded rows, unsettled captures | divergence is fixed at capture time; only divergence-targeted capture (close favorites) reliably produces it. row count monotonic -- cannot regress |
| insufficient_market_coverage (marketCoverage) | 0.5918 (58/98) | >= 0.60 | FAIL | **+2 covered directional settled rows** (if no uncovered rows added): (58+2)/(98+2) = 0.6000 exactly | any settled directional row carrying a market baseline (every backed_depth capture row qualifies; divergence rows count double-duty) | uncovered directional rows actively HURT: each one added requires +1.5 covered rows to offset (0.6/0.4) | smallest numeric gap but fragile in reverse: future enriched_market_missing directional settles (3 exist) would push coverage down. the 40 legacy uncovered rows are a permanent drag |
| populatedRegionCount | 2 | >= 2 | PASS | 0 | n/a (band n only grows) | n/a | cannot regress. NOTE: a third band becoming populated adds an adjacency constraint (see discrimination fragility below) |
| discrimination_inverted | inverted; gte_0.80 acc 0.5000 vs bar 0.5382 | higher populated band >= lower band acc - 0.05 (adjacent pairs) | FAIL | not a row-count gap -- **outcome-dependent** (see section 7, scenario 4) | settled gte_0.80 directional rows that evaluate CORRECT (raise the top band); mathematically, incorrect 0.75-band rows also lower the bar (flagged as an artifact, not a goal) | volume alone: more rows with the current outcome mix leave it inverted; disagreement/coverage rows at 0.75 do not touch the top band | cannot be purchased with capture volume. if the true gte_0.80 rate genuinely sits > 0.05 below the 0.75 band, this sub-gate stays false CORRECTLY -- that is the defect it exists to catch |
| insufficient_slates (informational; passing) | 12 | >= 3 | PASS | 0 | any new settled slate date | n/a | comfortable margin |
| no_enriched_market_missing (passing) | 3 directional | >= 1 | PASS | 0 | n/a | n/a | but see coverage fragility: more emm rows hurt coverage |
| insufficient_directional_sample (passing) | 98 | >= 40 | PASS | 0 | n/a | n/a | comfortable margin |
| conclusionsAllowed | false | all reasons empty | FAIL | all three failing sub-gates must clear at the SAME read | -- | -- | interlock: new rows move several measures at once; scenarios below account for that |

## 6. evidence types -- what future rows do

- **valid settled directional rows**: grow validDirectionalN and the confidence regions;
  neutral-to-negative for coverage unless market-covered.
- **market-covered rows** (any backed_depth capture): numerator+denominator of coverage;
  +2 (with no uncovered additions) closes the coverage sub-gate exactly.
- **marketAgreement=false rows (settled divergence)**: the ONLY rows that move
  marketDisagreementN; also covered rows, so they serve coverage simultaneously. 5 needed.
- **settled divergence/candidate-edge rows**: same as above; their correct/incorrect split
  is what a future edge-over-market read will use once n >= 10 -- but the split is not
  gated, only the n.
- **confidence-region population rows**: rows landing in lt_0.70 / 0.70_0.74 could
  populate a third region (needs n >= 15; currently 6 and 8). observed capture yield into
  these bands over the last two cohorts: 0 of 13 -- dai confidence clusters at 0.75/0.80,
  so this path is structurally rare. it is also NOT desirable under the current rule (see
  fragility note in section 7).
- **outcome rows that affect discrimination direction**: settled gte_0.80 directional rows.
  observed gte_0.80 capture yield: 4 of 13 runs across the 07-04 and 07-06 cohorts
  (~0.31/run, tiny n). only their evaluated outcomes move the inversion.

## 7. projection scenarios (planning estimates, not evidence)

### scenario 1 -- mechanical minimum (best case, every row perfectly useful)

**5 settled divergence rows** close both numeric sub-gates simultaneously:
disagreement 5 -> 10 (= threshold), coverage (58+5)/(98+5) = 0.6117 (> 0.60).
the discrimination sub-gate is NOT volume-closable; best-case overlay: if >= 2 of those
divergence rows carried gte_0.80 confidence AND settled correct, the top band clears its
bar ((8+2)/(16+2) = 0.5556 >= 0.5382, holding the 0.75 band fixed). so the absolute
mechanical minimum is **5 new settled rows with ideal properties** -- realistic only as a
lower bound. (historical caution: all 5 existing divergence rows sit at 0.70-0.75
confidence, none at 0.80.)

### scenario 2 -- observed 2026-07-06 yield (weak anchor, n=6)

the divergence-TARGETED cohort (close-favorite prefilter) yielded 1 divergence per 6
captured runs (16.7%). at that yield, +5 divergence rows implies **~30 captured settled
directional runs = 5 six-run cohorts** (or ~3 cohorts at the 12-run cap). low confidence:
the yield estimate rests on one hit. the UNtargeted 07-04 cohort yielded 0/7; pooled
screened registry backed_depth yield is 1/13 (7.7%), which would imply ~65 runs. treat
30-65 runs as the observed-anchor band.

### scenario 3 -- sensitivity range (runs needed for +5 divergence rows)

| assumed divergence yield | captured runs needed (ceil 5/y) | six-run cohorts | est. model cost @ $0.00071/run |
| --- | --- | --- | --- |
| 5%    | 100 | 17 | ~$0.071 |
| 10%   | 50  | 9  | ~$0.036 |
| 16.7% (07-06 observed) | 30 | 5 | ~$0.021 |
| 25%   | 20  | 4  | ~$0.014 |
| 33%   | 15  | 3  | ~$0.011 |

coverage (+2 covered rows) is satisfied by the first settled cohort in every scenario;
divergence n is the binding numeric constraint throughout.

### scenario 4 -- discrimination recovery (outcome-dependent; cannot be bought)

the sub-gate needs acc(gte_0.80) >= acc(0.75_0.79) - 0.05 at a future read. holding the
0.75 band fixed at 0.5882 (bar 0.5382), with the top band at 8/16:

| assumed true rate of future settled gte_0.80 rows | additional gte_0.80 settled rows to clear | note |
| --- | --- | --- |
| 1.00 (all correct) | 2 | absolute floor |
| ~0.58 (current pooled directional rate 57/98) | ~15 | if the inversion is sampling noise around a common rate |
| 0.55 | ~52 | very slow convergence |
| <= 0.5382 | never (bar fixed) | a genuine inversion keeps failing -- correctly |

at the observed gte_0.80 capture yield (~0.31/run), the ~15-row noise scenario implies
**~48 captured runs (~8 six-run cohorts)** -- conveniently overlapping the divergence
capture program (same runs serve both if they land at 0.80).

**structural fragility flagged for a separate criterion review (NOT edited here):**
1. the bar is relative: incorrect 0.75-band settles LOWER it. mechanically, +5 incorrect
   0.75-band rows would drop the bar to 40/73 - 0.05 = 0.4979, beneath the top band's
   current 0.5000 -- the inversion clears via bottom-region decay, and with disagreement
   and coverage also cleared, conclusionsAllowed could flip TRUE on degraded performance.
   the readout template's fixed verdict language ("a TRUE requires merit verification")
   is the existing guard; the criterion review should decide whether non-inversion should
   be paired with an absolute floor.
2. populating a third region currently makes the gate HARDER: 0.70_0.74 sits at 0.75
   accuracy (n=8); if it reached n >= 15 with accuracy > 0.6382, the adjacent pair
   (0.70_0.74 -> 0.75_0.79) would register a NEW inversion regardless of the top band.
   low likelihood (0 rows landed there in the last 13 captures), but reasoning about the
   gate requires knowing this.

### unknowables at current n (what this projection cannot say)

- true divergence yield of targeted capture: 1 observed hit (95% binomial CI on 1/6 spans
  roughly 0.4%-64%); the 30-run estimate could be off by an order of magnitude.
- whether the divergence subset shows any candidate edge at n >= 10 (current 2/5 split is
  pure noise).
- whether the gte_0.80 inversion is noise or real (n=16); this decides if gate 4 is ~8
  cohorts away or structurally unreachable without a genuine calibration change.
- whether future divergence rows will ever land at gte_0.80 (0 of 5 so far).
- durable per-run cost: metering is a public-list ESTIMATE logged to stdout only; no
  durable per-run cost sink exists yet.

## 8. cost projection (documented anchors only)

- **anchor**: 2026-07-06 capture cohort, 6 runs, total estimated $0.004259
  (per-run $0.000699-$0.000735, mean ~$0.00071), model gpt-4o-mini, one call per run.
  source: `backed-depth-divergence-capture-2026-07-06-v2.md` sections 5/16; metering via
  model_metering.py public-list estimate -- **approximate, hand-recorded, not billing
  truth**. no durable per-run cost evidence exists (stdout-only sink).
- cost per captured run: ~$0.0007 (estimate).
- cost per settled divergence candidate at the observed 16.7% yield: ~$0.0043 (one six-run
  cohort per candidate).
- mechanical minimum (scenario 1, 5 perfect rows in one cohort): ~$0.004.
- sensitivity range (scenario 3): **~$0.011 to ~$0.071 total model cost** for 15-100
  captured runs.
- discrimination-noise overlay (scenario 4, ~48 runs): ~$0.034, largely shared with the
  divergence program.
- every scenario sits far below the per-cohort guardrails (MAX_PAID_RUNS 12, $0.05 cap);
  the practical constraints are calendar (one close-favorite morning slate per day) and
  the-odds-api quota (~7 units per screened cohort; 402 units remained at the 07-06
  capture), not model spend.

## 9. interpretation

what this projection says:
- the two numeric sub-gates are close: coverage needs +2 covered settled rows (one cohort);
  disagreement needs +5 settled divergence rows, which at observed-to-pessimistic yields
  means roughly 30-100 captured runs (~5-17 six-run morning cohorts) at negligible model
  cost.
- the binding uncertainty is NOT spend -- it is divergence yield (one observed hit) and the
  discrimination outcome mix, which no amount of capture volume can force.
- divergence-targeted capture is the only capture shape that serves all three failing
  sub-gates at once (covered + potential disagreement + occasional 0.80-band rows).

what this projection does not say:
- it does not say gate 4 WILL clear after N runs -- discrimination is outcome-dependent.
- it does not say the 2/5 disagreement split means anything (candidate edge signal only).
- it does not authorize any capture; every paid cohort remains individually
  approval-gated.

## 10. what this does not license

- no tuning
- no threshold edits
- no model replacement
- no buyer-facing accuracy, edge, or performance claims
- no gate edits
- no registry default-on change
- no new capture authorization
- no automatic cadence

## 11. recommended next slice

**Evidence Acquisition Cadence Proposal v1** -- the projection produced usable numeric
ranges (runs, cohorts, cost, quota, calendar) sufficient for cadence planning: propose a
morning-window capture cadence (frequency, cohort size, stop conditions, re-projection
checkpoints at disagreement n=7 and n=10), still approval-gated per cohort. carry the two
criterion fragilities (bottom-decay TRUE path; third-region adjacency) as INPUTS to a
separate, later criterion interpretation/review note -- they are documented here, do not
block cadence planning, and must not be edited casually.
