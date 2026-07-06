# HANDOFF: Calibration Sufficiency Criterion Review v1

## 1. Objective
No-spend, operator-gated review: is Gate 4's rule (n>=15 in every exact 2dp confidence bucket) the right sufficiency criterion given DAI's confidence distribution? Decide on principle without gaming the gate. No gate change implemented.

## 2. Outcome
**The exact-2dp-bucket rule is a legitimate criterion defect — it is structurally unsatisfiable** (DAI's leans cluster at 0.75/0.80, so the low-confidence buckets 0.63/0.68/0.70/0.72 never reach 15; the gate would report FALSE forever, even if real discrimination later appeared). Recommendation: **replace it with a discrimination-based hybrid criterion + fix the ExclusionReason filter, in one later approved slice. Net effect today: Gate 4 stays FALSE** (confidence shows no discrimination — 0.75 bucket 62% vs 0.80 bucket 53%; DAI-vs-market disagreement is 2-2 noise). A change that leaves the verdict FALSE is provably not goalpost-moving. Also recommend making Gate 4's by-purpose split explicit (block tuning/buyer-claims/model-replacement/Stage-3-mutation; allow capture + diagnostics).

## 3. Repo State
### Before
- dai: main, `c6d4f43`, dirty (pre-existing csproj phantom only), 0 ahead / 0 behind.
- dai-vault: main, `d2a2411`, 3 ahead / 0 behind, untracked `06 Execution/system-state-synopsis-v1.md`.
### After
- dai: main, `c6d4f43` (UNCHANGED), same csproj phantom, 0 ahead / 0 behind.
- dai-vault: main, `<tip = this slice's docs commit>`, **4 ahead / 0 behind**, synopsis still untracked.

## 4. Services Used
- devcore-sql + DevCore.Api :5007: read-only (calibration /rows, already up). agent-service / sports-app: NOT started. No paid services.

## 5. Work Performed
- Verified state; read `pooled_calibration.py` gate logic + CLI + `confidence-calibration-rules-v1.md` + evidence-readiness-gates.
- Recomputed 273 `/rows`: exact buckets, confidence bands, active-bucket thresholds, route-specific, discrimination (monotonicity), exclusion impact.
- Compared 6 candidate criteria (A-F) against current data; ran the gate-purpose review; made the decision.
- Wrote report + this handoff; committed docs (no push).

## 6. Files Changed
- `dai-vault/06 Execution/reports/calibration-sufficiency-criterion-review-2026-07-05-v1.md` — the review.
- `dai-vault/06 Execution/reports/calibration-sufficiency-criterion-review-handoff-2026-07-05-v1.md` — this handoff.
- dai: none.

## 7. DB Writes / External Side Effects
- none. Read-only `/rows`. No AgentRuns, no writes, no external calls.

## 8. Paid Calls / Cost
- paid model calls: 0 | cost: $0.00 | proof: no agent-service; read-only; AgentRuns unchanged (273).

## 9. Validation Proof
- Discrimination: populated buckets 0.75 (n=63, acc 0.619) and 0.80 (n=15, acc 0.533) -> no positive discrimination (mild inversion within noise).
- Candidate verdicts (current data): A FAIL (unsatisfiable), B 5% FAIL / 10% PASS (gaming dial), C FAIL (low n=6, med n=8), D FAIL (registry n=25 / legacy n=66), F FAIL (inversion + 2-2 disagreement).
- Exclusion impact: only 2 excluded directional-settled rows (both 0.75) -> verdict unchanged.
- dai `git status`: only csproj phantom; no runtime/calibration file changed.

## 10. What Did Not Change
- prompts / routing / confidence logic / calibration rules-gate / buyer copy / migrations-schema / reconciliation: unchanged.
- runtime behavior: unchanged (read-only; no gate change implemented; no services started).

## 11. Open Issues
- Gate 4 rule is broken (can never turn TRUE on real evidence) but currently returns the right answer (FALSE) for the wrong reason — replace it, but the fix doesn't unlock anything today.
- ExclusionReason not filtered in pooled_calibration.py (harmless now; fold fix into the criterion slice).
- Confidence shows no discrimination yet; disagreement sample thin (2-2). Both need more data (morning capture).
- dai-vault has 4 unpushed commits (3 prior + this). Push only on instruction.

## 12. Recommended Next Slice
**Best: Gate-4 Discrimination-Based Sufficiency Criterion v1** (implement the hybrid criterion + exclusion-filter fix, TDD; prove it returns FALSE on current data and TRUE only on a constructed discriminating corpus; /metrics byte-identical). **Runner-up: Backed-Depth Divergence Capture v2** (paid, morning) to grow the disagreement split the new gate needs. Defer buyer packaging / WNBA / more Interrogate.

## 13. Suggested Next Prompt
```text
SLICE: Gate-4 Discrimination-Based Sufficiency Criterion v1
Mode: implementation (calibration measurement-infra). APPROVAL: this changes Gate 4 sufficiency logic — proceed only if operator approves in this prompt; else deliver a design+test-plan memo only.
HARD: no change to decision/confidence PRODUCTION logic (SportsEvaluator constants untouched), no prompts/routing/buyer/schema change, no paid calls, no new AgentRuns, no reconciliation writes, do not push, no co-authored-by.
Context (calibration-sufficiency-criterion-review-2026-07-05-v1.md): the exact-2dp-bucket gate is structurally unsatisfiable (low-confidence buckets never fill); replace it with a discrimination-based hybrid that still returns FALSE today (0.75=62% vs 0.80=53% -> no discrimination; disagreement 2-2).
Phases:
0. Verify repo state (dai c6d4f43 + csproj phantom; dai-vault unpushed docs). Read-only services only.
1. In services/agent-service/app/services/pooled_calibration.py, add: (a) ExclusionReason filtering in _is_reconciled (drop rows with exclusionReason); (b) a hybrid sufficiency gate = total directional-settled >= N AND >=2 populated confidence regions (n>=15 each) AND no-severe-accuracy-inversion across populated regions AND marketDisagreement readable-split AND market-baseline coverage >= threshold. Keep the old fields for continuity; add the new gate as the authoritative conclusionsAllowed.
2. TDD: unit tests proving conclusionsAllowed=False on the current /rows corpus (fixture), True on a synthetic discriminating corpus, and that excluded rows are dropped.
3. Make the by-purpose split explicit in the summary output (blocks: tuning/buyer-claims/model-replacement/Stage-3; allows: capture/diagnostics) — reporting only, no enforcement change.
4. Prove /metrics BYTE-IDENTICAL (no runtime decision surface touched); pytest green.
Validation: 0 writes, 0 new runs, 0 paid calls; state test counts + /metrics delta (expect none).
Deliver a vault report + continuation-grade handoff. Commit docs+code separately; do not push.
```

---
Durable source of truth: `calibration-sufficiency-criterion-review-2026-07-05-v1.md` (full candidate table + discrimination evidence). This handoff is the compressed resume point.
