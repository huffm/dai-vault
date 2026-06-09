# Decision Artifact Persistence Review v1

**date:** 2026-06-09
**status:** read-only persistence review + decision-support. docs only. no runtime code, no migration, no schema change, no database-technology commitment, no artifact-contract/buyer-projection/confidence/posture/lean/prompt/model/cost-guardrail change.
**scope:** map what the factory currently remembers and define what it must remember next, before any Postgres or outcome-reconciliation work.

## Purpose

Before choosing Postgres, changing database technology, or building outcome reconciliation, inspect what the factory currently persists and define the memory requirements for artifact history, cost visibility, buyer validation, per-sport calibration, and future outcome reconciliation.

Primary question: what does the DAI factory need to persist to support those goals? This is a persistence review, not a migration or schema slice.

## Scope

- read-only inspection of the persistence layer (EF `AppDbContext`, domain entities, migrations), the AgentRun write/read path, cost-telemetry implementation, and the vault calibration artifact convention.
- document current state, memory requirements, a SQL-Server-vs-Postgres assessment, and a recommended storage posture.

## Non-goals

- no runtime code, EF migration, schema, or artifact-contract change.
- no Postgres migration or database-technology implementation.
- no outcome-reconciliation runtime, no cost-guardrail change, no buyer-projection/confidence/posture/lean/prompt/model/source change.
- no auth/tenant/billing/Stripe/dashboard/deployment change. no Jera change. no push.

## Current persistence state

Storage is SQL Server (`devcore` database) via EF Core (`DevCore.Data.AppDbContext`), run locally in the `devcore-sql` Docker container. The model is more complete than the "v3 artifact + cost log" framing implied. Persisted tables (DbSets):

- chat: Conversations, Messages
- identity: Tenants, Users, UserIdentities
- agentic: AgentProfiles, AgentPermissions, **AgentRuns**, **AgentRunOutcomes**, **AgentRunEvaluations**, ProbeRefreshMergeAudits (dormant probe-refresh audit ledger)
- sports reference (seeded, platform-level, not tenant-scoped): Sports, Teams

AgentRun (the manufactured unit) persists, per row:

- identity/keys: `AgentRunKey` (bigint PK), `AgentRunId` (GUID public id), `TenantKey`, `RequestedByUserKey`, `AgentProfileKey` -- tenant/user attribution is already threaded.
- run metadata: `RunType`, `Status` (pending/completed/failed, check-constrained), `CorrelationId`, `StartedUtc`, `CompletedUtc`, `DurationMs`, `ErrorMessage`.
- request + artifact: `InputJson` (the request) and **`OutputJson` (the full serialized `AgentRunExecutionResult` -- the entire v3 artifact)**.
- promoted first-class columns (migration `MakeCompetitionFirstClass`): `Competition` (<=32), `GameDate` (date), `LeanSide` (<=8).
- indexes: `(TenantKey, RequestedByUserKey, StartedUtc)`, `(TenantKey, RunType, StartedUtc)`, and a calibration-friendly `(TenantKey, Competition, GameDate)`.

Answers to the review's persistence questions:

- full artifact JSON persisted: **yes** (`OutputJson`, required).
- cognitiveProtocol persisted: **yes** -- inside `OutputJson` (the composer writes `CognitiveProtocol` into the execution result; the artifact endpoint deserializes it back).
- buyer projection persisted: **no** -- buyer-advertised strength, evidence-sufficiency tier, and the `advertised_strength_limited_by_evidence` humility reason are derived in the Angular projection only (Evidence-Sufficiency Band Gate v1; ledger entry 23). They are re-derivable from persisted inputs.
- raw confidence persisted: **yes** -- inside `OutputJson` (`Confidence`/`AggregateConfidence` and `AnalyzerConfidence`).
- evidenceRichness persisted: **yes** -- inside `OutputJson`.
- source/retrieve status persisted: **yes** -- `groundedSignals`, `missingSignals`, `signalAvailability`, `signalFollowUps`, and `pipelineSteps` all live inside `OutputJson`.
- cost telemetry persisted: **no** -- log-only to the `devcore.cost` logger (Artifact Cost Guardrails v1; ledger entry 22). Tokens/cost/latency/finishReason/requestId are emitted as a structured log line and lost after emission. **This is the one clear persistence gap.**
- model tokens/cost/latency queryable later: **no** -- not stored anywhere queryable.
- vault calibration JSONs the only durable artifact samples: **no** -- they duplicate (a human-browsable copy of) the `GET /artifact` DTO; the authoritative artifact lives in SQL `OutputJson`.
- what is in SQL vs only in vault: SQL holds the authoritative runs/artifacts/outcomes/evaluations; the vault holds calibration reports + artifact JSON snapshots (research/review copies) and all the slice reports.

