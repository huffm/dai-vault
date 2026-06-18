# Named Risk Grounding Review v1

**date:** 2026-06-18
**status:** SEAM DEFINITION - docs/contract only. No code, no Probe, no enforcement.
**classification:** design seam for an observed quality concept; not an implementation.

**Anchor:** The model can name a risk before the factory can prove it. That gap must become inspectable before it becomes actionable.

## 1. Executive Summary

Across 7/7 settled misses the artifact *named* the failure mode (a starter who might struggle, a lineup slump, missing sharp/public confirmation) but had **no grounded source** for that risk - or only a shallow one. The model's awareness was correct; the factory's evidence was absent. This document defines the **NamedRiskUngrounded** seam: a deterministic, niche-configurable way to detect a named risk, map it to a canonical source group, and classify whether that group is grounded, ungrounded, shallow, variance-only, or unmappable.

This is a **seam definition, not machinery**. It specifies the concepts, the mapping rules, the status model, and the future observed-projection shape. It implements nothing, triggers no Probe, changes no prediction. It is the evidence-backing counterpart to the already-implemented Artifact Direction Consistency Guard (which checks direction, not evidence).

## 2. Problem Statement

Structured `LeanSide` drives evaluation; prose explains it. When the prose raises a risk ("if Teng struggles early", "Twins could break out of their slump", "absence of sharp/public data"), a reader cannot tell whether the factory actually *checked* that risk or merely *mentioned* it. A mentioned-but-unchecked risk is an inspection blind spot: it reads like diligence but carries no grounded evidence. The seam makes the gap visible so a reviewer (and, much later, a Probe) can act on it - without pretending a mention is a check.

## 3. Relationship to Artifact Direction Consistency Guard v1

Different axis, deliberately opposite weighting:

| | Artifact Direction Consistency Guard (implemented) | Named Risk Grounding (this seam) |
|---|---|---|
| question | does prose direction match the structured side? | is a named risk backed by grounded evidence? |
| axis | WHICH SIDE | IS IT CHECKED |
| counter-case | **excluded** as a direction source (it names the opponent by design) | **primary input** - the counter-case is where risks are named |
| inputs | lean sentence, summary, team refs, LeanSide | counter-case/watch-for/factors/summary, grounded signals, source mapping, depth, feasibility |
| status | Consistent / PotentialMismatch / Ambiguous / NotEvaluable | Grounded / Ungrounded / DepthInsufficient / NotMappable / VarianceOnly / NotEvaluable |
| state | live observed projection on `/artifact` | not implemented; seam only |

The two are complementary observed quality checks; neither gates anything.

## 4. Named Risk Definitions

- **NamedRisk** - a discrete way-the-read-could-fail that the artifact prose explicitly raises (a stated hazard, not a generic hedge).
- **RiskSourceGroup** - the canonical source group whose evidence would substantiate or refute that risk.
- **RiskGroundingStatus** - the classification of a NamedRisk against its mapped group (status model in section 6).
- **NamedRiskUngrounded** - a NamedRisk whose mapped group is **not grounded** at all.
- **NamedRiskGrounded** - a NamedRisk whose mapped group is grounded **at decision-grade depth**.
- **NamedRiskDepthInsufficient** - the mapped group is grounded but **too shallow** to bear the risk's claim (e.g., starter identity present, starter form/health absent).
- **PregameActionableRisk** - a NamedRiskUngrounded / DepthInsufficient whose evidence is realistically obtainable **before first pitch** (vs. an in-game event).
- **VarianceOnlyRisk** - the risk describes an in-game/postgame-only event (a blowout, an in-game injury exit, a single-game RBI outlier); **not source-fixable**, must not become a source requirement.
- **ProbeCandidateRisk** - a PregameActionableRisk whose RiskSourceGroup has a feasible acquisition path (`ProbeFallbackCatalog` `supported` or `future_candidate`); the **future** Probe trigger, surfaced observed-only here.

Scope note: NamedRisk is about hazards in the *case against* the read. It is distinct from `whatWouldChangeTheRead` (flip conditions) and from posture hedges; those may be scanned but are weighted below the counter-case.

