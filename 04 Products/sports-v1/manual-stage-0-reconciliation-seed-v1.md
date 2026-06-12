# Manual Stage 0 Reconciliation Seed v1

**date:** 2026-06-12
**status:** runbook only (local dev database unreachable this session). docs-only; no code, no schema, no runtime change. no sample was fabricated.
**scope:** exercise the existing internal reconciliation path on a small real local/dev sample of settled NBA/MLB runs to produce calibration evidence, OR -- when suitable local data is unavailable -- the precise runbook to do so later. this session is the latter.

## Objective

Turn `POST /api/agent-runs/reconcile` (Outcome Reconciliation Matcher v1) from unused infrastructure into real reconciled-outcome evidence, and capture operational friction before any settlement-provider automation or internal calibration read surface is built. This is internal/manual only -- no buyer-visible track record, no dashboard, no scheduled job.

## Database readiness findings (this session)

- target: `Server=localhost,1433;Database=devcore` (appsettings.Development.json) -- local/dev, not production.
- `Test-NetConnection localhost:1433` -> `TcpTestSucceeded = False`. Nothing is listening on the SQL port.
- `dotnet ef migrations list` returns the codebase migration list but reports "Unable to determine which migrations have been applied ... an error occurs while accessing the database" -- i.e. the DB is unreachable.
- both required migrations exist in the codebase but applied-state is UNCONFIRMED:
  - `20260611152737_AddAgentRunGameIdentity`
  - `20260612121023_ExpandOutcomeStatusTaxonomy` (the prior handoff explicitly recorded this as not yet applied to the dev db).

Per the database guard, an unreachable DB with unconfirmed migrations means: do not start the server, do not fabricate a sample, produce the runbook. The dev SQL container (`devcore-sql`, per the troubleshooting doc) was not running; bringing it up and applying migrations is a normal dev step but was out of scope to do blindly this session.

## Local data availability

Unknown -- the database could not be queried. Whether any identity-bearing settled runs exist locally must be re-checked once the DB is up (queries below).

## Runbook

### 0. Bring the environment up (normal dev steps)

1. Start the dev SQL container (`devcore-sql`) per the sports troubleshooting doc; confirm `localhost,1433` is listening.
2. Apply migrations to the local `devcore` db (dev-only):
   `dotnet ef database update --project platform/dotnet/DevCore.Data --startup-project platform/dotnet/DevCore.Api`
   Confirm `20260611152737_AddAgentRunGameIdentity` and `20260612121023_ExpandOutcomeStatusTaxonomy` are present in `__EFMigrationsHistory`.
3. Run the API in Development with the dev auth bypass enabled (see auth note) so the reconcile endpoint can resolve a tenant without a real token.

### 1. Match key (exact, do not deviate)

Canonical key: `(AgentRun.SourceProvider, AgentRun.ExternalGameId)` == `(request.sourceProvider, request.externalGameId)`, tenant-scoped to the caller's `TenantKey`. Exact equality only. No display-name fallback (forbidden). Legacy/null-identity runs never match.

### 2. Sample selection rules

- 5-10 settled runs if available; include both basketball (NBA) and MLB if suitable runs exist.
- Only identity-bearing runs (`SourceProvider` and `ExternalGameId` not null). NBA market-grounded runs carry `odds_api` + the odds event id; MLB runs carry `mlb_statsapi` + the gamePk.
- A run is a candidate only when its game has actually finished (now > `ScheduledStartUtc` by a safe margin, e.g. +6h).
- Do not fabricate outcomes. Determine the real final result from a trusted source named in the friction log at reconcile time (e.g. the same provider that grounded the run: ESPN/odds-api box score for NBA, statsapi.mlb.com `gamePk` boxscore for MLB). Record `source` and `sourceRef` accordingly.
- Tenant scoping: the seeded runs must belong to the `Dev:TenantKey` the API resolves, or the matcher returns NoMatch even with a correct key.

### 3. Read queries (run against local/dev only)

