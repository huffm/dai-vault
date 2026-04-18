# orchestration

**date:** 2026-04-18
**status:** seam established. compose step named in code. agent/tool doctrine documented. see `decision-intelligence-model.md` for full context.

---

## current truth

there is no orchestration layer today. `AgentRunService` calls FastAPI directly via `FastApiClient.AnalyzeSportsMatchupAsync`. the call is one step: analyze. the RunType dispatch switch in `AgentRunService` is the only routing logic that exists.

**the orchestrator seam exists in code today:** `AgentRunService.ComposeDecisionArtifact` is a private static method in `DevCore.Api/AgentRuns/AgentRunService.cs`. it currently maps the analyzer output 1:1 to the decision artifact. when signal scoring exists, this method grows inputs and applies the orchestrator's composition logic — without changing the call signature from the controller's perspective.

```csharp
// today: trivial pass-through
private static AgentRunExecutionResult ComposeDecisionArtifact(SportsAnalysisResponse analyzerOutput)
    => new(Lean: ..., Summary: ..., Confidence: ..., Factors: ..., GroundedSignals: ...);

// future: receives scored signals, applies weights, produces final lean + confidence
private static AgentRunExecutionResult ComposeDecisionArtifact(
    SportsAnalysisResponse analyzerOutput,
    EvaluatorOutput evaluatorOutput,
    CollectorOutput collectorOutput)
    => ...;
```

**evidence quality is tracked in `OutputJson` today:** `AgentRunExecutionResult.GroundedSignals` carries which signal categories had real retrieved data (vs model priors). stored in `AgentRun.OutputJson`. not surfaced to the UI yet. the learning loop will use this to weight accuracy measurements per signal category.

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
CollectorOutput {
  sport, matchup, date,
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

**tools are shared platform infrastructure.** `OddsMarketClient`, future `InjuryClient`, `WeatherClient` are tools. they belong to the platform, not to a specific agent. an agent profile declares which tools it uses; the tools themselves stay sport-agnostic.

**profiles inject behavior.** the same collector role behaves differently for NFL vs NBA because the profile specifies different tools, different signal categories, and different null-handling rules. the role code stays stable; the profile drives variation.

**do not hardcode niche rules into shared role code.** a collector should not contain an if-statement that says "if NFL, also fetch weather." that rule belongs in the niche config or agent profile. the collector calls whatever tools the profile authorizes.

**current exception:** `AgentRunService.ExecuteSportsMatchupAsync` currently has a hard-coded `if (sport == "nfl")` check for market data. this is the correct deferral for now — there is only one sport with market enrichment. when a second sport uses market data, the if-statement moves into the niche config. do not generalize it prematurely.

---

## confidence ownership

**analyzer confidence is provisional.** the model emits confidence based on the inputs in the prompt. it is not calibrated against signal scoring rules, cross-source weighting, or completeness metrics. it reflects what the model believes, not what the evidence objectively supports.

**final confidence belongs to the orchestrator.** when the evaluate step exists:
- the evaluator scores each signal category against thresholds
- the scored signals produce an `aggregate_confidence` (high / medium / low or a calibrated float)
- `ComposeDecisionArtifact` uses this to compute the final confidence
- the analyzer's provisional value becomes one input, not the final answer

**current contract:** `SportsAnalysisResponse.Confidence` is labeled "analyzer-local confidence" in its comments. `AgentRunExecutionResult.Confidence` is labeled "final confidence (today: passes through from analyzer)." the comment on `ComposeDecisionArtifact` makes the seam explicit. no behavioral change.

---

## transition path

the current single-step analysis does not need to be replaced immediately. the path is:

1. RunType dispatch — done. `AgentRunService` routes by run type.
2. collect step started — `OddsMarketClient` fetches market data before the analyzer call. `GroundedSignals` tracks what was retrieved.
3. compose seam named — `ComposeDecisionArtifact` is the explicit seam in code. trivial today, grows when scoring exists.
4. next: promote the collect step to a typed `CollectorOutput` — structured, nullable fields, source statuses
5. then: add evaluate step — score signals against thresholds, produce `EvaluatorOutput`
6. then: `ComposeDecisionArtifact` consumes evaluator output and produces final lean + confidence
7. then: introduce the orchestrator as coordinator when two meaningfully different pipeline configurations exist

do not introduce the orchestrator coordinator until there are at least two meaningfully different pipeline configurations that justify the abstraction.
