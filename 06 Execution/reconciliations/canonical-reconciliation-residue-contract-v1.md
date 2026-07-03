---
title: "Canonical Reconciliation Residue Contract v1"
type: "reconciliation"
date: "2026-07-03"
status: "complete"
project: "DAI"
slice: "Canonical Reconciliation Residue Contract v1"
repos:
  dai: "code + tests (uncommitted)"
  dai-vault: "docs-only"
tags:
  - reconciliation
  - settlement
  - calibration
  - hardening
related:
  - "02 Platform/decisions/0006-canonical-reconciliation-residue-contract.md"
  - "06 Execution/reconciliations/outcome-reconciliation-follow-up-v7c.md"
---

# Canonical Reconciliation Residue Contract v1

**slice:** make it impossible / test-failing for any settlement writer to persist an outcome without
complete provenance (source + sourceRef + notes), so post-game residue is complete for calibration,
artifact, and prompt-selection review. non-semantic hardening only.
**status:** complete 2026-07-03. `dai` code+tests (uncommitted, not pushed); `dai-vault` docs-only.
**verification:** DevCore.Api.Tests 1043/1043; `/metrics` byte-identical; 0 reconciliation writes.

## writer-path inventory (exhaustive, both repos + scripts + vault)

| path | file / entrypoint | writes outcome | writes eval | sets SourceRef | sets Notes | canonical | risk |
|------|-------------------|:---:|:---:|:---:|:---:|:---:|------|
| per-run outcome | `AgentRunsController.RecordOutcome` `POST /{id}/outcome` -> `AddOutcomeAndEvaluation` | yes | yes | from body | from body | yes | guarded now |
| identity reconcile | `AgentRunsController.Reconcile` `POST /reconcile` -> `AddOutcomeAndEvaluation` | yes | yes | from body | from body | yes | guarded now |
| dev reconcile harness | `scripts/dev/sports/reconcile-calibration-outcomes.ps1` | via API | via API | **was $null** | was optional | yes | fixed (parity) |
| dev purge | `scripts/dev/sports/purge-dev-agent-runs.ps1` | DELETE only | DELETE only | n/a | n/a | no (raw SQL) | delete-only, localhost-guarded |
| test fixtures (x4) | `*Tests` direct DbContext `.Add(...)` | some | some | no | no | no | test-only, out of scope |
| reconcile services | `OutcomeReconciliationService` / `Matcher` / `ReconcilePrecheck` | no | no | - | - | - | read/classify only |
| python (agent-service) | calibration read paths only | no | no | - | - | - | read-only |
| vault runbooks | `04 Products/.../settlement-readiness...`, `06 Execution/handoffs`, reconciliations | via API | via API | operator | operator | yes | doc, canonical |
| future poller (roadmap) | `roadmap/sports-v1-roadmap.md`, `weekly-plans/week-01` | unbuilt | unbuilt | - | - | - | line-movement idea, not a settler |

**single production write choke point:** `AddOutcomeAndEvaluation` (`AgentRunsController.cs`). No
poller / auto-settle / IHostedService / BackgroundService exists in either repo.

## automated burst origin (2026-07-03T15:39:13-16Z)

**Conclusion: incomplete canonical-route usage, NOT a route bypass.** Evidence:
- the 7 rows carry `Source = statsapi_final` with `SourceRef`/`Notes` null (same Source string as the
  manual v7/v7b writes, which DO set sourceRef/notes);
- their `EvalStatus`/`WinningSide` are exactly what `RunEvaluator` derives, and every one of the 104
  outcomes (incl. these 7) shows `ResolvedUtc` a few ms before `EvaluatedUtc` -- the precise signature
  of `AddOutcomeAndEvaluation` (outcome constructed, eval a moment later). A bespoke SQL script would
  have to replicate both the evaluator derivation and that incidental ms gap; the API path explains
  both trivially;
- the dev harness `reconcile-calibration-outcomes.ps1` is a concrete example of a script that POSTs to
  the canonical API with `sourceRef = $null`.
