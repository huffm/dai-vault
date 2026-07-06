# HANDOFF: Backed-Depth Divergence Capture Readiness v1

## 1. Objective
No-spend preflight to make the next paid Backed-Depth Divergence Capture v2 (morning MLB, 10:00-13:00 ET) low-risk and mechanically ready. No paid generation, no gate changes.

## 2. Outcome
**READY — no blockers.** Calibration logic verified (discrimination_hybrid_v1 + exclusion filter present; 436 tests pass; live gate FALSE on merits; offline diagnostic has no runtime importer). Capture path verified (gpt-4o-mini single call, cost log, source-readiness + odds + StatsAPI + registry-canary all confirmed and reversible). Both repos are already fully pushed (0/0 — the prompt's "unpushed" note is stale). The only real risk is operational timing (must run in the morning ET pre-game window). Morning checklist + paste-ready capture prompt produced.

## 3. Repo State
### Before
- dai: main, `d79c38f`, dirty (pre-existing csproj phantom only), 0 ahead / 0 behind.
- dai-vault: main, `1aa0f4a`, 0 ahead / 0 behind, untracked `06 Execution/system-state-synopsis-v1.md`.
### After
- dai: main, `d79c38f` (UNCHANGED), same csproj phantom, 0 ahead / 0 behind.
- dai-vault: main, `<tip = this slice's docs commit>`, **1 ahead / 0 behind** (this readiness docs commit, unpushed), synopsis still untracked.

## 4. Services Used
- devcore-sql + DevCore.Api :5007: read-only health check (already up). agent-service :8000: confirmed DOWN (not needed; started only at capture time). sports-app: not started. No paid services.

## 5. Work Performed
- Verified repo state (both fully pushed; only known phantom/synopsis).
- Ran full agent-service suite (436 passed); confirmed live gate FALSE on merits; confirmed no runtime importer of pooled_calibration.
- Re-verified capture guardrails from source: gpt-4o-mini single call, cost-log sink, registry canary default-off + allowlist, source-readiness/odds/StatsAPI paths.
- Authored readiness report (with morning checklist) + this handoff; committed docs (no push).

## 6. Files Changed
- `dai-vault/06 Execution/reports/backed-depth-divergence-capture-readiness-2026-07-05-v1.md` — readiness report + checklist.
- `dai-vault/06 Execution/reports/backed-depth-divergence-capture-readiness-handoff-2026-07-05-v1.md` — this handoff.
- dai: none.

## 7. DB Writes / External Side Effects
- none. Read-only /rows + /metrics + service health. No AgentRuns, no writes, no external generation calls.

## 8. Paid Calls / Cost
- paid model calls: 0 | cost: $0.00 | proof: agent-service not started for generation; pytest offline; AgentRuns unchanged (273).

## 9. Validation Proof
- 436 agent-service tests passed.
- Live pooled summary: conclusionsAllowed=False, failingReasons [discrimination_inverted, insufficient_market_disagreement, insufficient_market_coverage].
- Guardrails: model="gpt-4o-mini" (sports_analyzer.py:647), 1 create() site, canary flag absent from .env, DEFAULT_ALLOWLIST includes backed_depth.
- Both repos 0 ahead / 0 behind (fully pushed). dai diff = only csproj phantom.

## 10. What Did Not Change
- prompts / prompt-registry recipes / routing / confidence generation / calibration gate / buyer copy / migrations-schema: unchanged.
- runtime behavior: unchanged (read-only preflight; no services started for generation; no flags flipped; .env untouched).

## 11. Open Issues
- Operational timing is the only risk: capture must run 10:00-13:00 ET (v1 failed by running too late).
- agent-service must be started at capture time with DAI_MLB_REGISTRY_PROMPT_CANARY=1 process-scoped, then reverted default-off.
- This readiness docs commit is unpushed (dai-vault 1 ahead). Push only on PUSH_APPROVED=true.
- Gate 4 remains FALSE (expected); the capture's purpose is to grow the readable disagreement + discrimination evidence.

## 12. Recommended Next Slice
Backed-Depth Divergence Capture v2 — paid, approval-gated, morning MLB pre-game window.

## 13. Suggested Next Prompt (paste-ready morning capture)
```text
SLICE: Backed-Depth Divergence Capture (PAID) v2
Date: <a game day>  |  RUN WINDOW: 10:00-13:00 ET (full pre-game slate required)
Mode: paid, approval-gated measurement capture.
PAID_CAPTURE_APPROVED=true
Params: SPORT=MLB, MAX_PAID_RUNS=12, MODEL_EXPECTED=gpt-4o-mini, MAX_MODEL_CALLS_PER_RUN=1,
TOTAL_COST_CAP_USD=0.05, SETTLEMENT_IN_THIS_SLICE=false, PUSH_ALLOWED=false, NAMED_GAMEPKS=auto.

Execute dai-vault/06 Execution/reports/backed-depth-divergence-capture-plan-2026-07-05-v1.md
using the checklist in backed-depth-divergence-capture-readiness-2026-07-05-v1.md.

Phase 0 — verify state: dai d79c38f (only csproj phantom), dai-vault fully pushed except any readiness docs; both should be clean/expected. Abort on unexpected drift.
Phase 1 — services (read-only): docker start devcore-sql -> wait SQL ready; DevCore.Api :5007 health 200. Do NOT start agent-service yet.
Phase 2 — guardrails (from source): model gpt-4o-mini (sports_analyzer.py:647), 1 create() call site, cost_log present, registry canary DAI_MLB_REGISTRY_PROMPT_CANARY default-off (absent from .env), DEFAULT_ALLOWLIST includes starter_enriched_market_backed_depth. STOP if any differs.
Phase 3 — slate: StatsAPI schedule for the date -> keep ONLY Pre-Game games; if < 4 pre-game -> report PARTIAL (too late) and stop. One the-odds-api h2h slate read -> per-game favorite + implied-prob gap + book count.
Phase 4 — prefilter: primary |gap| <= ~0.10, secondary >0.10 and <=0.15; EXCLUDE overwhelming favorites; require sufficient book count. Confirm each shortlisted game via GET /api/agent-runs/source-readiness (eligible starter_enriched_market_backed_depth, identity matched).
Phase 5 — FREEZE the slate doc (all considered + excluded + reasons + selected + pre-gen market favorite/gap/books + timestamp + caps) BEFORE any model call.
Phase 6 — generate: start agent-service with DAI_MLB_REGISTRY_PROMPT_CANARY=1 PROCESS-SCOPED (never .env), stdout -> log for cost lines. Generate <=12 via run-artifact-calibration.ps1 -Competition mlb. RECORD EVERY run (agreements + disagreements). Verify per run: promptSource=registry, selectedDataRegime=starter_enriched_market_backed_depth, model=gpt-4o-mini, 1 model call, identity present. STOP on any deviation / cost cap / missing market for too many games. Do NOT retry a game to change its lean.
Phase 7 — revert: restart agent-service DEFAULT-OFF (no env) or stop it; confirm .env has no registry flag.
Phase 8 — report + handoff: capture report (every run + agreement/disagreement counts + confidence distribution + evidenceRichness + identity safety + cost + settlement readiness). Do NOT settle this cohort (separate slice after finals).

Hard constraints: no prompt/routing/confidence/buyer/schema/migration change; registry default-off after; do not tune on results; do not push (PUSH_ALLOWED=false); no co-authored-by.
End with a continuation-grade handoff brief (per dai-agent-handoff canonical template).
```

---
Durable source of truth: `backed-depth-divergence-capture-readiness-2026-07-05-v1.md`. This handoff is the compressed resume point.
