# signal fallback ladder

**date:** 2026-05-11
**status:** v1 — first concrete extension of Cognitive Skill Pack Architecture v1. shipped with `SignalFollowUpEvaluator` extension and the updated Claude Code skill.
**owns:** the equivalence classification for any follow-up signal recommended in place of a missing primary.

---

## core claim

Not all fallbacks are equal. Two signals that point at the same dimension can answer different questions. Two signals that share a name can come from different providers with different fidelity. The platform must distinguish these cases so confidence and posture decisions are made against the truth, not against optimism.

This doc names the six tiers, the permissions each tier may grant, and the phase that owns each step.

---

## the six tiers

| tier | fallbackType | equivalence | confidencePermission | posturePermission | meaning |
|---|---|---|---|---|---|
| 1 | `exact_recovery` | `exact` | `confidence_preserved` | `aggressive_allowed_if_corroborated` | same signal, same source — retry or normalize. the desired signal is present or recoverable. |
| 2 | `source_substitution` | `near` | `confidence_mostly_preserved` | `aggressive_allowed_if_corroborated` | same signal, alternate source. fidelity comparable; provenance differs. |
| 3 | `fidelity_downgrade` | `partial` | `confidence_dampened` | `aggressive_blocked` | same concept, lower-fidelity version (e.g. directional-only sharp split). |
| 4 | `adjacent_proxy` | `adjacent` | `confidence_conservative` | `aggressive_blocked` | different signal that answers a closely related question in the same dimension. |
| 5 | `lateral_proxy` | `lateral` | `confidence_conservative` | `aggressive_blocked` | signal from a different dimension entirely. directional context only. |
| 6 | `unavailable_with_reason` | `none` | `confidence_reduced` | `aggressive_blocked` | no fallback exists. confidence must drop; posture must downgrade. |

Two invariants hold across all six tiers:

1. **Lateral proxies never lift confidence on their own.** A grounded lateral signal is still grounded — but it is not the missing signal. The ladder pins lateral_proxy at `confidence_conservative` and `aggressive_blocked` so a calibration tune cannot accidentally regress this.
2. **Only `exact_recovery` and `source_substitution` may permit corroborated aggression.** Even those two require an independent corroborating signal — they do not unilaterally re-enable `play`.

---

## why this ladder exists

Before this slice, `SignalFollowUpEvaluator` produced records that said "next look here" but did not say "and here is what that fallback is worth." A reviewer reading the artifact had to re-derive the equivalence judgment every time. The recommended next slice in the prior calibration report was "Line Movement Proxy v1" — implicitly treating `line_movement` as a substitute for `sharp_public`. That implication is wrong.

`sharp_public` answers: *what is the split between public betting behavior and sharper money?*
`line_movement` answers: *did the market price move, and which direction?*

These are different questions. Line movement is at best adjacent. Implementing it does not replace sharp/public — it adds a complementary lens. The ladder makes that visible at the data layer.

---

## phase ownership

The ladder spans three cognitive phases. Each owns a piece.

| phase | role in the ladder |
|---|---|
| interrogate | identifies missing or weak signals and names candidate follow-ups (today: `SignalAvailabilityRecord.FollowUpSignals`). does not grade equivalence. |
| discern | classifies each candidate against the ladder. decides whether the fallback is equivalent enough to lift confidence. produces the new `fallbackType`, `equivalence`, `confidencePermission`, `posturePermission` fields on `SignalFollowUpRecord`. |
| decide | enforces `posturePermission` and `confidencePermission` when assembling the final read. `aggressive_blocked` from any active fallback means `play` is not a valid posture for this run. |

Interrogate's responsibility is **what could fill the gap**.
Discern's responsibility is **how much that candidate is worth**.
Decide's responsibility is **what posture and confidence the resulting evidence permits**.

This split is why the Claude Code skill names `discern` as the owning phase for follow-up diagnostics, while interrogate's contract (in `phases/interrogate.md`) stays focused on naming candidates, not grading them.

