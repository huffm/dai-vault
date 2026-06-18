# Artifact Direction Consistency Guard v1

**date:** 2026-06-18
**status:** IMPLEMENTED - observed-only, read-only projection on the dev artifact surface.
**classification:** code (TDD) + docs. No prediction/buyer/advisory/enforcement change.

**Anchor:** Artifacts must be internally consistent before they can be trusted as decision products.

## 1. Problem

Settled Miss Corpus and Artifact Consistency Review v1 found the highest-leverage factory defect is not a single missing source - it is artifact-quality/contract risk. Structured `LeanSide` drives evaluation, but the buyer-visible prose (lean sentence, summary, factors) can imply a different side or be ambiguous, and legacy artifacts may lack the team reference needed to judge direction at all. No settled miss was caused by a structured error; the risk is a later reviewer being misled by the prose surface (EvaluationSurfaceRisk), and the pending `bdde423e` (structured `home` vs Twins prose) is a live instance.

This guard makes that inconsistency *inspectable* without changing any prediction.

## 2. Design

A pure, deterministic evaluator `ArtifactDirectionConsistencyEvaluator.Evaluate(...)` in `DevCore.Api/AgentRuns/ArtifactDirectionConsistency.cs`. It takes a small fail-soft input (decoupled from the live pipeline so it also runs derive-on-read against persisted/legacy artifacts):

- `LeanSide` (structured side), `HomeTeam`/`AwayTeam` refs, `Lean`/`Summary`/`CounterCase` prose, `ArtifactVersion`, optional `Factors`.

It compares the **structured** side against the **prose** side and returns a read-only `ArtifactDirectionConsistency` record. It is wired into the existing dev/diagnostic read surface `GET /api/agent-runs/{id}/artifact` (`AgentRunsController.GetArtifact`), derive-on-read, exactly mirroring the `PerceiveFulfillment` precedent - tenant+user scoped, never persisted, not the buyer route.

Heuristics are intentionally simple and deterministic (no NLP model):
- team mentions detected by **distinctive** alphanumeric tokens (length >= 3); tokens shared by both teams (e.g. "new"/"york") are removed so same-city teams are told apart by nickname;
- whole-word matching (`\b token \b`, case-insensitive);
- section weighting: **lean sentence > summary > factors**; the highest-weight section that names a team decides direction;
- the **counter-case is never a direction source** (rule 6) - it is the case against the lean and naturally names the opponent.

## 3. Statuses

- `Consistent` - prose direction supports the structured side.
- `PotentialMismatch` - prose direction points to the opposite side only.
- `Ambiguous` - both sides appear in the weighted prose / direction unclear.
- `NotEvaluable` - no structured lean, missing team refs, or no usable prose (fail-soft, never failure).

Result fields: `Status`, `StructuredLeanSide`, `StructuredLeanTeam`, `DetectedProseSides`, `CheckedSections`, `Warnings`, `Reason`, `ArtifactVersion`.

## 4. Rules

1. No structured `LeanSide` -> `NotEvaluable` (`reason=no_structured_lean`).
2. Missing/indistinct team refs or no usable prose -> `NotEvaluable` (`missing_team_reference` / `no_prose_sections` / `no_team_mention_in_prose`), never an exception.
3. Prose supports the structured side -> `Consistent`.
4. Prose points to the opposite side only -> `PotentialMismatch`.
5. Both sides appear in the weighted prose -> `Ambiguous`.
6. Counter-case is not parsed as final direction - lean/summary outweigh it.
7. No existing artifact field is altered.
8. No buyer output change.
9. Reads/reconciliation are never blocked (pure, read-only, fail-soft).

## 5. Version-Aware Behavior

The evaluator is version-tolerant and fail-soft: it reports `ArtifactVersion` but does not branch behavior on it. Legacy/`v2` artifacts that lack team references resolve to `NotEvaluable` (`missing_team_reference`) rather than a false verdict - which is exactly the legacy `5816433E` case from the corpus review (structured `home`, prose "Thunder", no team ref). This means the guard never manufactures a mismatch from a shape it cannot read.

## 6. Buyer / Advisory / Enforcement Non-Impact

- No model call, no generation, no reconciliation, no Probe, no migration, no source integration, no threshold change.
- No change to lean, confidence, posture, or any persisted artifact field; the value is computed per request and never stored.
- Surfaced only on the tenant/user-scoped dev `/artifact` inspection DTO (the `PerceiveFulfillment` surface), not the buyer artifact route; the field is additive and optional, so existing consumers are unaffected.
- Observed-only: it gates nothing and grants no enforcement authority (safety-engineering separation of warning sign from authority).

## 7. Test Coverage

TDD (red -> green), full `DevCore.Api.Tests` **722/722**.

Unit (`ArtifactDirectionConsistencyEvaluatorTests`, 10):
- consistent home lean; consistent away lean;
- opposite-side prose mismatch;
- ambiguous both-team language;
- null lean -> NotEvaluable;
- missing team refs (legacy) -> NotEvaluable; no usable prose -> NotEvaluable;
- counter-case names opponent but final direction stays Consistent;
- summary used as fallback when the lean sentence names no team;
- same-city teams distinguished by nickname.

Integration (`AgentRunsControllerTests`, 2):
- `/artifact` projects `Consistent` when prose matches the structured side (with team refs);
- `/artifact` projects `NotEvaluable` (`missing_team_reference`) for a run without team refs.

## 8. Recommended Next Slice

**Named Risk Grounding Review v1** (doc-first seam): map each named risk in `counterCase`/`watchFor` to its source group and define the observed seam where a *named-but-ungrounded* risk becomes a future Probe candidate - without wiring any Probe/enforcement. This pairs with this guard to complete the artifact-quality/contract inspection layer before any source-depth machinery.

Reminder: the 6 pending market-aware MLB runs must still settle and reconcile (after ~2026-06-19T02:00Z) before any source-priority or calibration decision. This slice changes no prediction, source, threshold, buyer surface, advisory/enforcement, or calibration state.
