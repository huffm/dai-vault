---
title: "Outcome Reconciliation Readiness + Next Slice Recommendation v1"
type: "reconciliation"
date: "2026-06-30"
status: "complete"
project: "DAI"
slice: "Outcome Reconciliation Readiness + Next Slice Recommendation v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - reconciliation
  - outcome
related:
  - "06 Execution/reconciliations/outcome-reconciliation-follow-up-v1.md"
---

# Outcome Reconciliation Readiness + Next Slice Recommendation v1 -- Evidence Report

**status:** complete (8-game live batch reconciled; 3 starter-missing soak runs still PENDING)
**date:** 2026-06-30

## purpose

Check whether any recent live MLB AgentRuns are now eligible for outcome reconciliation, reconcile only
finalized games, refresh prompt-route calibration metrics, and recommend the next highest-leverage slice.
Reconciliation/readiness slice only -- no prompt tuning, no cohort rerun, no OKF rewrite, no paid calls.

## repo / push state

- `dai`: clean, `main...origin/main`, 0 ahead / 0 behind. head `289777f` (thin tenant-scoped calibration
  metrics endpoint). No code edits this slice.
- `dai-vault`: clean except the expected untracked file. 0 ahead / 0 behind before this slice; the prior
  live-batch doc commit `4f90861` (`docs(execution): record live batch + settlement reconciliation gate`)
  was already pushed -- no push needed at start.
- Pre-existing untracked file `06 Execution/system-state-synopsis-v1.md` left untracked / excluded.

## service / db status

- `devcore-sql` docker container was `Exited (255)` after Docker Desktop restart; `docker start devcore-sql`
  brought it up healthy on `:1433`. SQL probe (`SELECT 1`) ready.
- `.NET` platform API built (`dotnet build` succeeded, 0 warn / 0 err) and run on `:5215`
  (`ASPNETCORE_ENVIRONMENT=Development`, dev bypass auth, TenantKey=1). `/health` -> `{"status":"ok"}`.
- FastAPI agent-service was **not** started -- reconciliation is a platform-only path (no model call).
- migration `20260629174632_AddAgentRunPromptRouteProvenance` present in `__EFMigrationsHistory`;
  `AgentRuns.PromptRouteProvenanceJson` exists, `nvarchar`, nullable. 246 AgentRuns total.
- `DEFAULT_ALLOWLIST` in `registry_prompt_canary.py` unchanged -- exactly four regimes
  (starter_enriched_market_backed_depth, starter_enriched_market_missing, starter_missing_market_missing,
  starter_missing_market_backed_depth).

## runs checked (11 recent live runs)

All TenantKey=1, RunType `sports.matchup.analysis`, SourceProvider `mlb_statsapi`, ExclusionReason NULL,
provenance persisted. Game status verified live against `statsapi.mlb.com/api/v1/schedule?gamePk=...`
(free public schedule API -- not a paid model call, no analysis run).

### 8-game live batch (2026-06-29 slate) -- now FINAL

| key | run | game (away @ home) | gamePk | status | final | lean | route |
|---|---|---|---|---|---|---|---|
| 260016 | 2dbd433e | White Sox @ Orioles | 824822 | Final | 8-2 | away | enriched (registry) |
| 260017 | 33bd433e | Pirates @ Phillies | 823444 | Final | 11-7 | away | enriched (registry) |
| 260018 | 36bd433e | Tigers @ Yankees | 823529 | Final | 7-3 | home | live fallback (assembly_error) |
| 260019 | 37bd433e | Mets @ Blue Jays | 822792 | Final | 1-2 | home | enriched (registry) |
| 260020 | 3dbd433e | Nationals @ Red Sox | 824741 | Final | 3-6 | home | enriched (registry) |
| 260021 | 42bd433e | Rangers @ Guardians | 824420 | Final | 6-3 | away | enriched (registry) |
| 260022 | 48bd433e | Reds @ Brewers | 823771 | Final | 3-5 | away | enriched (registry) |
| 260023 | 4cbd433e | Padres @ Cubs | 824662 | Final | 2-3 | home | enriched (registry) |

