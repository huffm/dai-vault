---
title: "Backed-Depth Divergence Capture Readiness v1"
type: "report"
date: "2026-07-05"
status: "READY -- no blockers; morning MLB paid capture is mechanically prepared; no-spend preflight"
project: "DAI"
slice: "Backed-Depth Divergence Capture Readiness"
repos:
  dai: "unchanged (d79c38f) -- read-only readiness"
  dai-vault: "docs-only (this report + handoff)"
tags:
  - calibration
  - capture
  - backed-depth
  - divergence
  - readiness
related:
  - "06 Execution/reports/backed-depth-divergence-capture-plan-2026-07-05-v1.md"
  - "06 Execution/reports/gate4-discrimination-sufficiency-criterion-2026-07-05-v1.md"
  - "06 Execution/reports/backed-depth-divergence-capture-2026-07-05-v1.md"
---

# Backed-Depth Divergence Capture Readiness v1

## 1. objective

No-spend preflight to make the next paid **Backed-Depth Divergence Capture v2** (morning MLB, 10:00-13:00 ET)
low-risk and mechanically ready. No paid generation, no new runs, no gate changes.

## 2. repo state

- dai: `d79c38f`, **0 ahead / 0 behind** (fully pushed); dirty only on pre-existing `DevCore.Data.csproj` phantom
  (empty diff, intentionally untouched).
- dai-vault: `1aa0f4a`, **0 ahead / 0 behind** (fully pushed); untracked `06 Execution/system-state-synopsis-v1.md`
  (pre-existing, not this slice).
- **Note:** the prompt's "known state" (dai >=1 unpushed, dai-vault several unpushed) is STALE -- both were pushed
  in the prior slice. Nothing is unpushed now. Classification: **safe for morning capture.**

## 3. recent context

Gate-4 discrimination criterion shipped (`discrimination_hybrid_v1` + exclusion filter; 436 tests; /metrics
byte-identical; live conclusionsAllowed=FALSE on the merits). Coverage/sport-supply diagnostic showed Gate 4 is
distribution/criterion-bound, not raw-supply-bound; WNBA is spread-only (not backed_depth). Divergence Capture v1
blocked (ran too late; only 2 non-close pre-game candidates). v2 = morning MLB, approval-gated.

## 4. calibration readiness -- READY

- `discrimination_hybrid_v1` present in `pooled_calibration.py` (criterion ref confirmed).
- ExclusionReason filtering present (drops excluded rows up front; `counts.excludedRowCount`).
- New-criterion tests present; **full agent-service suite = 436 passed**.
- Live data still fails on the merits: `conclusionsAllowed=False`, failingReasons
  `[discrimination_inverted, insufficient_market_disagreement, insufficient_market_coverage]`.
- **No runtime path imports the changed offline diagnostic** -- grep confirms `pooled_calibration` is imported only
  by its own module, the CLI (`scripts/pooled_calibration_report.py`), and its test. `/metrics` (the .NET endpoint)
  is unaffected.

## 5. capture path readiness -- READY

- Model: **`gpt-4o-mini`** hardcoded (`sports_analyzer.py:647`); **1** `chat.completions.create` call site (single
  model call per run; SDK transient retries are non-billing).
- Cost logging: `cost_log.info` sink present (`sports_analyzer.py`) -> capture via redirected agent-service stdout.
- Source-readiness screen: `GET /api/agent-runs/source-readiness?competition=mlb&homeTeam=&awayTeam=&gameDate=`
  (app-path, reuses generation retrieval; no model call/write). Returns identity, starter level, market level +
  BookCount + ConsensusSide, eligibility for `starter_enriched_market_backed_depth`.
- Odds screen: one the-odds-api h2h read (`baseball_mlb`, regions=us, markets=h2h) for favorite + implied-prob gap
  + book count across the slate.
- StatsAPI identity: `statsapi.mlb.com` schedule (gamePk, status, probables) for pre-game filtering.
- Generation: `scripts/dev/sports/run-artifact-calibration.ps1 -Competition mlb` (posts to DevCore.Api :5007, which
  calls agent-service; one model call per run).

## 6. cost/model guardrail readiness -- READY

Model gpt-4o-mini; <=1 model call/run; MAX_PAID_RUNS=12; TOTAL_COST_CAP_USD=0.05 (~$0.002/call, cost dominated by
the-odds-api quota, not model). STOP conditions: fallback / promptSource != registry / model != gpt-4o-mini /
cost-cap / missing identity / market missing for too many games.

## 7. registry/canary readiness -- READY, reversible

- Registry backed_depth route enabled by env **`DAI_MLB_REGISTRY_PROMPT_CANARY=1`** (+ optional `_REGIMES`;
  `DEFAULT_ALLOWLIST` already includes `starter_enriched_market_backed_depth`).