```sql
-- a. identity-bearing run count
SELECT COUNT(*) AS IdentityBearingRuns
FROM AgentRuns
WHERE SourceProvider IS NOT NULL AND ExternalGameId IS NOT NULL;

-- b. group identity-bearing runs by competition + provider
SELECT Competition, SourceProvider, COUNT(*) AS Runs
FROM AgentRuns
WHERE SourceProvider IS NOT NULL AND ExternalGameId IS NOT NULL
GROUP BY Competition, SourceProvider
ORDER BY Runs DESC;

-- c. settled candidates: game start in the past, still unreconciled
SELECT r.AgentRunId, r.Competition, r.SourceProvider, r.ExternalGameId,
       r.ScheduledStartUtc, r.HomeTeamRef, r.AwayTeamRef, r.LeanSide, r.Status
FROM AgentRuns r
LEFT JOIN AgentRunOutcomes o ON o.AgentRunKey = r.AgentRunKey
WHERE r.SourceProvider IS NOT NULL AND r.ExternalGameId IS NOT NULL
  AND r.ScheduledStartUtc < DATEADD(hour, -6, SYSUTCDATETIME())
  AND o.AgentRunOutcomeKey IS NULL
ORDER BY r.ScheduledStartUtc DESC;

-- d. duplicate-key groups (these will return MultipleMatches; do not reconcile blindly)
SELECT SourceProvider, ExternalGameId, COUNT(*) AS Runs
FROM AgentRuns
WHERE SourceProvider IS NOT NULL AND ExternalGameId IS NOT NULL
GROUP BY SourceProvider, ExternalGameId
HAVING COUNT(*) > 1;
```

### 4. Endpoint + auth

- `POST /api/agent-runs/reconcile` (tenant-scoped; same identity resolver as the per-run outcome endpoint).
- Auth options: a real bearer principal, OR the dev bypass -- in Development with `Dev:EnableBypassAuth=true` and valid `Dev:TenantKey` / `Dev:UserKey` (user-secrets / appsettings.Development.json). The resolved `TenantKey` must own the seeded runs.

### 5. Payload format (camelCase; `AddControllers()` default System.Text.Json)

```
ReconcileOutcomeRequest:
  sourceProvider   string   (required) e.g. "mlb_statsapi" or "odds_api"
  externalGameId   string   (required) e.g. "745804"
  outcomeStatus    string   (required) final: home_win | away_win | draw
                                       non-final: cancelled | postponed | suspended | void | unknown
  homeScore        int?     final score (null for non-final)
  awayScore        int?
  source           string   (required) trusted result source, e.g. "manual:statsapi"
  sourceRef        string?  external box-score id/url for audit
  notes            string?
  resolvedUtc      datetimeoffset?  back-fill timestamp; server uses UtcNow if omitted
```

Example (placeholder values only):

```
POST /api/agent-runs/reconcile
Content-Type: application/json
{
  "sourceProvider": "mlb_statsapi",
  "externalGameId": "PLACEHOLDER_GAMEPK",
  "outcomeStatus": "home_win",
  "homeScore": 5,
  "awayScore": 2,
  "source": "manual:statsapi",
  "sourceRef": "PLACEHOLDER_BOXSCORE_URL",
  "notes": "stage 0 manual seed"
}
```

Response (`ReconcileOutcomeResultDto`, camelCase):
`matchKind` (NotEvaluable | NoMatch | SingleMatch | MultipleMatches), `matchedRunIds` (Guid[]), `evaluatedRunId` (Guid?), `evalStatus` (correct | incorrect | inconclusive | null).

Expected behaviors: SingleMatch + final -> records outcome+evaluation, returns `evaluatedRunId` + `evalStatus`; SingleMatch already reconciled -> 409; NoMatch / MultipleMatches / NotEvaluable -> writes nothing.

### 6. Validation queries (after each reconcile)

```sql
-- outcomes recorded
SELECT o.AgentRunOutcomeId, r.AgentRunId, r.Competition, o.OutcomeStatus,
       o.HomeScore, o.AwayScore, o.Source, o.SourceRef, o.ResolvedUtc
FROM AgentRunOutcomes o JOIN AgentRuns r ON r.AgentRunKey = o.AgentRunKey
ORDER BY o.ResolvedUtc DESC;

-- evaluations by status (the calibration tally)
SELECT EvalStatus, COUNT(*) AS N
FROM AgentRunEvaluations
GROUP BY EvalStatus;

-- per-run evaluation detail
SELECT r.AgentRunId, r.Competition, e.EvalStatus, e.LeanSide, e.WinningSide, e.EvaluatedUtc
FROM AgentRunEvaluations e JOIN AgentRuns r ON r.AgentRunKey = e.AgentRunKey
ORDER BY e.EvaluatedUtc DESC;
```

### 7. Friction log template (one row per reconcile attempt)

```
| run AgentRunId | competition | provider | externalGameId | outcomeStatus | result source + ref | matchKind | evalStatus | friction / notes |
```

