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

---

## Outcome Reconciliation Follow-Up v1

**execution date:** 2026-06-12
**status:** checked against local/dev; no reconciliation performed because none of the four candidates had reached scheduled start yet. no outcome was fabricated and no non-final probe was posted.

### Environment Recheck

- `dai` preflight: clean at `f02b307`.
- `dai-vault` preflight: clean with `6b47130` present at HEAD; commit was still ahead of origin and not pushed.
- API `GET /api/competitions` returned 200 on `http://localhost:5007`.
- dev bypass config still shows `Dev:EnableBypassAuth=true`, `Dev:TenantKey=1`, `Dev:UserKey=1`.
- DB confirmed as local `devcore` inside `devcore-sql`; DB UTC clock was `2026-06-12 14:45:03Z`.
- required migrations confirmed in `__EFMigrationsHistory`: `AddAgentRunGameIdentity` and `ExpandOutcomeStatusTaxonomy`.

### Candidate Pre-State

The local artifact endpoint was used as the safe read path after the expanded SQL candidate query was rejected by the approval review. Every candidate still returned a completed artifact with the expected identity fields. Every `GET /api/agent-runs/{id}/evaluation` returned 404, so no candidate evaluation existed at follow-up time.

| AgentRunId | provider key | scheduled start | pre-state | evaluation |
|---|---|---|---|---|
| `c1f3423e-f36b-1410-8162-00373db4b724` | `odds_api` / `6cc5c3b9cfcb1d94bed9f3ca972b3114` | 2026-06-14T00:40:00Z | identity present; completed | 404 |
| `c3f3423e-f36b-1410-8162-00373db4b724` | `mlb_statsapi` / `822803` | 2026-06-12T23:37:00Z | identity present; completed | 404 |
| `caf3423e-f36b-1410-8162-00373db4b724` | `mlb_statsapi` / `824752` | 2026-06-12T23:10:00Z | identity present; completed | 404 |
| `d1f3423e-f36b-1410-8162-00373db4b724` | `mlb_statsapi` / `824826` | 2026-06-12T23:05:00Z | identity present; completed | 404 |

### Settlement Decision

No candidate was eligible for final reconciliation:

- current DB time: `2026-06-12T14:45:03Z`.
- earliest candidate scheduled start: `2026-06-12T23:05:00Z`.
- NBA candidate scheduled start: `2026-06-14T00:40:00Z`.

Because the system clock was before scheduled start for every candidate, there was no possible trusted final result to record. No final-score source was consulted or used, and no reconcile payload was sent. MLB future source references remain the exact statsapi gamePks; NBA still requires a trusted auditable final source after its scheduled game.

### Reconcile Summary

| candidate | reconcile attempted | matcher result | outcome recorded | evaluation recorded | reason |
|---|---|---|---|---|---|
| Knicks at Spurs | no | none | no | no | pre-start |
| Yankees at Blue Jays | no | none | no | no | pre-start |
| Rangers at Red Sox | no | none | no | no | pre-start |
| Padres at Orioles | no | none | no | no | pre-start |

New Stage 0 evaluation count from this follow-up:

| evalStatus | count |
|---|---:|
| correct | 0 |
| incorrect | 0 |
| inconclusive | 0 |

### Matcher Findings

- `SingleMatch`: not attempted; no final candidate.
- `NoMatch`: not attempted.
- `MultipleMatches`: not attempted; no duplicate-resolution decision was needed.
- `NotEvaluable`: not posted intentionally; the games were pre-start rather than settled non-final.

### Evaluator Observations

`RunEvaluator` was not invoked. The Yankees/Blue Jays run still has no directional lean value in the captured candidate report, so a later final reconciliation may produce an inconclusive evaluation; that remains expected and should not be treated as an evaluator defect.

### Read Surface Readiness

Internal Calibration Read Surface v1 is still not ready. This follow-up produced zero reconciled outcomes and zero new evaluations. The next attempt should run only after the scheduled games have actually settled and should use trusted final sources for the exact provider keys above.

---

## Manual Stage 0 Reconciliation Execution v1 (settled)

**execution date:** 2026-06-15
**status:** all four Stage 0 candidates reconciled live through `POST /api/agent-runs/reconcile`. Every candidate returned `SingleMatch` and persisted exactly one outcome + one evaluation. Trusted final sources consulted (MLB statsapi by gamePk; ESPN NBA scoreboard). No outcome fabricated. No code, schema, prompt, confidence/posture/lean, buyer, analyzer, or matcher change.
**classification:** true Stage 0 calibration sample (first live `SingleMatch` reconciliations on identity-bearing runs).

### Environment

- workspace `C:\Users\trolo\source\repos\dai-workspace`; both repos clean on `main`, 0/0 ahead/behind at slice start.
- brought the stack up via documented path only: Docker Desktop -> `docker start devcore-sql` (was `Exited (137)`; log reached "Recovery is complete", `0.0.0.0:1433->1433`) -> .NET API on `:5007` (`ASPNETCORE_ENVIRONMENT=Development`, `http` profile). FastAPI analyzer not started -- the reconcile path touches only DB + matcher (no model call).
- DB confirmed local `devcore` inside `devcore-sql`; dev bypass auth resolved `TenantKey=1` (candidate evaluations returned 404 pre-state, not 401).
- readiness: `GET /api/competitions` 200; all four `GET /api/agent-runs/{id}/evaluation` returned 404 before reconcile (no prior evaluation).

### Trusted outcome sources

