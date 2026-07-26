---
title: "WI-0037 Slice 2-ii-a Contract Conformance 2026-07-26 v1"
type: "evidence-report"
date: "2026-07-26"
status: "implemented local -- RED-first, all suites green; NOT reviewed, NOT integrated, NOT pushed"
project: "DAI"
slice: "WI-0037 Slice 2-ii-a: contract 1.1, xunit corpus runner, GameStatusResolver, structural bracket routing, frozen-state validation"
repos:
  dai: "code+tests+contract+corpus on local branch wi/0037-game-state-correctness-slice-2-ii-a (base dd760f9); NOT integrated"
  dai-vault: "docs on local branch wi/0037-game-state-correctness-slice-2-ii-a (base 59f32e4); NOT pushed"
tags:
  - system-development
  - sports-v1
  - correctness
  - implementation
related:
  - "02 Platform/system-development/work-items/WI-0037-game-state-correctness-v1.md"
  - "06 Execution/reports/wi-0037-slice-2-ii-architecture-review-2026-07-26-v1.md"
---

# wi-0037 slice 2-ii-a contract conformance 2026-07-26 v1

## opening, planning publication, workspaces

Gate 2026-07-26 14:42Z: dai main == origin == direct remote == `dd760f9`; vault main
== origin == direct remote == `642d8f2`; planning tip `59f32e4` (parent `642d8f2`,
docs-only, local); ops branch `642d8f2` clean/0-unique; wi/0035 preserved worktree
six paths, hash `86aa8b74` (unchanged at close). Planning commit `59f32e4` reviewed
(docs-only; citations re-verified; supplementary sports-app sweep consistent -- no
material omission per the review rule) -> **SLICE2II_PLANNING_REVIEW_PASS** ->
PUBLISHED UNCHANGED: vault main -> `59f32e4c151b1c0de762346454e27b5470090f1b`
(remote verified; only main pushed). Ops branch ff `642d8f2 -> 59f32e4` (not
pushed). Slice branches `wi/0037-game-state-correctness-slice-2-ii-a`: dai base
`dd760f9`, vault base `59f32e4`.

## pre-edit map (verified against source)

Response shape `MlbScheduleResponse{Dates[{Date,Games}]}` retains bucket identity
until the flattens at `MlbStarterClient.cs:278` (single) and `:338` (batch) discard
it; `MlbEventResolver.Resolve(games, home, away, pk?)` has no bracket input; frozen
`ScheduleState` reaches `NormalizeScheduleState` unvalidated (JSON null -> NRE); no
xunit corpus consumer existed; test csproj had no content links. Proposed files and
types matched the published architecture -- no contradiction.

## red evidence (all preserved in session scratch, captured BEFORE any production edit)

- **RED A** (scalar dates, current CLI): `{ "dates": "not-an-array" }` ->
  `reason=bracket_missing` exit 1; expected 1.1 `bucket_malformed`.
- **RED B** (cross-bucket reachability): new characterization test
  `batch_entry_reports_the_bracket_buckets_state_not_the_first_buckets` against the
  two-bucket 823042-shaped payload -> `Expected "Final" / Actual "Postponed"` --
  the flattened exact-pk path observes the out-of-bracket postponed record first.
- **RED C** (frozen-state null): slate JSON with `"scheduleState": null` through the
  REAL System.Text.Json deserialization seam + `RunPreflightAsync` ->
  `System.NullReferenceException` (uncontrolled); expected typed rejection.
- **RED D** (structural): no corpus runner file existed; zero references to
  `contractVersion`/`game-status-resolution` in C# tests; no version gate anywhere.

## contract 1.1 and corpus migration

`game-status-resolution-contract-v1.md` -> **1.1**: absent/null/non-array `dates`
is normatively `bucket_malformed`; `bracket_missing` reserved for a valid array
lacking the requested bracket; schema-compatible minor version (same shapes, same
six-reason vocabulary); accepted-version discipline documented (runners fail loudly
on unknown versions, never silently reinterpret). Corpus: `contractVersion` 1.1;
**25 fixtures** (+`gsr-25` scalar-dates -> bucket_malformed, consumers
operator+csharp_resolver); `csharp_resolver` tags added ADDITIVELY to all fixtures;
no second checked-in copy anywhere.

## powershell 1.1 conformance

`check-game-status.ps1` stage 1 now refuses absent/null/non-array `dates` (string
or non-enumerable container) as `bucket_malformed`; version literals -> 1.1;
everything else untouched (exit table, JSON purity, single resolution path, no F4
URL change, no F3 finals-harness change -- that file only carries the version
literal in its comment). Harness: version 1.1 asserted, 25 fixtures, gsr-25
executes, corpus-driven expectations, unknown versions rejected.

## c# implementation

