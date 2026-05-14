# cognitive protocol runtime

**date:** 2026-05-14
**status:** doctrine slice 1. canonical vocabulary defined. runtime code unchanged.

## core claim

DAI is a protocol-orchestrated decision engine, not an autonomous agent framework.

A user request creates a shared decision artifact. Deterministic platform code moves that artifact through a fixed set of cognitive protocols. Each protocol has a clear responsibility, a list of allowed actions, a list of forbidden behaviors, and a typed input and output contract. LLM calls happen inside specific protocols, on bounded inputs, against a known output schema.

The protocol set is stable. The artifact is the unit that travels. Niches plug in by overriding retrieval, prompt content, posture vocabulary, and scoring thresholds, not by inventing new protocols.

## what a protocol owns

Each protocol owns:

- a single responsibility named in one sentence
- a set of allowed tools (platform clients, retrievers, evaluators, model calls)
- a typed input contract read from the shared decision artifact
- a typed output contract written back to the shared decision artifact
- a list of forbidden behaviors enforced by code or schema
- quality rules that flag drift after the protocol runs
- a fallback path when inputs are missing or degraded
- a token budget when the protocol calls a model
- calibration hooks for measuring whether the protocol earned its place

## the four macro protocols and their micro-actions

The runtime moves the decision artifact through four cognitive protocols, then a final Synthesize layer. Each protocol has exactly three micro-actions. The 12 micro-actions are the canonical cognitive surface.

### 1. Perceive

Surface what is, name what is missing, and aim attention at what matters most.

| micro-action | responsibility |
|---|---|
| Detect | identify available signals, anomalies, and missing context |
| Frame | state the factual context for the decision |
| Aim | name the factors that matter most for this decision |

Perceive may trigger platform retrieval. Retrieval itself is deterministic platform work, not a cognitive micro-action.

### 2. Interrogate

Apply pressure to the initial read. Ask what is uncertain, probe gaps, and verify claims against staged evidence.

| micro-action | responsibility |
|---|---|
| Question | name the strongest open question or counter-case against the emerging read |
| Probe | targeted investigation of signal gaps, evidence needs, source follow-ups, or tool-backed questions |
| Verify | confirm or reject claims by checking them against staged evidence and platform guardrails |

Probe may request platform retrieval through allowed tools. It is not the retrieve stage.

### 3. Discern

Judge evidence quality after collection and interrogation. Separate grounded from weak, contrast competing signals, and stress test the read.

| micro-action | responsibility |
|---|---|
| Weigh | grade signals by quality and decision use |
| Contrast | interpret alignment or divergence between signals (market, sharp/public, situational) |
| Stress | name the fragility, key risk, or condition under which the read fails |

Stress lives here in the canonical model because it operates on weighed evidence, not on raw initial impressions.

### 4. Decide

Turn discerned evidence into a calibrated stance. Resolve the read, set a position, and justify the calibration.

| micro-action | responsibility |
|---|---|
| Resolve | reconcile the weighed evidence into a single direction or non-direction |
| Position | set the decision posture from the niche posture vocabulary |
| Justify | one-sentence calibration rationale for why the stated confidence fits the evidence |

Position means decision posture. It is not a pick, not a lock, and not a recommendation to act.

## the final layer: Synthesize

Synthesize integrates what survived the prior protocols into the final consumable artifact. It is not a fifth macro protocol and it does not contribute to the 12 cognitive micro-actions.

Synthesize may use the operations Integrate, Compose, and Deliver as internal synthesis steps:

- Integrate: combine validated material from prior protocols without adding new claims
- Compose: assemble the final decision artifact shape
- Deliver: return the consumable result and persist the full artifact

These are platform operations, not cognitive actions. They are deterministic in the current implementation and owned by composer code.

## the shared decision artifact

A run produces one artifact. The artifact is the working surface that every protocol reads from and writes to.

Conceptual shape:

```text
DecisionArtifact {
  run_id,
  input,
  perceive   { detect[], frame, aim[] },
  interrogate{ question, probe, verify },
  discern    { weigh, contrast, stress },
  decide     { resolve, position, justify },
  synthesize { integrate, compose, deliver }
}
```

The current runtime stores a subset of this shape inside `OutputJson.CognitivePhases` using legacy field names. The canonical shape is the target, not a description of what is implemented today. See `protocol-vocabulary-map.md` for the legacy-to-canonical mapping and `current-agent-run-contract.md` for the implemented contract.

## where the model is allowed

Model calls are reserved for cognitive work that genuinely benefits from interpretation:

- Question and Verify inside Interrogate
- Contrast and Stress inside Discern
- Resolve and Justify inside Decide

Probe inside Interrogate may request platform retrieval but the retrieval itself is deterministic. Position inside Decide is a clamped enum, not free-form model output. Synthesize is deterministic in the current implementation.

Detect, Frame, and Aim inside Perceive are produced by the same single model call as the rest in the current implementation. The protocol model allows future slices to move Detect and Frame closer to the retriever surface when deterministic extraction is cheaper than a model call.

## what this runtime is not

- not 12 separate agents
- not 12 model calls
- not a workflow engine
- not a multi-agent debate loop
- not a freeform autonomy layer
- not a pick or prediction service

The protocol runtime is a small, fixed set of bounded responsibilities applied to one shared artifact.

## current vs target

The protocol runtime is the target shape. The current implementation expresses the four protocols as prompt blocks inside one model call, with deterministic perceive retrieval before the call and deterministic synthesize after. See `cognitive-skill-pack-architecture-v1.md` for the worker pack target shape and `protocol-vocabulary-map.md` for the legacy vocabulary that persists in code and stored artifacts.

Future code slices land .NET and FastAPI changes in lockstep. Persisted `OutputJson` records and calibration reports written before a rename retain the legacy vocabulary.

## references

- `02 Platform/decisions/0004-cognitive-protocol-runtime.md`
- `02 Platform/architecture/cognitive-factory/protocol-vocabulary-map.md`
- `02 Platform/architecture/cognitive-worker-doctrine.md`
- `02 Platform/architecture/cognitive-factory/cognitive-skill-pack-architecture-v1.md`
- `02 Platform/architecture/cognitive-factory/phases/perceive.md`
- `02 Platform/architecture/cognitive-factory/phases/interrogate.md`
- `02 Platform/architecture/cognitive-factory/phases/discern.md`
- `02 Platform/architecture/cognitive-factory/phases/decide.md`
- `02 Platform/architecture/cognitive-factory/phases/synthesize.md`
- `02 Platform/architecture/current-agent-run-contract.md`
