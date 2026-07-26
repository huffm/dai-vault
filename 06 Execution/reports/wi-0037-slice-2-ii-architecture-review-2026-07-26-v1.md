---
title: "WI-0037 Slice 2-ii Architecture Review 2026-07-26 v1"
type: "evidence-report"
date: "2026-07-26"
status: "architecture bound -- three sub-slices (2-ii-a/b/c) selected; contract 1.1 required; NO implementation authorized"
project: "DAI"
slice: "WI-0037 Slice 2-ii pre-implementation architecture and contract-binding review"
repos:
  dai: "unchanged (read-only inspection at main dd760f9)"
  dai-vault: "docs-only; local branch wi/0037-game-state-correctness-slice-2-ii-planning from 642d8f2"
tags:
  - system-development
  - sports-v1
  - correctness
  - architecture
related:
  - "02 Platform/system-development/work-items/WI-0037-game-state-correctness-v1.md"
  - "06 Execution/reports/wi-0037-slice-2-status-resolution-design-2026-07-24-v1.md"
  - "06 Execution/reports/wi-0037-slice-2-i-status-resolution-2026-07-26-v1.md"
  - "06 Execution/patterns/game-status-recheck-discipline-v1.md"
---

# wi-0037 slice 2-ii architecture review 2026-07-26 v1

## 1. opening truth and preservation

Gate 2026-07-26: dai main == origin == direct remote == `dd760f9` (clean; contract
`game-status-resolution/1.0`, 24 fixtures, six-reason vocabulary verified on the
tip); vault main == origin == direct remote == `642d8f2` (S1 closed, S2-i closed,
S2-ii defined/unauthorized, WI-0037 in-progress); ops branch realigned at `642d8f2`
(clean, 0 unique, untouched); wi/0035 preserved worktree six dirty paths, hash
`86aa8b74`. No source changed after the S2-i closeout; ancestry linear.

## 2. authority set and requirements matrix