Record every NoMatch (wrong tenant? null identity? stale id?), MultipleMatches (duplicate runs for one game), NotEvaluable (non-final), and 409 (already reconciled).

## Runs attempted / outcomes recorded / no-match / multiple-match / non-final cases

None this session -- DB unreachable. To be filled when the runbook is executed.

## Observed friction (structural, found this session)

- the reconcile endpoint is tenant-scoped: a Stage 0 operator must seed/own runs under the resolved `Dev:TenantKey`, or every reconcile returns NoMatch despite a correct key. This is the most likely first-run trip-up.
- the result source for the real final score is manual and out-of-band; the endpoint records whatever `source`/`sourceRef` the operator supplies but cannot verify it. Source-reliability discipline is on the operator.
- MultipleMatches has no resolution path yet (matcher surfaces it, writes nothing) -- duplicate runs for one game must be reconciled by a future decision, not in Stage 0.
- the `ExpandOutcomeStatusTaxonomy` migration must be applied before any non-final status (`suspended`/`void`/`unknown`) can be recorded, or the check constraint rejects it.

## Source reliability notes

- prefer the same provider that grounded the run for the settlement result (odds_api / ESPN for NBA, statsapi.mlb.com gamePk for MLB) so the settled game and the run reference the same event identity.
- record `source` as `manual:<provider>` and `sourceRef` as the external box-score id/url; this preserves provenance for later weighting of manual vs automated sources.

## Evaluator concerns

- none found requiring a code change. `RunEvaluator` correctly yields inconclusive for draw and for null lean; correctness stays lean-direction-vs-outcome only; buyer-advertised strength is excluded from correctness (per contract). No evaluator behavior change is proposed.
- watch item (not a bug): runs whose `LeanSide` is null (analyze-failure or no directional read) will evaluate inconclusive on a final outcome -- expected, but they add no calibration signal; the Stage 0 sample should prefer runs with a non-null `LeanSide`.

## Stage 0 completion criteria

Stage 0 is complete when: at least ~5-10 real settled NBA/MLB runs have been reconciled through `/reconcile`; the AgentRunEvaluations tally shows a non-trivial mix of correct/incorrect (not all inconclusive); at least one NoMatch, one MultipleMatches (if duplicates exist), and one non-final NotEvaluable have been observed and logged; and the friction log captures source-reliability and tenant-scoping notes. The output is evidence + a friction record, not a buyer claim.

## Recommendation: is Internal Calibration Read Surface v1 ready next?

Not yet. A read surface that aggregates AgentRunEvaluations is premature until Stage 0 has produced a real, non-trivial body of correct/incorrect evaluations -- otherwise the surface would aggregate an empty or all-inconclusive set and prove nothing. Sequence: (1) bring up the dev DB + apply migrations, (2) run this runbook to seed real evaluations and a friction log, (3) only then build Internal Calibration Read Surface v1 against actual data. If the dev DB has no settled identity-bearing runs at all, generate a few via the normal analyzer path against recently completed games first.

---

# Execution: Manual Stage 0 Reconciliation Execution v1

**execution date:** 2026-06-12
**status:** executed against local/dev. environment is now ready (DB up, both required migrations applied, reconcile endpoint verified live). no calibration sample produced -- blocked on the absence of identity-bearing runs. one model-free plumbing-only validation of the endpoint's non-writing paths was performed. no calibration evidence fabricated.
**classification:** plumbing-only (live endpoint validation). NOT a true calibration sample.

## Environment confirmed local/dev

- connection string `Server=localhost,1433;Database=devcore` (appsettings.Development.json) -- localhost, not production.
- SQL Server runs in the local docker container `devcore-sql` (`mcr.microsoft.com/mssql/server:2022-latest`), confirmed `Up` via `docker ps`.
- dev auth bypass on: `Dev:EnableBypassAuth=true`, `Dev:TenantKey=1`, `Dev:UserKey=1`.
- environment identity is unambiguous (localhost container, devcore db). guard satisfied: no production, no remote production-like database touched.

## DB readiness result

- `localhost,1433` reachable this session (`Test-NetConnection` -> `TcpTestSucceeded=True`) -- the prior session's blocker (port closed) is resolved; the container was already running.
- `devcore` database present (`sys.databases`).
- sqlcmd executed inside the container: `docker exec devcore-sql /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P <pw> -C -d devcore -Q ...`.

## Migration status

