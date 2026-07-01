---
title: "Apply AgentRun Provenance Migration to Dev SQL v1"
type: "evidence-report"
date: "2026-06-29"
status: "complete"
project: "DAI"
slice: "Apply AgentRun Provenance Migration to Dev SQL v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - provisioning
  - provenance
related:
  - "06 Execution/reports/dotnet-agentrun-prompt-provenance-persistence-v1.md"
---

# Apply AgentRun Provenance Migration to Dev SQL v1

**status:** complete (migration applied + verified against dev SQL; docs-only commit)
**date:** 2026-06-29

## purpose

Clear the standing operational debt: apply and verify the existing AgentRun prompt provenance migration against
the local/dev SQL database, so platform-side prompt provenance persistence works against the real dev database
and not only in-memory EF tests. Operational DB verification slice -- no schema redesign, no code change.

## start state

dai `289777f` (main, synced, on origin): the four prompt-provenance feature commits are pushed; the EF model has
`AgentRun.PromptRouteProvenanceJson`; the migration `20260629174632_AddAgentRunPromptRouteProvenance` is in the
chain; `GET /api/agent-runs/prompt-route-calibration/metrics` exists; `DEFAULT_ALLOWLIST` = four regimes.
dai-vault `509acc9` (main, synced) with the prior provenance/export/metrics/endpoint docs. Only the pre-existing
untracked `06 Execution/system-state-synopsis-v1.md` was dirty (unrelated, untouched).

## migration name

`20260629174632_AddAgentRunPromptRouteProvenance` (adds nullable `AgentRuns.PromptRouteProvenanceJson`,
`nvarchar(max)`).

## database target

`Server=localhost,1433; Database=devcore` (SQL Server in the `devcore-sql` Docker container; sa login). The
connection string lives in `platform/dotnet/DevCore.Api/appsettings.Development.json` (dev password not
reproduced here).

## container status

Before: Docker daemon down (`com.docker.service` Stopped); `devcore-sql` in `Exited (137)` state (persists
across reboots, per the dev README). After: Docker Desktop started, daemon up; `devcore-sql` started -> `Up`,
listening on `0.0.0.0:1433->1433/tcp`; logs reported "SQL Server is now ready for client connections".

## commands run

```
# from repo root
Start-Process "C:\Program Files\Docker\Docker\Docker Desktop.exe"   # bring up the daemon
docker ps                                                          # confirm daemon (polled until up)
docker start devcore-sql                                           # start the dev SQL container
docker logs devcore-sql                                            # confirm "SQL Server is now ready..."

# from platform/dotnet, ASPNETCORE_ENVIRONMENT=Development (so the dev connection string is used)
dotnet ef migrations list   --project DevCore.Data --startup-project DevCore.Api --no-build   # showed (Pending)
dotnet ef database update    --project DevCore.Data --startup-project DevCore.Api --no-build  # applied it

# schema + data verification (sqlcmd inside the container, against the devcore database)
docker exec devcore-sql /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P <dev-pw> -C -d devcore -Q "<checks>"

# application-level verification
dotnet build DevCore.Api.Tests
dotnet test DevCore.Api.Tests --filter "PromptRouteProvenance|PromptRouteCalibration|AgentRunsController"
```

## applied / already-applied status

**Applied this slice.** `dotnet ef migrations list` showed `20260629174632_AddAgentRunPromptRouteProvenance
(Pending)` (the only pending migration; all others applied). `dotnet ef database update` then ran
`ALTER TABLE [AgentRuns] ADD [PromptRouteProvenanceJson] nvarchar(max) NULL;` and inserted the
`__EFMigrationsHistory` row. A re-list afterward shows it with no `(Pending)` marker.

## schema verification

Verified against the `devcore` database via sqlcmd:
- `__EFMigrationsHistory` contains `20260629174632_AddAgentRunPromptRouteProvenance` (count = 1).
- `INFORMATION_SCHEMA.COLUMNS`: `PromptRouteProvenanceJson`, type `nvarchar`, `IS_NULLABLE = YES`.

## data safety check

- `SELECT COUNT(*) FROM AgentRuns` = **235** rows -- existing data intact, nothing reset or deleted.
- `SELECT COUNT(*) FROM AgentRuns WHERE PromptRouteProvenanceJson IS NOT NULL` = **0** -- the column is null for
  every historical row, as expected for a nullable additive column. **No backfill was performed.**
- The migration is purely additive (one nullable column); no existing column or row was modified.

## tests run

- `dotnet ef migrations list ...` -> `20260629174632_AddAgentRunPromptRouteProvenance` no longer `(Pending)`.
- `dotnet build DevCore.Api.Tests` -> 0 errors.
- `dotnet test DevCore.Api.Tests --filter "PromptRouteProvenance|PromptRouteCalibration|AgentRunsController"`
  -> **90 passed, 0 failed**.
- (Full suite was 971 passed at the prior slice; no code changed here, so it is unaffected.)
- No paid model calls.

Note: the .NET test suite uses the EF Core in-memory provider, so the tests do not themselves exercise the dev
SQL column; the dev-SQL column is verified directly via sqlcmd above. The tests confirm no code path requires a
non-null provenance value.

## code changes

None. DB-only operational slice; the EF model, migration, and snapshot were already committed in
.NET AgentRun Prompt Provenance Persistence v1. No appsettings/connection-string change was needed.

## doc changes

NEW `06 Execution/apply-agentrun-provenance-migration-to-dev-sql-v1.md` (this doc) + appended the slice result to
`06 Execution/handoffs/current-slice.md`.

## paid-call status

None.

## buyer-facing impact

None. Additive nullable column applied to the dev database; no buyer UX/copy/body, no prompt/template/recipe
change, no model call.

## default allowlist status

Unchanged at four regimes.

## rollback plan

Not performed (not needed -- the column is nullable, additive, and currently all-null). If ever required:
`dotnet ef database update 20260622205819_AddMarketSnapshotLedger --project DevCore.Data --startup-project
DevCore.Api` (the migration immediately prior) runs the `Down()` `DROP COLUMN [PromptRouteProvenanceJson]`. Note
that rollback would discard any `PromptRouteProvenanceJson` data written after this point; today there is none.

## risks / deferred items

- Applied to the LOCAL dev database only. Staging/production databases still need the same `database update` (or
  the deploy migration step) before the platform-side export/metrics run against those environments.
- `devcore-sql` persists in `Exited` state across reboots; future sessions just need `docker start devcore-sql`
  (the dev README documents this).
- No real provenance rows exist yet in dev (`prov_not_null = 0`); the column will populate as new MLB analyzer
  runs flow through the .NET capture path. A later slice could exercise an end-to-end run to populate one.

## next recommended slice

Live-Scheduled Starter-Missing Soak v1, OR Broad Cohort Rerun Grouped by Prompt Recipe v1, OR Calibration
Metrics Export Download v1.

Related: [[dotnet-agentrun-prompt-provenance-persistence-v1]],
[[platform-side-prompt-provenance-calibration-export-v1]], [[calibration-outcome-metrics-by-prompt-route-v1]],
[[thin-tenant-scoped-calibration-metrics-endpoint-v1]].
