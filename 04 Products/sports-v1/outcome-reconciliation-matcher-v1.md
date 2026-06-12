# Outcome Reconciliation Matcher v1

**date:** 2026-06-12
**status:** runtime matcher foundation. narrow platform change + one minimal EF migration + tests in `dai`; report/handoff/ledger in `dai-vault`. no OutputJson artifact-contract change, no buyer-facing change, no settlement provider, no scheduled job, no calibration change.
**scope:** make outcome reconciliation executable -- given a settled game outcome, deterministically find the identity-bearing run(s) to evaluate against it -- without automating settlement or exposing any track record.

## Purpose

Stable Game Identity Capture v1 and MLB Game Identity Capture v1 put the canonical key `(SourceProvider, ExternalGameId)` on every buyer-ready run at generation time. The reconciliation scaffold (AgentRunOutcomes, AgentRunEvaluations, `RunEvaluator`, per-run `POST /api/agent-runs/{id}/outcome`) already evaluates a lean against a manually attached outcome. What was missing is the reverse direction: starting from a settled outcome, find which run(s) it should settle. This slice adds that matcher and the minimal settlement-status taxonomy the outcome reconciliation contract requires, turning identity-bearing runs into reconcilable calibration samples.

## What already existed (not duplicated)

- `AgentRunOutcome` (status, scores, source, sourceRef, resolvedUtc, notes; one-per-run unique index) and `AgentRunEvaluation` (evalStatus, leanSide, winningSide).
- `RunEvaluator.WinningSide` / `Evaluate` -- pure lean-vs-outcome verdict (correct / incorrect / inconclusive).
- `POST /api/agent-runs/{id}/outcome` -- manual per-run path that writes outcome + derived evaluation in one transaction, tenant-scoped, 409 on duplicate.
- `OutcomeStatuses` constants (home_win/away_win/draw/cancelled/postponed) mirrored by the `CK_AgentRunOutcomes_OutcomeStatus` check constraint.

The matcher, the reverse `(provider, id) -> run` lookup, and the suspended/void/unknown settlement states did not exist.

## Match key and matcher design

Canonical key: `(SourceProvider, ExternalGameId)` -- exact, provider-scoped equality. No display-name fallback, no identity inference from team names (the contract forbids it; names drift and collide). Legacy/null-identity runs are never matched.

The matcher is split into a pure core and a thin db-backed query service:

- `OutcomeReconciliationMatcher.Match(request, candidates)` -- pure, no i/o. Classifies into `OutcomeMatchKind`:
  - `NotEvaluable` -- the outcome status is not a final settlement; checked first, before any matching.
  - `NoMatch` -- no identity-bearing run shares the key.
  - `SingleMatch` -- exactly one run matches.
  - `MultipleMatches` -- more than one run matches; surfaced for review, never silently collapsed to one.
- `OutcomeReconciliationService.MatchAsync(tenantKey, provider, externalGameId, status)` -- loads the tenant's runs that share the key (the SQL equality naturally excludes null-identity rows) and delegates to the pure matcher. It only finds and classifies; it never writes.

The matcher distinguishes evaluability from direction deliberately. `IsFinalSettlement` is true only for home_win/away_win/draw. `draw` is a final settlement but yields an inconclusive verdict (no directional winner) -- that distinction is the evaluator's job, not the matcher's.

## Settlement taxonomy

Expanded `OutcomeStatuses` to the contract's full settlement axis by adding `suspended`, `void`, `unknown` to the existing `cancelled`, `postponed` (non-final) and `home_win`/`away_win`/`draw` (final-with-direction). Added a `FinalSettlement` set + `IsFinalSettlement(status)` classifier as the single source of truth for evaluability. The contract's `final` settlement state maps to the directional trio (this schema keeps direction fused into the final statuses rather than adding a bare directionless `final`); this reconciliation is documented so the fused-vs-split choice is explicit.

One minimal EF migration (`ExpandOutcomeStatusTaxonomy`) drops and recreates `CK_AgentRunOutcomes_OutcomeStatus` with the expanded set, keeping the check constraint mirroring `OutcomeStatuses` as the constraint comment mandates. Clean `Down`. No table, column, or index change.