Important caveat: most artifact-internal fields (confidence, evidenceRichness, posture, cognitiveProtocol, signalAvailability) are queryable only by parsing `OutputJson`; only `Competition`, `GameDate`, `LeanSide`, `Status`, and the timestamps are promoted to indexed columns.

## Current SQL / devcore storage observations

- 17 EF migrations exist; the agentic/outcome chain landed across `AddAgentRuns`, `MakeCompetitionFirstClass`, `AddAgentRunOutcomeColumns`, `AddAgentRunOutcome`, `AddAgentRunEvaluation`. So outcome/evaluation storage was deliberately built ahead of a runtime that fully uses it.
- delete safety is intentional: outcomes and evaluations use `OnDelete(NoAction)` ("historical outcome data is more durable than a run row"); tenants never cascade-hard-delete.
- calibration-oriented indexes already exist: `AgentRunOutcomes (OutcomeStatus, ResolvedUtc)` and `AgentRunEvaluations (EvalStatus, EvaluatedUtc)`.

## Vault calibration artifact observations

- the calibration harness writes per-run artifact JSON to `04 Products/sports-v1/calibration/artifacts/<ts>-<comp>-<shortId>.json` plus a markdown calibration report. These are the `GET /artifact` DTO captured for human review; they are not the system of record (SQL is). They are committed intentionally as research samples under the accepted convention.
- this is the right place for research/calibration snapshots, but it should not be confused with durable factory memory -- vault artifacts are a curated subset, not a queryable history.

## Cost telemetry persistence status

Log-only. Artifact Cost Guardrails v1 emits one structured JSON cost record per run to the `devcore.cost` logger inside the FastAPI analyzer; `.NET` does not capture model usage at all (it only HTTP-calls the analyzer). Nothing writes cost to SQL. Consequences: no per-run unit cost history, no cost-per-competition rollup, no cost-vs-evidence analysis, no spend trend. Closing this is the highest-value database-agnostic persistence improvement and is already tracked as ledger entry 22 (cost sink deferred).

## Artifact identity and game matching findings

What exists to identify a run/game:

- `AgentRunId` (GUID) -- stable unique run id.
- `Competition` (e.g. "mlb") + `GameDate` (date) as promoted columns.
- home/away team **display names** inside `InputJson`/`OutputJson` (e.g. "Baltimore Orioles").
- a seeded `Teams` reference table (`SportKey`, `Name`), but AgentRun does **not** store `Team` foreign keys -- it stores names in JSON.
- `LeanSide` (home/away/null) promoted as a column -- the field the evaluator compares against the winning side.
- `AgentRunOutcome.SourceRef` (<=512) can hold an external reference, and `Source` names the provider -- but only on the outcome row, populated at outcome-ingestion time.

Can a generated artifact be matched to a final game outcome later? Today only by `(competition, gameDate, homeTeam name, awayTeam name)`. That works for the manual outcome endpoint but is fragile for automation: team-name drift, no stable provider/event id on the run, doubleheaders/postponements not disambiguated, and no link to the provider that grounded the signals. **The main matching gap is the absence of a stable external game/event id on the AgentRun** (or a Team-FK pair) to reliably join an artifact to a settled outcome. This is the key thing Outcome Reconciliation Contract v1 must define.

## Factory memory requirements

What the factory needs to remember, by goal:

- buyer artifact history: the artifact payload + run metadata + a stable run id -- already met (`OutputJson` + AgentRun).
- cost / unit economics: per-run model usage and estimated cost, queryable and rollup-able by competition/time -- **not met** (log-only).
- evidence-sufficiency analysis: evidenceRichness, grounded/missing signals, signalAvailability per run -- met inside `OutputJson`; queryability limited.
- confidence calibration: raw + analyzer confidence, posture, lean per run, joinable to outcomes -- inputs met; the join key for reliable outcome matching is weak.
- outcome reconciliation: a settled outcome per run + a derived evaluation -- data model met (AgentRunOutcomes/AgentRunEvaluations); the settlement source and stable match key are missing.
- per-sport validation: filter/group by competition -- met (`Competition` indexed).
- tenant/economic boundary later: per-tenant attribution on every run -- already threaded (`TenantKey`); cost-per-tenant blocked only by the cost-persistence gap (ledger entries 11, 22).
- audit/debugging: correlation id, pipeline steps, quality warnings, error message -- met.

## Candidate persistence categories

Separating the memory needs (so storage can be reasoned about per category):

