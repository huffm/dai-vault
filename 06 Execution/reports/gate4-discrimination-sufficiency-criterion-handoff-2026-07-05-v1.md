# HANDOFF: Gate-4 Discrimination-Based Sufficiency Criterion v1

## 1. Objective
Approval-gated (`GATE4_CRITERION_CHANGE_APPROVED=true`) TDD implementation: replace the structurally defective exact-2dp confidence-bucket Gate-4 rule with a discrimination-based hybrid criterion, and fix the ExclusionReason filtering gap in `pooled_calibration.py`. Must return `conclusionsAllowed=False` on current live data.

## 2. Outcome
Shipped. New `discrimination_hybrid_v1` criterion + exclusion filter in `pooled_calibration.py`. TDD: 9 new tests red → implement → 15 green; full agent-service suite **436 passed**. **On live data `conclusionsAllowed=False` on the merits** — failingReasons `[discrimination_inverted, insufficient_market_disagreement, insufficient_market_coverage]` (0.75 region acc 0.619 vs 0.80 region acc 0.533 = inverted; disagreement 4<10; coverage 0.565<0.60). 16 excluded rows now correctly dropped. Nothing new unlocked → change is not goalpost-moving.

## 3. Repo State
### Before
- dai: main, `c6d4f43`, dirty (pre-existing csproj phantom only), 0 ahead / 0 behind.
- dai-vault: main, `f4b6953`, 4 ahead / 0 behind, untracked `06 Execution/system-state-synopsis-v1.md`.
### After
- dai: main, **`d79c38f`** (commit: fix(calibration): use discrimination-based gate4 sufficiency), same csproj phantom uncommitted, **1 ahead / 0 behind**.
- dai-vault: main, `<tip = this slice's docs commit>`, **5 ahead / 0 behind**, synopsis still untracked.

## 4. Services Used
- devcore-sql + DevCore.Api :5007: read-only (/metrics + /rows spot-check, already up). agent-service / sports-app: NOT started (pytest runs standalone). No paid services.

## 5. Work Performed
- Verified state; confirmed pooled_calibration.py has no runtime/endpoint consumer (offline CLI + test only).
- TDD: wrote 10 criterion tests (+ preserved 5 structural), ran red (9 fail), implemented, ran green (15).
- Implemented `discrimination_hybrid_v1` (band regions + non-inversion discrimination + disagreement readability + coverage + min-sample) and ExclusionReason filtering, additively.
- Live-validated against /rows (273): FALSE on the merits; ran full suite (436 passed); confirmed /metrics unchanged.
- Wrote report + this handoff; committed dai code+tests and dai-vault docs (no push).

## 6. Files Changed
- `dai/services/agent-service/app/services/pooled_calibration.py` — discrimination_hybrid criterion + exclusion filter (offline diagnostic).
- `dai/services/agent-service/tests/test_pooled_calibration.py` — 10 new criterion tests + preserved structural tests.
- `dai-vault/06 Execution/reports/gate4-discrimination-sufficiency-criterion-2026-07-05-v1.md` — implementation report.
- `dai-vault/06 Execution/reports/gate4-discrimination-sufficiency-criterion-handoff-2026-07-05-v1.md` — this handoff.

## 7. DB Writes / External Side Effects
- none. No AgentRuns, no outcomes/evaluations, no DB writes. /rows + /metrics read-only.

## 8. Paid Calls / Cost
- paid model calls: 0 | cost: $0.00 | proof: no agent-service started for generation; pytest offline; AgentRuns unchanged (273).

## 9. Validation Proof
- TDD: 9 red → 15 green; full agent-service suite 436 passed.
- Live: conclusionsAllowed=False; failingReasons [discrimination_inverted, insufficient_market_disagreement, insufficient_market_coverage]; excludedRowCount=16; slatesMet/emmMet preserved true.
- /metrics unaffected: totalRows 273 / reconciled 94 / matched 57 / matchRate 0.6064 (.NET endpoint untouched).
- dai diff = pooled_calibration.py + test + pre-existing csproj phantom only.

## 10. What Did Not Change
- confidence generation (SportsEvaluator constants), prompts, prompt registry recipes, routing, buyer copy, schema/migrations, ProbeRefresh, artifact generation, the .NET /metrics endpoint: unchanged.
- runtime behavior: unchanged (pooled_calibration.py is offline diagnostic-only; no importer other than CLI+test).

## 11. Open Issues
- Gate 4 remains FALSE (correct): confidence not yet informative (0.80 doesn't beat 0.75); disagreement thin (4); coverage 0.565.
- To move it, need a corpus with positive confidence discrimination + >=10 readable disagreements + >=60% coverage — the morning divergence capture is the instrument to grow disagreement.
- Thresholds (40/2/15/0.05/10/0.60) are conservative first choices; revisit once more data accrues.
- dai has 1 unpushed commit; dai-vault 5 unpushed. Push only on instruction.

## 12. Recommended Next Slice
Backed-Depth Divergence Capture v2 (paid, morning MLB, approval-gated) — grows the readable DAI-vs-market disagreement sample the new gate needs (and provides positive-discrimination evidence). The gate is now unbroken, so future evidence can actually move it.

## 13. Suggested Next Prompt
```text
SLICE: Backed-Depth Divergence Capture (PAID) v2
RUN WINDOW: 10:00-13:00 ET on a game day (full pre-game slate required)
PAID_CAPTURE_APPROVED=true | Caps: max 12 paid runs, gpt-4o-mini only, 1 model call/run, total cost cap $0.05.
Execute dai-vault/06 Execution/reports/backed-depth-divergence-capture-plan-2026-07-05-v1.md:
1. Verify repo state (dai d79c38f + csproj phantom; dai-vault unpushed docs). No unexpected drift.
2. StatsAPI schedule -> keep ONLY Pre-Game games; abort if <4 pre-game (report PARTIAL).
3. One the-odds-api h2h slate read -> favorite + implied-prob gap + book count per game.
4. Divergence prefilter: prefer close favorites (|gap| <= ~0.10), mixed books; EXCLUDE overwhelming favorites.
5. Confirm shortlist via GET /api/agent-runs/source-readiness (eligible starter_enriched_market_backed_depth, identity matched).
6. FREEZE the slate doc BEFORE any model call.
7. agent-service with DAI_MLB_REGISTRY_PROMPT_CANARY=1 process-scoped; generate <=12 via run-artifact-calibration.ps1 -Competition mlb; record EVERY run; restart agent-service default-off.
8. Do NOT settle this cohort (separate slice after finals).
Hard constraints: no prompt/routing/confidence/buyer/schema/migration change; STOP on any fallback / promptSource!=registry / model!=gpt-4o-mini / cost-cap / missing identity; do not push; no co-authored-by.
End with a continuation-grade handoff brief.
```

---
Durable source of truth: `gate4-discrimination-sufficiency-criterion-2026-07-05-v1.md`. This handoff is the compressed resume point.
