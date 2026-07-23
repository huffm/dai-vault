---
title: "Provider-Event Binding Slice C -- Verbatim Binding to Flight Provenance + WI-0036 Board-2.3 Migration 2026-07-22 v1"
type: "evidence-report"
date: "2026-07-22"
status: "Slice C implemented locally and green on both runtimes; UNINTEGRATED and unavailable to live workflow; authorizes no flight or runtime use; execution retrieval remains a separate open Slice-D gap"
project: "DAI"
slice: "WI-0035 provider-event binding vertical, Slice C (verbatim binding to flight plan + provenance; WI-0036 atomic migration to board 2.3; compat-seam retirement)"
repos:
  dai: "local branch wi/0035-provider-event-binding; NOT pushed"
  dai-vault: "local branch wi/0035-provider-event-binding; NOT pushed"
tags:
  - evidence-operations
  - sports-v1
  - market-contrast
  - wildcard-flight
  - implementation
related:
  - "06 Execution/reports/provider-event-binding-slice-b-2026-07-22-v1.md"
---

# provider-event binding slice C -- verbatim binding to flight provenance and WI-0036 board-2.3 migration

## what is now true

The Slice-B-validated `provider_event_binding` now propagates VERBATIM from the board (2.3)
into the WI-0036 flight plan and the flight-selection provenance, and the entire WI-0036
flight vertical has been MIGRATED off the frozen board-2.2 compatibility seam onto the live
board 2.3. The frozen `daily_evidence_planner_compat_v22` reproducer is **deleted**.

## the atomic migration invariant (satisfied)

These formed one compatibility unit and migrated from board 2.2 to board 2.3 **together, in
one dai commit**: the board reproduction source, the board schema version accepted by the
flight planner, the validated binding, the replay reference, the flight-plan schema, the
flight-selection provenance schema, C# and Python serialization, the cross-runtime vectors,
the CLI behavior, and the tests and fixtures. No committed state reproduces or accepts board
2.3 while retaining board-2.2 flight semantics; there is no half-versioned intermediate.

## version cascade

| contract | from | to |
|---|---|---|
| wildcard-flight-plan | 1.1 | **1.2** (each candidate row carries the verbatim binding + replay reference) |
| wildcard-flight-planner | 1.1 | **1.2** |
| flight-selection-provenance | 1.0 | **1.1** (carries the verbatim binding + replay reference) |
| flight board consumed | daily-evidence-board/2.2 (frozen) | **daily-evidence-board/2.3** (live) |

Unchanged: `wildcard-flight-plan-request/1.1`, `wildcard-flight-realization/1.1`,
`wi0036-candidate-lane/1.0`, and the lane/signal/novelty registries.

## verbatim canonical byte authority

The binding carried into the flight plan and provenance is the **exact canonical binding
object the board validated** -- never a projection, reconstruction, or reserialization that
alters its canonical bytes. The authoritative "canonical binding byte sequence" is
`peb.canonical_wire_json(binding)`. A consumer may parse the binding for validation, but the
canonical bytes remain authoritative for fingerprint verification and cross-runtime parity.
Proven: the propagated bytes equal the board bytes; the propagated binding strictly validates
and its fingerprint verifies; and mutating any propagated content -- a property-name mutation,
a scalar-value mutation, or a fingerprint mutation -- fails closed.

## bound vs unbound (precise rule)

- an explicitly UNBOUND candidate (e.g. a market-absent wildcard with no market-contrast
  screen) remains unbound: `provider_event_binding` and `replay_reference` are null, and
  nothing is fabricated;
- a provider-BOUND candidate produces a bound plan row only when both a strictly valid binding
  AND the board replay reference are present;
- a candidate that structurally indicates binding but is malformed, or is bound without a
  replay reference, FAILS CLOSED (`BOUND_CANDIDATE_INVALID_BINDING` /
  `BOUND_CANDIDATE_MISSING_REPLAY_REFERENCE`); absence of binding data never silently
  downgrades a malformed bound candidate to unbound.

