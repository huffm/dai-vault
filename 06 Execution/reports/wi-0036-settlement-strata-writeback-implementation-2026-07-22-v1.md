---
title: "WI-0036 Settlement Strata and Realized-Position Writeback Implementation v1"
type: "evidence-report"
date: "2026-07-22"
status: "complete, independently verified, and INTEGRATED 2026-07-22 (dai main 48a2931, dai-vault main fa31a3e, each == origin/main; migration generated but NOT applied)"
project: "DAI"
slice: "WI-0036 Slice 3 remainder"
repos:
  dai: "code+tests+migration INTEGRATED to main 48a29313988beac34cb8dad388371c472ff21d73 (migration generated, NOT applied)"
  dai-vault: "docs INTEGRATED to main fa31a3e91a09753602e34ca8cee3d38e059d9dd0"
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

*(Superseded 2026-07-22 by the addendum below: the next action was an independent review
of the coordinated branches, probing stratum non-pooling, all three realization modes,
substituted-for identity fidelity, legacy/unknown behavior, 400-before-row failure,
migration shape, buyer exclusion, and decision/matching non-drift, with integration
gated behind a later explicit prompt. That review PASSED and integration executed.)*

## addendum 1 -- independently verified and integrated (2026-07-22)

**Independent verification: PASS. No correction commit required.** The review was
performed against live repository state rather than this report, and re-derived from
source: commit ancestry and `WI: WI-0036` trailers; protected-state hashes; that
`Validate` then `TryCreate` both precede `db.AgentRuns.Add` and `SaveChangesAsync`; the
closed fail-closed strata mapping (`core`/`reserve` -> core, `wildcard` -> wildcard, and
`null`/unknown/`excluded`/`blocker` -> unclassified); that `Shape` runs in memory so the
mapping carries no EF-translation hazard; absence of any Flight/Lane/Stratum member on the
buyer surface; migration symmetry (9 nullable additions / 9 removals) with all nine
properties in the model snapshot; that `FlightSelectionProvenance.cs` and
`services/agent-service` are untouched; and that the additive export keys are inert to the
python consumer, which reads exclusively via `.get()` (the C#-side superset is the
pre-existing pattern -- market, settlement, and attribution fields were already absent
from `CALIBRATION_FIELDS`). Suites re-run independently: **1536/1536** .NET and
**617/617** pytest; strict snapshot 25/6/0; `diff --check` and scans clean; build
warning-free on every changed file.

**Operator-accepted semantics (binding; recorded so a future reviewer does not reopen
them as unresolved defects):**

1. **An AgentRun is an executed realization.** Provenance supplied at AgentRun creation
   must carry `realizedPosition` and `realizedVia`; omission returns 400 before service
   invocation and before any run row. Plan-time provenance (exported without availability)
   remains valid for non-run uses. This is a deliberate endpoint-context requirement, not
   an incidental side effect, and does NOT change `flight-selection-provenance/1.0`.
2. **Scheduled-mode projection coverage is sufficient for this slice.** The writeback
   performs no `realizedVia`-specific branching, while genuine producer vectors cover
   `core_reserve` and `wildcard_substitution` end-to-end through the controller host. A
   producer-generated scheduled vector is a non-blocking future test enhancement, not an
   integration prerequisite.

**Integration executed** by coordinated fast-forward, `--ff-only`, ordinary non-force
pushes, no merge commit and no history rewrite:

- dai: `ce34a9e7 -> 48a29313988beac34cb8dad388371c472ff21d73` (`main -> main`)
- dai-vault: `6e667b5c -> fa31a3e91a09753602e34ca8cee3d38e059d9dd0` (`main -> main`)

Post-fetch proof: local `main` == `origin/main` == the verified tip in both repositories.
Both pushes succeeded on first attempt; no asymmetric published state occurred. Protected
state byte-identical throughout. The migration
`20260722100648_AddAgentRunFlightSelectionWriteback` remains **generated but NOT applied**;
no database-dependent command was run and no database write occurred. Repository
publication granted **no runtime or commercial authority**; **$0**.

Next action: none authorized. Applying the migration and any Slice-4 decision each require
their own explicit operator authorization. Slices 4-6, the July 22 observation, and every
paid wildcard flight remain separately governed.

### Slice Synopsis

**Change:** Added nullable realized-selection writeback on `AgentRun` and an additive
settlement-joined row stratum that keeps core/reserve, wildcard, and unknown evidence
separate.
**Reason:** The integrated artifact seam preserved selection provenance but did not make
the realized slot queryable on the run row or prevent downstream reads from silently
pooling wildcard evidence with core evidence.
**Proof:** RED-first seam, focused 37/37, full .NET 1536/1536, pytest 617/617, unchanged
1.0 contracts and producer vectors, migration unapplied, $0.
**State:** Independently verified (PASS, no correction commit) and INTEGRATED by
coordinated fast-forward -- dai main `48a2931`, dai-vault main `fa31a3e`, each ==
origin/main; migration generated but NOT applied; nothing activated.
**Next:** None authorized. Applying the migration and any Slice-4 decision each require
their own explicit operator authorization.