- before: latest applied migration was `20260425145646_AddAgentRunEvaluation`; three migrations pending, including both required ones. `AgentRuns` existed but lacked the identity columns (`SourceProvider`, `ExternalGameId`, `ScheduledStartUtc`, ...).
- applied via the documented dev flow (runbook step 0.2):
  `dotnet ef database update --project platform/dotnet/DevCore.Data --startup-project platform/dotnet/DevCore.Api`
  this applied all three pending migrations: `20260605123908_AddProbeRefreshMergeAuditStore`, `20260611152737_AddAgentRunGameIdentity`, `20260612121023_ExpandOutcomeStatusTaxonomy`.
- after: both required migrations confirmed present in `__EFMigrationsHistory`. The `CK_AgentRunOutcomes_OutcomeStatus` check constraint now includes `suspended`/`void`/`unknown`. local/dev only; no production migration.

## API readiness result

- started the platform api only (Development, `http` profile, `http://localhost:5007`) via `dotnet run --project platform/dotnet/DevCore.Api --launch-profile http`. the reconcile endpoint touches only the DB + matcher, so the FastAPI agent service and a funded model key were not required.
- readiness confirmed: `GET /api/competitions` -> 200.
- the api process was stopped cleanly after the probes (no lingering dev process).

## Candidate query results

| query | result |
|---|---|
| total AgentRuns | 115 |
| identity-bearing (`SourceProvider` AND `ExternalGameId` not null) | **0** |
| rows with `SourceProvider` not null | 0 |
| rows with `ExternalGameId` not null | 0 |
| rows with `ScheduledStartUtc` not null | 0 |
| settled candidates (start in past, no outcome) | 0 (none can qualify -- no identity) |
| duplicate `(SourceProvider, ExternalGameId)` groups | 0 (no identity rows to group) |
| existing AgentRunOutcomes | 8 |
| existing AgentRunEvaluations | 8 |

competition/provider breakdown of the 115 runs: 59 null-competition, 27 mlb, 27 nba, 2 nfl -- **all with null `SourceProvider`**. the 8 pre-existing outcomes (6 mlb, 2 nba, all `Source=manual`) were attached via the per-run `POST /outcome` path, not the matcher; their evaluation tally is 3 correct / 2 incorrect / 3 inconclusive.

## Candidate selection decision

No selection possible. Every one of the 115 runs predates identity capture (all identity columns null), so the matcher -- which keys exactly on `(SourceProvider, ExternalGameId)` and never falls back to display names -- returns NoMatch for all of them. There are zero true calibration candidates and zero rows on which a live SingleMatch could be exercised.

## True calibration candidates found

None. A true calibration sample requires an identity-bearing run generated before its game settled. No run carries identity at all. No calibration evidence was produced or fabricated.

## Plumbing-only candidates generated

No new runs were generated (that would require model spend). Instead, the reconcile endpoint's two non-writing classifications were exercised live against the migrated DB to validate the path mechanics. This is plumbing-only and must not be counted as model-performance or calibration evidence.

## Endpoint payloads used

`POST http://localhost:5007/api/agent-runs/reconcile`, `Content-Type: application/json`, camelCase, dev-bypass auth (no bearer).

NoMatch probe (final status, nonexistent id):
```json
{ "sourceProvider": "mlb_statsapi", "externalGameId": "plumbing-nomatch-0001",
  "outcomeStatus": "home_win", "homeScore": 1, "awayScore": 0,
  "source": "plumbing:none", "sourceRef": "stage0-plumbing", "notes": "plumbing-only nomatch probe" }
```

NotEvaluable probe (non-final status):
```json
{ "sourceProvider": "mlb_statsapi", "externalGameId": "plumbing-nomatch-0001",
  "outcomeStatus": "postponed", "source": "plumbing:none", "sourceRef": "stage0-plumbing",
  "notes": "plumbing-only noteval probe" }
```

## Outcomes attempted / recorded

- attempted: 2 plumbing-only reconcile calls (both intentionally non-writing classifications).
- recorded: 0. Neither call wrote an outcome or evaluation, by design.

## Validation query results

- `outcomes_before = 8`, `outcomes_after = 8`; `evaluations_after = 8`. Confirmed the endpoint wrote nothing for NoMatch and NotEvaluable -- matching the matcher contract.

## NoMatch / MultipleMatches / NotEvaluable findings

