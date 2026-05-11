# phase: discern

**date:** 2026-05-11
**status:** v1 doctrine — discern is the most code-heavy phase today. signal quality and follow-up diagnostics already live in dedicated evaluators.

---

## responsibility

Discern judges evidence quality. It separates grounded evidence from weak or absent evidence, listens to external signals when present, and decides which signals get to influence the lean.

In a sentence: **discern decides what counts as evidence and how much weight each signal carries.**

This is the phase that owns the signal follow-up diagnostics work. The first concrete skill pack lives here.

---

## current code mapping

| current step | role in discern |
|---|---|
| `analyze` prompt `discern` block | `test`, `listen`, `filter` emissions from the model. |
| `SignalQualityEvaluator` | deterministic: per-signal `Quality`, `DecisionUse`, `FollowUpSignals`, `ConfidenceEffect`. |
| `SignalFollowUpEvaluator` | deterministic: resolves each follow-up signal to a status + reason + decision use. |
| `SportsEvaluator` calibration | maps grounded count and confidence-effect flags to a confidence band. |

Discern has the richest deterministic surface in the current platform. The model contributes the listen/filter narrative; the platform owns the structured judgment.

---

## owns

- per-signal quality classification (`strong / usable / unavailable`)
- per-signal decision use (`primary_signal / confirmation / proxy_candidate / directional_only`)
- follow-up signal resolution (`grounded / missing / unavailable / not_implemented / candidate`)
- confidence effect per signal (`support / support_cautiously / dampen / block_aggressive_posture / neutral`)
- listening to external signals (market, sharp/public split) when present
- filtering grounded vs weak vs absent evidence in the model's `filter` block

---

## must not

- treat model interpretation as grounded fact
- admit stale or weak evidence as equivalent to strong grounded evidence
- ignore missing information
- emit null for `listen` when external signal blocks were provided in the prompt
- override the platform's grounded signal count with a model claim

---

## inputs

- perceive output: `SportsRetrievalOutput`, `SignalAvailability[]`
- interrogate output (the counter-case, stress, reframe)
- the model's emerging read

---

## outputs

- `SignalAvailabilityRecord[]` enriched with quality + decision use + follow-up suggestions + confidence effect
- `SignalFollowUpRecord[]` with resolved status per (parent signal, follow-up) pair
- a structured `discern` block in the cognitive artifact: `test`, `listen`, `filter`
- inputs to decide: confidence band, blocked postures, evidence-quality flags

---

## guardrails the platform enforces

- `SignalFollowUpEvaluator` runs immediately after `SignalQualityEvaluator` inside `SportsRetrievalOutput` — no model call is allowed between them
- `block_aggressive_posture` is propagated to decide and read by the calibration report (`signal_quality_blocks_aggressive_posture` flag)
- signal vocabulary is canonicalized by `SportsQualityChecker` rule 4 before grounded-set membership checks (e.g. `rest_fatigue → rest_schedule`, `public_sharp → sharp_public`)
- `line_movement` is centralized in `SignalFollowUpEvaluator.NotImplementedSignals` so a future "Line Movement Proxy v1" slice flips it in one place

---

## calibration hooks

A discern failure looks like:

- a signal that should have been graded `usable` was treated as `strong`
- a missing primary signal did not trigger a follow-up record
- `block_aggressive_posture` did not propagate to the calibration report
- a proxy candidate was recommended that does not actually substitute for the missing primary

When calibration finds these, the fix lands in `SignalQualityEvaluator`, `SignalFollowUpEvaluator`, or the discern prompt block — never in decide or compose.

---

## the first skill pack lives here

`dai-signal-follow-up-diagnostics` is a Claude Code skill that helps a reviewer read an artifact's discern surface and recommend the next slice. It does not change discern code. It produces a structured diagnosis that names:

- which signals are missing, weak, stale, conflicting, or proxy-only
- which signals block aggressive posture
- which signals should reduce confidence
- recommended follow-up sources or proxies
- the owning cognitive phase (almost always discern)
- the recommended next slice

See `dai/.claude/skills/dai-signal-follow-up-diagnostics/SKILL.md`.

---

## generalization beyond sports

For crypto: discern judges on-chain signal freshness, sharp-wallet activity, and market regime. For stocks: discern judges fundamental versus narrative signals. The vocabularies and evaluators differ. The doctrine is the same: discern owns evidence quality and follow-up routing.

---

## sharp/public fallback ladder v1

Discern now owns explicit fallback equivalence classification. When a follow-up signal is recommended in place of a missing primary, discern grades the candidate against a six-tier ladder:

1. `exact_recovery` (same signal, same source — retry / normalize)
2. `source_substitution` (same signal, alternate source)
3. `fidelity_downgrade` (same concept, lower fidelity)
4. `adjacent_proxy` (different signal, same dimension)
5. `lateral_proxy` (signal from a different dimension)
6. `unavailable_with_reason` (no fallback present)

The ladder is implemented by `SignalFollowUpEvaluator` and emitted on `SignalFollowUpRecord` as `FallbackType`, `Equivalence`, `ConfidencePermission`, and `PosturePermission`. The full table, invariants, and concrete classifications live in `../signal-fallback-ladder.md`.

Key rule: **a lateral proxy never lifts confidence**, and only `exact_recovery` or `source_substitution` may permit `aggressive_allowed_if_corroborated`. This is the rule that prevents `line_movement` from being treated as a substitute for missing `sharp_public`.
