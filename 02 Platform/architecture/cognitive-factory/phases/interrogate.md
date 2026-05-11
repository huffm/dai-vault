# phase: interrogate

**date:** 2026-05-11
**status:** v1 doctrine — implemented today as a prompt block inside the single FastAPI model call.

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
