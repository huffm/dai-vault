---
title: "WI-0036 Settlement Strata and Realized-Position Writeback Implementation v1"
type: "evidence-report"
date: "2026-07-22"
status: "complete locally; independent review and integration pending"
project: "DAI"
slice: "WI-0036 Slice 3 remainder"
repos:
  dai: "local branch wi/0036-wildcard-settlement-strata-writeback from ce34a9e7; not pushed or integrated"
  dai-vault: "local branch wi/0036-wildcard-settlement-strata-writeback from 6e667b5c; not pushed or integrated"
tags:
  - system-development
  - evidence-operations
  - implementation
  - reconciliation
related:
  - "02 Platform/system-development/work-items/WI-0036-wildcard-evidence-discovery-loop-v1.md"
  - "02 Platform/architecture/cognitive-factory/wildcard-evidence-discovery-loop-v1.md"
  - "06 Execution/reports/wi-0036-wildcard-capture-flight-plan-implementation-2026-07-21-v1.md"
---

# wi-0036 settlement strata and realized-position writeback implementation v1

## result

The WI-0036 Slice-3 remainder is implemented on coordinated local branches and stops
before integration. A run row now records the realized frozen selection facts needed for
audit and settlement/reconciliation reads. The existing settlement-joined prompt-route
calibration row read exposes those facts with an explicit evidence stratum, preventing a
wildcard capture from silently reading as core evidence.

`flight-selection-provenance/1.0` and `wi0036-candidate-lane/1.0` are unchanged. No
cross-runtime vector changed. No prompt, decision, routing, confidence, lean, posture,
matching, buyer, retrieval, model, source, gateway, scheduler, or spend path changed.

## opening state and gates

- dai main and origin/main: `ce34a9e74659b42c71317267c64901a24ceb7091`;
- dai-vault main and origin/main: `6e667b5c2ce2feb7430d4ac8f33ab72bbf419aea`;
- coordinated branch: `wi/0036-wildcard-settlement-strata-writeback` in both repos;
- protected worktree hashes captured before edits and required byte-identical at close;
- path-portability map resolved through `<DAI_WORKSPACE_ROOT>`, `<DAI_REPO_ROOT>`, and
  `<DAI_VAULT_ROOT>`; both repo roots verified with git;
- skills gate selected slice-runner, system-development, test-discipline, code-review,
  docs-architect, vault-grill, agent-handoff, and token-tight instructions. The DAI skills
  were available as checked-in instruction files rather than callable session skills, so
  they were applied directly as the documented fallback.

## gap audit and ownership

The integrated provenance block already carries the producer-derived selection facts:
flight/freeze identity, lane and wildcard plan role, scheduled and realized positions,
realization mode, and optional substituted-for identity. The gap was persistence and a
stratified read, not a new producer contract. Therefore the 1.0 provenance and lane
contracts remain authoritative and unchanged.

The existing read seam is `PromptRouteCalibrationExporter`: it already joins run,
recorded outcome, market snapshot, and settlement provenance. `OutcomeReconciliationService`
owns matching and remains untouched under the slice's matching prohibition. The existing
aggregate metrics calculator also remains untouched; consumers that need core coverage
must use the row-level evidence stratum rather than pool rows.

## implemented contract

### realized-position writeback

Nine nullable columns were added to `AgentRun`:

- flight id and flight-plan freeze fingerprint;
- selection lane and optional wildcard plan role;
- scheduled position and realized position;
- `realizedVia` (`scheduled | core_reserve | wildcard_substitution`);
- optional substituted-for source provider and external event id.

`FlightSelectionRunWriteback.TryCreate` requires realized position and realization mode
for any run that carries flight-selection provenance. It copies the already validated
provenance fields directly; it never re-ranks, re-realizes, infers an identity, or grants
authority. Failure returns 400 before service invocation and before the pending run row
is written. Absence of provenance preserves the legacy all-null path and unchanged input
JSON behavior.

### settlement/reconciliation read strata

`PromptRouteCalibrationRow` gained trailing optional selection fields and the required
read classification `evidenceStratum`:

| persisted selection lane | evidence stratum |
|---|---|
| `core` | `core` |
| `reserve` | `core` |
| `wildcard` | `wildcard` |
| null or unknown | `unclassified` |

`reserve` stays a distinct selection lane and is classified as core evidence only because
it is a core-qualified backup. A wildcard substitute stays wildcard. Unknown or legacy
data is never silently promoted to core. Settlement state is still supplied by the
existing recorded-outcome join; no settlement fact is re-derived.

