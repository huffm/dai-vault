# Stage 0 True Calibration Candidate Capture v1

**date:** 2026-06-12
**status:** candidate capture complete against local/dev. no outcomes recorded. no reconciliation performed. no calibration read surface built. no runtime code changed.
**scope:** capture a small, capped set of identity-bearing pre-settlement NBA/MLB analyzer runs so they can become true calibration evidence after settlement.

## Environment

- workspace: `C:\Users\trolo\source\repos\dai-workspace`
- implementation repo: `dai`
- vault repo: `dai-vault`
- environment: local/dev only
- database: local SQL Server container `devcore-sql`, database `devcore`
- API: `http://localhost:5007`
- FastAPI agent service: `http://127.0.0.1:8000`
- dev tenant/user observed on candidate rows: `TenantKey=1`, `RequestedByUserKey=1`

## DB/API Readiness

- `GET /api/competitions` returned 200.
- `GET http://127.0.0.1:8000/api/ping` returned `status=ok`.
- SQL Server was confirmed through the local `devcore-sql` container.
- required migrations present in `__EFMigrationsHistory`:
  - `20260611152737_AddAgentRunGameIdentity`
  - `20260612121023_ExpandOutcomeStatusTaxonomy`
- baseline carried into this slice: 115 agent runs, 0 identity-bearing.
- after candidate capture: 119 agent runs, 4 identity-bearing.

## Model Spend

Budget rule: max 6 analyzer runs for this slice.

Observed spend:

| source | runs | note |
|---|---:|---|
| Claude | 1 | NBA candidate already generated before this continuation |
| Codex | 3 | MLB candidates present in DB at resume, generated through normal analyzer path |
| total | 4 | under the max 6 cap |

No additional analyzer calls were made after the resumed validation found the three planned MLB candidates already present.

## Candidate Selection

Normal analyzer path:

`POST /api/agent-runs` with `runType=sports.matchup.analysis` and input `{ competition, homeTeam, awayTeam, gameDate }`.

MLB upcoming re-check on 2026-06-12 returned 15 games for `GET /api/competitions/mlb/upcoming?days=3`, including all three selected games:

- New York Yankees at Toronto Blue Jays
- Texas Rangers at Boston Red Sox
- San Diego Padres at Baltimore Orioles

All three generated MLB rows were created at about `2026-06-12T14:19Z`, before their scheduled starts. No started/settled game was intentionally selected.

## Candidate Records

| AgentRunId | source | matchup | gameDate | createdUtc | provider | external id | scheduledStartUtc | season | homeTeamRef | awayTeamRef | leanSide | posture | confidence | evidenceRichness | artifactVersion | identity complete | outcome/eval |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---:|---:|---|---|---|
| `c1f3423e-f36b-1410-8162-00373db4b724` | Claude | New York Knicks at San Antonio Spurs | 2026-06-13 | 2026-06-12T14:16:30.0664878Z | odds_api | `6cc5c3b9cfcb1d94bed9f3ca972b3114` | 2026-06-14T00:40:00Z | 2025-26 | san-antonio-spurs | new-york-knicks | home | monitor | 0.675 | 2 | sports_decision_artifact_v3 | yes | 0/0 |
| `c3f3423e-f36b-1410-8162-00373db4b724` | Codex | New York Yankees at Toronto Blue Jays | 2026-06-12 | 2026-06-12T14:19:29.2690242Z | mlb_statsapi | `822803` | 2026-06-12T23:37:00Z | 2026 | toronto-blue-jays | new-york-yankees | null | wait | 0.375 | 0 | sports_decision_artifact_v3 | yes | 0/0 |
| `caf3423e-f36b-1410-8162-00373db4b724` | Codex | Texas Rangers at Boston Red Sox | 2026-06-12 | 2026-06-12T14:19:35.4700842Z | mlb_statsapi | `824752` | 2026-06-12T23:10:00Z | 2026 | boston-red-sox | texas-rangers | home | monitor | 0.75 | 1 | sports_decision_artifact_v3 | yes | 0/0 |
| `d1f3423e-f36b-1410-8162-00373db4b724` | Codex | San Diego Padres at Baltimore Orioles | 2026-06-12 | 2026-06-12T14:19:43.1961044Z | mlb_statsapi | `824826` | 2026-06-12T23:05:00Z | 2026 | baltimore-orioles | san-diego-padres | home | monitor | 0.75 | 1 | sports_decision_artifact_v3 | yes | 0/0 |

