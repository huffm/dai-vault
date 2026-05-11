# phase: synthesize

**date:** 2026-05-11
**status:** v1 doctrine — synthesize is fully platform-owned today (no model call).

---

## responsibility

Synthesize integrates what survived the prior phases into the final consumable artifact. It does not add new claims, override evidence quality, or hide uncertainty.

In a sentence: **synthesize plates what is already true — it does not cook new evidence.**

In vault language this is also called **Manifest**. The conceptual → code → user-facing mapping is:

| layer | name |
|---|---|
| vault / conceptual | Manifest |
| software / doctrine | Synthesize |
| pipeline stage code | Compose |
| user-facing output | Deliver |

---

## current code mapping

| current step | role in synthesize |
|---|---|
| `compose` (`SportsComposer.Compose`) | reads `SportsRetrievalOutput`, `SportsAnalysisResponse`, `EvaluatorOutput` from the artifact. assembles `AgentRunExecutionResult`. maps compact delivery fields onto `AgentRunResultDto`. |
| `AgentRunService.UPDATE` | persists `OutputJson` and `DurationMs`. |
| `AgentRunsController` response | hands the compact `AgentRunResultDto` back to the frontend. |

Synthesize is intentionally deterministic and code-only today. No second model call. No model is asked to "summarize" the prior phases — that responsibility belongs inside the single analyze call's `decide.voice` and `interrogate.balance` fields.

---

## owns

- integrating validated material from prior phases
- assembling the final decision artifact shape (`AgentRunExecutionResult`)
- mapping internal cognitive phases into compact delivery fields
- preserving evidence ownership: `EvidenceRichness` from the retriever, not the model
- preserving `AnalyzerConfidence` separately for the future learning loop
- failing safely: missing phase fields land as null without breaking the run

---

## must not

- invent new evidence
- introduce unsupported arguments or claims
- hide uncertainty surfaced in prior phases
- override evidence quality with more confident language
- call the model for "polish" or "summary"
- promote the model's local confidence to the final confidence

---

## inputs

- the complete `SportsRunArtifact`: retrieval output, analysis response, evaluator output, quality warnings, signal availability, signal follow-ups
- the agent run row metadata (`AgentRunId`, `CorrelationId`, `Competition`, `GameDate`)

---

## outputs

- `AgentRunExecutionResult` persisted to `OutputJson`
- compact `AgentRunResultDto` returned to the caller
- artifact inspection surface (`GET /api/agent-runs/{id}/artifact`) reads from the same persisted shape

---

## guardrails the platform enforces

- `EvidenceRichness` derives from `SportsRetrievalOutput.GroundedSignals.Length` — never from model output
- `Confidence` in the final DTO is `EvaluatorOutput.AggregateConfidence` — never `SportsAnalysisResponse.Confidence`
- `Posture` is the validated, clamped value from FastAPI
- compact delivery fields fall back through documented sources (e.g. `posture` falls back from `phases.decide.posture`)
- missing phase fields fail safely to null — they do not fail the run

---

## calibration hooks

A synthesize failure looks like:

- a final field that does not match its underlying phase source
- a delivery field that drifts from the cognitive artifact
- a missing fallback that surfaces an empty cell instead of a meaningful null
- a future regression where a new phase field is added but not surfaced through synthesize

When calibration finds these, the fix lands in `SportsComposer` or the DTO mapping — not in the prompt and not in evaluator code.

---

## generalization beyond sports

Synthesize is the most reusable phase. Every niche assembles its decision artifact from grounded inputs and validated phase output. The compose code per niche differs only in which fields are mapped where. The doctrine is identical: integrate, do not invent.
