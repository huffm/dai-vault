# protocol vocabulary map

**date:** 2026-05-14
**status:** canonical runtime active. legacy vocabulary retired from runtime.

## purpose

This document is the single source of truth for the canonical Cognitive Protocol Runtime vocabulary and the historical mapping from retired runtime names.

The active runtime now accepts and persists only the canonical `protocol` / `CognitiveProtocol` shape:

- prompt block key: `protocol`
- persisted artifact field: `CognitiveProtocol`
- analyzer protocol fields: `question`, `verify`, `probe`, `contrast`, `weigh`, `stress`, `justify`, `position`, `resolve`, plus scalar `detect`, `frame`, `aim`
- read-side projection: `ProtocolView` from canonical `CognitiveProtocol` only

Legacy vocabulary remains documented here only for historical interpretation:

- pre-canonical persisted dev records are not rewritten
- calibration reports written under the legacy vocabulary remain as written
- active readers no longer translate legacy-only records into `ProtocolView`

## canonical vocabulary

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

## retired legacy to canonical mapping

The names below are retired from the active runtime. Use this table only when reading pre-canonical artifacts, old calibration reports, or historical notes.

### Perceive

| legacy field | canonical micro-action | notes |
|---|---|---|
| `perceive.detect` | Perceive.Detect | name already aligned |
| `perceive.frame` | Perceive.Frame | name already aligned |
| `perceive.aim` | Perceive.Aim | name already aligned |

### Interrogate

| legacy field | canonical micro-action | notes |
|---|---|---|
| `interrogate.balance` | Interrogate.Question | retired runtime name for the strongest counter-case against the lean |
| `interrogate.stress` | Discern.Stress | retired runtime location; active runtime emits Stress under Discern only |
| `interrogate.reframe` | Interrogate.Verify | retired runtime name for an alternate explanation tested against staged evidence |
| `SignalFollowUpRecord[]` (post-2026-05-14, Probe Population v1) | Interrogate.Probe | populated deterministically by `CognitiveProtocolBuilder.BuildProbe` from existing follow-up data. each missing primary signal identified via `Reason = "primary_signal_missing"`, `DecisionUse = "missing_confirmation"`, or `FallbackType = "lateral_proxy"` is mapped through a doctrinal template (one templated sentence per signal). signals without a template are dropped to avoid fabricating injury, form, or travel claims. line_movement is excluded because it is permanently not_implemented. Probe stays null when no template matches. no model call. |

### Discern

| legacy field | canonical micro-action | notes |
|---|---|---|
| `discern.test` | Discern.Stress | retired runtime name for the strongest challenge to the read |
| `discern.listen` | Discern.Contrast | retired runtime name for market or external signal interpretation |
| `discern.filter` | Discern.Weigh | retired runtime name for separating grounded from weak signal |

`SignalQualityEvaluator.Quality`, `DecisionUse`, `FollowUpSignals`, and `ConfidenceEffect` are the deterministic surface of Discern.Weigh. They remain unchanged.

### Decide

| legacy field | canonical micro-action | notes |
|---|---|---|
| `decide.calibrate` | Decide.Justify | retired runtime name for the calibration rationale sentence |
| `decide.posture` | Decide.Position | retired runtime field name; the posture enum and `Read Stance` UI label do not change |
| `decide.voice` | Decide.Resolve | retired runtime name for framing the read without hype |

The earlier proposal of `Decide.Choose` is rejected. Position is the canonical word. Future code that introduces a `Choose` field is a regression against decision 0004.

### Synthesize

| legacy concept | canonical operation | notes |
|---|---|---|
| `SportsComposer.Compose` | Synthesize.Compose | unchanged in code |
| integration of validated protocol material | Synthesize.Integrate | unchanged in code |
| `AgentRunResultDto` mapping and persistence | Synthesize.Deliver | unchanged in code |

## name collisions resolved by this map

### Retrieve

Retrieve is not a cognitive micro-action and never will be. It stays as a platform and pipeline concept owned by:

- `SportsRetriever`, `SportsRetrievalOutput`, `RetrieveAsync`
- `SportsRunArtifact.RecordRetrieve` and the `retrieve` pipeline step label
- future vector search, tool calls, API clients, and source fetching

The Interrogate micro-action that requests further investigation is Probe. Probe may invoke platform retrieval; it is not itself retrieval.

### Stress

`interrogate.stress` is legacy and retired from the active runtime. The canonical home for Stress is Discern.

- in current code and persisted v3 runs: Stress lives inside `protocol.discern.stress` / `CognitiveProtocol.Discern.Stress`
- in pre-canonical records and old calibration reports: Stress may appear as `interrogate.stress` or `discern.test`

### Choose

`Choose` was floated as the Decide micro-action and is rejected. The canonical word is Position. The sports product language depends on Position not reading as a pick.

## canonical enforcement rule

Do not reintroduce alias scaffolding for the retired names above. Changes to the canonical protocol shape must update FastAPI, .NET, Angular, dev tooling, and vault docs in lockstep. The relevant active surfaces are:

- FastAPI Pydantic models in `dai/services/agent-service/app/models/sports.py`
  (`SportsCognitiveProtocol`, `SportsPerceiveProtocol`, `SportsInterrogateProtocol`, `SportsDiscernProtocol`, `SportsDecideProtocol`)
- the analyze prompt body in `dai/services/agent-service/app/services/sports_analyzer.py`
- the .NET cognitive records and the `SportsAnalysisResponse.Protocol` shape consumed by the .NET caller
- `SportsQualityChecker` rules that match against canonical field names
- the artifact inspection endpoint response and the `/dev/artifacts` Angular page
- dev calibration tooling in `dai/scripts/dev/sports/run-artifact-calibration.ps1`

Persisted `OutputJson` records and calibration markdown written under the legacy vocabulary are not rewritten. Current runtime readers do not project legacy-only records; this map is for human interpretation of historical material.

## scope guard for this map

This document is the canonical name registry. It does not:

- define how each protocol implements its work
- enumerate forbidden phrases or guardrails per phase
- describe the calibration loop

Those concerns live in the protocol docs, `cognitive-protocol-runtime.md`, and `cognitive-worker-doctrine.md`.
