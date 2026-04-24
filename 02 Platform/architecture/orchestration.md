# orchestration

**date:** 2026-04-20
**status:** four explicit pipeline steps in place — retrieve, analyze, evaluate, compose. typed SportsRetrievalOutput and EvaluatorOutput exist. calibrated confidence comes from the evaluator, not the analyzer. the compose step assembles AgentRunExecutionResult from all prior stage outputs. competition is the internal routing key for the sports slice. nba + ncaamb have grounded `rest_schedule` and `market` signals, and the evaluator distinguishes partial vs full basketball grounding richness. agent/tool doctrine documented. see `decision-intelligence-model.md` for full context.

---

## current truth

there is no orchestration layer today. `AgentRunService` coordinates four explicit steps for sports runs: retrieve, analyze, evaluate, compose. the RunType dispatch switch in `AgentRunService` is the only routing logic that exists.

**four explicit steps in `RunSportsMatchupPipelineAsync`:**
1. `ISportsRetriever.RetrieveAsync` — retrieves grounded evidence for the selected competition; records `SportsRetrievalOutput` on the artifact
2. `ISportsAnalyzer.AnalyzeAsync` — sends user input + retrieval evidence to FastAPI; records `SportsAnalysisResponse` on the artifact
3. `ISportsEvaluator.Evaluate` — scores evidence richness + analyzer confidence; records `EvaluatorOutput` on the artifact
4. `ISportsComposer.Compose` — assembles all prior outputs into `AgentRunExecutionResult`; sets `artifact.FinalResult`

**`SportsComposer.Compose` is the compose seam.** final `Confidence` comes from `EvaluatorOutput.AggregateConfidence`, not the analyzer. `Lean`, `Summary`, `Factors` come from the analyzer. `GroundedSignals` comes from `SportsRetrievalOutput`.

**`EvaluatorOutput` is the typed evaluate step contract:** defined in `DevCore.Api/AgentRuns/EvaluatorOutput.cs`. contains `AggregateConfidence` (calibrated), `AnalyzerConfidence` (stored for learning loop comparison), and `ConfidenceBand` (`"high"` / `"medium"` / `"low"`).

**confidence ownership is now established:** `SportsAnalysisResponse.Confidence` is the analyzer's local estimate — used as one input to `Evaluate`, not stored as the final value. `AgentRunExecutionResult.Confidence` is now `EvaluatorOutput.AggregateConfidence`. both are stored in `OutputJson` for the learning loop.

**current calibration model (honest proxy):**
- zero grounded signals (any competition when its retriever fails): dampen analyzer confidence by 0.75; clamp to [0.30, 0.60]. the model reasoned from priors — high claimed confidence is narrative quality, not signal quality.
- partial grounding richness (`0 < groundedCount < CompetitionMaxGroundedSignals(competition)`): dampen analyzer confidence by 0.90; clamp to [0.35, 0.75]. this is a conservative interim path for runs that have some real data but not the full grounded set the current platform can support. today this mainly means basketball `1 of 2`.
- full grounding richness (`groundedCount == CompetitionMaxGroundedSignals(competition)`): clamp analyzer confidence to [0.35, 0.85]. the model had the full currently supported grounded set for that competition, but the ceiling still stays conservative because this is not outcome-validated scoring yet.
- `CompetitionMaxGroundedSignals(competition)` defines the current maximum grounded signals possible per supported competition. update the competition catalog when a new grounded source is added.

**`SportsRetrievalOutput` is the typed retrieve step contract:** defined in `DevCore.Api/AgentRuns/SportsRetrievalOutput.cs`. contains `FootballMarketContext?`, `MlbStarterContext?`, `BasketballScheduleContext?`, `BasketballMarketContext?`, and `GroundedSignals` (computed from non-null fields). the single source of truth for which signals had real retrieved data.

**evidence quality is stored in `OutputJson`:** `AgentRunExecutionResult.GroundedSignals` and `AgentRunExecutionResult.AnalyzerConfidence` are both stored for the future learning loop.

---

## target model

the platform should orchestrate reusable agents and tools without embedding niche-specific rules into the core execution path.

the orchestrator's job is coordination, not analysis. it does not know how to score a spread or call an odds API. it knows which agents to invoke, in what order, and with what inputs for a given run type.

**what the orchestrator coordinates:**
- input collection (collector agents, potentially parallel)
- signal evaluation (evaluator agent, per-niche scoring rules)
- synthesis (synthesizer agent, produces the decision artifact)
- compliance checks (if required before delivery)
- delivery formatting and dispatch

**what the orchestrator does not own:**
- sport-specific rules or data source logic — those belong in the collector agents and niche config
- prompt construction — that belongs in the agent profile
- output formatting — that belongs in the synthesizer and delivery layer

---

## agent roles

five reusable roles. each role stays stable across niches. niche-specific behavior is injected through agent profiles and niche config, not through role code.

| role | responsibility |
|---|---|
| collector | retrieve and normalize raw signals from external sources; flag stale or missing data explicitly |
| evaluator | score each signal category against niche-specific thresholds; produce a structured scored signal set |
| synthesizer | receive scored signals; produce the decision artifact (lean, confidence, signals, counter-signals, narrative) |
| compliance | check output against content rules before delivery; block if thresholds are violated |
| delivery | format and dispatch the artifact (email, webhook, UI response) |

---

## agent profiles

profiles are the niche-specific configuration injected into each agent at runtime. they separate the agent's behavioral parameters from the execution code.