- **MLB (mlb_statsapi):** `statsapi.mlb.com/api/v1.1/game/{gamePk}/feed/live` -- all three gamePks resolved `abstractGameState=Final` / `game_finished`, matchups confirmed by the statsapi `id` slug (nyamlb-tormlb, texmlb-bosmlb, sdnmlb-balmlb). Scores from `/api/v1/game/{gamePk}/linescore`.
- **NBA (odds_api run):** the odds-api event id is opaque (no public lookup without a key), so the auditable final was taken from ESPN's NBA scoreboard for 2026-06-13 (`site.api.espn.com/.../basketball/nba/scoreboard?dates=20260613`), `status=Final`, NY 94 @ SA 90. `source=manual:espn`. Provenance recorded; the run identity (odds_api key) still drove the match, the ESPN box score only supplied the directional result.

### Four-candidate result table

| candidate | sport / league | teams (home) | scheduled start UTC | prior run id | captured identity (provider / key) | final (home-away) | outcome source | matchKind | evalStatus | reason |
|---|---|---|---|---|---|---|---|---|---|---|
| Knicks at Spurs | NBA | New York Knicks at San Antonio Spurs (home: Spurs) | 2026-06-14T00:40:00Z | `c1f3423e-f36b-1410-8162-00373db4b724` | odds_api / `6cc5c3b9cfcb1d94bed9f3ca972b3114` | SA 90 - NY 94 (away_win) | ESPN NBA scoreboard 20260613 (`manual:espn`) | SingleMatch | **incorrect** | lean was home (Spurs); Knicks (away) won |
| Yankees at Blue Jays | MLB | New York Yankees at Toronto Blue Jays (home: Blue Jays) | 2026-06-12T23:37:00Z | `c3f3423e-f36b-1410-8162-00373db4b724` | mlb_statsapi / `822803` | TOR 8 - NYY 5 (home_win) | statsapi gamePk 822803 (`manual:statsapi`) | SingleMatch | **inconclusive** | no directional lean on the run (lean null) |
| Rangers at Red Sox | MLB | Texas Rangers at Boston Red Sox (home: Red Sox) | 2026-06-12T23:10:00Z | `caf3423e-f36b-1410-8162-00373db4b724` | mlb_statsapi / `824752` | BOS 10 - TEX 1 (home_win) | statsapi gamePk 824752 (`manual:statsapi`) | SingleMatch | **correct** | lean home (Red Sox); Red Sox won |
| Padres at Orioles | MLB | San Diego Padres at Baltimore Orioles (home: Orioles) | 2026-06-12T23:05:00Z | `d1f3423e-f36b-1410-8162-00373db4b724` | mlb_statsapi / `824826` | BAL 7 - SD 3 (home_win) | statsapi gamePk 824826 (`manual:statsapi`) | SingleMatch | **correct** | lean home (Orioles); Orioles won |

Artifact id: identity lives on the AgentRun row (not in `OutputJson`); the run id above is the artifact anchor. All four artifacts are v3, `Status=completed`, `TenantKey=1`.

### Persistence verification

- each reconcile returned the matched run id + evalStatus; all four `GET /api/agent-runs/{id}/evaluation` then returned 200 with consistent `leanSide`/`winningSide`/`evalStatus` (e.g. Knicks: lean `home`, winning `away`, `incorrect`).
- idempotency: re-POST of the Rangers/Red Sox key returned **409** (one-outcome-per-run guard holds on the matcher path, single-sourced with the per-run endpoint).
- DB tally (sqlcmd in `devcore-sql`): outcomes/evals moved **8/8 -> 12/12**; eval tally now correct 5 / inconclusive 4 / incorrect 3. The +4 delta (+2 correct, +1 incorrect, +1 inconclusive) is exactly these four candidates; the four identity-bearing outcome rows confirm provider/key/score/eval.

### What this proves / does not prove

- **Proves:** the matcher contract end-to-end on real settled data -- first live `SingleMatch` reconciliations. The canonical `(SourceProvider, ExternalGameId)` key matched each settled game to its one identity-bearing run with no display-name fallback; the deterministic evaluator produced the expected correct/incorrect/inconclusive verdicts; persistence and the 409 guard hold. The data gap the prior execution hit (zero identity-bearing runs) is closed for this sample.
- **Does not prove:** nothing global. This is a **Stage 0 sample reconciled successfully** -- four settled candidates the current matcher handled cleanly. It is not "fully validated", not "production ready", not an automated feedback loop. `MultipleMatches` was still not exercised (no duplicate-key runs exist), and no settlement-provider automation was built or implied.

### Failure patterns

None. Zero NoMatch, zero MultipleMatches, zero NotEvaluable, zero errors across the four. The only non-`correct` results are by design: the null-lean run evaluates inconclusive (expected, low calibration value) and the Spurs lean was genuinely wrong.

### Is the matcher good enough for Stage 0?

Yes for Stage 0. The matcher cleanly handled all four settled candidates on the canonical key, single-sourced the write path with the per-run endpoint, and guarded duplicates with 409. The unexercised edge (`MultipleMatches` resolution) remains a deferred decision (ledger entry 25), not a Stage 0 blocker -- no duplicate-key runs exist to trigger it.

### Recommended next slice

**More Stage 0 samples** before any automation. Four games (one of them a no-lean inconclusive) is a successful contract proof but a thin calibration base. Generate and reconcile a larger identity-bearing batch (both sports, prefer non-null leans) to build a non-trivial correct/incorrect body. Internal Calibration Read Surface v1 is closer but still premature on a 4-sample base; matcher hardening and identity-capture hardening are not indicated (both performed correctly here). Buyer-artifact outcome feedback stays gated on a larger reconciled sample (ledger entry 12).