### schema migration

EF migration `20260722100648_AddAgentRunFlightSelectionWriteback` adds only the nine
nullable columns above and has a symmetric drop rollback. The migration and model
snapshot were generated mechanically. The migration was not applied.

## TDD evidence

The first focused .NET run was RED at compile time because the new run-row fields,
stratum classifier, and additive row fields did not exist. The permanent regressions then
went GREEN for:

- producer-generated wildcard-substitution and core-reserve vectors persisted with the
  exact realized slot and substituted-for identity;
- a pure scheduled-mode writeback projection with no substitution identity;
- missing `realizedPosition` or `realizedVia` rejected before the no-I/O service and run
  row;
- absent provenance preserving the legacy null writeback;
- settled core, reserve, wildcard, and legacy rows reading as core, core, wildcard, and
  unclassified respectively;
- an unknown lane reading as unclassified;
- all persisted selection facts surfacing through the settlement-joined row read.

Verification results:

- focused .NET seam: **37/37**;
- full DevCore.Api.Tests: **1536/1536** (integrated baseline 1530);
- full agent-service pytest through the repo virtual environment: **617/617**;
- `git diff --check`: clean before documentation closeout;
- no provenance/lane schema or vector diff;
- no changed-file warning was introduced (reported package/compiler/analyzer warnings are
  pre-existing and outside this slice).

## files changed

### dai

- `platform/dotnet/DevCore.Domain/Agentic/AgentRun.cs` -- nullable writeback fields;
- `platform/dotnet/DevCore.Data/AppDbContext.cs` -- column mappings;
- `platform/dotnet/DevCore.Data/Migrations/20260722100648_AddAgentRunFlightSelectionWriteback.cs`
  plus designer and model snapshot -- additive migration;
- `platform/dotnet/DevCore.Api/AgentRuns/FlightSelectionRunWriteback.cs` -- direct
  validated-provenance projection;
- `platform/dotnet/DevCore.Api/AgentRuns/EvidenceStrata.cs` -- closed read classifier;
- `platform/dotnet/DevCore.Api/Controllers/AgentRunsController.cs` -- fail-closed
  writeback before row creation;
- `platform/dotnet/DevCore.Api/AgentRuns/PromptRouteCalibrationExport.cs` -- additive
  settlement-joined stratum and position fields;
- focused exporter, writeback, and controller-host tests.

### dai-vault

- WI-0036 current state and Slice-3 disposition;
- wildcard architecture current-state and next-slice recommendation;
- system-development MOC and platform delivery timeline;
- this closeout report;
- append-only current-slice handoff.

## external activity and posture

No live source, model, Tool Gateway, AgentRun, capture, screening, settlement,
reconciliation, scheduler, observation, or paid-flight action occurred; paid cost was
**$0**. One read-only `dotnet ef migrations list` verification command attempted the
configured SQL connection and timed out; it applied no migration and performed no
database write. All implementation tests used in-process/no-I/O or in-memory paths.

The new columns and reads do not authorize execution. Every provenance authority ledger
remains the exact closed eight-key all-false set. The default-off wildcard posture is
unchanged.

## rollback and next action

Rollback is the local migration down operation plus reverting the two local slice commits;
because the migration was never applied and the branches are unintegrated, no external
state rollback is presently required.

Next action: independently review the complete coordinated branches. Review must probe
stratum non-pooling, all three realization modes, substituted-for identity fidelity,
legacy/unknown behavior, 400-before-row failure, migration shape, buyer exclusion, and
decision/matching non-drift. Only a later explicit prompt may authorize coordinated
fast-forward integration. Slices 4-6, the July 22 observation, and every paid wildcard
flight remain separately governed.

### Slice Synopsis

**Change:** Added nullable realized-selection writeback on `AgentRun` and an additive
settlement-joined row stratum that keeps core/reserve, wildcard, and unknown evidence
separate.
**Reason:** The integrated artifact seam preserved selection provenance but did not make
the realized slot queryable on the run row or prevent downstream reads from silently
pooling wildcard evidence with core evidence.
**Proof:** RED-first seam, focused 37/37, full .NET 1536/1536, pytest 617/617, unchanged
1.0 contracts and producer vectors, migration unapplied, $0.
**State:** Coordinated local branches only; not pushed, merged, integrated, or activated.
**Next:** Independent review, then separately authorized coordinated integration only if
the review passes.
