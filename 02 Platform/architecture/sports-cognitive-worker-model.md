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
| synthesize | integrate | combine validated material from prior phases without adding new claims |
| synthesize | compose | assemble the final decision artifact shape |
| synthesize | deliver | return the consumable `AgentRunResultDto` and persist the full `OutputJson` |

The first 4 phases (12 actions) are expressed through a single model call. The 13th function — Synthesize — is the platform-owned layer that integrates, compresses, and presents the validated artifact. It maps to `SportsComposer` in code. In vault language: Manifest. It does not invent; it plates what survived the prior phases.

---

## what this is not

- not 12 separate agents
- not 12 model calls
- not a workflow engine
- not 12 UI sections
- not a gambling pick service
- not a lock or certainty claim

The current retrieve -> analyze -> evaluate -> quality_check -> compose spine remains intact.

---

## pipeline mapping

| pipeline stage | cognitive role | current implementation |
|---|---|---|
| retrieve | perceive | collects grounded context, signal categories, and evidence availability |
| analyze | perceive, interrogate, discern, decide | one FastAPI model call emits the compact 4-phase cognitive artifact plus top-level delivery extracts |
| evaluate | discern, decide | deterministic .NET calibration uses grounded signal count and owns final confidence/evidence richness |
| quality_check | discern | deterministic .NET artifact quality warnings for internal feedback loops |
| compose | synthesize | builds `AgentRunExecutionResult`, integrates validated phase material, stores the full artifact in `OutputJson`, and maps compact delivery fields to `AgentRunResultDto` |

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

Artifact Quality v1 adds platform-owned `MissingSignals` and `ArtifactQualityWarnings` in `OutputJson`.
These are deterministic internal quality-loop fields.
They are not user-facing.
They do not implement the full future `ArtifactSources` taxonomy.
The full `known_facts`, `ai_interpretations`, `missing_information`, and `excluded_inputs` taxonomy remains deferred.

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

## synthesize rule

Synthesize integrates, compresses, augments, and presents validated artifact material.
It does not invent. It does not add unsupported claims. It does not override evidence quality.
It produces the consumable decision artifact from what survived the prior phases.

In this implementation, synthesize is owned by `SportsComposer` — deterministic code that reads all prior stage outputs from the `SportsRunArtifact` and maps them into the final `AgentRunExecutionResult`. Evidence richness is derived from the retriever's grounded signal count, not from model output.

---

## guardrails

Each phase has hard boundaries. These apply to the model role — prompt design must enforce them.

| phase | must not |
|---|---|
| perceive | decide posture, invent missing data, overstate signal strength |
| interrogate | add new facts, attack for noise, decide final stance |
| discern | treat interpretation as fact, admit weak evidence as strong, ignore missing information |
| decide | let confidence exceed evidence, use hype language, call uncertain reads certain |
| synthesize | invent evidence, introduce unsupported claims, hide uncertainty, override evidence quality |

The platform enforces additional guardrails in code:
- posture is validated against the allowed vocabulary and clamped to null on invalid values
- lean_side is validated to only "home", "away", or null
- signals_used is validated against the known signal category vocabulary
- `MissingSignals` is computed by the platform from expected competition signals minus grounded signals
- `ArtifactQualityWarnings` is produced by exactly five deterministic quality rules and remains internal
- missing phase fields fail safely to null — they do not fail the run

---

## deferred

- exposing the nested 4-phase artifact directly in Angular
- phase-level scoring or analytics
- phase-level database columns
- a separate evaluator for each action
- a multi-agent deliberation workflow
- richer evidence sources for injury, weather, travel, or line movement
- confidence calibration against historical outcomes
