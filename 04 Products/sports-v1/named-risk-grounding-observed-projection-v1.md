# Named Risk Grounding Observed Projection v1

**date:** 2026-06-18
**status:** IMPLEMENTED - observed-only, read-only projection on the dev artifact surface.
**classification:** code (TDD) + docs. No prediction/buyer/advisory/enforcement/Probe change.

**Anchor:** The model can name a risk before the factory can prove it. That gap must be inspectable before it becomes actionable.

## 1. Problem

Named Risk Grounding Review v1 defined the seam: a risk named in artifact prose (counter-case, watch-for, factors, summary) often has no grounded source group, or only a shallow one. This slice implements the pure deterministic evaluator + read-only projection, mirroring Artifact Direction Consistency Guard v1. It makes the named-risk-vs-evidence gap inspectable; it changes no prediction and triggers no Probe.

## 2. Design

Pure evaluator `NamedRiskGroundingEvaluator.Evaluate(...)` in `DevCore.Api/AgentRuns/NamedRiskGrounding.cs`, decoupled from the live pipeline (runs derive-on-read against persisted/legacy artifacts). It scans the prose sections, maps each named risk to a canonical `SourceGroups` value via a niche-owned keyword lexicon, and classifies it against the run's grounded groups. Wired into `GET /api/agent-runs/{id}/artifact` (`AgentRunsController.GetArtifact`), additive optional DTO field, derive-on-read - the same safe pattern as Artifact Direction Consistency and PerceiveFulfillment (tenant/user-scoped dev surface, never persisted, not the buyer route).

Grounded groups are taken from the already-computed `SourceSufficiency.Groups` (`IsGrounded`), so the projection is consistent with the sufficiency band.

Determinism, no NLP model:
- section weighting `counter_case` (primary) > `watch_for` > `factors` > `summary`; a group is recorded once across sections (first/highest-priority wins);
- depth-sensitivity inferred from quality/form terms (e.g. "struggles", "weakness", "form", "command"); a coarse per-competition `ShallowGroundedGroups` descriptor marks identity-only groups (MLB `starting_pitching`);
- variance markers ("in-game", "exits the game", "walk-off", ...) classify a risk as `VarianceOnly` (group-agnostic);
- `NotMappable` markers ("home-field advantage", "momentum", ...) are recorded as taxonomy-review candidates - no new group is created.

## 3. Statuses

- `Grounded` - the risk's mapped group is grounded at sufficient depth.
- `DepthInsufficient` - mapped group grounded but only at a shallow dimension (e.g. starter identity, no form/health).
- `Ungrounded` - mapped group not grounded; `ProbeCandidate=true` when a feasible pregame source exists.
- `NotMappable` - the risk maps to no canonical group (taxonomy-review candidate).
- `VarianceOnly` - in-game/postgame-only event; not source-fixable.
- `NotEvaluable` - no mappable named risk, or no prose.

Top-level `Status` is a rollup (most severe wins): `Ungrounded > DepthInsufficient > NotMappable > VarianceOnly > Grounded`; `NotEvaluable` only when there are no findings. Per-finding fields: `RiskText`, `Section`, `RiskSourceGroup`, `Status`, `ProbeCandidate`, `DepthNote`, `Reason`.

## 4. Rules (as implemented)

1. Scan `counterCase` (primary), `watchFor`, `factors`, `summary`.
2. Counter-case is a primary risk input (opposite of the direction guard, which excludes it).
3. Map only to existing canonical groups; reuse the taxonomy's vocabulary (`sharp_public` -> `market_movement`, etc.).
4. No new taxonomy groups - unmappable risks become `NotMappable` + a review candidate.
5. Group present but shallow + risk references quality/form -> `DepthInsufficient`.
6. In-game/postgame-only risk -> `VarianceOnly`.
7. No mappable risk -> `NotEvaluable`.
8. Never throws on legacy/missing fields (fail-soft).
9. Derive-on-read only; never persisted.
10. `ProbeCandidate=true` is an observed flag only - no Probe execution.

## 5. Source-Group Mappings (initial lexicon)

| risk language | RiskSourceGroup |
|---|---|
| starter / starting pitcher / mound / rotation | `starting_pitching` |
| lineup / (missing/key) bats / hitter / scratch | `lineup_injury` |
| bullpen / reliever / closer / late-inning | `bullpen_availability` |
| slump / surge / offensive form / run support | `team_form` |
| weather / wind / rain / ballpark | `weather_park` |
| sharp / public / line movement | `market_movement` |
| travel / fatigue / back-to-back / rest | `rest_travel` |
| run line / moneyline / spread / market read | `market_odds` |
| home-field advantage / momentum / intangible | `NotMappable` |

## 6. Buyer / Advisory / Enforcement / Probe Non-Impact

- No model call, no generation, no reconciliation, no Probe execution, no migration, no source integration, no threshold/confidence/posture change.
- No change to lean/confidence/posture or any persisted artifact field; computed per request, never stored.
- Surfaced only on the tenant/user-scoped dev `/artifact` DTO (additive optional field), not the buyer route; existing consumers unaffected.
- `ProbeCandidate` is an observed flag; it does not promote any `ProbeFallbackCatalog` path, enable `AllowProbeFallback`, or call the Tool Gateway. Observed warning sign != authority.

## 7. Test Coverage

TDD (red -> green). Full `DevCore.Api.Tests` **722 -> 735** (+13).

Unit (`NamedRiskGroundingEvaluatorTests`, 11):
- starter identity risk + starting_pitching grounded -> Grounded;
- starter quality risk + identity-only depth -> DepthInsufficient;
- lineup risk + lineup_injury missing -> Ungrounded;
- sharp/public risk + market_movement missing -> Ungrounded + ProbeCandidate;
- home-field risk -> NotMappable;
- counter-case is scanned;
- no mappable risk -> NotEvaluable;
- all sections missing -> NotEvaluable;
- multiple risks, mixed statuses (rollup most-severe);
- in-game event -> VarianceOnly;
- null inputs do not throw.

Integration (`AgentRunsControllerTests`, 2):
- `/artifact` projects DepthInsufficient for a starter-quality counter-case (mlb, starting_pitching grounded);
- `/artifact` projects NotEvaluable when prose names no mappable risk.

## 8. Recommended Next Slice

Two options, both deferred until chosen:
- **Settlement-completion reconcile** of the 6 pending market-aware MLB runs (separate pass, after they go Final ~2026-06-19T02:00Z) - the gating step before any source/calibration decision.
- **Source-depth enrichment** (e.g. MLB Starting Pitching Quality/Form Enrichment v1) - now that DepthInsufficient is observable, it has a measurable target; but it is an analyzer-behavior change and stays deferred until the corpus is larger.

Reminder: this slice changes no prediction, source, threshold, buyer surface, advisory/enforcement, Probe, or calibration state. If the 6 pending games go Final during related work, do not reconcile here - keep it a separate completion pass.
