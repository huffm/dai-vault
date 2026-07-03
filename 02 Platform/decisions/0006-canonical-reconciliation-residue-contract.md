# decision 0006: canonical reconciliation residue contract

**date:** 2026-07-03
**status:** accepted (implemented in `dai`; docs-only in vault)

## context

Outcome Reconciliation v7c found that the seven remaining 2026-07-02 games were already settled by an
out-of-process burst at 2026-07-03T15:39:13-16Z. The outcomes were correct (scores matched StatsAPI,
idempotency held, evaluations were right), but the settlement **residue was thin**: `Source` was set
(`statsapi_final`) while `SourceRef` and `Notes` were null -- unlike the manual v7/v7b writes which
carry `SourceRef = "gamePk N"` and a descriptive note. A DB scan showed this is not unique to the
burst: across 104 outcomes, `manual` had 8 null-`SourceRef`, `mlb_statsapi` had 7 null-`Notes`. The
root cause is that the canonical settlement API (`POST /{id}/outcome`, `POST /reconcile`) accepted
`SourceRef`/`Notes` as optional and persisted whatever it was given.

A full writer-path inventory (both repos + scripts + vault runbooks) confirmed there is exactly **one**
production write path -- the shared helper `AddOutcomeAndEvaluation` in `AgentRunsController`, reached
only through those two endpoints. There is **no** poller / auto-settle / background service; the
"all-final poller" of v7b was an ad-hoc manual/script invocation of the canonical API. The dev harness
`scripts/dev/sports/reconcile-calibration-outcomes.ps1` was the concrete example of the defect: it
POSTs to the canonical API with `sourceRef = $null`. So the burst was **incomplete canonical-route
usage**, not a route bypass.

## decision

Establish a single **Canonical Reconciliation Residue Contract**: every settlement WRITE must carry
complete provenance, enforced at the one choke point, so no future write -- manual, script, or any
future automation -- can produce thin residue.

1. **Required residue fields on any outcome write:** `source`, `sourceRef`, `notes` (all non-blank).
   Enforced by `SettlementProvenance.MissingFields` and the controller guard
   `SettlementProvenanceRefusalOrNull`, which returns **422** listing the missing fields and writes
   nothing.
2. **Enforcement point + ordering:** the guard runs in both settlement endpoints immediately before
   `AddOutcomeAndEvaluation`, **after** idempotency (409) and direction-integrity (422). Only real
   write attempts require residue; non-writing outcomes (NotEvaluable / NoMatch / MultipleMatches /
   non-final / postponed / 409 / direction-refusal) are unaffected. This includes no-decision
   (inconclusive, null-lean) settlements, which ARE writes and so DO require complete residue.
3. **Writer path / capture mode:** carried in `Notes` (the required structured-provenance field) as a
   convention -- a settlement note names the pass + writer path (e.g. "reconcile v7c via
   reconcile-calibration-outcomes.ps1"). No new column (no migration this slice). An explicit
   `CaptureMode` column is documented as optional future schema hardening.
4. **Readout:** the `/rows` calibration export now surfaces `settlementSource` / `settlementSourceRef`
   / `settlementNotes` (additive, trailing-optional, metrics-ignored) so a residue audit can see who
   settled a run and detect a thin write from the row alone.
5. **Operational parity:** `reconcile-calibration-outcomes.ps1` now always sends a populated
   `SourceRef` (`runId <id>`) and `Notes` (`$PassLabel`), so the harness cannot produce thin residue.
6. **Idempotency preserves residue:** a duplicate settlement attempt is 409 and never mutates the
   residue already recorded (kept from the existing symmetric idempotency guard; now test-locked).

## approved vs forbidden writer paths

- **Approved (canonical):** `POST /api/agent-runs/{id}/outcome` and `POST /api/agent-runs/reconcile`
  -> `AddOutcomeAndEvaluation`. Both now enforce the residue contract. Any future poller/automation
  MUST use these endpoints and therefore inherits the guard -- it cannot write thin residue.
- **Forbidden for settlement writes:** direct DbContext or raw SQL against `AgentRunOutcomes` /
  `AgentRunEvaluations`. The only raw-SQL toucher, `purge-dev-agent-runs.ps1`, is delete-only and
  dev/localhost-guarded (not a settler); it stays as-is. Test fixtures that seed outcomes via
  DbContext are test-only and out of contract scope.

## what NOT to backfill

The seven already-settled burst rows (and the older null-`SourceRef`/null-`Notes` rows) are **not**
backfilled. Backfilling would be a data mutation on settled records with no authorization and no
recoverable original provenance; the correct posture is forward-only enforcement. The `/rows` readout
now makes these thin rows visible for audit without altering them.

## consequences

- Incomplete settlement residue is now impossible through the canonical path and test-failing (7 new
  tests: manual + automation missing-field refusals, complete-residue persistence, no-decision residue,
  idempotency stability, and the exporter surfacing residue).
- Metrics unchanged: `/metrics` verified byte-identical (263 / reconciled 87 / matched 52 / 0.5977);
  only `/rows` gains three trailing nullable fields, following the additive pattern of ADR 0005.
- No decisioning / prompt-selection / confidence / buyer / denominator behavior changed -- this is a
  metadata-completeness guard only.

## references

- Origin: `06 Execution/reconciliations/outcome-reconciliation-follow-up-v7c.md` (the provenance-thin
  finding) and this slice `06 Execution/reconciliations/canonical-reconciliation-residue-contract-v1.md`.
- Code: `platform/dotnet/DevCore.Api/AgentRuns/SettlementProvenance.cs`,
  `Controllers/AgentRunsController.cs` (guard), `AgentRuns/PromptRouteCalibrationExport.cs` (readout),
  `scripts/dev/sports/reconcile-calibration-outcomes.ps1` (parity).
- Pattern lineage: additive `/rows` fields from ADR 0005 (persist assembly error detail).
