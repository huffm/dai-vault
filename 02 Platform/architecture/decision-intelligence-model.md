# decision intelligence model

**date:** 2026-04-18
**status:** strategic direction document — separates current truth from near-term and longer-term targets explicitly

---

## doctrine

this platform is not building a report generator. it is building a decision intelligence system.

the distinction matters:
- a report generator collects data and formats it. it produces output on demand.
- a decision intelligence system understands what decision a user is trying to make, gathers evidence relevant to that decision, synthesizes a structured view with explicit signal quality and counter-arguments, and learns which signals actually predicted outcomes.

the sports matchup analyzer is the first concrete expression of this doctrine. every architectural choice should be evaluated against one question: does it help a user make a better-informed decision, faster?

---

## 1. decision-first product model

**current truth:** the current product asks "who's playing?" and produces a summary + factor list. the decision — bet or don't bet, which side to lean toward — is implicit. the user must infer it from summary text.

**near-term direction:** introduce a structured lean field in the decision artifact. lean is not a pick. it is a directional signal with explicit confidence and counter-signals. the user is still making the decision; the platform is surfacing what the evidence suggests.

**longer-term target:** the platform knows what type of decision each user is evaluating (against the spread, moneyline, totals, props). analysis is scoped to the relevant decision surface, not general game commentary. the signal table maps directly to decision-relevant categories, not generic team facts.

---

## 2. evidence-backed architecture

**current truth:** a single LLM call (gpt-4o-mini, temperature=0.3) produces the entire analysis. the model uses its training weights as the evidence base. no external data is retrieved at runtime. no source citations exist.

**near-term direction:** the FastAPI sports analyzer already receives team names, sport, and date. the next concrete step is attaching real retrieved signals as explicit prompt inputs: current spread + line movement, sharp money %, public betting %, injury report status, and weather for outdoor NFL. these become grounded inputs, not things the model guesses from memory.

**longer-term target:** a collector layer retrieves and normalizes signals from external sources (odds api, actionnetwork, rotowire, open-meteo). signals arrive as typed, nullable fields — stale or missing data is explicit, not silently absent. the evaluator scores each signal category against sport-specific thresholds. the synthesizer receives scored signals, not raw data, and produces a structured decision artifact from them.

**signal categories (sports niche):**
- line movement (source: odds api)
- sharp vs public money split (source: actionnetwork)
- injury / availability (source: rotowire)
- weather conditions (source: open-meteo, NFL outdoor only; null for NBA)
- situational context (back-to-back, divisional game, rest differential — derivable from schedule data)

---

## 3. orchestrated synthesis

**current truth:** `AgentRunService` calls FastAPI directly. there is one step: analyze. there is no orchestration layer.

**near-term direction:** the service boundary is already explicit (RunType dispatch). the next structural move is separating analysis into discrete steps with typed handoffs between them: collect → evaluate → synthesize. each step can fail, log, and be retried independently without re-running the full pipeline.

**longer-term target:** an orchestrator coordinates reusable agents without embedding sport-specific rules into the core execution path. the orchestrator knows which agents to invoke for a given run type, in what order, and with what inputs. sport-specific rules live in agent profiles and niche configs, not in orchestrator code.

**orchestrator responsibilities:**
- load the niche config and agent profile for the run type
- dispatch collector agents (parallel where possible)
- pass collector output to the evaluator
- pass evaluator output to the synthesizer
- assemble the decision artifact
- invoke delivery if configured

**orchestrator non-responsibilities:**
- it does not know NFL scoring rules
- it does not know which external APIs exist or how to call them
- it does not format the brief
those concerns belong in the agents and niche config.

---

## 4. decision artifact design

**current truth:** the API returns `{ summary, confidence, factors[] }`. there is no lean field, no signal table, no counter-signals, no source citations, no provenance.

**near-term direction:** add `lean` to the backend response. lean is a compact directional signal — which side the evidence favors and how strongly — distinct from the narrative summary. this is the smallest meaningful step toward a structured decision artifact.

**longer-term target:** the full decision artifact is the platform's primary output unit. it is what the UI renders, what delivery formats, and what the learning loop stores as the baseline for outcome comparison.