---

## current code mapping

- contract: `SignalFollowUpRecord` in `AgentRunContracts.cs` — four new optional fields (`FallbackType`, `Equivalence`, `ConfidencePermission`, `PosturePermission`).
- rules: `SignalFollowUpEvaluator.cs` — each existing branch picks an equivalence label and the new `Classify` helper returns the matching ladder row.
- tests: `SignalFollowUpEvaluatorTests.cs` — six new tests cover the ladder invariants.

The ladder is additive on the wire. Older records persisted before this slice have `null` ladder fields; the Claude Code skill explicitly handles that case.

## current concrete classifications

| parent | follow-up | parent state | follow-up state | fallbackType | equivalence |
|---|---|---|---|---|---|
| sharp_public | line_movement | missing | not_implemented | adjacent_proxy | adjacent |
| sharp_public | line_movement | grounded | candidate | adjacent_proxy | adjacent |
| sharp_public | market | missing | grounded | lateral_proxy | lateral |
| sharp_public | market | missing | missing | unavailable_with_reason | none |
| market | line_movement | sharp missing | not_implemented | adjacent_proxy | adjacent |
| market | line_movement | sharp grounded | candidate | adjacent_proxy | adjacent |
| market | sharp_public | missing | grounded | exact_recovery | exact |
| market | sharp_public | missing | missing | unavailable_with_reason | none |
| generic | * | * | grounded | exact_recovery | exact |
| generic | not_implemented signal | * | not_implemented | adjacent_proxy | adjacent |
| generic | * | * | other | unavailable_with_reason | none |

---

## what this slice did not do

- did not implement an alternate sharp/public provider (would be `source_substitution`).
- did not implement a schema repair on ActionNetwork's playoff-null fields (would be `exact_recovery`).
- did not implement `line_movement` retrieval. `line_movement` remains `not_implemented` and classified `adjacent_proxy` regardless of implementation status.
- did not add competition-specific ladder rules. today's rules are competition-agnostic.
- did not wire a `confidencePermission`-aware adjustment into `SportsEvaluator`. the field is observational on the artifact for now. the evaluator continues to use its existing grounded-count and confidence-effect clamps. wiring `confidencePermission` into the calibration path is a deliberate future slice that should be scoped against outcome data.

---

## calibration feedback loop

If outcome data ever shows that a tier is misclassified (e.g. `source_substitution` from a new provider is actually closer to `fidelity_downgrade` because of latency), the fix lands inside `SignalFollowUpEvaluator.Classify` or the per-parent branch. The fix never moves into decide-phase code or the calibration clamp without a separate slice.

Calibration always updates the responsible pack — in this case, the discern-phase fallback rules.

---

## relationship to Line Movement Proxy v1 (deferred)

Line Movement Proxy v1 is now explicitly deferred. The slice was previously the recommended next step. It is the *adjacent* slice, not the substitute slice. The platform should walk the ladder top down:

1. **Sharp/Public Schema Repair v1** or **Sharp/Public Alternate Provider v1** — exact_recovery or source_substitution. These attempt to actually recover the missing primary.
2. **Sharp/Public Directional-Only v1** — fidelity_downgrade. A partial recovery, accepting reduced fidelity.
3. **Line Movement Proxy v1** — adjacent_proxy. Only after the higher rungs have been tried or ruled out.

The ladder makes the sequencing explicit so the team does not jump tiers under pressure.

---

## references

- `cognitive-skill-pack-architecture-v1.md` — the architecture this slice extends.
- `phases/interrogate.md` — names the boundary: interrogate identifies, does not grade.
- `phases/discern.md` — owns the grading.
- `phases/decide.md` — enforces the permission downstream.
- `dai/.claude/skills/dai-signal-follow-up-diagnostics/rules.md` — the Claude Code skill rule that mirrors this ladder.
- `dai/platform/dotnet/DevCore.Api/AgentRuns/SignalFollowUpEvaluator.cs` — the implementation.