The two cross-runtime vectors exercise both cases: `CROSS_RUNTIME_VECTOR` (824766) is an
unbound wildcard substitution (null binding); `RESERVE_SUBSTITUTION_VECTOR` (824004) is a
bound core reserve carrying the full verbatim binding and replay reference.

## historical board-2.2 rejection contract

The migrated flight planner consumes board 2.3 only. A historical board 2.2 (or any non-2.3
board) is rejected with a stable, machine-readable `UNSUPPORTED_BOARD_SCHEMA_VERSION` error
carrying structured context: `expected_schema_version = daily-evidence-board/2.3`,
`actual_schema_version`, and `consumer = wildcard_flight_plan`. There is no fallback to the
retired reproducer. Tests assert the code and the structured fields, not prose.

## compat-seam retirement

`daily_evidence_planner_compat_v22.py` and its characterization test are **deleted**. Scans
prove no active import, call, alias, dynamic loader, CLI switch, fallback branch, or
`CONSUMED_BOARD_SCHEMA_VERSION` pin remains; a documentation comment may mention the retired
name, but no active code line references it. Dated historical reports (Slice B) that mention
board 2.2 remain as history and are not rewritten.

## authorized provenance boundary

The binding and replay reference are permitted only on the board artifact, the internal
flight plan, and the flight-selection provenance (plus replay/parity fixtures). A leakage
scan confirms the binding payload, replay reference, provider identifiers, and fingerprint do
NOT appear in any buyer-facing surface, the buyer decision brief, `SportsRetriever`,
`OddsMarketClient`, `MarketSpreadInput`, `MarketSnapshot`, the Tool Gateway handlers, or any
capture/execution/settlement surface. Slice C introduced no execution-contract change.

## verification

| gate | result |
|---|---|
| full `DevCore.Api.Tests` | **1746/1746** |
| full agent-service pytest | **691/691** |
| WI-0036 flight suites (core + CLI, now on board 2.3) | **54/54** |
| C#/Python flight-provenance cross-runtime vectors | byte-identical (bound + unbound) |
| C#/Python binding + bracket vectors | byte-identical (unchanged from Slice B) |
| fresh-process determinism | flight + binding suites 77/77 on rerun |
| four execution-gap tests | **4/4 green, explicitly unresolved** |
| strict planning snapshot | 25 work items / 6 timeline entries / **0 warnings** |
| `git diff --check` both repos | clean |
| protected hash `DevCore.Data.csproj` | `63EF2488...` unchanged |
| compat removal | module + test deleted; no active reference remains |
| leakage scan | no binding/replay token on buyer or execution surfaces |
| stale-reference scan | no `compat_v22` / `v22` / `CONSUMED_*` in active code |

RED evidence: the flight suites failed against `6d4cd32` before the migration (board-2.2
consumption, flight/provenance schema 1.1/1.0, no binding fields, no structured historical
rejection). The cross-runtime vectors and the binding-propagation proofs were regenerated
from the real exporter and pinned on both runtimes.

## what remains unsafe and unfinished

Execution retrieval is untouched and remains a separate open **Slice D** gap: team/date
fallback, doubleheader mis-binding, no by-ID re-verification, and no `MarketSpreadInput`
binding member. Its four characterization tests remain green and explicitly unresolved --
green characterization does not mean the gap is corrected. Slice D carries the binding to the
execution boundary and enforces independent by-ID re-verification.

## posture

No StatsAPI, Odds `/events`, Odds `/odds`, model, Tool Gateway, or other network call. No
paid call, AgentRun, capture, flight freeze, settlement, reconciliation, scheduling, or
activation. No database access, migration, or service start. No integration, push, PR, remote
branch, or history rewrite. **$0.** Slice C is local, unintegrated, and authorizes no flight
or runtime use. The gamePk `823438` canary remains untouched historical pre-binding evidence.
