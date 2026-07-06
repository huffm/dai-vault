# HANDOFF: Gate-4 Coverage Diagnostic + Sport Supply Expansion Audit v1

## 1. Objective
No-spend diagnostic: find the fastest/safest path to Gate 4 (Calibration Sufficiency) and decide whether sport-supply expansion (esp. WNBA) belongs in the measurement strategy. Real `/rows` counts + code-cited sport audit. No implementation.

## 2. Outcome
Completed. **Decisive finding: Gate 4 is NOT blocked by sample supply, slate timing, or market coverage — it is blocked by DAI's confidence distribution vs the gate criterion.** 3 of 4 gate conditions are MET (slates, enriched_market_missing, market-disagreement n=4); the ONLY failure is `not below_n` — confidence buckets 0.63/0.68/0.70/0.72 (~14 legacy/low-confidence runs) each <15. New backed_depth capture lands at 0.75/0.80 (already >=15), so it never fills them; verified that range-bucketing and registry-corpus-scoping still FAIL (low-confidence directional leans are structurally rare). Market backfill rejected (uncovered runs are old/unbackfillable). WNBA is the only other in-season sport (July) and is CaptureReadyWithSmallConfig but yields spread-baseline only (not backed_depth) and doesn't advance Gate 4.

## 3. Repo State
### Before
- dai: main, `c6d4f43`, dirty (pre-existing csproj phantom only), 0 ahead / 0 behind.
- dai-vault: main, `38d77bf`, 2 ahead / 0 behind, untracked `06 Execution/system-state-synopsis-v1.md`.
### After
- dai: main, `c6d4f43` (UNCHANGED), same csproj phantom, 0 ahead / 0 behind.
- dai-vault: main, `<tip = this slice's docs commit>`, **3 ahead / 0 behind**, synopsis still untracked.

## 4. Services Used
- devcore-sql + DevCore.Api :5007: used read-only (calibration /rows, already up). agent-service / sports-app: NOT started. No paid services.

## 5. Work Performed
- Verified repo state; read Gate-4 gate logic (`pooled_calibration.py`) + CLI.
- Pulled 273 calibration `/rows`; computed active/settled/directional counts, confidence buckets (2dp exact per the gate), market coverage, disagreement — real numbers.
- Tested 4 alternative bucket/corpus definitions against the gate (all still fail).
- Ran a parallel code-cited sport-supply audit (WNBA/NBA/NFL/NHL/EPL/NCAA).
- Wrote report + this handoff; committed docs (no push).

## 6. Files Changed
- `dai-vault/06 Execution/reports/gate4-coverage-sport-supply-diagnostic-2026-07-05-v1.md` — the diagnostic.
- `dai-vault/06 Execution/reports/gate4-coverage-sport-supply-diagnostic-handoff-2026-07-05-v1.md` — this handoff.
- dai: none.

## 7. DB Writes / External Side Effects
- AgentRuns created: 0 | Outcomes: 0 | Evaluations: 0 | DB writes: 0. Read-only `/rows` only. No external API calls.

## 8. Paid Calls / Cost
- paid model calls: 0 | estimated cost: $0.00 | proof: no agent-service started; read-only /rows; AgentRuns unchanged (273).

## 9. Validation Proof
- Gate-4 counts computed from live `/rows` (273): directional settled 92 (56c/36i); buckets 0.63(1)/0.68(5)/0.70(6)/0.72(2)/0.75(63)/0.80(15); only `not below_n` fails.
- Market coverage 52/92; 40 missing = 37 legacy MLB (no snapshot) + 3 market-missing nba -> NotBackfillable.
- Disagreement n=4 (2c/2i); backed_depth 2/23.
- dai `git status`: only pre-existing csproj phantom; no runtime/calibration/config file changed.

## 10. What Did Not Change
- prompts / routing / confidence logic / calibration rules / buyer copy / migrations-schema / reconciliation: unchanged.
- runtime behavior: unchanged (read-only; no services started, no flags flipped, no sport implemented).

## 11. Open Issues
- Gate 4 blocked by criterion-vs-distribution; unclearable by supply. conclusionsAllowed = FALSE.
- `pooled_calibration.py` does not filter ExclusionReason (latent infra gap; barely moves numbers).
- DAI-vs-market disagreement sample thin (4, split 2-2) — uninformative about edge.
- WNBA add is code-not-config (small branch widen) + spread-baseline only; season-label cross-year caveat.
- dai-vault has 3 unpushed commits (2 prior + this). Push only on instruction.

## 12. Recommended Next Slice
**Best (no-spend, operator-gated): Calibration Sufficiency Criterion Review v1** — the only lever on the actual Gate-4 blocker (supply can't clear it). Decide, on principle (not to force a pass), whether n>=15-per-exact-bucket is the right sufficiency test given clustered confidence; fix the ExclusionReason filter gap; consider range bucketing. **Runner-up (paid, morning): Backed-Depth Divergence Capture v2** to grow the disagreement sample (substantive, gate-independent). Defer WNBA enablement (do a no-spend feasibility doc if factory breadth is the goal); reject market backfill.

## 13. Suggested Next Prompt
```text
SLICE: Calibration Sufficiency Criterion Review v1
Mode: no-spend measurement-doctrine review. APPROVAL-GATED (touches calibration-sufficiency doctrine).
HARD: no paid calls, no new AgentRuns, no reconciliation writes, no change to decision/confidence PRODUCTION logic, no prompt/routing/buyer/schema change, do not push, no co-authored-by. This is a review + proposal; implement a gate change ONLY with explicit operator approval in this prompt.
Context (from gate4-coverage-sport-supply-diagnostic-2026-07-05-v1.md): Gate 4's only failing sub-gate is `not below_n` (pooled_calibration.py); it fails on sparse confidence buckets 0.63/0.68/0.70/0.72 that NO capture can fill (DAI confidence clusters at 0.75/0.80). Range-bucketing and registry-corpus-scoping both still fail. pooled_calibration.py does not filter ExclusionReason.
Phases:
0. Verify repo state (dai c6d4f43 + csproj phantom; dai-vault unpushed docs). Read-only services only.
1. Read evidence-readiness-gates-v1.md + confidence-calibration-rules-v1.md + pooled_calibration.py; restate the intent of the n>=15-per-bucket gate.
2. Enumerate options with evidence: (a) keep exact-2dp buckets (accept Gate 4 stays blocked until low-confidence samples accrue naturally — may be never); (b) range buckets; (c) assess sufficiency on populated buckets + a separately-tracked disagreement sample; (d) scope the denominator to the attribution-complete registry corpus; (e) fix the ExclusionReason filter. For each: does it clear the gate with CURRENT data? is it principled or goalpost-moving?
3. Recommend a principled criterion + whether Gate 4 clearance is even the right near-term target (it gates tuning + buyer-perf-claims, which the platform is not doing).
4. If (and only if) the operator approves a specific change in this prompt, implement it TDD with /metrics-impact stated; else deliver a decision memo only.
Validation: 0 writes, 0 new runs, 0 paid calls; cite counts; if code changed, tests green + state /metrics delta.
Deliver a vault report + continuation-grade handoff. Commit docs-only; do not push.
```

---
Durable source of truth: `gate4-coverage-sport-supply-diagnostic-2026-07-05-v1.md` (full tables + evidence). This handoff is the compressed resume point.