Read in full: WI-0037 spec, S2 design, S2-i execution + adversarial review + F1
correction + closeout, doctrine, contract, all 24 fixtures, both production scripts,
both harnesses. Requirements matrix (requirement / authority / contract clause /
fixtures / PS implementation / remaining C# surface):

| # | requirement | authority | contract | fixtures | PS today | remaining C# |
|---|---|---|---|---|---|---|
| R1 | xunit corpus consumption | S2 design (option B) | whole contract | all applicable | harnesses consume | runner absent |
| R2 | C# staged conformance | design D-invariant | stages 1-5 | gsr-01..24 | guard+CLI conform | resolver path unbound |
| R3 | resolver date-scope enforcement | design G2 | stages 2-4 | gsr-04..10 | n/a | MlbEventResolver callers flatten buckets |
| R4 | DH-safe discovery | design D2 | identity principle | gsr-02/03 analog | n/a | OddsScheduleClient :127/:223 |
| R5 | batch duplicate boundary | design note | stage 4 analog | gsr-08/24 analog | n/a | GetStartersForDateBatchAsync last-write |
| R6 | frozen-state validation | design section 8 | slate contract (NOT gsr vocab) | none (slate-side) | n/a | adapter ValidationError |
| R7 | F2 container classification | S2-i review F2 | stage 1 (ambiguous) | gsr-11 family | bracket_missing today | convergence needed |
| R8 | F3 harness tally | S2-i review F3 | n/a | n/a | finals harness detail strings | none |
| R9 | F4 live context semantics | S2-i review F4 | context clause | gsr-04..07 | CLI live URL date=+gamePks= | none |
| R10 | WI closure boundary | this review | -- | -- | -- | -- |

## 3. c# runtime call graph (verified against source at dd760f9)

StatsAPI transport (`MlbStarterClient` HttpClient) -> deserialization
(`MlbScheduleResponse { Dates: MlbScheduleDate[] { Date, Games } }` -- the bucket
date IS already modeled) -> **flatten** (`:278` single, `:338` batch --
`SelectMany(d => d.Games)`; bucket identity discarded HERE) ->
`MlbEventResolver.Resolve(games, home, away, pk)` (`:64` exact-pk FirstOrDefault
over the FLAT list; fail-closed, ambiguity-refusing, 24 pinned scenarios; no date
input) -> grounding (`GroundFromResolutionAsync :359-450`) -> consumers
(`MarketContrastSourceAdapter` pre-elimination + facts; `SportsRetriever`;
starters tool). Separate odds-reference chain: the-odds-api ->
`OddsScheduleClient` (`OddsApiEvent { id, commence_time, home, away }`) ->
`MatchupEventDto(Date, HomeTeam, AwayTeam)` (provider event id and start DISCARDED)
-> `ScheduleMatchupDatesHandler` -> `SportsReferenceController.GetMatchupDates
:142` -> sports-app `sports-api.service.ts`. Boundary notes: duplicate input pks to
the batch are last-write-wins on the caller key (`:341-350`), shielded today only
by the adapter's slate validation; frozen `ScheduleState` is an unvalidated
non-nullable string (JSON null -> NRE in `NormalizeScheduleState`).

## 4. fixture-to-surface binding

The corpus `consumers` field currently carries `operator` / `finals_guard`. It is
runtime-neutral but SURFACE-vocabulary-incomplete for C#. Decision: extend the
values ADDITIVELY in the 1.1 bump with `csharp_resolver` (and keep generic binding
-- the xunit runner selects fixtures by consumer tag, no manual switch, no skipping
without an explicit tag). Binding: gsr-01..19, 22 -> `csharp_resolver` (staged
resolution + identity); gsr-20/21/23/24 -> guard-semantics vectors, `csharp_resolver`
applicable for resolution outcome only (READY/DEFECT mapping stays PS-side);
gsr-13 additionally pins ET-bracket semantics; no fixture is legitimately
PS-only except exit-code assertions, which live in harness logic, never in the
corpus (verified). No duplicated C# constants, no rewritten copies, no alternate
outcomes permitted.

## 5. xunit runner architecture

Location: `platform/dotnet/DevCore.Api.Tests/Sports/GameStatusResolutionCorpusTests.cs`
(+ `GameStatusResolutionModels` if not production-shared). Corpus file consumed
from the repo path via link/copy-to-output (`<Content Include>` of
`scripts/dev/sports/fixtures/game-status-resolution-v1.json`, CopyToOutputDirectory
PreserveNewest) -- single source of truth, no checked-in copy. Deserialization:
System.Text.Json, camelCase, `long` gamePk, nullable reference-annotated expected
model (`string? reason`), status as string (matching NormalizeScheduleState
output; no enum so the vocabulary stays contract-owned). Runner phases distinguish
(1) corpus schema validation (version == expected, 24+ unique ids, refusals within
vocabulary -- hard fail on unknown contract version or malformed entry BEFORE
behavioral tests via a fixture-source theory that throws), (2) contract conformance
(staged resolver over each payload -> disposition/reason/bucket/context-count
equality, fixture id in the display name), (3) surface-specific behavior (adapter/
batch tests reference fixtures by id where applicable). PowerShell exit codes never
appear in C# expectations. Future versions: runner pins an accepted-versions set;
a corpus bump outside the set fails loudly.

## 6. contract-version decision: V1_1_REQUIRED

Contract 1.0 stage 1 says "`dates` must be an array" but assigns NO refusal to a
non-array container; PS currently refuses scalar `dates` as `bracket_missing`
(S2-i review F2). Assigning `bucket_malformed` normatively CHANGES a currently
valid observable result -> not a clarification -> **1.1** (result schema and
vocabulary unchanged; classification newly normative). 1.1 scope: contract doc
stage-1 sentence ("a `dates` container that is absent, null, or not an array is
`bucket_malformed`"), corpus `contractVersion` -> 1.1 + ONE new fixture (gsr-25,
scalar `dates` -> `bucket_malformed`) + additive `csharp_resolver` consumer tags;
PS CLI stage-1 conformance change (+ its harness count/vector updates); C# runner
born on 1.1. Owner: Slice 2-ii-a (single contract authority; both runtimes converge
in the same slice). Compatibility: additive fixture + one reclassified refusal on a
malformed-input path; no consumer parses that reason programmatically today.

## 7. resolver enforcement decision: OPTION B (typed pre-resolved boundary)

New `platform/dotnet/DevCore.Api/Sports/GameStatusResolver.cs` implementing the
contract stages over the EXISTING `MlbScheduleResponse` (buckets intact -- zero DTO
change): `internal static GameStatusResolution Resolve(MlbScheduleResponse
schedule, string bracketDate, long gamePk, ExpectedMatchup? expected)` returning
`Resolved(bucketDate, MlbScheduleGame game, IReadOnlyList<RescheduleContext>)` |
`Refused(GameStatusRefusal reason, context)` with the refusal enum mapping 1:1 to
the contract vocabulary. `MlbStarterClient` single+batch paths route through it
(the `:278`/`:338` flattens are REMOVED; the fetch's `date=` value becomes the
bracket input -- date scope becomes structural, not disciplinary);
`MlbEventResolver` is UNCHANGED and keeps matchup/identity validation over the
exactly-one candidate (it validates, no longer effectively selects across buckets).
External caller contracts unchanged (`GetStartersAsync` signatures intact).
Failure mapping: refusals surface through the existing grounding vocabulary
(`identity_unresolved`/`identity_ambiguous` classes) without new wire strings --
mapping table in the slice. Option A rejected (threads bucket data through a
resolver whose 24 pinned scenarios would all churn); Option C rejected (leaves the
implicit discipline the design ordered eliminated). RED: multi-bucket schedule
response through the batch path -- current flatten lets `:64` FirstOrDefault take
the FIRST cross-bucket match, i.e. wrong-bucket status is reachable if a caller is
ever handed a multi-bucket payload; the new boundary makes that structurally
impossible.

## 8. frozen `ScheduleState` decision

Validation at the SLATE boundary (`MarketContrastSourceAdapter` request validation
`:306-314`): absent/null/empty/whitespace -> typed validation rejection
`"candidate schedule state is required"` (rejected request, same class as the
existing candidate validations -- NOT a per-candidate pre-elimination, NOT
`status_malformed`, NOT `unknown`; the reason belongs to the slate/caller-state
contract, outside the game-status-resolution vocabulary, exactly per the published
design). Unrecognized NON-EMPTY strings keep the pinned normalize-to-`unknown`
fail-closed path. Tests: request-level rejection fixtures incl. JSON-null
deserialization. Owner: 2-ii-a. No logging change.

## 9. discovery identity and deduplication (canonical key)

The odds surface has NO MLB gamePk; it DOES have a provider event id
(`OddsApiEvent.id`, `OddsScheduleClient.cs:18`). Canonical identity hierarchy:
(1) exact MLB gamePk when the surface carries it (StatsAPI surfaces only);
(2) provider event id within provider scope (the odds schedule surfaces -- THE
replacement dedup key for `:127` and `:223`); (3) explicitly typed composite
(date, commenceTime, home, away) only where no id exists. Team-plus-date is
REJECTED as an authoritative key. Replacement behavior: legitimate DH (two ids,
same pair, same ET date) -> two rows; postponed+makeup -> distinct events by id;
same-teams-same-date distinct ids -> two rows; rescheduled start -> same id, one
row (dedupe by id, deterministic order by Date then CommenceTime then id);
duplicate transport records of one id -> one row.

## 10. matchup-date consumer impact

`MatchupEventDto(Date, HomeTeam, AwayTeam)` is a flat list; it CANNOT represent two
same-day games distinguishably and carries no gamePk (none exists on this surface)
-- fixing only the DistinctBy key WOULD surface indistinguishable duplicate rows,
so the shape MUST change additively: `MatchupEventDto(Date, HomeTeam, AwayTeam,
StartUtc, ProviderEventId)`. Consumer table:

| consumer | current assumption | dh-safe impact | required change | test |
|---|---|---|---|---|
| `SportsReferenceController.GetMatchupDates :142` | passthrough list | additive fields flow | none (shape passthrough) | contract test |
| `GetUpcoming :108` | passthrough | same | none | contract test |
| `ScheduleMatchupDatesHandler` | passthrough | same | none | existing DI test |
| sports-app `sports-api.service.ts` | typed list of matchup dates | may assume one entry/day | VERIFY in 2-ii-b; additive fields tolerated by JSON; add a DH rendering check only if the page groups by date | targeted spec if needed |

## 11. batch-boundary decision

`GetStartersForDateBatchAsync` (`:341-350`) currently last-write-wins on duplicate
caller gamePks (unreachable via the adapter, unpinned at the client boundary).
Contract: **duplicate input gamePks are REJECTED at the batch boundary (fail
closed)** -- identical-vs-conflicting duplicates cannot be distinguished cheaply
and no legitimate caller sends them; distinct DH pks are, and must remain,
independent keys. RED tests: two DH pks both survive with correct per-game
grounding; repeated identical pk -> typed rejection; input order does not change
output; no last-write nondeterminism observable. Owner: 2-ii-b.

## 12-14. F2 / F3 / F4 dispositions

- **F2**: resolved by the 1.1 contract decision (section 6); owner 2-ii-a; PS +
  C# converge on `bucket_malformed` for absent/null/non-array `dates`.
- **F3**: smallest correction = compute failure-detail strings null-safely (bind
  `$reason = if ($null -ne $j) { $j.games[0].reason } else { '<no json>' }` before
  the Assert; never dereference inside the interpolation). Preserves every
  failure line, the final tally, nonzero exit, fixture ids; cannot convert fail to
  pass; zero production change. Owner: **2-ii-c** (no 2-ii-a/b RED depends on the
  finals harness; not a prerequisite commit).
- **F4**: **AUTHORITY_PLUS_CONTEXT_SELECTED.** The CLI live URL drops `date=` and
  fetches broadly by `gamePks=` (parity with the accepted finals-guard fetch);
  local bracket-first staging is unchanged and remains the sole authority; live
  and offline context semantics align with the multi-bucket fixtures; transport
  stays bounded/fail-closed (single GET, existing timeout). No completeness field
  needed once live matches offline semantics; `sourceMode`/`sourceRef` already
  disclose provenance. Owner 2-ii-c; NO live call in any 2-ii slice.

## 15. decomposition: OPTION 2 -- three sub-slices (SELECTED)

| slice | goal | exact likely paths | RED | GREEN | review boundary | deps |
|---|---|---|---|---|---|---|
| **2-ii-a** contract conformance | xunit corpus runner; contract 1.1 (+gsr-25, consumer tags); `GameStatusResolver` + starter-client routing; frozen-state validation; PS CLI F2 conformance | C# prod 2 (`GameStatusResolver.cs`, `MlbStarterClient.cs`) + adapter validation touch (1); C# tests 2; contract+corpus 2; PS 2 (CLI + its harness); vault 3 | no runner exists; multi-bucket wrong-bucket reachability; null state NRE; scalar dates misclassified | new runner all-vectors; DevCore suite; both PS harnesses | conformance only, no discovery | none |
| **2-ii-b** discovery correctness | id-keyed dedup in `OddsScheduleClient`; additive `MatchupEventDto` shape; batch duplicate rejection; first schedule-client unit tests | C# prod 2 (`OddsScheduleClient.cs`, `MlbStarterClient.cs` batch guard) + controller passthrough check; C# tests 2 new; app spec 0-1; vault 2 | DH collapse at `:223`; repeated-pk last-write | new client tests; DevCore suite; app spec if touched | discovery only | 2-ii-a (corpus 1.1) |
| **2-ii-c** operator/harness parity | F3 fix; F4 CLI live-URL change; cross-runtime parity vectors; doctrine sync | PS 3 (finals harness, CLI, CLI harness); vault 2 (doctrine, records) | harness crash repro; CLI URL asserts date= today | both PS harnesses; parity checks | scripts/doctrine only | 2-ii-a |

Path counts stay <= ~8 production/test sources per slice; 2-ii-a spans two runtime
domains (C# + narrow PS conformance) -- justified: single contract-authority
domain, and splitting the PS conformance out would create exactly the floating
contract-only work the design forbade. One mega-slice rejected (three RED domains,
~15 paths); PS-only F2 slice rejected (contract must stay single-authority).

## 16. verification plan

Per slice: RED first (preserved), then focused new tests, full
`DevCore.Api.Tests` suite (baseline 1780 on `dd760f9`), both PS harnesses
(baseline 40/181; counts shift with 1.1 vectors -- new baselines recorded in the
slice), strict snapshot, diff-check, scans. No live calls anywhere in 2-ii.

## 17. wi-0037 completion boundary

WI-0037 CLOSES when 2-ii-a, 2-ii-b, and 2-ii-c are complete: all corpus vectors
(1.1) conform in both runtimes where applicable, discovery is doubleheader-safe,
date scope is structurally enforced, frozen state is validated, F2/F3/F4 are
resolved, and no game-state rule exists only in prompts. Reconciliation CLI and
admin UI are explicitly NOT closure prerequisites (they are consumers of, not
parts of, game-state correctness).

## 18. exclusions

Reconciliation CLI/application-service redesign, admin pages, settlement mutation,
paid capture, live validation, WI-0034 slices, provider-event-binding redesign,
planner changes, database/schema/migration changes, r7a26 beyond F3, broad
StatsAPI client refactors, unrelated cleanup.

## 19. risks and residuals

Contract 1.1 shifts harness vector counts (documented, versioned); starter-client
routing touches the most-pinned identity path (mitigated: MlbEventResolver
unchanged, 24 existing scenarios must stay green plus corpus vectors); matchup-date
shape change may expose a sports-app one-per-day assumption (explicit verify step
in 2-ii-b); batch rejection is a behavior change at an internal boundary currently
shielded by the adapter (defense-in-depth, no external contract change).

## 20. next authorization

**WI-0037 Slice 2-ii-a** exactly as scoped in section 15 (copy-ready prompt in the
planning handoff/final report).
