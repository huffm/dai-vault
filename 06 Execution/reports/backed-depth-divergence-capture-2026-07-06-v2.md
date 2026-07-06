---
title: "Backed-Depth Divergence Capture 2026-07-06 v2 -- CAPTURED (6-run cohort, settlement pending)"
type: "report"
date: "2026-07-06"
status: "COMPLETE -- paid backed-depth divergence cohort captured; 6 runs, 1 divergence; settlement pending"
project: "DAI"
slice: "Backed-Depth Divergence Capture (PAID) v2"
repos:
  dai: "unchanged (d79c38f)"
  dai-vault: "docs-only"
tags:
  - calibration
  - cohort
  - backed-depth
  - divergence
  - capture
related:
  - "06 Execution/reports/backed-depth-divergence-candidate-slate-2026-07-06-v2.md"
  - "06 Execution/reports/backed-depth-divergence-capture-2026-07-05-v1.md"
  - "06 Execution/reports/backed-depth-divergence-capture-readiness-2026-07-05-v1.md"
  - "06 Execution/reports/backed-depth-divergence-capture-handoff-2026-07-06-v2.md"
---

# Backed-Depth Divergence Capture 2026-07-06 v2 -- CAPTURED

## 1. objective

Capture a bounded, settlement-safe cohort of registry-routed `backed_depth` MLB runs where
DAI-market divergence is plausible (close favorites), freezing the candidate slate before
paid generation. Measurement only -- no settlement, no tuning, no reconciliation. Paid,
approval-gated (`PAID_CAPTURE_APPROVED=true`).

## 2. prior blocked-v1 lesson

v1 (2026-07-05) ran at 17:44 ET; 13 of 15 games were already Final/In-Progress, leaving
only 2 pre-game (both overwhelming favorites). It blocked correctly pre-spend (0 runs). The
lesson -- run in the morning ET pre-game window -- was applied: this slice screened at
08:37 ET with all 8 games pre-game and ~5.5h of margin to first pitch.

## 3. repo state before

- dai: `d79c38f`, 0 ahead / 0 behind, dirty only on the pre-existing `DevCore.Data.csproj`
  phantom (empty textual diff).
- dai-vault: `7fef9e8`, 1 ahead / 0 behind (unpushed readiness docs commit), untracked
  `06 Execution/system-state-synopsis-v1.md`.
- No unexpected drift -> Phase 0 gate passed.

## 4. services used

- Docker Desktop: started (was down). devcore-sql container: started (was Exited).
- DevCore.Api :5007: built + run (`ASPNETCORE_ENVIRONMENT=Development`), health 200,
  upcoming + source-readiness + agent-runs endpoints used.
- agent-service :8000: started AFTER slate freeze, with the registry canary process-scoped;
  stopped after capture. sports-app: not started (not needed).

## 5. cost / model guardrail proof