## 5. Source-Group Mapping Rules

Map risk language to the canonical groups (`SourceSignalTaxonomy.SourceGroups`); reuse the existing signal->group aliases. Mapping is **niche-owned** (a sports lexicon), the mechanism is platform-level.

| named-risk language | RiskSourceGroup | tier |
|---|---|---|
| starter weakness / return from injury / "if X struggles" / starter form | `starting_pitching` | Critical |
| lineup weakness / missing bats / scratches / key hitter out | `lineup_injury` | Supporting |
| bullpen fatigue / late-inning collapse risk / overworked pen | `bullpen_availability` | Supporting |
| recent offensive surge or slump / run-support inconsistency | `team_form` | Contextual |
| weather / wind / park run environment | `weather_park` | Contextual |
| line movement / sharp-vs-public disagreement / missing confirmation | `market_movement` (incl. `sharp_public`) | Supporting |
| travel / rest / schedule fatigue | `rest_travel` | Supporting |
| market/run-line read itself | `market_odds` | Critical |

Mapping rules:
1. Map to exactly one **primary** group; a risk may carry a secondary group (record but classify on the primary).
2. Use the existing alias table (`sharp_public`/`public_sharp` -> `market_movement`; `injury_report` -> `lineup_injury`; `rest_fatigue` -> `rest_travel`; `lineup_form`/`matchup_style`/`situational`/`home_court` -> `team_form`; `weather`/`ballpark` -> `weather_park`).
3. **Do not invent taxonomy groups.** If a risk does not map cleanly (e.g., "home-field advantage", pure managerial/intangible), mark `NotMappable` and log a candidate for a later taxonomy review. NotMappable is a signal about the taxonomy, not a defect of the run.
4. Generic hedges with no hazard ("anything can happen") are not NamedRisks - they yield `NotEvaluable` for that section.

## 6. Grounding Status Model

Per NamedRisk, against its mapped RiskSourceGroup G and the run's grounded signals:

- **Grounded** - G is grounded and its grounded depth is decision-grade for the risk's dimension.
- **DepthInsufficient** - G is grounded but only at a shallow dimension (identity/availability) while the risk references quality/form/health. (Ties to SourcePresence-vs-SourceDepth from the miss-corpus review.)
- **Ungrounded** - G is not grounded at all; if G is feasible pregame -> also `PregameActionableRisk` (and `ProbeCandidateRisk` when a catalog path exists).
- **NotMappable** - the risk maps to no canonical group.
- **VarianceOnly** - the risk is an in-game/postgame-only event; not source-fixable.
- **NotEvaluable** - no NamedRisk present, or insufficient prose to identify one.

Depth determination for v1 is a **coarse per-group descriptor** (e.g., MLB `starting_pitching` is currently identity-only -> shallow), not a fuzzy NLP score. Refining depth scoring is deferred.

## 7. Examples From Settled Misses

| run | named-risk prose | RiskSourceGroup | status | actionable? |
|---|---|---|---|---|
| 3D03433E | "if Teng struggles early or Melton performs unexpectedly well" | starting_pitching | DepthInsufficient (identity-only) | pregame (starter form) |
| 5403433E | "Twins lineup could break out of their slump" | team_form | Ungrounded | pregame -> ProbeCandidate |
| B0DE423E | "Pirates may have a stronger lineup against right-handers" | lineup_injury / team_form | Ungrounded | pregame -> ProbeCandidate |
| B4DE423E | "Orioles could exploit any weaknesses in Kirby's performance, but specific data is lacking" | starting_pitching | DepthInsufficient (artifact literally says data lacking) | pregame (starter form) |
| 5816433E / C1F3423E | "absence of sharp/public data raises concerns" | market_movement (sharp_public) | Ungrounded | pregame -> ProbeCandidate |
| 6416433E | "Marlins' home field could provide a competitive edge" | (home-field) | NotMappable | taxonomy review candidate |
| (variance) | O'Hearn 6-RBI game, Julio in-game exit, 12-0 blowout | n/a | VarianceOnly | not source-fixable |

