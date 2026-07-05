---
title: "Backed-Depth Divergence Capture 2026-07-05 v1 -- BLOCKED (pre-spend)"
type: "report"
date: "2026-07-05"
status: "PARTIAL -- capture blocked before paid generation; 0 paid calls, 0 new runs; slate exhausted"
project: "DAI"
slice: "Backed-Depth Divergence Capture (PAID) v1"
repos:
  dai: "unchanged (c6d4f43)"
  dai-vault: "docs-only"
tags:
  - calibration
  - cohort
  - backed-depth
  - divergence
  - capture
related:
  - "06 Execution/reports/backed-depth-divergence-capture-plan-2026-07-05-v1.md"
  - "06 Execution/reports/backed-depth-divergence-candidate-slate-2026-07-05-v1.md"
  - "06 Execution/reports/backed-depth-divergence-capture-handoff-2026-07-05-v1.md"
---

# Backed-Depth Divergence Capture 2026-07-05 v1 -- BLOCKED (pre-spend)

## 1-3. title / date / objective

Execute the divergence capture plan: generate <=12 registry-routed backed_depth runs favoring plausible
DAI-market divergence, preserving a settlement-safe, measurable cohort. Paid, approval-gated
(`PAID_CAPTURE_APPROVED=true`).

## 4. prior calibration context

The prior 7-game backed_depth registry cohort (settled 2026-07-05) had DAI-market agreement on all 7 directional
games -> reconciliation mechanics validated, independent signal NOT validated. This slice sought games where
DAI-market divergence is plausible (close favorites) to enable an honest edge measurement later.

## 5. repo state before

- dai: `c6d4f43`, 0 ahead/0 behind, dirty only on pre-existing `DevCore.Data.csproj` phantom.
- dai-vault: `245e4dd`, 0 ahead/0 behind, untracked `06 Execution/system-state-synopsis-v1.md`.

## 6. services used

- devcore-sql: used (already up; DB reads only).
- DevCore.Api :5007: used (already up; `/source-readiness` screen + counts). health 200.
- agent-service :8000: **NOT started** (no generation reached). sports-app: not started.

## 7. cost/model guardrail proof (verified from source, pre-generation)

- model hardcoded `gpt-4o-mini` at `services/agent-service/app/services/sports_analyzer.py:647`.
- single model call per run: one `client.chat.completions.create` (line 650); no custom retry (SDK transient
  retries only, non-billing).
- cost metering present: `model_metering.py` (gpt-4o-mini 0.15/0.60 per 1M) + `cost_log` JSON line at
  `sports_analyzer.py:713`.
- caps confirmed: MAX_PAID_RUNS=12, TOTAL_COST_CAP_USD=0.05.
- registry route: canary env `DAI_MLB_REGISTRY_PROMPT_CANARY` (+ `_REGIMES`); `DEFAULT_ALLOWLIST` includes
  `starter_enriched_market_backed_depth`; absent from `.env` -> default-off. Never enabled this slice.

## 8. candidate selection method

NAMED_GAMEPKS=auto. StatsAPI schedule (free) for the 2026-07-05 slate + status; `/source-readiness` (app path) for
eligibility/identity/favorite/books on pre-game games; one the-odds-api h2h read for implied-probability gaps.

## 9. frozen slate summary

15-game slate; 13 already Final/In-Progress at 17:44 ET; **2 pre-game** (823931 SD@LAD, 824010 BOS@LAA). Both
eligible backed_depth (9 books, enriched, identity matched). Full detail + freeze:
`backed-depth-divergence-candidate-slate-2026-07-05-v1.md`.

## 10. generated runs

**None.** 0 runs generated.

## 11-14. distributions

Not applicable -- 0 runs generated. (Pre-generation market profile: 823931 LAD fav implied 0.690 gap 0.344; 824010
BOS fav implied 0.613 gap 0.188.)

## 15. identity safety / settlement readiness

Both pre-game candidates were identity-safe (StatsAPI gamePk matched). No cohort was created, so nothing is pending
settlement from this slice.

## 16. failures and exclusions

- 823931 SD@LAD -- EXCLUDED: overwhelming favorite (implied 0.690, gap 0.344); the plan says avoid overwhelming
  favorites.
- 824010 BOS@LAA -- EXCLUDED (marginal): moderate favorite (0.613, gap 0.188), ~2x the `<=0.10` close-favorite
  target; a single marginal game is not a divergence cohort.
- 13 games -- ineligible: already Final/In-Progress (not pre-game).

## 17. paid call count and cost

- paid model calls: **0**. estimated model cost: **$0.00**.
- external reads (screening only, no model): 2x `/source-readiness` (each triggers one the-odds-api call) + 1
  direct the-odds-api h2h slate read = ~3 the-odds-api quota units. No OpenAI spend.

## 18. what did not change

runtime code, model prompts, prompt registry recipes, routing, confidence logic, buyer copy, reconciliation logic,
schema/migrations: all unchanged. registry default-off: preserved (canary never enabled). 0 DB writes, 0 outcomes,
0 evaluations, 0 new AgentRuns.

## 19. validation proof

- AgentRuns count = 273 before and after (0 new runs).
- agent-service :8000 DOWN throughout (never started) -> 0 paid model calls.
- `$env:DAI_MLB_REGISTRY_PROMPT_CANARY` empty -> registry default-off preserved; `.env` untouched.
- `git status` dai: only the pre-existing csproj phantom; no runtime/registry/config file changed.

## 20. repo state after

- dai: `c6d4f43` (unchanged). dai-vault: `245e4dd` + this slice's docs commit (see handoff).

## 21. recommended next slice

Reschedule the paid divergence capture to a game day executed 10:00-13:00 ET (full pre-game slate available),
screen via `/source-readiness` + one the-odds-api slate read, select close favorites (gap `<= ~0.10`) first, keep
all guardrails identical. No edge conclusion yet; conclusionsAllowed remains false until a real divergence cohort
is captured and reconciled.
