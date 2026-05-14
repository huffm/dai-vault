# decision 0004: cognitive protocol runtime

**date:** 2026-05-14
**status:** accepted (doctrine slice 1; docs only)

## decision

DAI is modeled as a Cognitive Protocol Runtime: deterministic code moves a shared decision artifact through four macro cognitive protocols, each composed of three micro-actions, followed by a final Synthesize layer.

The four macro protocols and their canonical micro-actions are:

1. Perceive: Detect, Frame, Aim
2. Interrogate: Question, Probe, Verify
3. Discern: Weigh, Contrast, Stress
4. Decide: Resolve, Position, Justify

Final layer: Synthesize.

This replaces the implicit framing of DAI as a set of freeform autonomous agents. Cognition is bounded by protocols, not delegated to agent identity.

## naming decisions captured in this slice

### Retrieve is not a cognitive micro-action

Retrieve remains a deterministic platform and pipeline concept owned by the runtime, the retriever surfaces, tool calls, API clients, vector search, and source fetching. Keeping Retrieve outside the cognitive protocol set avoids collision with `SportsRetriever`, `RetrieveAsync`, `SportsRunArtifact.RecordRetrieve`, and the existing `retrieve` pipeline step label.

### Interrogate.Probe replaces the earlier proposed Interrogate.Retrieve

Probe means a targeted investigation of signal gaps, evidence needs, source follow-ups, or tool-backed questions raised by the model during interrogation. It is a cognitive action that may trigger platform-owned retrieval, but it is not the retrieval step itself.

### Discern.Stress is the canonical home for Stress

Stress testing happens after evidence has been collected and weighed. It belongs in Discern in the future canonical model. The current implementation emits `interrogate.stress` inside the analyze prompt; that is legacy vocabulary, not the canonical future location.

### Decide.Position replaces Decide.Choose

Position names a decision posture, not a gambling pick. This protects the sports product language and keeps the existing no-locks and no-picks rule intact. The implemented posture vocabulary (`play`, `pass`, `monitor`, `wait`, `compare`, `avoid`) and the `Read Stance` UI label remain unchanged.

### Synthesize is the final artifact layer

Synthesize is not counted as a fifth macro protocol. It may keep Integrate, Compose, and Deliver as internal synthesis operations, but those are not part of the 12 cognitive micro-actions.

## what this slice does not do

This slice is docs only. It does not:

- rename existing code symbols
- change FastAPI Pydantic models, .NET records, or the `CognitivePhases` JSON shape
- rename pipeline step labels (`retrieve`, `analyze`, `evaluate`, `quality_check`, `compose`)
- modify persisted `OutputJson` payloads
- update `current-agent-run-contract.md` (that doc tracks the implemented runtime shape and will be updated only after a future code slice)
- update existing calibration reports (they record legacy vocabulary as it stood at the time)

## why this matters

Future runtime work has to update .NET and FastAPI in lockstep because the cognitive artifact crosses both layers. Establishing the canonical vocabulary in doctrine first lets the future code slices land against a stable target instead of negotiating naming every time.

It also keeps the no-pick framing intact for the sports product: Position is the canonical word, and any future code introducing a `Choose` action will be flagged as a regression against this decision.

## references

- `02 Platform/architecture/cognitive-factory/cognitive-protocol-runtime.md`
- `02 Platform/architecture/cognitive-factory/protocol-vocabulary-map.md`
- `02 Platform/architecture/cognitive-worker-doctrine.md`
- `02 Platform/architecture/cognitive-factory/cognitive-skill-pack-architecture-v1.md`
- `02 Platform/architecture/cognitive-factory/phases/perceive.md`
- `02 Platform/architecture/cognitive-factory/phases/interrogate.md`
- `02 Platform/architecture/cognitive-factory/phases/discern.md`
- `02 Platform/architecture/cognitive-factory/phases/decide.md`
- `02 Platform/architecture/cognitive-factory/phases/synthesize.md`
