---
title: "WI-0037 Game-State Correctness v1"
type: "plan"
date: "2026-07-24"
status: "in-progress"
project: "DAI"
slice: "WI-0037 planning only: parent scope + Slice 1-2 definitions; NO implementation authorized"
repos:
  dai: "unchanged (planning; implementation awaits separate slice authorizations)"
  dai-vault: "docs-only (this WI, MOC registration, current-slice)"
tags:
  - system-development
  - work-item
  - sports-v1
  - correctness
related:
  - "02 Platform/system-development/work-items/WI-0034-daily-evidence-planner-stage-0.md"
  - "02 Platform/system-development/work-items/WI-0035-market-contrast-candidate-screen.md"
  - "02 Platform/system-development/work-items/WI-0036-wildcard-evidence-discovery-loop-v1.md"
  - "06 Execution/reports/dai-work-item-completion-audit-2026-07-24-v1.md"
---

# wi-0037 game-state correctness v1

## problem  <!-- LITE -->

Two verified game-state correctness defects surfaced during the July 23 daily-evidence
operation, and neither is owned by an open work item:

1. **Caller-state over-constraint (twice-demonstrated paid cost).**
   `MarketContrastSourceAdapter.PreEliminationReason`
   (`platform/dotnet/DevCore.Api/AgentRuns/MarketContrastSourceAdapter.cs:348-351`)
   pre-eliminates a caller candidate on any ordinal inequality of normalized schedule
   state, even though line 349 already admits both `scheduled` and `pregame` as
   screenable. Routine `Scheduled -> Pre-Game` progression between slate freeze and
   screen therefore kills valid candidates. This fired at 16:11Z and again at 17:57Z on
   2026-07-23, eliminating the day's primary anchor (gamePk 823042) from both paid
   screens. The branch has zero test coverage; only the sibling `caller_start_mismatch`
   branch is asserted (`DevCore.Api.Tests/AgentRuns/MarketContrastSourceAdapterTests.cs:268`).

2. **Distributed, partially implicit date-scoping discipline (latent).**
   The observed July 23 false-postponement misread came from an operator-side ad-hoc
   StatsAPI query reading the wrong date bucket (no first-party code contains positional
   `dates[0]` selection -- verified 2026-07-24). But authoritative date-scoping remains
   distributed and unenforced: `MlbEventResolver.cs:64` resolves by `FirstOrDefault` on
   gamePk and is identity-safe only while every caller stays date-scoped (an unenforced
   invariant); `OddsScheduleClient.cs:223` deduplicates by `DistinctBy(Date)` and can
   collapse legitimate doubleheaders on the discovery path (`:127` keys on
   date+home+away); the finals guard flattens buckets and can false-DEFECT legitimate
   makeup/final pairs; operator-facing status checks have no canonical resolver script.

## desired behavior

Deterministic, test-pinned agreement with authoritative MLB game-state truth across
planning, screening, capture, finality, and settlement boundaries: safe monotonic
schedule-state progression is never treated as an identity threat, and every status
decision is derived from a frozen Eastern date bracket plus an exact gamePk resolving to
exactly one authoritative candidate.

## affected surfaces

- `platform/dotnet/DevCore.Api/AgentRuns/MarketContrastSourceAdapter.cs` (Slice 1)
- `platform/dotnet/DevCore.Api.Tests/AgentRuns/MarketContrastSourceAdapterTests.cs` (Slice 1)
- `platform/dotnet/DevCore.Api/Sports/MlbEventResolver.cs` (Slice 2)
- `platform/dotnet/DevCore.Api/Sports/OddsScheduleClient.cs` (Slice 2)
- `scripts/dev/sports/check-settlement-finals.ps1`, `preflight-settlement.ps1`,
  planner preflight / pre-screen / pre-capture gates, operator status-check scripts
  (Slice 2 inspection scope)
- relevant C# and PowerShell tests for each converged surface (Slice 2)

## non-goals

This WI stays narrowly on game-state correctness. It is NOT a general schedule,
provider, planner, or orchestration redesign. Explicitly excluded from the whole WI:
provider-binding changes (WI-0035 closed scope), planner-board schema changes (WI-0034/
WI-0036), schedule-adapter automation (WI-0034 Slice 3), operating-skill integration
(WI-0034 Slice 4), broad StatsAPI client rewrite, start-instant tolerance policy change
(characterize only; any tolerance is a separately decided follow-up), paid calls,
captures, settlement of live data, and database writes.

