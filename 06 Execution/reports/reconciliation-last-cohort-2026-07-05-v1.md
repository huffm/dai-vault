---
title: "Reconciliation -- Last Captured Cohort (Registry-Routed v8 Backed-Depth) v1"
type: "reconciliation"
date: "2026-07-05"
status: "complete -- 7/7 settled, identity-safe, official-source final; directional read only (n=7)"
project: "DAI"
slice: "Settle-and-Reassess Registry-Routed v8 Backed-Depth Cohort"
repos:
  dai: "unchanged (32180df) -- no code edits"
  dai-vault: "docs-only (this report)"
tags:
  - reconciliation
  - calibration
  - cohort
  - registry
  - backed-depth
related:
  - "06 Execution/reconciliations/registry-routed-v8-backed-depth-cohort-v1.md"
  - "06 Execution/reconciliations/paid-registry-routing-canary-v1.md"
  - "06 Execution/handoffs/current-slice.md"
---

# Reconciliation -- Last Captured Cohort (Registry-Routed v8 Backed-Depth) v1

**scope:** reconciliation-only slice. settle the most recent unreconciled captured cohort against the official
final outcome source, preserve exclusions, produce a defensible calibration read. no new analyses, no new agent
runs, no paid model calls, no prompt/routing/confidence/buyer/schema changes.

## 1. cohort discovered (evidence-based)

The most recent unreconciled captured cohort is the **Registry-Routed v8 Backed-Depth Cohort**, captured
2026-07-04, documented in `06 Execution/reconciliations/registry-routed-v8-backed-depth-cohort-v1.md` with status
"settlement pending finality," and named as the next actionable slice in `current-slice.md`
("Settle-and-Reassess Registry-Routed v8 Cohort ... settle each via the residue contract when Final").

- **capture date:** 2026-07-04 (first pitch 07-04T20:10Z .. 07-05T01:40Z)
- **sport/league:** MLB
- **run count:** 7 (all registry-routed, all directional)
- **regime/recipe:** `starter_enriched_market_backed_depth` @ `mlb.pregame.analysis.starter_enriched_market_backed_depth.v1` v1, `promptSource=registry`
- **provider identity coverage:** 7/7 persisted `SourceProvider=mlb_statsapi`, `ExternalGameId=gamePk`
- **outcomes already written:** 0/7 (all `hasOutcome=0` pre-slice)
- **finality as of 2026-07-05:** all 7 Final (verified against MLB StatsAPI, below)

## 2. repo state before

| repo | branch | HEAD | tree | upstream |
|---|---|---|---|---|
| `dai` | main | `32180df` | pre-existing empty-diff `DevCore.Data.csproj` (CRLF/filemode phantom; `git diff` empty) | 0 ahead / 0 behind |
| `dai-vault` | main | `5860e2b` | 1 untracked pre-existing doc (`06 Execution/system-state-synopsis-v1.md`) | 0 ahead / 0 behind |

The csproj phantom is the same pre-existing line-ending delta already recorded in the OKF hygiene report; it was
not touched in this slice.

## 3. services used

- **devcore-sql** (docker container `devcore-sql`, mssql 2022) -- was `Exited (137)`; started for this slice. read + write.
- **DevCore.Api** :5007 -- started via `dotnet run` (ASPNETCORE_ENVIRONMENT=Development, dev-bypass tenant 1). precheck + reconcile + calibration reads.
- **MLB StatsAPI** `statsapi.mlb.com/api/v1.1/game/{gamePk}/feed/live` -- official final outcome source, read-only.
- **NOT started (not needed, no spend):** agent-service (:8000 model path), sports-app (:4201). Zero paid model calls.

## 4. official final outcomes (StatsAPI, official source only)

All 7 games `abstractGameState=Final`, `detailedState=Final`, `statusCode=F`. Home/away identity confirmed to
match each run's persisted `HomeTeamRef`/`AwayTeamRef` (no display-name-only matching, no home/away inversion).