Observation: the dominant settled-miss statuses are **Ungrounded** and **DepthInsufficient** on pregame-actionable groups - the same conclusion as the miss-corpus review, now expressed per-risk.

## 8. Future Observed Projection Shape (NOT implemented)

When implemented later, mirror the Artifact Direction Consistency Guard / PerceiveFulfillment precedent: a pure, deterministic, fail-soft evaluator surfaced read-only on `GET /artifact`, never persisted, gating nothing.

```
NamedRiskGroundingProjection {
  ArtifactVersion,
  Risks: [ {
    RiskText, Section,            // counter_case | watch_for | factors | summary
    RiskSourceGroup,              // canonical group or null (NotMappable)
    Status,                       // Grounded | DepthInsufficient | Ungrounded | NotMappable | VarianceOnly | NotEvaluable
    PregameActionable,            // bool
    ProbeCandidate,               // bool (catalog path exists)
    Reason
  } ],
  Counts,                         // status histogram
  OverallFlag                     // e.g. any Critical-tier risk Ungrounded/DepthInsufficient
}
```

Section weighting (opposite of the direction guard): `counter_case` primary, then `watch_for`/`what_would_change_the_read`, then `factors`/`summary`.

## 9. Future Probe-Candidate Behavior (NOT implemented)

`ProbeCandidateRisk` is the natural **future** Probe trigger: a pregame-actionable risk whose source group has a catalog acquisition path could, in an enforcement-aware future, prompt a Probe to fetch that group before/with the read. In the observed seam it is **surfaced only**. Probe execution stays disabled (MLB `AllowProbeFallback=false`; most MLB risk groups are `future_candidate`, not `supported`). Defining ProbeCandidateRisk does **not** promote any catalog path, enable Probe, or change `SportSufficiencyProfile`.

## 10. Explicit Deferred Decisions

- **Probe execution / triggering** - deferred; this seam only names the candidate.
- **Advisory / soft / hard enforcement** - deferred; observed warning sign never equals authority.
- **Confidence dampening** on ungrounded critical-tier risks - deferred; a threshold change that requires calibration evidence (and the 6 pending market-aware runs are unsettled).
- **Buyer copy** ("what we could not verify") - deferred; high overclaim risk.
- **New taxonomy groups** - deferred; NotMappable accrues a review list, nothing is created casually.
- **Depth scoring** beyond a coarse per-group descriptor - deferred.
- **The observed projection implementation itself** - deferred to its own slice (section 11).

## 11. Product / System Review + Recommended Next Slice

1. **All-niche or sports-only?** All-niche **mechanism** (named risk -> group -> grounding status), with a **niche-owned lexicon + depth descriptors** (like `SportSufficiencyProfile`). Build it platform-level, configure per niche.
2. **Observed projection later?** Yes - same read-only `/artifact` pattern as the direction guard. Not now.
3. **Feed Probe later?** Yes, via `ProbeCandidateRisk`, but only after the observed projection exists and shows value. Deferred.
4. **Affect buyer copy later?** Possibly (honesty line); deferred, overclaim risk.
5. **Affect confidence later?** Possibly (dampen on ungrounded critical risk); deferred, needs calibration evidence.
6. **Minimum safe path?** This seam doc -> then a pure observed `NamedRiskGroundingEvaluator` + read-only `/artifact` projection (no Probe/enforcement/confidence), TDD, mirroring the direction guard.
7. **What would overbuild look like?** Wiring Probe execution, enforcement, confidence math, or buyer copy now; a fuzzy NLP risk extractor; speculative new taxonomy groups.

**Recommended next slice:** *Named Risk Grounding Observed Projection v1* - implement the pure deterministic evaluator + read-only `/artifact` projection defined here, observed-only, TDD, no Probe/enforcement/confidence/buyer change.

Reminder: the 6 pending market-aware MLB runs must still settle and reconcile (after ~2026-06-19T02:00Z) **as a separate completion pass** before any source-priority or calibration decision. If those games go Final during this work, do not reconcile here. This slice changes no prediction, source, threshold, buyer surface, advisory/enforcement, or calibration state.
