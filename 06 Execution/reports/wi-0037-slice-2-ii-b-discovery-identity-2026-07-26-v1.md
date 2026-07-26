---
title: "WI-0037 Slice 2-ii-b Discovery Identity 2026-07-26 v1"
type: "evidence-report"
date: "2026-07-26"
status: "implemented local -- RED-first, all suites green; NOT reviewed, NOT integrated, NOT pushed"
project: "DAI"
slice: "WI-0037 Slice 2-ii-b: provider identity preservation, doubleheader-safe discovery, matchup representation, atomic batch validation"
repos:
  dai: "code+tests+frontend on local branch wi/0037-game-state-correctness-slice-2-ii-b (base 841ae26); NOT integrated"
  dai-vault: "docs on local branch wi/0037-game-state-correctness-slice-2-ii-b (base 664cd4a); NOT pushed"
tags:
  - system-development
  - sports-v1
  - correctness
  - implementation
related:
  - "02 Platform/system-development/work-items/WI-0037-game-state-correctness-v1.md"
  - "06 Execution/reports/wi-0037-slice-2-ii-architecture-review-2026-07-26-v1.md"
---

# wi-0037 slice 2-ii-b discovery identity 2026-07-26 v1

## opening and pre-edit map

Gate 2026-07-26 17:59Z: dai main == origin == direct remote == `841ae26`; vault main
== origin == direct remote == `664cd4a`; ops branch `664cd4a` clean/0-unique; wi/0035
preserved worktree six paths, hash `86aa8b74` (unchanged at close). Branches
`wi/0037-game-state-correctness-slice-2-ii-b` (dai base `841ae26`, vault base
`664cd4a`), dedicated worktrees. Data-flow map verified against source: odds json ->
`OddsApiEvent{id, commence_time, home, away}` -> pre-fix projection DISCARDED id and
commence -> `MatchupEventDto(Date, Home, Away)` -> handler/controller passthrough ->
TS mirror (`agent-run.model.ts:24`; embedded `game:` at `:85`) ->
`sports-api.service.ts` (+ stub generator) -> analyzer (`dates` computed `:97`, pills
`@for track d` html `:112`, `selectDate` find-first-by-date `:500`). Batch path:
`GetStartersForDateBatchAsync` had no duplicate-input validation; `byGamePk[gamePk]=`
is last-write. **Existing Date semantics: DATE_IS_EASTERN_OPERATIONAL_DATE**
(ET conversion, comment `OddsScheduleClient.cs:41-42`) -- preserved unchanged.

## red evidence (preserved BEFORE any production edit)

C# RED run: exit 1, **11 failures** (`2iib-red2.txt`): RED A pair-path DH collapse
(2 ids -> 1 row via `DistinctBy(Date)` at `:223`); RED B sampler collapse (`:127`);
RED C representation loss (`MatchupEventDto` has no `ProviderEventId`/`StartUtc`);
RED D duplicate batch input `[1001,1001]` and `[1001,1002,1001]` both proceeded (no
typed rejection, schedule request made, last-write dictionary); plus the id-contract
cases (conflicting same-id, blank id, distinct-ids-identical-fields, deterministic
ordering) all failing pre-fix. RED E frontend distinguishability (static analysis,
file:line): pills tracked BY DATE STRING (duplicate track keys on a DH day),
selection find-first-by-date, TS model without identity fields, bare 3-field literal
in `review-feedback.spec.ts:19`.

## provider identity contract (implemented)

Within odds provider scope, `OddsApiEvent.id` is the AUTHORITATIVE identity (this
surface has no mlb gamePk; date/teams/commence/array-position rejected as
authoritative keys). Shared `NormalizeEvents` pipeline now feeds BOTH discovery
paths:

- **blank/whitespace id** -> malformed provider event, FAIL CLOSED (omitted; never
  date/team-deduped; structured warn log with home/away/commence -- no payload dump);
- **same id, equivalent payload** (equivalence over id+home+away+utc commence) ->
  one row, deterministic coalescing;
- **same id, CONFLICTING payload** -> provider-integrity failure: the WHOLE id is
  excluded (never first/last selection, never a fabricated canonical event),
  structured warn log, deterministic and order-independent (pinned both orders);
- **distinct ids** -> always distinct rows, even with identical teams/date/commence
  (legitimate doubleheader).

Ordering: eastern `Date`, then `StartUtc`, then `ProviderEventId` (ordinal);
source-order independence pinned by a reversed-input test.

## representation (additive)