conceptual shape:
```
{
  lean: { direction, strength, basis },
  confidence: { level, rationale },
  signals: [{ category, direction, flag }],
  counter_signals: [string],
  summary: string,
  sources: [{ label, retrieved_at }],
  generated_at: datetime,
  sport: string,
  matchup: { team_a, team_b, date },
  agent_run_id: uuid
}
```

`lean.direction` is not a pick. it is the directional signal the evidence supports: which side has the weight of available signals behind it. the user decides what to do with it.

`counter_signals` is not boilerplate hedging. it documents the strongest argument against the lean, so users can weigh it themselves.

`sources` documents provenance. users and operators can verify what data was used.

---

## 5. learning loop

**current truth:** runs are stored in the `AgentRun` table with `InputJson` and `OutputJson`. no outcome data exists. no signal accuracy is tracked.

**near-term direction:** no action required now. the storage foundation is already in place. outcome tracking is correctly deferred until the decision artifact is worth comparing outcomes against.

**longer-term target:** after a game concludes, an outcome collector retrieves the final score and spread result. it compares the outcome against the stored decision artifact for that game. over time, signal accuracy and lean-direction accuracy accumulate per sport, signal category, and confidence band. this is the feedback loop that informs signal weight calibration.

conceptual outcome record:
```
{
  agent_run_id: uuid,
  game_result: { winner, spread_covered, total_result },
  lean_correct: bool,
  signal_accuracies: [{ category, was_favorable, outcome_aligned }]
}
```

the learning loop does not change model behavior in real-time. it informs signal weight tuning in a future offline calibration pass.

---

## 6. storage by purpose

**current truth:** one SQL database holds everything. `AgentRun` stores input and output as raw JSON blobs. no vector store, no analytics layer.

**near-term direction:** no new storage layers required. do not build vector or analytics infrastructure before structured signals exist to store in them.

**longer-term target:** four storage roles, each holding what it is best suited for:

| layer | technology | holds |
|---|---|---|
| relational | SQL (current) | run metadata, agent run records, tenant config, subscription state, sport/team reference data |
| object | blob store (Azure Blob or S3) | full decision artifacts by run id, full collector output snapshots, delivery receipts |
| vector | vector DB (not yet) | embedded signal summaries for retrieval-augmented synthesis — defer until a specific retrieval use case justifies it |
| analytics | OLAP (not yet) | signal accuracy over time, lean direction win rates, confidence calibration — defer until outcome data exists |

the vector and analytics layers are intentionally deferred. building them before structured signal data exists produces empty infrastructure.

---

## 7. profile-driven behavior

**current truth:** `AgentProfileKey` is stored null. no profile is loaded at runtime. prompts are hardcoded in `sports_analyzer.py`.

**near-term direction:** no profile system required in this slice. sport-specific prompt variations are managed in FastAPI directly. that is the right call for now.

**longer-term target:** agent profiles separate behavioral configuration from execution code. a profile defines the model, temperature, system prompt template, output schema, and signal weighting rules for a specific agent × niche combination.

conceptual profile shape:
```
{
  profile_key: "sports.nfl.evaluator.v1",
  model: "gpt-4o-mini",
  temperature: 0.3,
  system_prompt_template: "...",
  output_schema: "SportsDecisionArtifact",
  signal_weights: {
    line_movement: 0.30,
    sharp_money: 0.25,
    injury: 0.25,
    weather: 0.10,
    situational: 0.10
  }
}
```

profiles are versioned. changing a model or signal weight creates a new profile version, so historical artifacts can be linked to the exact profile that generated them.

---

## sequencing

do not build toward this model from the outside in. build from the current working layer outward:

1. **now (current slice):** run metadata visibility — surface `durationMs`, expose `agentRunId`, wire `GET /api/agent-runs/{id}`
2. **next:** structured lean in the backend response — smallest step toward a real decision artifact
3. **then:** attach real signal inputs to the FastAPI call — replace model guessing with retrieved data
4. **then:** split collect / evaluate / synthesize into distinct typed steps
5. **later:** orchestrator coordination, agent profiles, outcome tracking, vector retrieval

each step must prove its layer before the next begins. the brief must be worth delivering before the learning loop has anything to learn from.