### 3 starter-missing soak runs -- still PENDING

| key | run | game (away @ home) | gamePk | status | eligible? |
|---|---|---|---|---|---|
| 260013 | 1fbd433e | Marlins @ Rockies (06-30) | 824338 | Scheduled / Preview | NO (today, not final) |
| 260014 | 25bd433e | Giants @ Dbacks (06-30) | 825066 | Scheduled / Preview | NO (today, not final) |
| 260015 | 28bd433e | White Sox @ Orioles (07-01) | 824818 | Scheduled / Preview | NO (future) |

## reconciliation actions performed

All writes went through the real API guards (direction-integrity guard + `RunEvaluator`), no direct-SQL
outcome writes, no fabricated scores. Scores/winner taken verbatim from the StatsAPI final.

- **Identity-driven `/reconcile` (7 clean SingleMatch games):** 260016, 260017, 260018, 260019, 260020,
  260021, 260022. Each returned `SingleMatch` and recorded outcome + evaluation.
- **MultipleMatches guard demonstrated on gamePk 824662 (Padres@Cubs):** `/reconcile` returned
  `MultipleMatches` (matched the live batch run 260023 **and** prior-cohort run 250030, both non-excluded,
  same provider+gamePk) and **wrote nothing** -- the matcher correctly refused to guess.
- **Explicit per-run `/{id}/outcome` for 260023:** wrote the outcome keyed directly on the live batch run's
  AgentRunKey (home_win 3-2), with a note recording the 824662 identity collision. This targets exactly the
  intended live run without touching the older cohort run 250030.

The 3 soak runs were left untouched (not final). No game that was not Final was reconciled.

## AgentRunOutcome evidence

`AgentRunOutcomes` rows 76 -> 84 (+8, all for keys 260016-260023). Each new outcome has a paired
`AgentRunEvaluation`:

| key | game | outcome | scores (away-home) | lean | eval |
|---|---|---|---|---|---|
| 260016 | White Sox @ Orioles | away_win | 8-2 | away | **correct** |
| 260017 | Pirates @ Phillies | away_win | 11-7 | away | **correct** |
| 260018 | Tigers @ Yankees | away_win | 7-3 | home | **incorrect** |
| 260019 | Mets @ Blue Jays | home_win | 1-2 | home | **correct** |
| 260020 | Nationals @ Red Sox | home_win | 3-6 | home | **correct** |
| 260021 | Rangers @ Guardians | away_win | 6-3 | away | **correct** |
| 260022 | Reds @ Brewers | home_win | 3-5 | away | **incorrect** |
| 260023 | Padres @ Cubs | home_win | 2-3 | home | **correct** |

Batch directional record: **6 correct / 2 incorrect** (0.75). The 3 soak runs have 0 outcome rows (pending).

## metrics endpoint results

`GET /api/agent-runs/prompt-route-calibration/metrics` (tenant-scoped), before -> after this slice:

| metric | baseline | after |
|---|---|---|
| totalRows | 246 | 246 |
| reconciledRows | 68 | **76** |
| unreconciledRows | 170 | **162** |
| noDecisionRows | 8 | 8 |
| unknownRouteRows | 236 | 236 |
| fallbackRows | 1 | 1 |
| registryRows | 10 | 10 |
| liveRows | 1 | 1 |
| matchedRows | 41 | **47** |
| unmatchedRows | 27 | **29** |
| matchRate | 0.6029 | **0.6184** |

Route-level (after):

- `starter_enriched_market_backed_depth` (registry): total 7, reconciled **7/7**, matched 6, unmatched 1,
  matchRate **0.857**. (The 7 registry batch runs; the one unmatched is 260022 Reds@Brewers.)
