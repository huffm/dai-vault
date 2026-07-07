---
title: "HANDOFF: Gate-4 Evidence-Sufficiency Projection v1 -- DONE (2026-07-07)"
type: "handoff"
date: "2026-07-07"
status: "COMPLETE -- read-only projection report shipped; no writes, no spend"
project: "DAI"
slice: "Gate-4 Evidence-Sufficiency Projection v1"
related:
  - "06 Execution/reports/gate4-evidence-sufficiency-projection-2026-07-07-v1.md"
  - "06 Execution/reports/backed-depth-divergence-settlement-handoff-2026-07-07-v1.md"
  - "06 Execution/reports/gate4-evidence-readout-backed-depth-divergence-2026-07-06-v1.md"
---

# HANDOFF: gate-4 evidence-sufficiency projection v1 (2026-07-07)

## 1. objective

read-only, docs-only projection quantifying the additional settled evidence each failing
gate-4 sub-gate needs under the live discrimination_hybrid_v1 criterion, with capture-volume
and cost ranges from documented anchors only.

## 2. outcome

COMPLETE. report shipped:
`06 Execution/reports/gate4-evidence-sufficiency-projection-2026-07-07-v1.md`.
headline numbers: coverage needs +2 covered settled directional rows (exact: 60/100 = 0.60);
disagreement needs +5 settled divergence rows; at observed targeted yield (1/6, weak
anchor) that is ~30 captured runs / ~5 six-run cohorts, sensitivity band 15-100 runs
(~$0.011-$0.071 est. model cost); discrimination is outcome-dependent and NOT
volume-purchasable (floor 2 all-correct gte_0.80 settles; ~15 if the inversion is noise
around the pooled 0.58 rate; never if the true 0.80-band rate is > 0.05 below the 0.75
band). two criterion fragilities documented and flagged for a SEPARATE review (bottom-decay
TRUE path; third-region adjacency trap) -- criterion NOT edited. fresh live read matched
the filled readout on every value (no discrepancy). recommended next = Evidence Acquisition
Cadence Proposal v1.

## 3. repo state before / after

- dai before and after: main @ `dbda7a8`, 0 ahead / 0 behind, dirty only on the phantom
  `DevCore.Data.csproj`. NO changes, NO commit, NO push.
- dai-vault before: main @ `075a8ee`, 0/0, untracked = known manifest + synopsis noise.
- dai-vault after: projection report + this handoff committed and pushed (sha recorded in
  the session closeout); same untracked noise retained, not committed.

## 4. services used

devcore-sql (docker, already up) + DevCore.Api :5007 (already up from the settlement
slice) for GET /rows only. agent-service NOT running (port 8000 verified not listening);
pooled summary via local `scripts/pooled_calibration_report.py`. no external calls beyond
localhost.

## 5. work performed

- skills gate run (dai-skill-router); selected slice-runner doctrine + dai-agent-handoff +
  verification-before-completion; no missing skills.
- phase 0 baselines recorded; services verified; no paid path needed.
- phase 1 fresh binding read: /rows -> 279 rows / 116 valid settled / 98 directional /
  58 covered / 5 disagreement (all dai-home-vs-market-away, conf 0.70-0.75, none gte_0.80);
  full discrimination_hybrid_v1 source read (adjacent-populated-band inversion rule,
  >= comparators, coverage = covered/validDirectionalN). cross-check vs filled readout:
  identical.
- phase 2 sub-gate gap table with exact gaps (+5 disagreement; +2 covered rows for
  coverage incl. the 1.5x offset rule for uncovered additions; populated regions passing
  and monotonic; inversion expressed as the outcome condition acc(gte_0.80) >= 0.5382).
- phase 3 scenarios: mechanical minimum (5 ideal rows), observed-yield (30 runs, weak
  1/6 anchor; untargeted 0/7 and pooled 1/13 give a 30-65 band), sensitivity table
  (5%-33% -> 100-15 runs), discrimination recovery table (2 / ~15 / ~52 / never by assumed
  true rate) + the two structural fragilities.
