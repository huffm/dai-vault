# phase: decide

**date:** 2026-05-11 (updated 2026-05-14)
**status:** v1 doctrine — decide is partially model-owned (posture, voice) and partially platform-owned (final confidence calibration).

---

## cognitive protocol runtime alignment (2026-05-14)

Decide is the fourth macro protocol of the Cognitive Protocol Runtime. Its three canonical micro-actions are:

| canonical micro-action | responsibility |
|---|---|
| Resolve | reconcile the weighed evidence into a single direction or non-direction |
| Position | set the decision posture from the niche posture vocabulary |
| Justify | one-sentence calibration rationale for why the stated confidence fits the evidence |

### Position is decision posture, not a pick

Position is the canonical word. It names the decision posture the platform supports, drawn from the niche posture vocabulary. In sports today that vocabulary is `play`, `pass`, `monitor`, `wait`, `compare`, `avoid`. The UI label remains `Read Stance`. Position must never be labeled `Pick`, must never read as a lock, and must never become a recommendation to act.

An earlier proposal made Decide.Choose the canonical name. That is rejected. Any future code that introduces a `Choose` field is a regression against decision 0004.

### legacy field mapping

Today the analyze prompt emits `decide.calibrate`, `decide.posture`, and `decide.voice`. The canonical mapping is:

| legacy field | canonical micro-action |
|---|---|
| `decide.calibrate` | Decide.Justify |
| `decide.posture` | Decide.Position |
| `decide.voice` | Decide.Resolve |

The posture enum, the FastAPI clamp, the `SportsEvaluator` confidence calibration, and the `Read Stance` UI label are unchanged. See `../protocol-vocabulary-map.md` for the full map.

---

## responsibility

Decide turns the discerned evidence into a calibrated stance. It names the posture, explains why the confidence fits the evidence, and chooses the voice — without hype, without false certainty.

In a sentence: **decide sets the read stance and the calibration narrative that justifies it.**

---

## current code mapping

| current step | role in decide |
|---|---|
| `analyze` prompt `decide` block | the model emits `calibrate`, `posture`, `voice`. |
| posture clamp (FastAPI) | invalid posture values are clamped to null before leaving the analyzer. |
| `SportsEvaluator.Evaluate` | deterministic confidence calibration; uses grounded signal count and confidence-effect flags. |
| `EvaluatorOutput.AggregateConfidence` | the final confidence value the user sees; **not** the analyzer's local estimate. |

Final confidence belongs to the platform, not the model. The model's local confidence is stored separately as `AnalyzerConfidence` for the future learning loop.

---

## owns

- calibrate: a one-sentence rationale for why the stated confidence matches the evidence
- posture: the read stance from the allowed vocabulary (`play`, `pass`, `monitor`, `wait`, `compare`, `avoid`)
- voice: the framing — calibrated, no hype, no lock language
- final confidence value (`AggregateConfidence`) — produced deterministically by the platform

---

## must not

- let confidence exceed what the evidence supports
- use hype or lock language
- call uncertain reads certain
- produce a posture not supported by evidence quality
- pick an aggressive posture (`play`) when discern raised `block_aggressive_posture`
- treat the model's local confidence as the final value

---

## inputs

- discern output: signal availability, signal quality, follow-up records, confidence effect flags
- interrogate output: balance, stress, reframe
- the platform's deterministic calibration model (`SportsEvaluator`)

---

## outputs

- structured `decide` block: `calibrate`, `posture`, `voice`
- compact deliver field: `posture` (UI label: `Read Stance`)
- final calibrated confidence (`AggregateConfidence`)
- `ConfidenceBand` (`high / medium / low`)

---

## guardrails the platform enforces

- posture enum clamping in FastAPI
- `block_aggressive_posture` is honored: if discern flagged it, an aggressive posture is downgraded
- confidence calibration clamps:
  - zero grounded signals: dampen by 0.75; clamp to [0.30, 0.60]
  - partial grounding: dampen by 0.90; clamp to [0.35, 0.75]
  - full grounding: clamp to [0.35, 0.85]
- the UI labels posture as `Read Stance` — never `Pick`

---

## calibration hooks

A decide failure looks like:

- posture aggressive but stress notes were severe
- confidence above 0.75 with only one grounded signal
- voice that uses hype or lock language and slipped past quality rules
- calibrate sentence that does not actually justify the band

When calibration finds these, the fix lands in `SportsEvaluator` clamps, the decide prompt block, or new quality rules — not in compose or interrogate.

---

## generalization beyond sports

The posture vocabulary is niche-specific. The decide doctrine is not. For stocks, postures might be `enter`, `accumulate`, `hold`, `trim`, `exit`. For kalshi, `take`, `pass`, `monitor`. The owning code (calibration clamp, enum validator) stays platform-shaped; the vocabulary lives in niche config.

---

## sharp/public fallback ladder v1 (downstream impact)

Discern now emits two new permissions per follow-up record:

- `ConfidencePermission` — the most this fallback may contribute to final confidence (`confidence_preserved`, `confidence_mostly_preserved`, `confidence_dampened`, `confidence_conservative`, `confidence_reduced`).
- `PosturePermission` — the strongest posture this fallback may permit (`aggressive_allowed`, `aggressive_allowed_if_corroborated`, `aggressive_blocked`).

Decide must respect these. If any active fallback emits `aggressive_blocked`, `play` is not a valid posture for the run. If every relevant fallback emits `confidence_conservative` or `confidence_reduced`, decide must not produce a high-band confidence value.

**This wiring is observational in v1.** The ladder fields are persisted on the artifact but `SportsEvaluator` has not yet been adapted to consume them — it still uses the existing grounded-count and confidence-effect clamps. Wiring `ConfidencePermission` into the calibration path is a future slice that should be scoped against outcome data. Until that slice lands, decide reads the ladder via the Claude Code diagnostics skill, not via deterministic code enforcement.

See `../signal-fallback-ladder.md`.