- `unknown` route: total 236, reconciled 69 (+1 = run 260018, the live fallback run, evaluated incorrect),
  matched 41, unmatched 28, matchRate 0.594.
- `starter_missing_market_missing` (registry): total 2, reconciled **0** (2 of the 3 soak runs, pending).
- `starter_missing_market_backed_depth` (registry): total 1, reconciled **0** (1 soak run, pending).

`matchRate` here is the directional correctness rate among reconciled directional runs (matched = lean correct).
`registryRows`/`liveRows` are provenance-source counters and are unchanged by reconciliation, as expected.

## assembly_error follow-up note

Run 260018 (Tigers@Yankees, gamePk 823529) carries
`fallbackReason=assembly_error`, `promptSource=live`, `legacyFallbackUsed=true`, with
`routingReason="registry assembly failed for regime starter_enriched_market_backed_depth; live prompt
authoritative"`. The safe live fallback worked -- the run completed and reconciled cleanly (incorrect).

**Attribution gap worth a diagnostic:** because the fallback run has no `selectedPromptRecipeId`, the metrics
endpoint buckets it under the `unknown` route, not the `starter_enriched_market_backed_depth` regime it was
*intended* for. So its reconciled outcome counts toward `unknown`'s match rate, and the enriched route shows
n=7 instead of n=8. This is a calibration-attribution leak, not a runtime failure. Recommend a small
non-paid **Registry Assembly Error Diagnostic v1** to (a) find why registry assembly failed for that regime,
and (b) decide whether assembly_error fallback runs should retain their intended regime label for calibration.

## OKF evaluation

Recommendation: **D -- defer OKF until after outcome reconciliation matures.** The active calibration thread
now has its first reconciled live evidence and a concrete attribution defect (assembly_error -> unknown
bucket); that is higher leverage than front-matter work right now. If a docs slice is wanted in parallel,
prefer **E -- a very small docs-only OKF pilot** (YAML front matter schema applied to 5-10 recent execution
docs only, content preserved, rollout rules documented, no mass churn). Do **not** do a full-vault OKF revamp.

## recommended next slice

- **Primary: Registry Assembly Error Diagnostic v1** (non-paid). Investigate the
  `starter_enriched_market_backed_depth` assembly failure behind run 260018 and the calibration-attribution
  leak (fallback runs landing in `unknown`). Protects calibration integrity, which is the point of the
  provenance work. No paid calls, no prompt changes.
- **Backup: Outcome Reconciliation Follow-up v1** (non-paid, time-gated). Reconcile the 3 starter-missing
  soak games once they go Final (824338 + 825066 settle the night of 06-30; 824818 on 07-01). Trivial rerun
  of this slice's reconcile path; populates the two `starter_missing` registry routes' match rates.

A live batch capture or broad cohort rerun would grow sample size but requires paid model calls -- out of
scope for this lane; defer until the diagnostic and the soak follow-up are done.

## paid-call status

**None.** No model calls. StatsAPI schedule reads are the free public MLB endpoint. All reconciliation went
through platform DB/API only.

## buyer-facing impact

**None.** Internal calibration/reconciliation only. No buyer UX, copy, body, or pricing touched.

## default allowlist status

**Unchanged** -- exactly four regimes in `DEFAULT_ALLOWLIST`.

## risks / deferred items

- The 3 soak runs remain unreconciled by design; their routes show 0 match rate until they settle.
- gamePk 824662 has two non-excluded runs (live 260023 + cohort 250030); 250030 stays unreconciled and will
  keep producing `MultipleMatches` on any identity-driven reconcile for that game until it is excluded or
  reconciled on its own. Left as-is (out of scope).
- assembly_error attribution leak (run 260018 -> `unknown` route) deferred to the recommended diagnostic.
- API + devcore-sql left running at slice end for any immediate follow-up; stop when no longer needed.
