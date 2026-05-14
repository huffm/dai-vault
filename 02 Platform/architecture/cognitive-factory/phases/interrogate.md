# phase: interrogate

**date:** 2026-05-11 (updated 2026-05-14)
**status:** v1 doctrine — implemented today as a prompt block inside the single FastAPI model call.

---

## cognitive protocol runtime alignment (2026-05-14)

Interrogate is the second macro protocol of the Cognitive Protocol Runtime. Its three canonical micro-actions are:

| canonical micro-action | responsibility |
|---|---|
| Question | name the strongest open question or counter-case against the emerging read |
| Probe | targeted investigation of signal gaps, evidence needs, source follow-ups, or tool-backed questions |
| Verify | confirm or reject claims by checking them against staged evidence and platform guardrails |

### why Probe and not Retrieve

An earlier proposal made Retrieve a cognitive micro-action under Interrogate. That collided with the existing platform retrieval surface (`SportsRetriever`, `RetrieveAsync`, `SportsRunArtifact.RecordRetrieve`, and the `retrieve` pipeline step label).

Probe replaces it. Probe is a cognitive action that may request platform retrieval through allowed tools, but it is not retrieval itself. Retrieval remains deterministic platform work owned by the retriever surfaces. See `../protocol-vocabulary-map.md` for the full mapping and `../../decisions/0004-cognitive-protocol-runtime.md` for the naming decision.

### legacy field mapping

Today the analyze prompt emits `interrogate.balance`, `interrogate.stress`, and `interrogate.reframe`. The canonical mapping is:

| legacy field | canonical micro-action |
|---|---|
| `interrogate.balance` | Interrogate.Question |
| `interrogate.stress` | Discern.Stress (moves protocols in the canonical model) |
| `interrogate.reframe` | Interrogate.Verify |
| (none today) | Interrogate.Probe |

`interrogate.stress` stays where it is in code and persisted runs. A future code slice will move it to Discern. Persisted `OutputJson` records and calibration reports written before that slice keep the legacy location.

Probe has no current field. Platform-side follow-up investigation today is performed deterministically by `SignalFollowUpEvaluator`. A future slice may introduce a model-emitted Probe field for cases where the model itself should request a tool-backed follow-up.

---

## responsibility

Interrogate applies pressure to the initial read. It does not generate the read and it does not finalize the read. It stress-tests what perceive surfaced.

In a sentence: **interrogate names the strongest case against the lean, the fragile assumption, and the alternate explanation.**

---

## current code mapping

| current step | role in interrogate |
|---|---|
| `analyze` prompt `interrogate` block | the model emits `balance`, `stress`, `reframe` against the perceive context. |
| prompt guardrails v1.5 | forbidden phrases and concrete-fragility rules live in the prompt today. |
| `SportsQualityChecker` | post-call quality rules flag interrogate emissions that drift into invention or platitude. |

Interrogate has no dedicated code surface yet. Its content is produced by the model and validated deterministically by the quality checker.

---

## owns

- balance: the strongest concrete case against the lean, grounded in staged signals or named as a missing-signal limitation
- stress: a concrete fragility (large spread risk, missing sharp_public, equal rest limiting schedule edge, missing injury data)
- reframe: an alternate explanation that does not invoke ungrounded traits

---

## must not

- add new facts not in the staged context
- attack for the sake of noise or word count
- decide final stance
- invent a counter-case when evidence is thin — name the limitation instead
- reference hidden strengths, resilience, trends, form, travel, weather, injuries, or player availability without grounded context

---

## inputs

- the perceive output: staged context, frame, aim
- the model's emerging read

---

## outputs

- a structured `interrogate` block inside the cognitive artifact: `balance`, `stress`, `reframe`
- compact deliver fields: `counter_case`, `watch_for`

---

## guardrails the platform enforces

- prompt-level forbidden phrases (no "could outperform", "has potential", "may surprise", "talent", "strong team", "recent performance" without grounded context)
- post-call quality rules in `SportsQualityChecker` (see rule 4 for canonical signal vocabulary)
- compact delivery fields fail safely to null when interrogate fields are absent or malformed

---

## calibration hooks

An interrogate failure looks like:

- a counter-case that does not cite a specific signal or a named limitation
- a fragility statement that is generic ("anything could happen")
- a reframe that invents a team trait or trend
- absence of an interrogate block when evidence quality is mixed

When calibration finds these, the fix lands in the interrogate prompt block, the forbidden-phrase list, or the quality rule that detects the drift — never in decide.

---

## generalization beyond sports

In other niches, interrogate is the same responsibility expressed differently. For a stock entry decision: balance might cite a missing earnings catalyst; stress might name liquidity risk; reframe might offer a sector-rotation explanation. The doctrine is identical. The forbidden vocabulary and concrete-fragility examples differ.

---

## boundary with discern on the fallback ladder

Interrogate **names candidates**. Discern **grades them**.

When a primary signal is missing, interrogate identifies which signals could conceivably fill the gap. Today this is implemented by `SignalAvailabilityRecord.FollowUpSignals` — a plain list of signal names attached to each availability record by `SignalQualityEvaluator`.

Interrogate must not classify those candidates against the Sharp/Public Fallback Ladder. That grading is discern's responsibility. Interrogate's contribution is honest naming: "here is what could be looked at next." Discern then decides whether the candidate is `exact_recovery`, `source_substitution`, `fidelity_downgrade`, `adjacent_proxy`, `lateral_proxy`, or `unavailable_with_reason`.

See `../signal-fallback-ladder.md`.