- **Default-off preserved:** the flag is ABSENT from `services/agent-service/.env` (0 matches). Enable it
  **process-scoped only** when starting agent-service for the capture window, then restart agent-service WITHOUT the
  env (or stop it) to restore default-off. Never write the flag to `.env`.
- Reversibility: stopping/​restarting agent-service without the env fully reverts; no config drift.

## 8. morning operating checklist (10:00-13:00 ET)

```
PRECHECK (no spend)
[ ] git state: dai d79c38f (only csproj phantom), dai-vault 1aa0f4a (synopsis untracked); both 0/0. Abort if unexpected drift.
[ ] services: docker start devcore-sql -> wait SQL ready; DevCore.Api :5007 health 200.
[ ] StatsAPI schedule for today -> keep ONLY Pre-Game games. If < 4 pre-game -> report PARTIAL (too late), stop.
[ ] one the-odds-api h2h slate read -> per-game favorite + implied-prob gap + book count.
[ ] divergence prefilter: primary |gap| <= ~0.10, secondary >0.10 and <=0.15; EXCLUDE overwhelming favorites; need book count sufficient.
[ ] GET /source-readiness per shortlisted game -> eligible starter_enriched_market_backed_depth + identity matched + BookCount adequate.
[ ] FREEZE the candidate slate doc (all considered + excluded + reasons + selected) BEFORE any model call.

CAPTURE (paid, <=12 runs, $0.05 cap)  -- only after operator PAID_CAPTURE_APPROVED=true
[ ] start agent-service with DAI_MLB_REGISTRY_PROMPT_CANARY=1 PROCESS-SCOPED (never .env); stdout -> log for cost lines.
[ ] generate <=12 via run-artifact-calibration.ps1 -Competition mlb; RECORD EVERY run (agreements + disagreements).
[ ] verify per run: promptSource=registry, selectedDataRegime=starter_enriched_market_backed_depth, model=gpt-4o-mini, 1 model call, identity present. STOP on any deviation.
[ ] after capture: restart agent-service DEFAULT-OFF (no env) / or stop it; confirm .env still has no registry flag.

CLOSE
[ ] write capture report (every run + agreement/disagreement counts + cost + identity safety + settlement readiness).
[ ] do NOT settle this cohort in the capture slice (separate settlement slice after StatsAPI finals).
[ ] continuation-grade handoff. Commit docs. Do not push unless PUSH_APPROVED=true.
```

Candidate preference: pre-game only; stable StatsAPI identity; backed_depth source-readiness eligible; book count
sufficient; implied-prob gap primary <=~0.10, secondary >0.10 and <=0.15; exclude overwhelming favorites; freeze
before generation; record every run; no retries to change lean; no settlement in-slice.

## 9. blockers -- NONE (before morning)

| candidate blocker | status | severity | fix |
|---|---|---|---|
| repo dirty beyond phantom | none (only phantom) | - | n/a |
| unpushed code commit | none (all pushed) | - | n/a |
| test failures | none (436 passed) | - | n/a |
| cost logging | present | - | n/a |
| model mismatch | gpt-4o-mini confirmed | - | n/a |
| registry route uncertainty | confirmed (env-scoped, allowlisted) | - | n/a |
| source-readiness endpoint | confirmed (app-path) | - | n/a |
| canary override | confirmed (process-scoped, reversible) | - | n/a |
| stale services | devcore-sql + DevCore.Api up; agent-service started at capture time | low (operational) | start at capture |
| API keys | odds key present (used read-only in v1); OpenAI key present (agent-service .env) | low | verify at capture |
| candidate slate process | defined (StatsAPI -> odds -> source-readiness -> freeze) | - | n/a |
| handoff/report convention | continuation-grade standard in place | - | n/a |

**The only real risk is operational timing** -- the capture MUST run in the morning ET pre-game window (the v1
failure mode). No code/repo blocker exists.

## 10. push readiness

**Nothing to push** -- both repos are 0 ahead / 0 behind (the prior slice pushed dai `d79c38f` + dai-vault
`1aa0f4a`). `PUSH_APPROVED=true` was not provided; no push performed. This readiness slice's docs commit will be a
new unpushed commit, safe to push later.

## 11. validation proof

Read-only. 436 agent-service tests passed; live pooled summary conclusionsAllowed=False (merits); grep-confirmed no
runtime importer of the offline diagnostic; model/single-call/canary/cost-log confirmed from source; services
health-checked read-only. 0 paid calls, 0 new AgentRuns, 0 DB writes, agent-service NOT started.

## 12. what did not change

No runtime code, prompts, prompt registry recipes, routing, confidence generation, calibration gate logic, buyer
copy, schema/migrations. No new AgentRuns, no DB writes, no paid calls, agent-service/sports-app not started. dai
unchanged at `d79c38f`.

## 13. recommended next slice

**Backed-Depth Divergence Capture v2** -- paid, approval-gated, executed in the 10:00-13:00 ET pre-game window,
following the checklist above and the capture plan. Paste-ready prompt in the handoff.
