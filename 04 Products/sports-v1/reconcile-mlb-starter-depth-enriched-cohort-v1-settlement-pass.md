# MLB Starter-Depth Enriched Cohort Reconciliation v1

## Summary

- **date:** 2026-06-22
- **slice:** Reconcile MLB Starter-Depth Enriched Cohort v1 -- Settlement Completion Pass
- **cohort size:** 6 (AgentRunKey 190013-190018)
- **reconciled count:** 6 of 6
- **pending count:** 0
- **pre-state totals:** AgentRunOutcomes = 29, AgentRunEvaluations = 29
- **post-state totals:** AgentRunOutcomes = 35, AgentRunEvaluations = 35
- **correct / incorrect / inconclusive:** 6 / 0 / 0

All six games were Final at reconcile time; all six runs carry structured `LeanSide=home`; all six
games were home wins, so every run evaluated `correct`. Read strictly within the enriched starter-depth
regime (`SourceDepth.starting_pitching=enriched` 6/6). Do not pool with the identity-only market-aware
cohort (180013-180020).

## Settlement Manifest

Source: MLB StatsAPI `schedule` (sportId=1, hydrate=linescore), checked 2026-06-22. Status read from
`status.abstractGameState`; all six = Final. OutcomeStatus derived from final score (home > away ->
home_win). Not reconciled from memory or expected winners.

- **190013** / f0a9433e / gamePk 824264 / White Sox at Tigers / Final / CHW 3, DET 4 -> **home_win** / source manual / statsapi:824264
- **190014** / f5a9433e / gamePk 823534 / Reds at Yankees / Final / CIN 0, NYY 5 -> **home_win** / source manual / statsapi:823534
- **190015** / fba9433e / gamePk 823853 / Giants at Marlins / Final / SF 3, MIA 4 -> **home_win** / source manual / statsapi:823853
- **190016** / fda9433e / gamePk 822966 / Nationals at Rays / Final / WSH 2, TB 5 -> **home_win** / source manual / statsapi:822966
- **190017** / 01aa433e / gamePk 824910 / Brewers at Braves / Final / MIL 2, ATL 3 -> **home_win** / source manual / statsapi:824910
- **190018** / 02aa433e / gamePk 822886 / Padres at Rangers / Final / SD 7, TEX 9 -> **home_win** / source manual / statsapi:822886

## Reconciliation Results

Endpoint: `POST /api/agent-runs/reconcile` (tenant-scoped, dev bypass auth tenant 1). Each call returned
`SingleMatch` and wrote exactly one outcome + one evaluation row.

- **190013** / gamePk 824264 / SingleMatch / EvalStatus **correct** / lean home vs winner home
- **190014** / gamePk 823534 / SingleMatch / EvalStatus **correct** / lean home vs winner home
- **190015** / gamePk 823853 / SingleMatch / EvalStatus **correct** / lean home vs winner home
- **190016** / gamePk 822966 / SingleMatch / EvalStatus **correct** / lean home vs winner home
- **190017** / gamePk 824910 / SingleMatch / EvalStatus **correct** / lean home vs winner home (see Special Handling)
- **190018** / gamePk 822886 / SingleMatch / EvalStatus **correct** / lean home vs winner home

## Special Handling

- **190017 / gamePk 824910 (Brewers at Braves)** was evaluated on the structured `AgentRun.LeanSide=home`
  column. The persisted evaluation row confirms `EvalStatus=correct`, `LeanSide=home`, `WinningSide=home`.
- This run carried `ArtifactDirectionConsistency=PotentialMismatch` (a prose-vs-structured risk recorded
  at capture). That prose mismatch risk did **not** control evaluation: the deterministic `RunEvaluator`
  grades the denormalized structured `LeanSide`, never the prose. Recorded here as a standing artifact-
  quality observation, not an evaluation defect.

## Verification

- **API endpoint used:** `POST /api/agent-runs/reconcile`; read-back via DB inspection (and a pre-mutation
  `GET /api/agent-runs/{id}/evaluation` returning 404 for 190013, confirming no prior evaluation).
- **Pre-state (sqlcmd in container devcore-sql):** AgentRunOutcomes=29, AgentRunEvaluations=29; cohort
  190013-190018 each identity-confirmed (`mlb_statsapi` + matching gamePk, `LeanSide=home`,
  `ExclusionReason=null`, `Status=completed`) with 0 outcome / 0 eval rows.
- **Post-state (sqlcmd):** AgentRunOutcomes=35, AgentRunEvaluations=35. Each cohort run has exactly one
  outcome and one evaluation; all `EvalStatus=correct`; all eval `LeanSide=home`, `WinningSide=home`;
  `Source=manual`, `SourceRef=statsapi:<gamePk>`.
- **Cohort tally:** correct 6, incorrect 0, inconclusive 0, not-evaluated 0.
- **Settlement source:** MLB StatsAPI schedule endpoint, all six `Final`, 0 in progress.
- **Tests:** none run. No code changed -- this slice only wrote outcome/evaluation rows through the
  existing reconciliation runtime. Services started for the pass: docker container `devcore-sql` and
  `DevCore.Api` on `http://localhost:5007`.

## Next Recommended Slice

**Reconciled-Outcome Calibration Read v1.** With 35 settled outcomes now spanning the identity-only
market-aware cohort and this enriched starter-depth cohort, the next highest-value move is a read-only
calibration analysis that groups settled runs by source-depth regime, evidence-sufficiency tier, raw
confidence band, and posture, keeping the enriched cohort separate from the identity-only cohort. This
is the doctrine-sanctioned consumer of reconciled evidence (Outcome Reconciliation Contract v1, calibration
feedback-loop section) and the prerequisite for any future confidence-calibration change, which remains
gated on reconciled-outcome evidence. Note for that slice: this enriched cohort is a clean 6/6 home-lean /
home-win sweep, so it confirms the loop end-to-end but adds no incorrect/inconclusive contrast on its own;
interpretation should pool cautiously across regimes for variance, never merging the regimes.