a profile binds:
- which model and temperature to use
- the system prompt template
- the expected output schema
- signal weights (for the evaluator)
- any niche-specific rules or null handling

profiles are versioned. a profile key is stored on the `AgentRun` record so historical artifacts can be traced to the exact configuration that produced them.

**current truth:** `AgentProfileKey` is stored null. no profile is loaded at runtime. this is the correct deferral — profile infrastructure is not needed until there is more than one run type with distinct behavioral requirements.

---

## handoff shapes (target)

each step produces a typed output consumed by the next. the orchestrator does not inspect or transform these — it passes them through.

```
RetrievalOutput {
  competition, matchup, date,
  signals: { line_movement?, sharp_money?, injury?, weather?, situational? }
  retrieved_at, source_statuses: [{ source, status, stale }]
}

EvaluatorOutput {
  scored_signals: [{ category, direction, score, flag }],
  aggregate_confidence: high | medium | low,
  signal_gaps: [string]
}

SynthesizerOutput  =  DecisionArtifact  (see decision-intelligence-model.md §4)
```

---

## agent/tool design doctrine

**agents differ by role, not by exclusive tool ownership.** the same external API (e.g., The Odds API) may be called by a collector agent and by a signal-scorer agent using the same underlying client. what differs is the agent's objective, interpretation, and output contract — not which tools it can call.

**tools are shared platform infrastructure.** `OddsMarketClient`, `OddsScheduleClient`, `MlbStarterClient`, future `InjuryClient`, and `WeatherClient` are tools. they belong to the platform, not to a specific agent. an agent profile declares which tools it uses; the tools themselves stay platform-owned.

**profiles inject behavior.** the same collector role behaves differently for NFL vs NBA because the profile specifies different tools, different signal categories, and different null-handling rules. the role code stays stable; the profile drives variation.

**do not hardcode niche rules into shared role code.** a collector should not contain an if-statement that says "if football, also fetch weather." that rule belongs in the niche config or agent profile. the collector calls whatever tools the profile authorizes.

**current exception:** `SportsRetriever.RetrieveAsync` contains competition-specific branches for `nfl`, `ncaaf`, `mlb`, `nba`, and `ncaamb` data retrieval. this is the correct deferral — there is no profile system yet to inject competition-specific tool lists. when additional competition-specific sources are added, this logic should move into niche config. do not generalize prematurely.

---

## confidence ownership

**analyzer confidence is provisional.** the model emits confidence based on the inputs in the prompt. it reflects narrative consistency of the model's reasoning — not signal quality, not calibration against outcomes, not cross-source weighting. a model with no real data can claim 0.80 confidence because its story is internally consistent. that reading is not equivalent to 0.80 real predictive confidence.

**final confidence belongs to the evaluator.** the evaluate step exists:
- `ISportsEvaluator.Evaluate` in `AgentRunService` scores evidence richness against analyzer confidence
- it produces `EvaluatorOutput.AggregateConfidence` — the calibrated final value
- `ComposeDecisionArtifact` uses `AggregateConfidence`, not `analyzerOutput.Confidence`
- the analyzer's provisional value is stored separately in `AnalyzerConfidence` for learning loop comparison

**current calibration is an honest proxy, not a final model.** the dampening factor (0.75) and clamp ranges are documented choices that will be tuned once outcome data exists. the structure is correct — the seam is established — but the numbers are conservative estimates, not statistically validated.

**future confidence calibration:** when outcome tracking accumulates enough data to measure which confidence bands were accurate, the calibration parameters can be updated. when a real signal-scoring evaluator agent exists (with per-category weights), it produces a new `EvaluatorOutput` — nothing in the current pipeline needs to change except the implementation of `Evaluate`.

---

## transition path

the current single-step analysis does not need to be replaced immediately. the path is:

1. RunType dispatch — done. `AgentRunService` routes by run type.
2. retrieve step started — `OddsMarketClient` and `MlbStarterClient` fetch grounded data before the analyzer call.
3. compose seam named — `SportsComposer.Compose` is the explicit seam.
4. retrieve step typed — done. `SportsRetrievalOutput` is the explicit contract. `GroundedSignals` ownership is with the retriever.
5. evaluate step added — done. `EvaluatorOutput` is the typed contract. `Evaluate` produces calibrated confidence from evidence richness + analyzer input. `ComposeDecisionArtifact` now receives all three pipeline outputs.
6. competition-first routing slice added — done. internal analysis now keys off explicit competition codes (`nfl`, `ncaaf`, `nba`, `ncaamb`, `mlb`) while the UI stays at sport family + level.
7. basketball rest/schedule grounding added — done. `nba` and `ncaamb` now send explicit `BasketballScheduleContext` into the analyzer when the retriever grounds both teams' recent schedule context.
8. basketball market grounding added — done. `nba` and `ncaamb` now also send explicit `BasketballMarketContext` into the analyzer when current spreads are available.
9. evaluator richness distinction added — done. `Evaluate` now uses current grounded count plus `CompetitionMaxGroundedSignals(competition)` to distinguish zero grounded, partially grounded, and fully grounded runs. today that mainly changes basketball `0 of 2`, `1 of 2`, and `2 of 2`.
10. next: when calibration parameters are outcome-validated, `Evaluate` can grow into a real scoring pass with per-category weights
11. then: introduce the orchestrator as coordinator when two meaningfully different pipeline configurations exist

do not introduce the orchestrator coordinator until there are at least two meaningfully different pipeline configurations that justify the abstraction.