## Manual / internal reconcile path

Added `POST /api/agent-runs/reconcile` (tenant-scoped, same identity resolver as the per-run endpoint). It runs the matcher and:

- `NotEvaluable` / `NoMatch` / `MultipleMatches` -> returns the classification, writes nothing. MultipleMatches returns every matched run id for human review.
- `SingleMatch` -> records the outcome + derived evaluation for that one run, reusing the exact same write path as the per-run endpoint (extracted into a shared `AddOutcomeAndEvaluation` helper) and the same one-outcome-per-run 409 guard.

The write path is single-sourced: the per-run endpoint and the reconcile endpoint produce identical outcome+evaluation rows through `AddOutcomeAndEvaluation` + `RunEvaluator`. Buyer-advertised strength, confidence, posture, and lean are untouched -- correctness remains lean-direction-vs-outcome only.

## Changes made

backend (`dai`):
- `AgentRunContracts.cs`: `OutcomeStatuses` += Suspended/Void/Unknown + `FinalSettlement` set + `IsFinalSettlement`; new `ReconcileOutcomeRequest` / `ReconcileOutcomeResultDto`.
- `OutcomeReconciliationMatcher.cs` (new): `OutcomeMatchKind`, `RunIdentityCandidate`, `OutcomeMatchRequest`, `OutcomeMatchResult`, pure `Match`.
- `OutcomeReconciliationService.cs` (new): `IOutcomeReconciliationService` + db-backed `MatchAsync`.
- `AgentRunsController.cs`: inject the service; extract `AddOutcomeAndEvaluation` from `RecordOutcome` (behavior-preserving); add `Reconcile` endpoint.
- `Program.cs`: register `IOutcomeReconciliationService` (scoped).
- `AppDbContext.cs`: expand the check constraint; `ExpandOutcomeStatusTaxonomy` migration + snapshot.

tests (`dai`, test-first):
- `OutcomeReconciliationMatcherTests.cs` (new, 13): single/no/multiple match; legacy-null ignored; non-final not-evaluable (all five); final evaluable (all three).
- `AgentRunsControllerTests.cs` (9 new reconcile tests): single match records outcome+evaluation; non-final writes nothing (all four non-final); no match; multiple matches writes nothing and does not pick; legacy-null ignored. Existing `record_outcome*` tests unchanged (per-run compatibility).

## Verification

- TDD throughout; red observed before green at each step (pure matcher: missing types/constants; reconcile: missing DTOs/endpoint).
- bounded ladder (dai-test-discipline): matcher class 13/13; reconcile 9/9; outcome/eval + identity-adjacent 73/73.
- final verification: full .NET suite 659 passed / 0 failed (+22 vs prior 637); solution build succeeded; `npm test` 47/47; `ng build` emitted bundle; zero diffs under `apps/` and `services/`; `git diff --check` clean; single migration; added-line ascii scan clean.
- OutputJson artifact contract unchanged; no prompt/parser/confidence/posture/lean/cost change.

## Deferred (explicitly)

- **display-name fallback: forbidden**, not merely deferred -- the matcher matches only on the exact provider-scoped key.
- **settlement provider integration: deferred** -- no external feed; the reconcile path is internal/manual.
- **scheduled settlement: deferred** -- no jobs, no automation, no bulk.
- **multiple-match auto-evaluation: deferred** -- surfaced for review; evaluating every matched run (vs one) is a future decision, not taken here.
- **buyer-visible track record: deferred** -- this slice writes evaluation rows for internal calibration only; nothing buyer-facing.
- **confidence calibration changes: deferred** -- evaluations accumulate as samples; no threshold or confidence math moves (ledger entry 12 still gated on reconciled-outcome evidence, which this slice now begins producing).

## Recommended next work

- Calibration read surface v1 (internal): aggregate AgentRunEvaluations by evidence-sufficiency tier / sport / posture / raw-confidence band for the delayed feedback loop -- still internal, no buyer claim.
- A decision on MultipleMatches: evaluate-all vs human-pick, once real duplicate runs appear.
- Begin manual Stage 0 reconciliation on settled NBA/MLB runs via the new reconcile endpoint to learn outcome-source reliability before any provider integration.