Notes:

- MLB identity matches the expected contract: `SourceProvider=mlb_statsapi`, `ExternalGameId` is the statsapi `gamePk` string, `ScheduledStartUtc` is populated, `Season=2026`, and team refs are ascii slugs.
- The Yankees/Blue Jays run has `LeanSide=null`; it is still an identity-bearing pre-settlement candidate, but a final directional outcome would evaluate inconclusive unless the stored lean-side semantics change, which they should not for this batch.
- Artifact output remains v3. Identity is stored on the run row, not inside `OutputJson`.

## Identity Validation

SQL validation against the four candidate run ids showed:

- every candidate has non-null `SourceProvider`, `ExternalGameId`, `ScheduledStartUtc`, `Season`, `HomeTeamRef`, and `AwayTeamRef`.
- every candidate is `Status=completed`.
- every candidate is under `TenantKey=1`, `RequestedByUserKey=1`.
- candidate outcome rows: 0.
- candidate evaluation rows: 0.
- global identity-bearing run count: 4.

## Duplicate Findings

Duplicate query over identity-bearing rows:

```sql
SELECT SourceProvider, ExternalGameId, COUNT(*) AS RunCount
FROM AgentRuns
WHERE SourceProvider IS NOT NULL AND ExternalGameId IS NOT NULL
GROUP BY SourceProvider, ExternalGameId
HAVING COUNT(*) > 1;
```

Result: no duplicate `(SourceProvider, ExternalGameId)` groups.

## Tenant Notes

All four candidates align to the same dev tenant/user (`TenantKey=1`, `RequestedByUserKey=1`). Reconciliation after settlement must be performed with the same resolved tenant or the matcher will correctly return `NoMatch`.

## Outcome/Evaluation Status

No outcomes or evaluations were recorded for these candidates. This slice intentionally stopped at candidate capture. There are older outcome/evaluation rows in the dev database from prior per-run experiments, but none are attached to the four captured candidates.

## Classification

| candidate | classification | reason |
|---|---|---|
| Knicks at Spurs | true pending | identity-bearing, pre-settlement, no outcome/evaluation |
| Yankees at Blue Jays | true pending, limited calibration value | identity-bearing, pre-settlement, no outcome/evaluation; null lean side likely yields inconclusive later |
| Rangers at Red Sox | true pending | identity-bearing, pre-settlement, no outcome/evaluation |
| Padres at Orioles | true pending | identity-bearing, pre-settlement, no outcome/evaluation |

No plumbing-only runs are included as calibration candidates in this report.

## Scratch Handling

Scratch scripts observed in `dai`:

- `.probe.ps1`
- `.waitready.ps1`
- `.upcoming.ps1`
- `.mlb.ps1`
- `.gen.ps1`

They were only used as local probes/generation helpers and were removed from `dai` after validation.

## Follow-Up After Settlement

After each game is final:

1. Confirm the game has settled from a trusted source before calling any reconcile endpoint.
2. Re-check duplicate keys for the candidate's `(SourceProvider, ExternalGameId)`.
3. Reconcile through `POST /api/agent-runs/reconcile`, not the per-run fallback path, using the same tenant.
4. For MLB, prefer statsapi settlement by `gamePk` and record `source=manual:statsapi` plus a concrete `sourceRef`.
5. For NBA, prefer a source that resolves the odds-api event identity or a clearly auditable box-score source; record `source` and `sourceRef`.
6. Validate that exactly one outcome and one evaluation were written for the matched run.
7. Append the result and any friction to this file or a follow-up Stage 0 settlement note.

Do not build the internal calibration read surface until at least a small non-trivial set of these pending candidates has settled and been reconciled into real `AgentRunEvaluations`.