## evidence basis

The July 23 operational chain, canonical records on published vault main:

- false-postponement interpretation and forward correction:
  `06 Execution/reports/daily-evidence-late-slate-reevaluation-2026-07-23-v1.md`
- first paid screen (capture blocked by the misread; first caller_state_mismatch
  exclusion): `06 Execution/reports/daily-evidence-paid-screen-pass2-capture-blocked-2026-07-23-v1.md`
- second paid screen (second caller_state_mismatch exclusion; successful TB@TOR
  capture): `06 Execution/reports/daily-evidence-second-screen-capture-2026-07-23-v1.md`
- authoritative settlement of that capture:
  `06 Execution/reconciliations/daily-evidence-second-screen-settlement-2026-07-24-v1.md`
- completion-audit verification of the defects, coverage gap, and ownership analysis:
  `06 Execution/reports/dai-work-item-completion-audit-2026-07-24-v1.md`
- free-cycle context: `06 Execution/reports/daily-evidence-planner-free-preflight-2026-07-23-v1.md`

## ownership and relationships

New corrective WI; WI-0035 is NOT reopened. Semantic ownership, not file history, drives
this: the caller-state predicate was discovered by WI-0035 operational validation but the
parent invariant (game-state truth across all boundaries) spans WI-0034 planner gates,
WI-0035 screening, finals/settlement guards, and WI-0036 flight/provenance consumers.

- discovered by: WI-0035 operational validation (2026-07-23 paid screens)
- corrects: game-state behavior exercised through WI-0034 planning and WI-0035 screening
- related consumers: WI-0036 flight realization and provenance reads where schedule
  state participates in candidate qualification

## decomposition

### Slice 1 -- monotonic caller-state progression (IMPLEMENTED LOCAL 2026-07-24; review pending)

> State (2026-07-24, superseding "defined; NOT authorized"): authorized and implemented
> RED-first on local branch `wi/0037-game-state-correctness-slice-1` (dai `0a9129b` from
> base `85af96d`): 18-row transition matrix + July 23 regression fixture +
> start-instant precedence proof (2 RED before the fix); narrow predicate correction
> admits ONLY frozen `scheduled` -> authoritative `pregame`; focused 47/47, related
> 343/343, full suite 1780/1780. NOT reviewed, NOT integrated, NOT pushed. Evidence:
> [[wi-0037-slice-1-caller-state-progression-2026-07-24-v1]]. Slice 2 remains
> unauthorized and unimplemented; WI-0037 stays `in-progress`.

**Problem.** As above: `PreEliminationReason` rejects `Scheduled -> Pre-Game` because it
compares normalized states for ordinal equality although both states are independently
screenable, conflating identity with freshness.

**Intended behavior.** Admit only explicitly safe monotonic progression within the
screenable state set. Initial required transition: `Scheduled -> Pre-Game`. Continue
failing closed for: any transition to postponed, suspended, or canceled; in-progress
(live) states; final; unknown or unsupported states; backward progression
(`Pre-Game -> Scheduled`); start-instant mismatch (sibling branch, unchanged); game
identity mismatch; ambiguous live state.

**Required implementation proof (for the future authorization):**
1. RED characterization tests pinning the CURRENT predicate first;
2. a complete state-transition matrix (every ordered pair over the normalized state set);
3. tests for the sibling start-instant mismatch branch;
4. the narrowest predicate correction that admits the safe transition set;
5. no schema or policy-version change unless proven necessary in review;
6. focused adapter tests green;
7. full DevCore.Api.Tests suite green;
8. an operationally realistic fixture reproducing the July 23 transition (slate frozen
   `Scheduled` at 17:44Z, caller `Pre-Game` at screen time);
9. no threshold relaxation anywhere else in the screen policy;
10. no change to paid-screen authority.

**Exclusions.** Start-instant tolerance policy change (characterization may name the
future question but must not alter behavior); date-bucket resolver work (Slice 2);
schedule-adapter automation; operating-skill integration; provider-binding changes;
planner schema changes; paid validation calls.

### Slice 2 -- canonical date-bracketed status resolution (defined; NOT authorized)

**Problem.** As above: date-scoping discipline is distributed and partially implicit;
latent risks are resolver callers assumed date-scoped, `FirstOrDefault` in
identity-sensitive paths, doubleheader collapse via date-keyed deduplication, makeup and
historical bucket ambiguity, finals-guard behavior across legitimate paired buckets, and
operator queries outside the canonical discipline.

