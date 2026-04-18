# orchestration

**date:** 2026-04-18
**status:** near-empty today. this document describes the target model. see `decision-intelligence-model.md` for full context on each layer.

---

## current truth

there is no orchestration layer today. `AgentRunService` calls FastAPI directly via `FastApiClient.AnalyzeSportsMatchupAsync`. the call is one step: analyze. the RunType dispatch switch in `AgentRunService` is the only routing logic that exists.

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

## transition path

the current single-step analysis does not need to be replaced immediately. the path is:

1. RunType dispatch is done — `AgentRunService` routes by run type
2. next: split the FastAPI call into typed collector input + analyzer output (begin structuring the handoff)
3. then: promote collect / evaluate / synthesize to discrete service steps
4. then: introduce the orchestrator as the coordinator of those steps

do not introduce the orchestrator layer until there are at least two meaningfully different pipeline configurations that justify the abstraction.