- phase 4 cost projection from the documented 07-06 anchor ($0.004259 / 6 runs, metering
  estimate, not billing truth; no durable per-run sink) -> $0.011-$0.071 total band; noted
  real constraints = calendar + the-odds-api quota, not model spend.
- phases 5-8: report written to the required path/structure; validation; vault-only
  commit; push.

## 6. files changed

dai: none.
dai-vault (one commit):
- `06 Execution/reports/gate4-evidence-sufficiency-projection-2026-07-07-v1.md` (new)
- `06 Execution/reports/gate4-evidence-sufficiency-projection-handoff-2026-07-07-v1.md`
  (this file)

## 7. db writes / side effects

0 db writes (GET-only; /rows counts 279/118/116 identical before and after the slice).
no service state changed; stack left running.

## 8. paid calls / cost

0 paid model calls, $0.00. agent-service never started; only localhost reads.

## 9. validation proof

- every current value traceable: /rows fresh read (counts, regions, disagreement rows,
  coverage), pooled_calibration.py (thresholds, comparators, band rule), filled readout
  (cross-check: identical), capture doc v2 sections 5/16 (cost anchor).
- /metrics not used anywhere as a denominator.
- language check: no "demonstrated edge" (candidate edge signal only), no buyer-facing
  claims, no tuning/threshold recommendation (fragilities flagged for a separate review,
  explicitly not edited).
- dai git status: phantom csproj only; no code changed. no migrations. no captures.
- port 8000 not listening -> no paid path existed during the slice.

## 10. what did not change

pooled_calibration.py, discrimination_hybrid_v1, thresholds, prompts, routing, confidence,
models, buyer copy, registry flags (default-off), database contents, /metrics, dai repo.

## 11. open issues

- divergence-yield uncertainty dominates the projection (one observed hit; 95% CI on 1/6
  spans roughly 0.4%-64%); re-project at disagreement n=7.
- criterion fragilities need a home: (a) inversion can clear via bottom-region decay ->
  a TRUE could arrive on degraded performance (merit-verification language in the readout
  template is the current guard); (b) populating 0.70_0.74 (acc 0.75, n=8) would add an
  adjacency pair that currently registers a NEW inversion if its accuracy stays > 0.6382.
  both belong to a criterion interpretation/review slice, approval-gated, NOT this one.
- durable per-run cost sink still missing (stdout-only metering); Durable Cost Evidence v1
  remains a ranked interim slice from the 07-06 audit.
- known untracked vault noise (manifest json, synopsis) still intentionally uncommitted.

## 12. recommended next slice

**Evidence Acquisition Cadence Proposal v1** (docs-only, no-spend): the projection has the
numeric clarity cadence planning needs (runs/cohorts/cost/quota/calendar per scenario).
propose morning-window capture cadence with per-cohort approval gates, stop conditions,
and re-projection checkpoints at disagreement n=7 and n=10; carry the criterion
fragilities as inputs to a separate later criterion interpretation note.

## 13. suggested next prompt

"SLICE: Evidence Acquisition Cadence Proposal v1. mode: read-only, docs-only, no-spend, no
captures, no reconciliation, no code, no gate/criterion edits, push after verification.
read first: 06 Execution/reports/gate4-evidence-sufficiency-projection-2026-07-07-v1.md,
backed-depth-divergence-settlement-handoff-2026-07-07-v1.md, patterns/
cohort-selection-and-run-discipline-v1.md, and backed-depth-divergence-capture-plan +
readiness docs. deliverable: 06 Execution/plans/evidence-acquisition-cadence-proposal-v1.md
proposing capture cadence (frequency, window 10:00-13:00 ET, cohort size 6-12, per-cohort
approval gate preserved, guardrails MAX_PAID_RUNS 12 / $0.05 / registry canary
process-scoped), settlement-next-morning pairing, stop conditions (quota floor, yield
collapse, gate movement), and re-projection checkpoints at marketDisagreementN 7 and 10.
cadence is a PROPOSAL; each cohort still requires explicit operator approval. no capture
authorization is implied. end with a 13-section handoff."
