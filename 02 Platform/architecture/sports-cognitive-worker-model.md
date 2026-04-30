# sports cognitive worker model

**date:** 2026-04-30  
**status:** implemented as cognitive artifact v1 for `sports.matchup.analysis`

---

## design rule

Myth inspires the worker.  
Logic defines the job.  
The artifact proves the value.

The symbolic origin is design lineage only. The implementation uses operational names, typed contracts, evidence ownership, and the existing sports pipeline.

---

## model

Cognitive Artifact v1 represents a 12-action decision worker through 4 macro phases. The 13th layer is the delivered consumable artifact.

| macro phase | action | software job |
|---|---|---|
| perceive | detect | identify the available signals, anomalies, and missing context |
| perceive | frame | state the factual matchup context |
| perceive | aim | name the factors that matter most |
| interrogate | balance | state the strongest counter-case |
| interrogate | stress | name the risk, fragile assumption, or uncertainty |
| interrogate | reframe | offer an alternate explanation |
| discern | test | challenge the emerging read |
| discern | listen | interpret market or external signal when present |
| discern | filter | separate grounded evidence from weak or absent evidence |
| decide | calibrate | explain why confidence matches the evidence |
| decide | posture | set the read stance |
| decide | voice | phrase the read without hype or false certainty |
| deliver | final artifact | return the consumable `AgentRunResultDto` and persist the full `OutputJson` |

This is intentionally 4 phases with 3 actions each because the product needs a compact, inspectable reasoning artifact. It gives the system enough structure to challenge a read without creating a 12-step UI, 12 agents, or 12 model calls.

---

## what this is not

- not 12 separate agents
- not 12 model calls
- not a workflow engine
- not 12 UI sections
- not a gambling pick service
- not a lock or certainty claim

The current retrieve -> analyze -> evaluate -> compose spine remains intact.

---

## pipeline mapping

| pipeline stage | cognitive role | current implementation |
|---|---|---|
| retrieve | perceive | collects grounded context, signal categories, and evidence availability |
| analyze | perceive, interrogate, discern, decide | one FastAPI model call emits the compact 4-phase cognitive artifact plus top-level delivery extracts |
| evaluate | discern, decide | deterministic .NET calibration uses grounded signal count and owns final confidence/evidence richness |
| compose | deliver | builds `AgentRunExecutionResult`, stores the full artifact in `OutputJson`, and maps compact fields to `AgentRunResultDto` |

The model can describe its reasoning, but it does not own evidence richness. Evidence richness is the count of grounded signal categories from the retriever/evaluator path.

---

## implemented fields

The nested cognitive artifact is stored internally in `OutputJson` as `CognitivePhases`.

```text
CognitivePhases {
  Perceive { Detect[], Frame?, Aim[] }
  Interrogate { Balance?, Stress?, Reframe? }
  Discern { Test?, Listen?, Filter? }
  Decide { Calibrate?, Posture?, Voice? }
}
```

The API response and Angular UI receive only the compact delivery fields:

| field | source | ui label |
|---|---|---|
| `posture` / `posture` | validated analyzer deliver extract, fallback from `phases.decide.posture` | Read Stance |
| `counter_case` / `counterCase` | analyzer deliver extract, fallback from `phases.interrogate.balance` | Counter Case |
| `watch_for` / `watchFor` | analyzer deliver extract, fallback from `phases.interrogate.stress` | Watch For |
| `what_would_change_the_read` / `whatWouldChangeTheRead` | analyzer deliver extract | What Would Change the Read |
| `evidence_richness` / `evidenceRichness` | .NET `retrieval.GroundedSignals.Length` | diagnostic only |

`evidenceRichness` is nullable in the final .NET DTO. `null` means an older record or response predates the field. `0` means a current run completed with no grounded signals.

---

## posture vocabulary

`posture` must be one of:

- `play`
- `pass`
- `monitor`
- `wait`
- `compare`
- `avoid`

Invalid posture values are clamped to null before they leave FastAPI. The UI labels this as `Read Stance`. It must not be labeled `Pick`, and it must not use lock language or gambling hype.

---

## compatibility rules

- new fields are additive
- missing phases do not fail parsing
- malformed phases become null
- missing deliver extracts remain null or empty
- old/minimal records remain compatible
- `CognitivePhases` is internal to `OutputJson` for now
- Angular renders each compact section only when populated

---

## deferred

- exposing the nested 4-phase artifact directly in Angular
- phase-level scoring or analytics
- phase-level database columns
- a separate evaluator for each action
- a multi-agent deliberation workflow
- richer evidence sources for injury, weather, travel, or line movement
- confidence calibration against historical outcomes
