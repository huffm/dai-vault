# decision 0008: source readiness preflight gate

**date:** 2026-07-03
**status:** accepted (implemented in `dai`; docs-only in vault)

## context

v8 halted twice at canary, each time AFTER spending a paid model call. Canary v2 (MIN@NYY) landed
`starter_missing_market_missing` because generation's `MlbStarterClient` uses
`statsapi /api/v1/schedule?hydrate=probablePitcher` and requires BOTH probables (home Yankees read TBD),
while the v8 free preflight had used a DIFFERENT endpoint (`feed/live`, which showed Rodon) -- so the free
check was optimistic and disagreed with what generation actually retrieves. The market dimension is
similarly not schedule-predictable: `OddsMarketClient` matches an odds event by exact team name inside an
ET-date window, and books post games at different times (~21h out, none of the 10 approved games matched).

The canary protects the cohort but wastes the individual call. We need a gate that predicts the regime
BEFORE the model call, using the SAME retrieval as generation.

## decision

Add a read-only, pre-model **Source Readiness Preflight Gate**: `GET /api/agent-runs/source-readiness`
(competition, homeTeam, awayTeam, gameDate). It runs the EXACT generation retrieval and predicts the
regime -- without a model call, an AgentRun row, or a market snapshot.

1. **Reuse, not re-implement.** the endpoint builds an ephemeral `SportsRunArtifact` (throwaway run id,
   never persisted) and calls the same `ISportsRetriever.RetrieveAsync` generation uses. RetrieveAsync
   fetches grounded evidence and records it on the in-memory artifact only; the model call and all DB
   persistence happen LATER in `AgentRunService`/the controller, which the gate never reaches. So the gate
   sees exactly what generation would see -- closing the endpoint-mismatch that fooled the free preflight.
2. **Deterministic classification (`SourceReadinessClassifier`)** mirrors the python observed-regime logic
   (`migration_readiness._starter_state`/`_market_state`, `dataregime.mlb_regime`): starter enriched iff
   both starters carry season quality (`MlbPitcherQuality.DataAvailable`, matching `format_pitcher_quality`);
   market `backed_depth` iff `BookCount >= 2 AND ConsensusSide`. `predictedObservedDataRegime` therefore
   equals the `observedDataRegime` the analyzer would stamp for the same retrieval.
3. **Eligibility gate:** `generationEligibleForTargetRegime = identity matched AND starter enriched AND
   market backed_depth`. Otherwise a composed `stopReason` names what is not ready.
4. **Contract fields:** sourceProvider, externalGameId, game, gameDate, identityStatus, starterReadiness
   (level/home/away/source/reason), marketReadiness (level/bookCount/consensusSide/reason),
   predictedObservedDataRegime, generationEligibleForTargetRegime, targetRegime, stopReason, checkedAtUtc.

## safety guarantees (proven by tests + a live run)

- **no model call:** the gate never invokes the analyzer/FastAPI/OpenAI; it stops after retrieval.
- **no AgentRun row / no market snapshot:** RetrieveAsync writes nothing; persistence is downstream.
  Verified live: screening all 10 candidates left `AgentRuns` unchanged (265). Integration test asserts
  0 AgentRuns + 0 MarketSnapshotBatches after a call.
- **no prompt / decision / confidence / buyer / denominator change:** it is a read-only advisory endpoint;
  it does not touch generation behavior.
- **cost:** the gate makes the same free StatsAPI + one PaidExternal odds call generation would make -- a
  readiness read is far cheaper than a wasted model generation.

## consequences

- v8 (and any measurement cohort) can screen a candidate for `starter_enriched_market_backed_depth` BEFORE
  spending a model call; only eligible games are generated. Live screening at ~21h out showed all 10
  ineligible (9 starter-enriched but market-missing; 1 starter+market missing) -- exactly the calls the
  gate now saves.
- The endpoint-mismatch lesson is institutionalized: readiness == generation retrieval, never a parallel
  free endpoint.
- No retrieval BEHAVIOR was changed this slice (the `MlbStarterClient` schedule-vs-feed/live discrepancy
  and odds timing are data-availability facts, not bugs to fix here). A future retrieval-robustness slice
  MAY add a schedule->feed/live starter fallback; deferred.

## references

- Origin: `06 Execution/reconciliations/v8-cohort-execution-canary-halt-v2.md`; this slice
  `06 Execution/reconciliations/source-readiness-preflight-gate-v1.md`.
- Code: `platform/dotnet/DevCore.Api/AgentRuns/SourceReadiness.cs`,
  `Controllers/AgentRunsController.cs` (GET source-readiness), reusing
  `AgentRuns/SportsRetriever.cs` + `Sports/MlbStarterClient.cs` + `Sports/OddsMarketClient.cs`.
- Attribution lineage: ADR 0007 (observed regime) -- the gate predicts what 0007 stamps.