- **`GameStatusResolver.cs`** (new, `DevCore.Api/Sports`): staged contract stages
  1-5 over the bucketed response; closed `GameStatusRefusal` enum mapping 1:1 to
  the vocabulary (`ReasonCode` renders snake_case; never stringly created);
  deterministic context ordering (bucket date, ordinal); `SelectBracketGames`
  (stages 1-2) for matchup-resolution callers; `NormalizeScheduleState` with the
  identical alias vocabulary; **`requireStatus` consumer flag** -- contract stage 5
  requires "the status fields the consuming operation needs": status-resolution
  consumers demand a detailed state, identity-retrieval consumers (starter
  grounding, whose pinned fixtures legitimately omit `status`) pass
  `requireStatus: false` and the adapter owns state gating downstream. No
  execution authority, no network, no clock.
- **`MlbStarterClient.cs`**: single (:278 region) and batch (:338 region) flattens
  REMOVED; the fetch's `date=` value is the explicit bracket input;
  pk paths: `GameStatusResolver.Resolve(..., requireStatus: false)` ->
  resolved => `MlbEventResolver.Resolve([game], home, away, pk)` (the exactly-one
  candidate through the smallest collection adapter -- matchup validation only,
  never a second bracket authority); `DuplicateInBracket` => fail-closed ambiguity
  diagnostics via the in-bracket duplicates WITHOUT a pk (never first-match);
  other refusals => empty candidate set (the exact pre-existing fail-closed
  NoMatch/IdentityMismatch shapes -- malformed input is never converted to a
  silent success and no new wire strings exist). no-pk path:
  `SelectBracketGames` + unchanged `MlbEventResolver` ambiguity semantics.
  External caller signatures UNCHANGED.
- **`MarketContrastSourceAdapter.cs`**: slate validation adds
  `"candidate schedule state is required"` for null/absent/empty/whitespace frozen
  state (typed invalid input, outside the gsr vocabulary; unrecognized NON-EMPTY
  values keep the pinned normalize-to-unknown path). One Slice-1 matrix row
  intentionally migrated: the empty-state row became a non-empty unknown alias
  ("TBD"), since empty is now request-level invalid input (documented in-test).
- **`GameStatusResolutionCorpusTests.cs`** (new): phase-1 corpus schema validation
  hard-fails at load (unknown version, malformed corpus) before any behavioral
  result; phase-2 generic conformance over every `csharp_resolver` fixture
  (fixture id in every failure; a no-skip test proves tag-selection completeness);
  targeted DH-independence and version-gate facts; corpus linked via csproj
  `Content` from `scripts/dev/sports/fixtures/` (single source of truth). One
  documented runtime nuance: a container that cannot bind to the typed array is
  malformed input (deserialization failure -> null -> `bucket_malformed`),
  matching the dynamic-runtime classification.
- **`StarterClientBracketAuthorityTests.cs`** (new): RED B/C now green + the
  routing proofs.
- csproj: the corpus content link (no copy).

## verification (exact results)

- full `DevCore.Api.Tests`: **1811 passed / 0 failed / 0 skipped** (baseline 1780 +
  31 new; ~2s). All 24 pinned doubleheader scenarios green through the new
  bracket-authority routing.
- operator harness: **187 passed / 0 failed** (was 181; 1.1 + gsr-25 vectors).
- finals harness: **40 passed / 0 failed** (count unchanged; only a comment version
  literal changed in that file -- F3 untouched).
- `git diff --check` clean; machine-path/secret scans clean; zero live calls; zero
  db; zero dependency changes; excluded-path scan clean (no OddsScheduleClient,
  MatchupEventDto, TypeScript, frontend, batch-duplicate, F3, F4, migration,
  reconciliation paths).

## scope

Exactly 12 dai paths: contract, corpus, CLI + its harness, finals harness
(version literal only), GameStatusResolver (new), MlbStarterClient, adapter,
adapter tests (matrix row), corpus runner (new), bracket-authority tests (new),
test csproj. Within the planning estimate.

## commits and branch state

- dai: **`f8c0962`** "feat(sports): enforce bracketed game-status resolution"
  (12 files, +580/-50), base `dd760f9`, local only, NOT pushed, NOT integrated.
- dai-vault: one docs commit on the matching branch (this record + spec state +
  handoff), base `59f32e4`, local only.

## residual risks

`requireStatus` is a deliberate consumer-scoped stage-5 application (documented
here and in code) -- the adversarial review should judge it against the contract's
"fields the consuming operation needs" clause; the adapter retains its private
normalizer copy (shared-normalizer harmonization is a 2-ii-c candidate); Slice-1's
empty-state matrix row migration is an intentional, documented behavior change
(empty now rejects the request instead of pre-eliminating the candidate).

## next governed action

Independent adversarial review of the 2-ii-a chain, then a separate integration
authorization. Slices 2-ii-b and 2-ii-c remain unauthorized.