| gamePk | matchup | final (away @ home) | winner |
|---|---|---|---|
| 823118 | TOR @ SEA | TOR 0 @ SEA 11 | home (SEA) |
| 824415 | CWS @ CLE | CWS 3 @ CLE 1 | away (CWS) |
| 824171 | TB @ HOU | TB 8 @ HOU 10 | home (HOU) |
| 824903 | NYM @ ATL | NYM 3 @ ATL 14 | home (ATL) |
| 824092 | PHI @ KC | PHI 6 @ KC 1 | away (PHI) |
| 824012 | BOS @ LAA | BOS 8 @ LAA 1 | away (BOS) |
| 825063 | MIL @ AZ | MIL 3 @ AZ 4 | home (AZ) |

None postponed/suspended/cancelled/tied. No doubleheader/rescheduling ambiguity.

## 5. precheck results (GET /reconcile-precheck)

All 7 classified **IdentitySafe** (advice=1): exactly 1 active (`ExclusionReason IS NULL`) run per identity, each
unreconciled, backlog run id == the v8 run. Identity `POST /reconcile` (SingleMatch) is the safe path -- no
PerRunRequired, no Ambiguous, no AlreadyReconciled.

## 6. reconciliation writes (7/7, existing app path)

Written via the existing `POST /api/agent-runs/reconcile` endpoint (SingleMatch), under the Canonical
Reconciliation Residue Contract v1: `source=statsapi_final`, `sourceRef="gamePk <pk>"`,
`notes="Registry-Routed v8 Backed-Depth Cohort; <final score>; via reconcile"`. All 7 returned SingleMatch and
persisted an `AgentRunOutcome` + derived `AgentRunEvaluation`. Residue verified non-thin (source/sourceRef/notes
all populated).

| gamePk | run8 | lean | conf | outcomeStatus | homeScore | awayScore | eval |
|---|---|---|---|---|---|---|---|
| 823118 | cb49433e | home | 0.80 | home_win | 11 | 0 | correct |
| 824415 | cd49433e | home | 0.80 | away_win | 1 | 3 | incorrect |
| 824171 | d049433e | home | 0.75 | home_win | 10 | 8 | correct |
| 824903 | d649433e | home | 0.80 | home_win | 14 | 3 | correct |
| 824092 | d949433e | away | 0.75 | away_win | 1 | 6 | correct |
| 824012 | dc49433e | away | 0.75 | away_win | 1 | 8 | correct |
| 825063 | dd49433e | away | 0.75 | home_win | 4 | 3 | incorrect |

## 7. exclusions

- **This cohort:** 0 exclusions. All 7 runs were active (`ExclusionReason IS NULL`), directional, identity-safe,
  final, and not-already-reconciled. No game hit any exclusion reason (postponed/suspended/cancelled/abandoned/
  duplicate identity/ambiguous mapping/missing provider identity/missing final score/unsupported/precheck-failed).
- **Out of scope (flagged, not settled):** `822882` DET@TEX -- a prior v8 registry backed_depth **canary** with
  **no lean** (no-decision). It is the 1 remaining unreconciled backed_depth registry row. Belongs to an earlier
  canary slice, not this cohort; settling it would evaluate inconclusive. Left for a separate cleanup decision.

## 8. calibration read (post-reconciliation, read-only)

