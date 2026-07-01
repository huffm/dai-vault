---
title: "Controlled Live Batch Capture v2"
type: "evidence-report"
date: "2026-06-30"
status: "complete"
project: "DAI"
slice: "Controlled Live Batch Capture v2"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - prompt-registry
  - calibration
  - capture
related:
  - "06 Execution/reports/targeted-live-batch-capture-v1.md"
---

# Controlled Live Batch Capture v2 -- Evidence Report

**status:** complete (8-run approved paid batch; all registry-authoritative; pending reconciliation)
**date:** 2026-06-30

## purpose

Identify today's best live MLB candidates, get explicit operator approval before paid calls, then run the
approved batch through the real .NET -> FastAPI path with durable prompt-route provenance and metrics
verification. Live evidence capture only -- no prompt tuning, no code change, no reconciliation (games not Final).

## start state

- `dai`: clean, synced, `0f563d6` (0/0). `dai-vault`: clean, synced, `d1ced0f` (0/0).
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- `DEFAULT_ALLOWLIST` unchanged -- exactly four regimes.
- Calibration Route Attribution Fix v1 present (PromptRouteCalibrationExport.cs line 147:
  `return $"{regime}::{fallback}";`).

## service / db status

- `devcore-sql` already up (1433). `AgentRuns.PromptRouteProvenanceJson` present + nullable; migration applied.
- FastAPI agent-service started: `uvicorn main:app :8000`, `DAI_MLB_REGISTRY_PROMPT_CANARY=1`,
  `DAI_MLB_ROUTE_PROVENANCE_PATH=<scratch>\route-provenance-v2.jsonl`. `/api/ping` healthy.
- .NET API started: `dotnet run :5215` (Development, dev bypass auth, tenant 1). `/health` ok.

## free schedule probe

`statsapi.mlb.com/api/v1/schedule?sportId=1&date=2026-06-30&hydrate=probablePitcher` (free, non-paid): 15 games,
all Scheduled. 14 both-starters-announced (enriched); 1 TBD starter (825066 Giants@Dbacks -> starter_missing).
Dedupe vs AgentRuns: `824338` (Marlins@Rockies) and `825066` (Giants@Dbacks) already captured as soak runs and
excluded. The only starter_missing game today was therefore already captured -- the slate offers no fresh
starter_missing diversity, so none was fabricated. The 13 remaining new games are all enriched.

## candidate table (approved batch -- 8 new enriched games)

| # | gamePk | away (SP) @ home (SP) | start UTC | expected regime | dupe |
|---|---|---|---|---|---|
| 1 | 824907 | Cardinals (Liberatore) @ Braves (Pérez) | 06-30 23:15 | starter_enriched_market_backed_depth | new |
| 2 | 824096 | Rays (Jax) @ Royals (Cameron) | 06-30 23:40 | starter_enriched_market_backed_depth | new |
| 3 | 824175 | Twins (Ryan) @ Astros (Burrows) | 07-01 00:10 | starter_enriched_market_backed_depth | new |
| 4 | 824984 | Dodgers (Wrobleski) @ Athletics (Springs) | 07-01 01:40 | starter_enriched (market sub TBD) | new |
| 5 | 823122 | Angels (Soriano) @ Mariners (Woo) | 07-01 01:40 | starter_enriched_market_backed_depth | new |
| 6 | 824661 | Padres (Sears) @ Cubs (Boyd) | 07-01 00:05 | starter_enriched_market_backed_depth | new |
| 7 | 823528 | Tigers (Skubal) @ Yankees (Schlittler) | 06-30 23:05 | starter_enriched_market_backed_depth | new |
| 8 | 822793 | Mets (McLean) @ Blue Jays (Gausman) | 06-30 23:07 | starter_enriched_market_backed_depth | new |

4 matchups recur from the 06-29 series but with NEW gamePks (distinct games, not duplicates). The other 4 are
fresh matchups (Cardinals@Braves, Rays@Royals, Twins@Astros, Dodgers@Athletics, Angels@Mariners) chosen for team
diversity vs the prior cohort.

## approval gate

Presented candidate table, recommended size, expected regimes, dupe-check result, expected model calls
(1/run), estimated cost (<$0.02), env vars, exact POST flow, and verification plan. **Operator approved 8 runs.**
No paid calls were made before approval.

## run evidence (all 8 completed)

| AgentRunKey | runId | game | gamePk | status | lean |
|---|---|---|---|---|---|
| 270013 | c240433e | Cardinals @ Braves | 824907 | completed | home |
| 270014 | c440433e | Rays @ Royals | 824096 | completed | away |
| 270015 | ca40433e | Twins @ Astros | 824175 | completed | away |
| 270016 | cf40433e | Dodgers @ Athletics | 824984 | completed | away |
| 270017 | d040433e | Angels @ Mariners | 823122 | completed | home |
| 270018 | d640433e | Padres @ Cubs | 824661 | completed | home |
| 270019 | db40433e | Tigers @ Yankees | 823528 | completed | home |
| 270020 | de40433e | Mets @ Blue Jays | 822793 | completed | home |