- model: `gpt-4o-mini` on all 6 runs (from each run's `devcore.cost` log line).
- one model call per run: exactly 6 `sports model-call cost` lines for 6 runs.
- per-run estimated cost $0.00070-0.00074; **total estimated cost $0.004259** (8.5% of the
  $0.05 cap). Metering source: `model_metering.py` public-list estimate (not billing truth).
- caps respected: 6 <= MAX_PAID_RUNS 12; $0.004259 < $0.05.

## 6. registry / canary handling

- Enabled process-scoped only: `DAI_MLB_REGISTRY_PROMPT_CANARY=1` set as an inline env
  prefix on the uvicorn process (never written to `.env`, never persisted to config).
- Confirmed default-off before (absent from `.env`, shell unset) and after (still absent
  from `.env`; agent-service stopped).
- Every run's `promptRouteProvenance`: `promptSource=registry`,
  `registryAuthoritativeEnabled=true`, `legacyFallbackUsed=false`, `regimeAllowlisted=true`,
  `selectedDataRegime=starter_enriched_market_backed_depth`,
  `selectedPromptRecipeId=mlb.pregame.analysis.starter_enriched_market_backed_depth.v1`,
  `attributionStatus=complete`, 64-char `assembledHash`. No fallback occurred.

## 7. candidate selection method

`NAMED_GAMEPKS=auto`. StatsAPI schedule (free) for the 2026-07-06 slate + status; one
the-odds-api `baseball_mlb h2h` slate read for de-vigged implied probs + gaps; read-only
`GET /api/agent-runs/source-readiness` (app path, no model, no db write) per candidate for
identity / starter level / market level+books / predicted regime / eligibility. Divergence
prefilter: primary gap `<= ~0.10`, exclude gap `> 0.15`.

## 8. frozen slate summary

Frozen 2026-07-06 08:36:51 EDT, AgentRuns=273. 8 games considered, 6 selected (close
favorites, gap 0.017-0.090, all backed_depth / 9 books / identity matched / enriched
starters), 2 excluded (overwhelming favorites: PHI@KC gap 0.319, COL@LAD gap 0.324). Full
detail: `backed-depth-divergence-candidate-slate-2026-07-06-v2.md`. No paid generation
occurred before freeze.

## 9. generated runs

6 registry-routed `backed_depth` runs via the existing app path
(`POST /api/agent-runs`, runType `sports.matchup.analysis`). The generation script
`run-artifact-calibration.ps1` selects the first-N upcoming games, which on this slate
begin and end with the two excluded overwhelming favorites; to honor the frozen divergence
slate exactly, the six selected games were POSTed directly to the same `/api/agent-runs`
endpoint and body the script uses internally (identical app path, one model call each).

| # | gamePk | matchup (away @ home) | agentRunId | DAI lean | conf | evR | adv | market fav | favP | gap | agreement | promptSource | regime | cost$ |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 822958 | New York Yankees @ Tampa Bay Rays | ac31433e-f36b-1410-8175-00373db4b724 | home | 0.75 | 2 | High | Tampa Bay Rays (home) | 0.514 | 0.027 | AGREE | registry | backed_depth | 0.000707 |
| 2 | 822712 | Houston Astros @ Washington Nationals | ad31433e-f36b-1410-8175-00373db4b724 | home | 0.75 | 2 | High | Washington Nationals (home) | 0.522 | 0.043 | AGREE | registry | backed_depth | 0.000699 |
| 3 | 824900 | New York Mets @ Atlanta Braves | b331433e-f36b-1410-8175-00373db4b724 | home | 0.80 | 2 | High | Atlanta Braves (home) | 0.545 | 0.090 | AGREE | registry | backed_depth | 0.000713 |
| 4 | 823036 | Milwaukee Brewers @ St. Louis Cardinals | b431433e-f36b-1410-8175-00373db4b724 | home | 0.75 | 2 | High | Milwaukee Brewers (away) | 0.521 | 0.042 | **DISAGREE** | registry | backed_depth | 0.000702 |
| 5 | 823282 | Arizona Diamondbacks @ San Diego Padres | b631433e-f36b-1410-8175-00373db4b724 | home | 0.75 | 2 | High | San Diego Padres (home) | 0.509 | 0.019 | AGREE | registry | backed_depth | 0.000735 |
| 6 | 823205 | Toronto Blue Jays @ San Francisco Giants | b731433e-f36b-1410-8175-00373db4b724 | away | 0.75 | 2 | High | Toronto Blue Jays (away) | 0.508 | 0.017 | AGREE | registry | backed_depth | 0.000704 |

## 10. DAI-market agreement/disagreement summary

- **5 agree, 1 disagree** of 6.
- The disagreement -- 823036 Milwaukee @ St. Louis -- is the signal-bearing run: DAI leaned
  the home Cardinals while the market's (close) favorite was the away Brewers (gap 0.042).
  This is the first readable DAI-vs-market divergence in the backed_depth cohort; v1's
  settled 7-game cohort had DAI agreeing with market on all 7.
- No edge conclusion yet -- agreement/disagreement is directional capture only. Whether the
  one divergence (or the five agreements) were correct depends on finals and belongs to a
  later settlement slice.

## 11. confidence distribution

- 0.75 x5, 0.80 x1. Range 0.75-0.80. The lone 0.80 is an agreement run (824900 Mets@Braves).
  The single divergence run sits at 0.75.

## 12. evidence / source-depth distribution

- `evidenceRichness` = 2 on all 6. `advertisedStrength.band` = High on all 6.
- `sourceDepth` per run: starting_pitching enriched, market_odds enriched (9 books,
  book-disagreement ~3-9%), bullpen_availability shallow (relievers-used proxy).

## 13. market baseline summary

- All 6 selected games: 9 books, backed_depth, de-vigged favorite implied prob 0.508-0.545
  (close favorites), gap 0.017-0.090. source-readiness consensus side matched the-odds-api
  favorite on all six -> internally consistent baseline captured pre-generation and frozen.

## 14. identity safety / settlement readiness

- All 6 runs carry `sourceProvider=mlb_statsapi` and `ExternalGameId` = the StatsAPI gamePk
  (identity matched at retrieve time), `Status=completed`, `ExclusionReason=NULL` (active).
- The cohort is measurement-ready and settlement-ready: a later reconciliation slice can
  match each run to its official final by gamePk.

## 15. failures and exclusions

- No generation failures (6/6 completed, HTTP 200, no fallback, no warnings that blocked).
- Excluded pre-generation: 824089 PHI@KC (gap 0.319) and 823930 COL@LAD (gap 0.324) --
  overwhelming favorites, low divergence signal. Left un-screened for source-readiness on
  purpose (excluded on market gap first). No secondary-band (0.10-0.15) games existed.

## 16. paid call count and cost

- paid model calls: **6** (one per run). estimated total cost: **$0.004259** (< $0.05 cap).
- external screening reads (no model): 1 the-odds-api slate read + 6 source-readiness
  (each 1 the-odds-api unit) = 7 the-odds-api units; quota remaining 402.

## 17. what did not change

runtime code, model prompts, prompt registry recipes, routing logic, confidence generation,
buyer copy, calibration gate logic, reconciliation logic, schema/migrations: all unchanged.
registry default-off posture: preserved (`.env` untouched; canary process-scoped only,
removed on stop). `dai` HEAD `d79c38f`, working tree dirty only on the pre-existing csproj
phantom.

## 18. validation proof

- AgentRuns 273 -> **279** (+6, exactly the generated count).
- AgentRunOutcomes **112 -> 112** (0 written). AgentRunEvaluations **112 -> 112** (0 written).
- 6 `devcore.cost` lines, all `model=gpt-4o-mini`, one per run.
- All 6 `promptRouteProvenance.promptSource=registry`, `legacyFallbackUsed=false`,
  `selectedDataRegime=starter_enriched_market_backed_depth`.
- `.env` has no `DAI_MLB_REGISTRY_PROMPT_CANARY`; `git status` dai = only csproj phantom;
  HEAD unchanged.

## 19. repo state after

- dai: `d79c38f` (unchanged), dirty only on csproj phantom, 0 ahead / 0 behind.
- dai-vault: `7fef9e8` + this slice's docs commit (see handoff); untracked synopsis retained.

## 20. recommended next slice

**Backed-Depth Divergence Settlement / Reconciliation v1** -- after official finals for
2026-07-06 are available (evening ET / next morning), settle these 6 runs via the identity
`/reconcile` residue contract, writing outcomes/evaluations. Gate 4 still depends on future
settlement: this cohort adds 6 backed_depth runs and the first readable disagreement, but
`conclusionsAllowed` stays FALSE until settled evidence accumulates. No edge conclusion yet.