`MatchupEventDto(Date, HomeTeam, AwayTeam, StartUtc, ProviderEventId)` --
StartUtc = exact provider commence instant, UTC-normalized `yyyy-MM-ddTHH:mm:ssZ`;
existing fields and JSON conventions unchanged; handler/controller passthrough
(zero changes needed). TS mirror updated with REQUIRED `startUtc` +
`providerEventId` (backend guarantees both); embedded `game: MatchupEventDto` at
`agent-run.model.ts:85` flows through; the bare literal in
`review-feedback.spec.ts` updated.

## analyzer distinguishability

Pills now iterate EVENTS (`@for (ev of matchupEvents(); track ev.providerEventId)`),
selection via `selectEvent(ev)` keyed by `providerEventId` (never date/index/teams);
`aria-pressed`/selected styling via `isSelectedEvent`; labels via `eventLabel` --
formatted date, plus localized start time ONLY when the date is shared by more than
one event (provider id never used as display text); dead `selectDate` and the
one-per-day `find(e => e.date === date)` REMOVED. Stub generator emits deterministic
`startUtc`/`providerEventId` and a day-3 stub DOUBLEHEADER (same date/teams,
distinct ids/times) so dev mode exercises the path.

## atomic batch validation

`GetStartersForDateBatchAsync` validates the full gamePk collection at the entrance,
BEFORE the ledger's first schedule attempt, any network call, cache activity,
per-game resolution, or dictionary population: any repeated gamePk -> typed
rejection `duplicate_gamepk_batch_input` (internal const, surfaced through the
existing `MlbStarterBatchResult.Failure` channel -- deliberately NOT a provider
source failure semantically; documented in code), structured warn log naming the
sorted duplicated pks, EMPTY result map, zero requests (pinned). `[1001,1002]`
valid; `[1001,1001]` and `[1001,1002,1001]` rejected; order-independent. Distinct
doubleheader pks unaffected (pinned). Last-write behavior is unreachable.

## verification (exact)

- C# focused (first-ever OddsScheduleClientTests + batch): **14/14** covering both
  production paths through the real HTTP/deserialization seam (normal event, DH
  survival x2 paths, identical-fields distinct ids, same-id coalesce, same-id
  conflict x2 orders, blank id x2, deterministic ordering + reversed source, empty
  response, http error, duplicate-batch x2, DH-batch-valid).
- full .NET: **1831 passed / 0 failed / 0 skipped** (baseline 1817 + 14 new); all
  2-ii-a corpus/resolver/bracket tests and the 24 pinned DH scenarios green.
- operator harness: **187/187**; finals harness: **40/40** (both untouched-green;
  no PowerShell file changed).
- frontend: `npm ci` (lockfile install only -- no dependency-version change), then
  vitest **136 passed / 0 failed (14 files)** incl. the new
  `matchup-event-distinguishability.spec.ts` (+2), and `npm run build` SUCCESS.
- `git diff --check` clean; machine-path/secret scans clean; zero live odds or
  statsapi calls; zero db/paid/model calls.

## scope

Exactly 9 changed paths: `OddsScheduleClient.cs`, `MlbStarterClient.cs`,
`OddsScheduleClientTests.cs` (new), `agent-run.model.ts`,
`sports-api.service.ts`, `analyzer.component.ts`, `analyzer.component.html`,
`review-feedback.spec.ts`, `matchup-event-distinguishability.spec.ts` (new).
UNTOUCHED (verified): GameStatusResolver, GameStatusPayloadReader,
MlbEventResolver, contract doc, fixture corpus, every PowerShell script, F3, F4,
adapter normalizer, requireStatus, reconciliation/settlement/db/schemas/migrations,
provider-event binding, planner, dependency versions.

## commits and branch state

- dai: **`5a11a2c`** "fix(sports): preserve provider game identity" (9 files,
  +363/-42), base `841ae26`, local only, NOT pushed, NOT integrated; tree clean.
- dai-vault: one docs commit on the matching branch (this record + spec state +
  handoff), base `664cd4a`, local only.

## residual risks

The provider-integrity exclusion (conflicting same-id -> whole id dropped) surfaces
only through logs on this reference surface -- acceptable here (reference/display
consumers; no settlement authority), flagged for the reviewer; the analyzer start-
time suffix uses the viewer's locale for display only (identity remains the id);
frontend component-DOM-level DH rendering is pinned at the model/selection level
plus compilation/build (the app has no analyzer component DOM spec seam -- smallest
permitted test added; noted per authorization).

## next governed action

Independent adversarial review of the 2-ii-b chain, then a separate integration
authorization. Slice 2-ii-c (F3, F4, requireStatus hardening, normalizer
consolidation, parity vectors) remains unauthorized and is the final slice before
WI-0037 closure.