- **Remaining unknown (evidence gap):** no request-audit / writer-path column exists, so the exact
  process and which of the two endpoints cannot be *proven*, only inferred. Closing that gap is what
  the `Notes` writer-path convention (and an optional future `CaptureMode` column) is for going forward.

## canonical residue contract

Required on every write: `source`, `sourceRef`, `notes` (non-blank). Enforcement:
`SettlementProvenance.MissingFields` + `SettlementProvenanceRefusalOrNull` (422), placed before the
write and after idempotency + direction-integrity. Idempotent retries stay 409 and never mutate
recorded residue. Writer-path/capture-mode carried in `Notes` (no migration). Full field mapping and
approved/forbidden paths: see ADR `0006-canonical-reconciliation-residue-contract`.

## changes made

- `platform/dotnet/DevCore.Api/AgentRuns/SettlementProvenance.cs` (new) -- the validator.
- `platform/dotnet/DevCore.Api/Controllers/AgentRunsController.cs` -- `SettlementProvenanceRefusalOrNull`
  guard wired into both write endpoints. behavior: writes with incomplete residue -> 422, nothing
  persisted; complete writes unchanged.
- `platform/dotnet/DevCore.Api/AgentRuns/PromptRouteCalibrationExport.cs` -- additive
  `settlementSource/SourceRef/Notes` on the row + RawRow projection. metrics-neutral readout of residue.
- `scripts/dev/sports/reconcile-calibration-outcomes.ps1` -- populates SourceRef + Notes (`$PassLabel`);
  can no longer post thin residue.
- tests: 7 new (missing-field refusals on manual + automation paths, complete-residue persistence,
  no-decision residue, idempotency-preserves-provenance, exporter surfacing residue) + existing writing
  tests updated to send complete residue.

## residue readout mapping

Every contract residue field maps to an existing `/rows` column or run-row field, EXCEPT capture
mode/writer path (carried in `settlementNotes`) and the precheck classification (reproducible from
`reconcile-precheck`, not persisted per row). Calibration residue (route, promptSource, fallbackReason,
fallbackDetail, sourceDepth via artifact, confidence, evidenceRichness, market fields, exclusionReason,
outcome, evaluation via resultSide/matchedOutcome) and prompt-selection residue (route,
promptSource, recipeId/version, assembledHash, fallbackReason/Detail, regime, provenance JSON) are all
already on the row. **Gap closed:** settlement source/sourceRef/notes are now surfaced. **Remaining
gap (deferred):** no per-row writer-path/capture-mode column -- convention in Notes for now.

## next targeted batch plan (do NOT run this slice)

- **Cohort:** top up `starter_enriched_market_missing` (currently n=3, 1/3) and add a second
  `starter_enriched_market_backed_depth` slate, both with market odds captured at generation so the
  market dimension is grounded. ~1 paid call/game.
- **Control/candidate:** run a control (current prompt selection) and candidate only if a specific
  prompt-selection hypothesis exists; none is justified yet, so v8 is measurement-only (no candidate arm).
- **Sample-size gate to calibrate:** the pooled report's binding gate -- at least one confidence bucket
  beyond 0.75 reaching n>=15 -- plus `conclusionsAllowed = true`. Until then, descriptive only.
- **Approval gate:** no paid call without named candidates + regimes + exact call count + explicit
  approval. New game runs remain 0 until that gate is passed.
- **Must remain unchanged during the batch:** prompt text, prompt selection, analyzer decisioning,
  confidence formula, advertised strength, evidence sufficiency, market agreement, buyer copy,
  calibration + metrics denominators.

## safety ledger

paid calls 0; new game runs 0; reconciliation writes 0 (verified via tests, not live DB); migrations 0;
prompt / prompt-selection / decision / buyer changes none; /metrics denominator byte-identical;
historical 7 rows NOT backfilled. code baseline DevCore.Api.Tests 1035 -> 1043 (all green).
