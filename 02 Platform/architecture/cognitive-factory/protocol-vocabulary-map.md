# protocol vocabulary map

**date:** 2026-05-14
**status:** doctrine slice 1. canonical vocabulary defined. runtime code unchanged.

## purpose

This document is the single source of truth for the mapping between current runtime vocabulary and the canonical Cognitive Protocol Runtime vocabulary.

It exists because the canonical names land in doctrine before any code rename. Until the future code slices land:

- current JSON field names, Pydantic models, C# records, prompt blocks, pipeline step labels, and persisted `OutputJson` payloads remain unchanged
- calibration reports written under the legacy vocabulary remain as written
- any reader can use this table to translate between what the code says and what the doctrine names

## canonical vocabulary (target)

The Cognitive Protocol Runtime defines four macro protocols, each with three micro-actions, plus a final Synthesize layer.

### 1. Perceive

| canonical micro-action | responsibility |
|---|---|
| Detect | identify available signals, anomalies, missing context |
| Frame | state the factual context for the decision |
| Aim | name the factors that matter most |

### 2. Interrogate

| canonical micro-action | responsibility |
|---|---|
| Question | name the strongest open question or counter-case |
| Probe | targeted investigation of signal gaps, evidence needs, source follow-ups, tool-backed questions |
| Verify | confirm or reject claims against staged evidence and guardrails |

### 3. Discern

| canonical micro-action | responsibility |
|---|---|
| Weigh | grade signals by quality and decision use |
| Contrast | interpret alignment or divergence between signals |
| Stress | name the fragility, key risk, or condition under which the read fails |

### 4. Decide

| canonical micro-action | responsibility |
|---|---|
| Resolve | reconcile weighed evidence into a single direction or non-direction |
| Position | set the decision posture from the niche posture vocabulary |
| Justify | one-sentence calibration rationale |

### Final layer: Synthesize

Synthesize is not a fifth macro protocol and is not counted among the 12 cognitive micro-actions. It may keep Integrate, Compose, and Deliver as internal synthesis operations.

## legacy to canonical mapping

Current implementations under `OutputJson.CognitivePhases`, FastAPI Pydantic models, the .NET `CognitivePhases` record, and the `analyze` prompt blocks use the legacy vocabulary below.

### Perceive

| legacy field | canonical micro-action | notes |
|---|---|---|
| `perceive.detect` | Perceive.Detect | name already aligned |
| `perceive.frame` | Perceive.Frame | name already aligned |
| `perceive.aim` | Perceive.Aim | name already aligned |

### Interrogate

| legacy field | canonical micro-action | notes |
|---|---|---|
| `interrogate.balance` | Interrogate.Question | the strongest counter-case against the lean is the canonical Question output |
| `interrogate.stress` | Discern.Stress | Stress moves to Discern in the canonical model; legacy emissions stay under interrogate until a future code slice |
| `interrogate.reframe` | Interrogate.Verify | an alternate explanation tested against staged evidence is the canonical Verify output |
| `SignalFollowUpRecord[]` (post-2026-05-14, Probe Population v1) | Interrogate.Probe | populated deterministically by `CognitiveProtocolBuilder.BuildProbe` from existing follow-up data. each missing primary signal identified via `Reason = "primary_signal_missing"`, `DecisionUse = "missing_confirmation"`, or `FallbackType = "lateral_proxy"` is mapped through a doctrinal template (one templated sentence per signal). signals without a template are dropped to avoid fabricating injury, form, or travel claims. line_movement is excluded because it is permanently not_implemented. Probe stays null when no template matches. no model call. |

### Discern

| legacy field | canonical micro-action | notes |
|---|---|---|
| `discern.test` | Discern.Stress | the current `test` field already names the strongest challenge to the read; in the canonical model that lives under Stress |
| `discern.listen` | Discern.Contrast | listening to market or external signals is the canonical Contrast output |
| `discern.filter` | Discern.Weigh | filter separates grounded from weak; in the canonical model the deterministic weighing is performed by `SignalQualityEvaluator` and the model contribution becomes Weigh |

`SignalQualityEvaluator.Quality`, `DecisionUse`, `FollowUpSignals`, and `ConfidenceEffect` are the deterministic surface of Discern.Weigh. They remain unchanged in this slice.

### Decide

| legacy field | canonical micro-action | notes |
|---|---|---|
| `decide.calibrate` | Decide.Justify | the calibration rationale sentence maps onto Justify |
| `decide.posture` | Decide.Position | Position is the canonical word; the posture enum and `Read Stance` UI label do not change |
| `decide.voice` | Decide.Resolve | the framing of the read without hype maps onto Resolve in the canonical model |

The earlier proposal of `Decide.Choose` is rejected. Position is the canonical word. Future code that introduces a `Choose` field is a regression against decision 0004.

### Synthesize

| legacy concept | canonical operation | notes |
|---|---|---|
| `SportsComposer.Compose` | Synthesize.Compose | unchanged in code |
| integration of validated phase material | Synthesize.Integrate | unchanged in code |
| `AgentRunResultDto` mapping and persistence | Synthesize.Deliver | unchanged in code |

## name collisions resolved by this map

### Retrieve

Retrieve is not a cognitive micro-action and never will be. It stays as a platform and pipeline concept owned by:

- `SportsRetriever`, `SportsRetrievalOutput`, `RetrieveAsync`
- `SportsRunArtifact.RecordRetrieve` and the `retrieve` pipeline step label
- future vector search, tool calls, API clients, and source fetching

The Interrogate micro-action that requests further investigation is Probe. Probe may invoke platform retrieval; it is not itself retrieval.

### Stress

`interrogate.stress` is legacy. The canonical home for Stress is Discern. Until a future code slice renames the field, both readings are correct:

- in code and persisted runs: Stress lives inside `interrogate`
- in doctrine: Stress lives inside Discern

### Choose

`Choose` was floated as the Decide micro-action and is rejected. The canonical word is Position. The sports product language depends on Position not reading as a pick.

## lockstep rule for future code slices

When code finally renames any of these fields, .NET and FastAPI must update in lockstep. The relevant surfaces are:

- FastAPI Pydantic models in `dai/services/agent-service/app/models/sports.py`
  (`SportsCognitivePhases`, `SportsPerceivePhase`, `SportsInterrogatePhase`, `SportsDiscernPhase`, `SportsDecidePhase`)
- the analyze prompt body in `dai/services/agent-service/app/services/sports_analyzer.py`
- the .NET cognitive records and the `SportsAnalysisResponse` shape consumed by the .NET caller
- `SportsQualityChecker` rules that match against legacy field names
- the artifact inspection endpoint response and the `/dev/artifacts` Angular page

Persisted `OutputJson` records and calibration markdown written under the legacy vocabulary are not rewritten. Readers translate via this map.

## scope guard for this map

This document is the canonical name registry. It does not:

- define how each protocol implements its work
- enumerate forbidden phrases or guardrails per phase
- describe the calibration loop

Those concerns live in the per-phase docs, `cognitive-protocol-runtime.md`, and `cognitive-worker-doctrine.md`.