**Parent invariant.** Every status decision derives from: (1) a frozen Eastern date
bracket; (2) an exact gamePk; (3) exactly one authoritative candidate within the
applicable resolution policy; (4) explicit handling of doubleheaders, makeup games,
historical buckets, and duplicates; (5) no positional bucket selection; (6) the same
test-pinned resolver semantics across all consumers.

**Required surfaces to inspect and, where justified, converge:** planner preflight;
market pre-screen; pre-capture gate; finality guard; settlement preflight; schedule
discovery; operator-facing status-check scripts; `MlbEventResolver`;
`OddsScheduleClient`; relevant C# and PowerShell tests.

**Required tests (minimum):** normal single game; legitimate doubleheader; same-team
same-date doubleheader; postponed original plus scheduled makeup; historical and current
buckets sharing identity context; duplicate gamePk; missing game; undated resolver
misuse; finals guard with legitimate makeup/final pairs; exact bracket boundary
behavior.

**Exclusions.** Caller-state progression correction (Slice 1); provider-binding
redesign; planner-board schema changes; broad StatsAPI client rewrite; automated
schedule-adapter implementation; settlement of live data; paid calls.

## acceptance criteria  <!-- LITE -->

Planning-slice criteria (this authorization only):

- WI-0037 exists with the parent purpose stated verbatim, is MOC-registered, and appears
  in the strict snapshot as the 26th registered work item with status `in-progress`;
- both slices are defined with problem, intended behavior, proof requirements, and
  exclusions, and NEITHER is authorized;
- ownership records name discovery (WI-0035), corrected surfaces (WI-0034/WI-0035), and
  related consumers (WI-0036) without reopening WI-0035;
- evidence basis cites the canonical July 23 chain records by full vault path;
- no source file, test, schema, or runtime state changes in this slice.

Parent-close criteria (future): both slices independently implemented, reviewed, and
integrated under their own authorizations with the proofs above.

## test plan  <!-- LITE, written BEFORE implementation -->

No tests in this planning slice. Slice 1: RED characterization matrix in
`MarketContrastSourceAdapterTests.cs` before any predicate change. Slice 2: the
ten-scenario minimum above across C# resolver/client tests and PowerShell guard tests.

## implementation notes

Slice order is 1 then 2 (smallest bounded fix with demonstrated paid cost first; the
resolver convergence benefits from the characterization discipline Slice 1 establishes).
WI-0034 Slice 4 (operating/skill integration) should follow both correctness slices so
the codified workflow references the canonical resolver. Sequencing recorded in the MOC.

## docs to update

This WI; `MOC - DAI System Development.md` (registration + backlog sequencing);
`06 Execution/handoffs/current-slice.md` (planning handoff at close of this slice).

## verification commands  <!-- LITE -->

This slice: strict snapshot (`build-next-slice-snapshot.ps1 -Strict`) showing 26 items /
0 warnings; `git diff --check`. Future slices: per their authorizations (focused test
filters + full `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj`).

## risks

Slice 1: widening the admitted transition set too far (mitigated by the explicit
fail-closed list and RED-first matrix). Slice 2: convergence pressure turning into a
client rewrite (mitigated by the inspect-then-converge-where-justified scope and the
non-goals list); finals-guard changes altering settlement behavior (any guard change
must keep the duplicate-DEFECT discipline pinned by its existing tests).

## links  <!-- LITE; all 8 required at close, per work-item-traceability -->

- work item: WI-0037 (ADO: AB#- when wired; no ADO item created)
- branch: `wi/0037-game-state-correctness` (dai-vault planning branch from published
  main `af69725`; implementation branches minted at slice authorization)
- pr: - (planning slice; no push authorized)
- commits: planning commit recorded at close of this slice
- tests: none this slice (planning only; future test surfaces named above)
- verification notes: strict snapshot 26 items / 0 warnings at close; audit report
  ownership analysis
- docs updated: this WI; MOC; current-slice at close
- lessons: none this slice (the operational lessons live in the July 23 records and the
  completion-audit report)

## final handoff requirements

Planning handoff appended to `06 Execution/handoffs/current-slice.md`; status stays
`in-progress` (no slice authorized); next governed action = a separate operator
authorization for Slice 1 implementation.