- **NoMatch:** observed live -> `{"matchKind":"NoMatch","matchedRunIds":[],"evaluatedRunId":null,"evalStatus":null}`. Wrote nothing.
- **NotEvaluable:** observed live (non-final `postponed`) -> `{"matchKind":"NotEvaluable","matchedRunIds":[],"evaluatedRunId":null,"evalStatus":null}`. Wrote nothing. Confirms evaluability is checked before matching.
- **MultipleMatches:** not reachable -- needs duplicate identity-bearing rows, of which there are zero.
- **SingleMatch:** not reachable live -- needs at least one identity-bearing run; covered today only by the 9 in-process reconcile integration tests.

## Tenant / auth friction

Dev bypass resolved `TenantKey=1` cleanly; the endpoint was reachable without a bearer token. Tenant-scoping friction noted in the seed runbook is unexercised here because no candidate rows exist; it remains the most likely first trip-up once identity-bearing runs are seeded under the resolved tenant.

## Source reliability notes

No real outcome source was consulted -- no final scores were written, so no provider settlement was needed. The plumbing payloads use `source: "plumbing:none"` precisely to mark them as non-evidential. When real reconciliation runs, prefer the same provider that grounded the run (statsapi `gamePk` boxscore for MLB, odds-api/ESPN for NBA) and record `source`/`sourceRef` for provenance.

## Evaluator observations

`RunEvaluator` was not invoked (no SingleMatch). No evaluator behavior change is warranted. The pre-existing 8 evaluations (3 correct / 2 incorrect / 3 inconclusive) were produced by the per-run path before this session and are unaffected.

## Root-cause of the data gap

Identity capture shipped at the code level (Stable Game Identity Capture v1, MLB Game Identity Capture v1), but every run in the dev DB was generated before those slices, so the production-shaped data has zero identity coverage. The reconcile/matcher path is correct but starved of input. Stage 0 cannot produce calibration evidence until fresh identity-bearing runs are generated through the analyzer and allowed to settle.

## Whether Internal Calibration Read Surface v1 is ready next

Still not ready. The environment is now proven (DB up, migrations applied, endpoint validated live), but no reconciled calibration evidence exists. Updated sequence: (1) DONE -- bring up dev DB, apply migrations, validate the reconcile endpoint live; (2) generate a small batch of fresh identity-bearing NBA/MLB runs via the normal analyzer path (model spend) for games about to play; (3) after they settle, reconcile them through `/reconcile` with trusted sources to seed real correct/incorrect evaluations; (4) only then build Internal Calibration Read Surface v1. Step (2) is the first model-spend action and should be an explicit, budgeted next slice.

---

# Execution: Stage 0 Outcome Reconciliation Follow-Up v1

**execution date:** 2026-06-12
**status:** local/dev follow-up performed too early to reconcile. no `POST /api/agent-runs/reconcile` calls were submitted. no outcomes or evaluations were written. no final results were fabricated.
**classification:** wait-only, no evidence produced.

## Candidate check

The four identity-bearing candidates from Stage 0 True Calibration Candidate Capture v1 were checked through local API artifact/evaluation reads. All four still had the expected provider key and completed artifact state; every evaluation endpoint returned 404.

## Settlement status

DB UTC time was `2026-06-12 14:45:03Z`, before all four scheduled starts:

- Padres at Orioles: `2026-06-12T23:05:00Z`
- Rangers at Red Sox: `2026-06-12T23:10:00Z`
- Yankees at Blue Jays: `2026-06-12T23:37:00Z`
- Knicks at Spurs: `2026-06-14T00:40:00Z`

Because all games were pre-start, no trusted final result source was available or used.

## Outcomes attempted / recorded

- reconcile payloads submitted: 0.
- outcomes recorded: 0.
- evaluations recorded: 0.
- correct / incorrect / inconclusive added by this follow-up: 0 / 0 / 0.

## Matcher findings

No `NoMatch`, `MultipleMatches`, or `NotEvaluable` findings were produced in this follow-up because no reconcile request was sent. This avoids turning a pre-start game into a synthetic non-final test. The previously documented plumbing-only `NoMatch` and `NotEvaluable` probes remain the only live matcher classification probes in this runbook.

## Evaluator observations

The evaluator was not invoked. The expected future caveat remains: the Yankees/Blue Jays run has no directional lean, so a later final outcome may evaluate inconclusive.

## Recommendation

Internal Calibration Read Surface v1 is still not ready. Run the follow-up again only after the games have settled, then reconcile the exact provider keys through `/api/agent-runs/reconcile` with trusted final sources.