1. artifact payload memory: the v3 artifact (`OutputJson`). Exists.
2. run metadata: status, timestamps, duration, correlation, competition, gameDate, leanSide, tenant/user. Exists (partly promoted to columns).
3. signal/source snapshot: grounded/missing/signalAvailability/signalFollowUps/pipelineSteps. Exists inside payload; not column-queryable.
4. buyer projection / advertised-strength memory: advertised band, evidence tier, humility reason. Not persisted (derived in Angular; re-derivable). Optional to persist (ledger entry 23).
5. model cost telemetry: tokens, cost, latency, finishReason, requestId, model. **Not persisted (gap; entry 22).**
6. outcome / reconciliation memory: outcome (status/scores/source/sourceRef/resolvedUtc) + derived evaluation (correct/incorrect/inconclusive). Data model + manual endpoint + evaluator exist; settlement source + match key missing.
7. tenant / billing attribution later: TenantKey present on runs; cost attribution blocked by category 5.
8. calibration / vault research artifacts: curated JSON snapshots + reports in the vault. Exists; not the system of record.

## SQL Server vs Postgres assessment

- Is current SQL Server enough for the next 30-60 days? **Yes.** The relational model already supports artifact history, run metadata, per-sport filtering, outcomes, and evaluations, with the right indexes. Dev volumes are tiny and there is no revenue or scale pressure.
- Would Postgres/jsonb materially improve artifact querying now? **Marginally, not materially.** The only friction is that confidence/evidenceRichness/signalAvailability live inside `OutputJson`; SQL Server can already query JSON via `JSON_VALUE`/`OPENJSON`, just without a GIN-style index. Postgres `jsonb` + GIN would help only if artifact-internal fields become hot, indexed query paths at real volume -- which is not the case yet.
- What would Postgres help with later? Heavy ad-hoc querying of artifact-internal JSON at scale, `jsonb` indexing, and (separately) pgvector if memory/embeddings are ever added (already a deferred, unrelated item).
- What would migrating now cost? Re-targeting the EF provider, rewriting 17 migrations' provider-specific bits (check constraints, `newsequentialid()`, `datetimeoffset` defaults), swapping the `devcore-sql` container and connection strings, and re-validating identity/seed/outcome behavior -- real attention and regression risk for zero current benefit.
- What should be deferred until revenue or stronger need? The database-technology choice itself.
- What minimal persistence improvements are database-agnostic (work on SQL Server now, portable later)? (a) persist cost telemetry (table or columns; entry 22); (b) add a stable external game/event id (or Team-FK pair) to AgentRun for reliable outcome matching; (c) optionally promote confidence/evidenceRichness/advertisedStrength to indexed columns for calibration queries. None of these require Postgres.

Doctrine check: do not assume scale before revenue; money is the only real validation; tenant is an economic boundary (already modeled); do not build storage architecture ahead of need. All point to staying put.

## Recommended storage posture

**Stay on SQL Server for now and improve documented memory requirements; defer Postgres.** Immediate migration is not warranted -- the evidence is the opposite of overwhelming. The next persistence implementation work (a cost-telemetry sink, a stable game-match key, optional column promotion for calibration) is database-agnostic and lands cleanly on the current SQL Server, keeping a future Postgres decision cheap and evidence-gated. Treat the database choice as a factory-memory decision driven by query/scale/revenue need, not a platform-fashion migration.

## Deferred decisions

- database technology (SQL Server vs Postgres): deferred -- recorded as new ledger entry 24.
- cost-telemetry persistence sink: already deferred (entry 22); reaffirmed as the highest-value database-agnostic next improvement.
- buyer-advertised-strength persistence / server authority: already deferred (entry 23); re-derivable, low urgency.
- stable game/event match key for reconciliation: identified here; should be specified by Outcome Reconciliation Contract v1 (the next slice), not implemented now.

## Recommended next slice

**Outcome Reconciliation Contract v1** (the planned step 5). This review shows the storage scaffold (AgentRunOutcomes, AgentRunEvaluations, the manual `/outcome` endpoint, `RunEvaluator`) already exists, so the contract slice should focus on: the settlement source and trigger, the stable game/event match key (the real gap), the settlement rules per competition (draw/cancelled/postponed/overtime), and aggregation/calibration queries over evaluations -- all database-agnostic on SQL Server. Per-Sport Buyer Validation Matrix v1 (step 6) can then read competition-grouped runs/evaluations.

## What was not changed

- no runtime code, EF migration, or schema
- no database-technology choice or migration
- no artifact contract, buyer projection, confidence, posture, lean, or decision logic
- no prompt, model-call, or cost-guardrail change
- no source/probe-refresh/pricing/Stripe/auth/tenant/dashboard/deployment change
- no Jera change

## Verification / unknowns

- claims about persisted fields are grounded in `AppDbContext.cs`, the AgentRun/AgentRunOutcome/AgentRunEvaluation domain entities, the AgentRunsController endpoints, and `RunEvaluator` (all read this slice).
- not inspected at the data level: whether any AgentRunOutcome/Evaluation rows actually exist yet in the dev database (runtime state, not code). Expectation: few or none, since no automated settlement source is wired (ledger entry 12 says gates flag only). Verify later with a read-only `SELECT COUNT(*)` if needed; not required for this review.