## prompt route provenance evidence

All 8 runs persisted non-null `PromptRouteProvenanceJson`. **8/8 registry-authoritative:**
`promptSource=registry`, `selectedDataRegime=starter_enriched_market_backed_depth`,
`selectedPromptRecipeId=mlb.pregame.analysis.starter_enriched_market_backed_depth.v1`, `version=v1`,
`fallbackReason=null`, `legacyFallbackUsed=false`. **No fallback occurred this batch** (no assembly_error, no
mismatch). `GET /api/agent-runs/{id}/artifact` verified to expose `promptRouteProvenance` (run c240433e:
src=registry, recipe ...backed_depth.v1, ver v1, assembledHash 107c1b9d81b7...).

Route diversity: none at the regime level -- the slate was enriched-heavy with liquid markets, so all 8 routed
to the same `starter_enriched_market_backed_depth` regime. This is honest reporting, not forced diversity.

## metrics before / after

`GET /api/agent-runs/prompt-route-calibration/metrics` (tenant 1):

| metric | before | after |
|---|---|---|
| totalRows | 246 | **254** (+8) |
| reconciledRows | 76 | 76 |
| unreconciledRows | 162 | **170** (+8) |
| matchedRows | 47 | 47 |
| unmatchedRows | 29 | 29 |
| matchRate | 0.6184 | 0.6184 |
| registryRows | 10 | **18** (+8) |
| liveRows | 1 | 1 |
| fallbackRows | 1 | 1 |
| unknownRouteRows | 235 | 235 |

Route `starter_enriched_market_backed_depth`: total 7 -> **15** (reconciled 7, unreconciled 8 = the new pending
games, matched 6, unmatched 1, registryRows 15). All deltas as expected: totalRows +8, unreconciled +8,
registryRows +8; matchRate unchanged (no new outcomes); liveRows/fallbackRows unchanged (no fallback).

## fallback route-key verification

No fallback occurred this batch, so no new fallback row was created. The pre-existing fallback route
`starter_enriched_market_backed_depth::assembly_error` (from run 260018, prior slice) remains correctly
attributed under its fallback key (total 1, src=live, reg=0, live=1, fb=1) -- NOT collapsed to `unknown`,
confirming Calibration Route Attribution Fix v1 holds after this batch.

## paid-call count and estimated cost

**8 paid gpt-4o-mini calls -- one completed analysis per run** (agent log: 8 "mlb analysis response" + 8
"mlb prompt routing decision"). The OpenAI SDK performed **1 automatic transient retry**
(`openai._base_client :: Retrying request to /chat/completions in 0.43s`) on a single request -- no 429 / rate
limit, no duplicate run or artifact, and it resolved to exactly 8 completed analyses. Documented per the
no-blind-retry rule. Estimated cost: **< $0.02**.

## buyer-facing impact

**None.** `SportsAnalysisResponse` unchanged; no prompt/recipe/template change; no chain-of-thought or prompt
text exposed. Provenance is run-row-adjacent metadata only.

## default allowlist status

**Unchanged** -- exactly four regimes.

## pending reconciliation notes

All 8 new games are Scheduled (first pitch 06-30 23:05 UTC onward, several after 07-01 00:00 UTC) -- none Final.
All 8 have 0 `AgentRunOutcome` rows and `ExclusionReason = null` (active). **PENDING -- not reconciled.** No
outcomes were fabricated and no future game was reconciled. Their route match rate will populate via a later
Outcome Reconciliation Follow-up once the games settle.

## risks / deferred items

- Regime evidence remains concentrated in `starter_enriched_market_backed_depth` (now n=15) -- the starter-missing
  and enriched_market_missing regimes still have thin samples (2, 1, 0). Growing those needs a slate that
  naturally offers TBD starters or thin markets; do not force it.
- No assembly_error fallback this batch, so the fallback route key was not freshly exercised (only the prior
  row verifies it). A future partial/asymmetric-evidence game would exercise it again.
- 8 new pending games + the 3 prior soak games are all awaiting settlement -> reconciliation backlog growing.

## next recommended slice

**Outcome Reconciliation Follow-up v1** (non-paid, time-gated). Once the 06-29-pending soak games and today's
8 batch games go Final, reconcile them via the `/reconcile` + per-run `/outcome` paths to populate route match
rates -- this is the highest-value non-paid step and clears the growing reconciliation backlog. Continue Daily
Live Batch Capture only if a fresh slate offers regime diversity worth the paid spend.