```
COHORT: Registry-Routed v8 Backed-Depth (captured 2026-07-04), MLB, regime starter_enriched_market_backed_depth

counts
  total cohort runs .............. 7
  valid candidate runs ........... 7   (settled AND ExclusionReason IS NULL, all directional)
  reconciled in this slice ....... 7
  previously reconciled .......... 0
  excluded ....................... 0
  pending ........................ 0

DAI lean vs outcome
  correct ........................ 5
  incorrect ...................... 2
  push / null (no lean) .......... 0
  hit rate ....................... 5/7  (0.71)   <-- directional read only, n=7

confidence bands
  0.80  n=3 ->  2 correct / 1 incorrect   (823118 ok, 824903 ok, 824415 miss)
  0.75  n=4 ->  3 correct / 1 incorrect   (824171 ok, 824092 ok, 824012 ok, 825063 miss)

market favorite baseline (marketConsensusSide vs winner)
  market correct ................. 5/7   (same two misses: 824415, 825063)
  DAI vs market agreement ........ 7/7 agree, 0 disagree   (marketAgreement=true on ALL 7)
  => DAI lean == market consensus on every game. This cohort measures whether the market-informed
     lean tracks outcomes; it does NOT measure DAI edge over market (no divergent games to test it).
     Both misses were market-aligned (market wrong too). No independent-edge signal available here.

evidence sufficiency / source depth
  evidenceRichness ............... 2 for all 7 (partial, uniform)
  market book count .............. 5-7 (valid backed_depth depth)
  posture ........................ monitor for all 7
  note: all 7 fit confidence_high_for_partial_evidence (richness<3 AND confidence>=0.70)

pooled route context (post-slice, whole system)
  starter_enriched_market_backed_depth registry route:
    reconciled 22 (matched 15 / unmatched 7), matchRate 0.682, avgConf 0.757
    1 unreconciled remains (822882, no lean)
```

**Small-sample caveat (explicit):** n=7 is **insufficient** for any accuracy conclusion. Per-bucket n=3/n=4 is
below any calibration threshold. Treat 5/7 as an **early directional read**, not a hit rate. Because DAI agreed
with the market on all 7, this cohort provides **no evidence of edge over the market**. `conclusionsAllowed`
remains gated. Do not tune on these results.

## 9. validation performed

- **no new agent runs:** `AgentRuns` count = 273 before and after (matches post-v8-generation baseline).
- **no paid model calls:** agent-service never started; only StatsAPI (free), devcore-sql, and DevCore.Api
  reconcile/read endpoints were called.
- **no sports analysis generation:** none.
- **writes match only eligible final identity-safe games:** 7 `POST /reconcile` -> 7 SingleMatch -> the 7 target
  runs each carry exactly one outcome + evaluation with correct scores and `sourceRef="gamePk <pk>"`.
- **residue complete:** source/sourceRef/notes non-null on all 7 (Residue Contract v1 satisfied; no thin residue).
- **no code / config drift:** `dai` working tree shows only the pre-existing empty-diff `DevCore.Data.csproj`
  phantom; no prompt/routing/confidence/buyer/migration/schema files changed.
- **exclusions preserved:** no run's `ExclusionReason` was altered; the excluded/no-lean 822882 was left untouched.

## 10. final repo state

| repo | branch | HEAD | tree change from this slice |
|---|---|---|---|
| `dai` | main | `32180df` | none (csproj phantom pre-existing) |
| `dai-vault` | main | `5860e2b` | +1 untracked report (this file) |

Nothing committed, nothing pushed. DB reconciliation writes are persisted in devcore-sql (7 outcomes + 7
evaluations).

## 11. next recommended slice

1. **Optional cleanup (no spend):** decide 822882 DET@TEX (registry backed_depth canary, no lean) -- either settle
   it (records outcome; evaluates inconclusive) or mark it excluded as a superseded canary, so the backed_depth
   registry route has 0 dangling unreconciled rows.
2. **Measurement gap:** this cohort had 7/7 DAI-market agreement, so it cannot measure DAI edge. A future
   measurement cohort (approval-gated, paid) should deliberately include games where the model's lean can diverge
   from market consensus, and track divergence explicitly -- otherwise backed_depth accuracy is indistinguishable
   from following the market favorite.
3. **Confidence-bucket gate:** the 0.80 and 0.75 buckets remain thin (n=3 / n=4 in this cohort). `conclusionsAllowed`
   stays gated until the settled backed_depth sample is materially larger. No tuning until then.
