---
title: "WI-0037 Game-State Correctness v1"
type: "plan"
date: "2026-07-24"
status: "complete"
project: "DAI"
slice: "WI-0037 game-state correctness: all slices implemented, reviewed, and locally integrated; local-only, unpublished, undeployed, activation-disabled, migration-unapplied"
repos:
  dai: "code (game-state correctness across Slices 1/2-i/2-ii-a/2-ii-b/2-ii-c+c1/2-iii; integrated to local main 4c6cd98; unpublished, undeployed)"
  dai-vault: "docs-only (this WI, MOC, current-slice, slice report, closeout report)"
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

> **WI-0037 CLOSEOUT -- COMPLETE (local integration + governance closeout, 2026-07-28).**
> All WI-0037 slices are implemented, independently reviewed, and integrated into the
> LOCAL engineering baseline: Slice 1 (caller-state progression), Slice 2-i (status
> resolution contract 1.0 + corpus + finals-guard bracket), Slice 2-ii-a (contract 1.1 +
> xunit corpus runner + typed GameStatusResolver), Slice 2-ii-b (discovery identity),
> Slice 2-iii (selected-event identity continuity, foundation + backend activation +
> frontend consumer continuity), and finally Slice 2-ii-c + c1 (operator/harness + status
> contract parity). The Slice 2-ii-c + c1 package was independently adversarially reviewed
> with verdict **WI0037_SLICE2IIC_C1_INDEPENDENT_REVIEW_PASSED_LOCAL_INTEGRATION_READY**
> (zero findings) and integrated by a pure local fast-forward: dai local `main` advanced
> `baf5e90` -> **`4c6cd985662caebec686d4b96a12a4104440e857`** (two reviewed commits 80e8d1a
> operator/harness parity, 4c6cd98 blank-state + comment correction). dai `origin/main`
> remains `aee1ade` (observational, deliberately not refreshed). This vault records branch
> is advanced into local vault `main` by an atomic local ref update; vault `origin/main`
> remains `e217a3f`. NOTHING was pushed and NOTHING was deployed.
>
> **Post-integration verification on source main at 4c6cd98:** PowerShell
> `test-check-game-status.ps1` 195/0 (exit 0); `test-check-settlement-finals.ps1` 40/0
> (exit 0) and `-Probe` reaches its tally with one pass + one deliberate failure (exit 1 by
> design, not a regression); focused `GameStatusResolutionCorpusTests` 63; combined focused
> .NET 189/0; full `DevCore.Api.Tests` **2059/2059, 0 skipped**; solution build 0 errors.
> Static gates: 69 added C#/PowerShell comment lines with zero uppercase and zero non-ascii;
> exactly one c# schedule-state alias table (`ScheduleStateNormalizer.cs`); every production
> `GameStatusResolver.Resolve` supplies the typed `GameStatusDetailRequirement`; the live URI
> uses `gamePks` with no `date=`, one GET, 30s timeout, local bracket selection; both runtimes
> exercise all 31 normalization vectors; corpus = 25 fixtures / 31 vectors / 6 refusals / 8
> normalized values. Warnings are pre-existing and NOT introduced by this package: NU1903
> advisories (`Microsoft.OpenApi` 2.0.0, `System.Security.Cryptography.Xml` 10.0.7) plus
> pre-existing compiler/nullability/xUnit warnings, every one located in files outside the
> 11-path changed set; not a warning-free build, not remediated. No network call.
>
> **Mandatory WI-completion glossary disposition pass** (rules: promote only a durable concept
> with demonstrated meaning beyond WI-0037; merge same-concept labels; otherwise retain
> WI-local; never erase from WI history; do not add a glossary entry for a merged or
> retained-WI-local term):
>
> | term | disposition | reason |
> |---|---|---|
> | provider-scoped candidate identity | **MERGED** into `provider-qualified duplicate identity` | Slice 2-iii-b2 F-B2-8 unified provider-scoped candidate discovery and provider-qualified duplicate identity as one WI-local concept over the governing `(SourceProvider, ExternalGameId)` pair; the surviving concept is `provider-qualified duplicate identity`. |
> | provider-qualified duplicate identity | **RETAINED WI-local** | load-bearing selected-event duplicate-detection evidence within WI-0037, but an implementation-level identity rule with no demonstrated use beyond WI-0037/selected-event binding; promoting it would distort the shared glossary. Not added to `glossary.md`. |
> | activation deployment evidence | **RETAINED WI-local** | the structured `{artifact, reference}` freshness-bounded evidence gating default-off `SelectedEventActivation` is specific to the selected-event activation runbook; no demonstrated platform-wide meaning. Not added to `glossary.md`. |
> | validated writer | **RETAINED WI-local** | the validated-writer contract (a provenance builder proven readable via `TryRead` before commit) is a useful WI-0037 selected-event write-path pattern but has no demonstrated reuse beyond WI-0037; default retain. Not added to `glossary.md`. |
>
> `domain execution provenance` was already PROMOTED to `01 Operating System/glossary.md`
> in a prior slice; it is confirmed present and NOT duplicated. Because no term was promoted,
> `glossary.md` is unchanged by this closeout.
>
> **Residuals (exact, preserved):**
> SELECTED_EVENT_IDENTITY_PROPAGATION_REQUIRED_BEFORE_WI0037_CLOSE = **RESOLVED** (Slice
> 2-iii-c local integration; do not reopen or erase);
> MULTI_INSTANCE_SELECTED_EVENT_ATOMICITY_REQUIRED_BEFORE_SCALE_OUT = **blocking before
> scale-out**, not a blocker to this local WI closeout;
> DURABLE_PREEXECUTION_SELECTION_DECISION_LEDGER_DEFERRED = **deferred/nonblocking**.
>
> **Release state (truthful):** LOCAL-ONLY; UNPUBLISHED; UNDEPLOYED; selected-event
> activation DISABLED (`SelectedEventActivation.Enabled=false`, blank DeploymentId/
> EvidencePath); migration `20260727133845_AddAgentRunDomainExecutionProvenance` present but
> UNAPPLIED. WI-0037 completion means the scoped implementation, conformance, review, local
> integration, and governance obligations are complete. It does NOT mean scale-out readiness,
> deployment readiness, product readiness, or production activation. Evidence:
> [[wi-0037-local-integration-and-governance-closeout-2026-07-28-v1]] and
> [[wi-0037-slice-2-ii-c-operator-harness-parity-2026-07-27-v1]].

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

### Slice 1 -- monotonic caller-state progression (COMPLETE: INTEGRATED + PUBLISHED 2026-07-24)

> State (2026-07-24 closeout, superseding "IMPLEMENTED LOCAL; review pending"):
> implemented RED-first, INDEPENDENTLY REVIEWED (verdict
> WI0037_SLICE1_INTEGRATION_READY; zero required corrections; findings recorded below),
> INTEGRATED and PUBLISHED by coordinated fast-forward: dai main `85af96d` ->
> `0a9129b5ab74158a2169653cedac1d898f09b67e` (remote verified), vault main `51b64f4` ->
> `3b66596b704773e7158f211ad930d4a80cd761c1` (remote verified). Post-integration full
> suite on published dai main: 1780/1780, 0 skipped. The predicate admits ONLY frozen
> `scheduled` -> authoritative `pregame`; every other unequal pair, backward
> progression, non-screenable state, and the start-instant blocker remain fail-closed.
> Evidence: [[wi-0037-slice-1-caller-state-progression-2026-07-24-v1]]. Review
> follow-ups (recorded, NOT implemented): (1) test-harness hygiene -- pre-existing
> global `Console.SetOut` capture race in
> `MarketContrastEventsGateTests.r7a26_command_di_named_client_seam` (flaked once in
> six full runs, passed isolated/class/rerun; outside Slice 1; needs its own
> hygiene-slice ownership decision); (2) Slice-2 rider -- null/absent frozen
> `ScheduleState` would NRE in the normalizer (pre-existing; characterize during
> Slice 2 planning); (3) alias note -- raw `Scheduled/Preview -> Warmup` passes because
> those values normalize to scheduled -> pregame (intentional, not an additional
> normalized transition). Slice 2 remains unauthorized and unimplemented; WI-0037 stays
> `in-progress`.

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

### Slice 2 -- canonical date-bracketed status resolution (2-i CLOSED; 2-ii-a CLOSED; 2-ii-b CLOSED; 2-iii INTEGRATED LOCAL; 2-ii-c + c1 INDEPENDENTLY REVIEWED PASS + LOCALLY INTEGRATED 2026-07-28 -- see the WI-0037 CLOSEOUT block at the top of this file)

> Slice 2-ii-c1 correction state (2026-07-27, on top of 2-ii-c): an independent review of
> the 2-ii-c package (source 80e8d1a / vault d98d1a5) found three issues, corrected in place
> (originals preserved; correction commits on top). Terminal state
> **WI0037_SLICE2IIC_C1_CORRECTION_IMPLEMENTED_LOCAL_REVIEW_REQUIRED**.
> F-1 (blocking): the PowerShell `Resolve-GameStatus` stage-5 check used truthiness, so a
> whitespace-only authoritative detailedState resolved as resolved/unknown while c# `Required`
> correctly refuses status_malformed. The original vector tests only exercised the direct
> normalizer (both runtimes normalize whitespace to unknown), so the staged divergence was
> invisible. Corrected RED-first: both runners now drive every vector through staged Required
> resolution (blank -> refused/status_malformed/null, nonblank -> resolved/expected); against
> 80e8d1a the strengthened PowerShell test failed for nv-31-whitespace-only only; the c# staged
> assertion already passed. Production fix is PowerShell-only: stage-5 now uses
> `[string]::IsNullOrWhiteSpace`; NO c# resolver/normalizer change (c# was the correct
> fail-closed side). F-2 (hygiene): the 2-ii-c report/current-slice wrongly claimed the
> added-comment audit was lowercase-ascii clean; the exact original audit over baf5e90..80e8d1a
> was 55 added comment lines, 19 uppercase-bearing, 0 non-ascii. Corrected: every added comment
> rewritten to generic lowercase prose (no code symbol/parameter/function/test/runtime-string
> renamed); corrected full-range audit = 69 added comment lines, 0 uppercase, 0 non-ascii. F-3
> (low): the contract md called the c# runner future work (corrected: it consumes the corpus
> now); the doctrine said 24 vectors (corrected: 25 scenario fixtures, 31 normalization vectors,
> six refusal reasons). Verification: PS test-check-game-status 195/0 (all vectors end-to-end),
> test-check-settlement-finals normal 40/0 + `-probe` exit 1; full .NET 2059/2059, 0 skipped;
> solution build 0 errors; 25 fixtures + 31 vectors + 6 reasons byte-for-byte unchanged; one c#
> alias table; live URL gamePks no date=; no network call; contract stays
> game-status-resolution/1.1. Warnings unchanged/pre-existing (NU1903 + compiler/nullability/
> xUnit; not remediated). Preserved wi/0035 checkout unchanged (de5791f, 86aa8b74); local-only,
> unpushed, undeployed; activation disabled; migration unapplied. Residuals: propagation RESOLVED
> (2-iii-c); scale-out blocking; decision-ledger deferred. Slice 2-ii-c + c1 is IMPLEMENTED
> LOCAL; independent review required over the corrected ranges; WI-0037 remains in-progress.
> Evidence: [[wi-0037-slice-2-ii-c-operator-harness-parity-2026-07-27-v1]] (slice 2-ii-c1
> correction section).

> Slice 2-ii-c state (2026-07-27, superseding "2-ii-c NOT authorized"): the operator
> authorized and this slice IMPLEMENTED LOCAL the final WI-0037 hardening obligations,
> review-required, on branch `wi/0037-game-state-correctness-slice-2-ii-c` (dai, from
> baf5e90; vault records from 849eb9d). Terminal state
> **WI0037_SLICE2IIC_IMPLEMENTED_LOCAL_REVIEW_REQUIRED**. Delivered: F3 null-safe
> finals-harness evidence (test-check-settlement-finals.ps1 only -- Get-FirstGameReason /
> Get-ReasonDetail null-safe binding replaces the two inline `$j.games[0].reason` detail
> dereferences; a self-contained `-Probe` mode proves the harness reaches its tally on
> malformed/non-json output and exits nonzero; normal harness still green 40/40; no
> production guard change); F4 authority-plus-context live query (check-game-status.ps1
> fetches `schedule?sportId=1&gamePks=<pk>` with no `date=`, one GET, 30s timeout,
> `-BracketDate` still the sole local bracket authority, offline path unchanged, truthful
> sourceRef; harness statically proves the URI shape with no live call); a mandatory typed
> `GameStatusDetailRequirement` (Required/NotRequired) replacing the defaulted boolean
> requireStatus at every resolver call site (starter = NotRequired, corpus + selected-event
> = Required; undefined enum value rejected; Resolved now accepts a truthful nullable
> normalized status with the null! suppression removed); one c# `ScheduleStateNormalizer`
> consolidating the resolver and adapter alias tables (both duplicates removed; PowerShell
> keeps its separate runtime normalizer by design); and a top-level `normalizationVectors`
> collection (31 vectors) in the canonical corpus consumed generically by both runners with
> no-skip proof. Contract stays `game-status-resolution/1.1`; the 25 scenario fixtures and
> six refusal reasons are unchanged; no runtime result shape, refusal, or normalized
> vocabulary changed. Baselines: PS 187/40, .NET 2025. After: PS test-check-game-status
> 195/0, test-check-settlement-finals normal 40/0 (probe 1/1 exit 1 by design); full .NET
> **2059/2059, 0 skipped** (2025 + 34); solution build 0 errors. Warnings (accurate,
> pre-existing, not introduced): NU1903 Microsoft.OpenApi 2.0.0 + System.Security.
> Cryptography.Xml 10.0.7, plus compiler/nullability/xUnit analyzer warnings; not a
> warning-free build; not remediated. NO frontend, database, migration, activation,
> reconciliation, or settlement change; local-only, unpushed, undeployed; activation
> disabled; migration 20260727133845 unapplied. Preserved wi/0035 checkout unchanged
> (de5791f, fingerprint 86aa8b74). NOT reviewed, NOT integrated, NOT pushed. Residuals:
> propagation RESOLVED (2-iii-c, unaffected); scale-out blocking; decision-ledger deferred.
> Slice 2-ii-c is IMPLEMENTED LOCAL; independent review required. WI-0037 remains
> in-progress. Next = one independent adversarial review of the complete source+vault
> 2-ii-c package; integration + closeout only under a later authorization. Evidence:
> [[wi-0037-slice-2-ii-c-operator-harness-parity-2026-07-27-v1]].

> Slice 2-ii-b closeout (2026-07-26, superseding "DR-2 corrected, delta review
> required"): the DR-2 delta review returned
> **WI0037_SLICE2IIB_DR2_DELTA_REVIEW_PASSED_INTEGRATION_READY** (zero findings),
> and the complete reviewed chains were INTEGRATED by coordinated fast-forward
> and PUBLISHED: dai main `841ae26` -> **`af598530217bf2558a7323fc301b20237eb62cee`**
> (five commits: 5a11a2c implementation, c5ab834 + c545117 F-A..F-G corrections,
> b8d0c0b DR-1, af59853 DR-2; remote verified, main-only push); vault main
> `664cd4a` -> **`dad6095a95eee3305797ecba2e1c6214d89ea98e`** (four docs commits;
> remote verified) plus this closeout commit on top. Post-publication on
> published dai main: .NET **1853/1853, 0 skipped**; operator **187/187**
> (contract 1.1, 25 fixtures, six-reason vocabulary); guard **40/40**; frontend
> **157/157 (17 files)** + build. Published guarantees: provider-id-authoritative
> doubleheader-safe discovery with whole-id fail-closed integrity before any
> filtering; STARTUTC_FIXED_WIDTH_UTC_100NS; providerEventId selection identity
> with one-selected-event = one-execution-entry parity; atomic
> duplicate_gamepk_batch_input rejection; collision-safe JSON-tuple label
> grouping. Mandatory unresolved residual:
> SELECTED_EVENT_IDENTITY_PROPAGATION_REQUIRED_BEFORE_WI0037_CLOSE (see Slice
> 2-iii). Slice 2-ii-b is CLOSED. Evidence:
> [[wi-0037-slice-2-ii-b-adversarial-review-and-corrections-2026-07-26-v1]]
> part 5 and [[wi-0037-slice-2-ii-b-discovery-identity-2026-07-26-v1]]. 2-ii-c
> remains unauthorized (F3, F4, requireStatus hardening, normalization
> consolidation, cross-runtime parity); 2-iii remains defined/unauthorized and
> REQUIRED before WI-0037 closure; WI-0037 stays `in-progress`.

> Slice 2-ii-b DR-2 state (2026-07-26, superseding "DR-1 corrected, DR-1 delta
> review required"): the independent DR-1 delta review CLOSED DR-1 (blob purity,
> runtime U+0000 equivalence, chain text visibility, no concealment, behavior
> identical) and returned one new finding, DR-2 (Medium): the collision-safety
> claim was overstated -- provider-controlled team names are not validated
> against control characters, and a NUL-embedded team name probe-survives the
> sampler path, letting the delimiter-concatenated group key alias distinct
> tuples ("A", NUL+"B") vs ("A"+NUL, "B") (label-grouping impact only; selection
> identity and paid-run cardinality unaffected). Corrected with NEW commit dai
> `af59853` ("fix(sports): use collision-safe matchup group keys"): the key is
> now `JSON.stringify([date, homeTeam, awayTeam])` -- unambiguous for arbitrary
> STRING content over the contract-guaranteed string tuple
> (GROUP_KEY_MEMBERS_STRING_GUARANTEED) -- with table-driven structural vectors
> (embedded NUL, printable delimiters, spaces/colons/pipes, quotes, backslashes,
> json-looking content, empty strings, unicode; both source orders; identical
> tuples still group) and the overstated claim removed/superseded in code and
> record. RED re-reproduced pre-fix (dr2-red.txt). Suites: .NET 1853/1853,
> operator 187/187, guard 40/40, frontend **157/157** + build. Chain
> `841ae26..af59853` fully text-visible (no binary source entries). NOT yet
> independently re-reviewed, NOT integrated, NOT pushed; integration readiness
> NOT claimed until the DR-2 delta review passes. 2-ii-c unauthorized; 2-iii
> defined/unauthorized; WI-0037 in-progress. Evidence:
> [[wi-0037-slice-2-ii-b-adversarial-review-and-corrections-2026-07-26-v1]]
> part 4.

> Slice 2-ii-b DR-1 state (2026-07-26, superseding "corrections applied, delta
> review required"): the independent delta review of the correction chain returned
> **WI0037_SLICE2IIB_DELTA_REVIEW_CORRECTIONS_REQUIRED** with every behavioral and
> architectural attack PASSING and exactly one finding -- DR-1 (Medium,
> audit-integrity, no runtime defect): `core/matchup-event-labels.ts` contained two
> raw U+0000 bytes as group-key separators, so git classified the source blob as
> binary (undiffable; skipped by text-based audit sweeps). Corrected with a NEW
> commit dai `b8d0c0b` ("fix(sports): make label grouping source text-safe"): the
> raw bytes are now six-character ascii escape syntax producing the IDENTICAL
> runtime NUL separator, plus one collision-resistance vector in the existing spec
> (required for runtime-equivalence proof: it fails under any printable separator).
> Proofs: zero NUL bytes in the committed blob; file classifies as javascript
> source text; complete-chain diff/numstat show the helper as text (63/0); no
> gitattributes/diff-driver concealment; the parent-pair diff alone still reads
> binary because the PARENT blob at c545117 contains the raw bytes (expected git
> nuance -- the authoritative checks are blob and complete-chain classification).
> Suites: .NET 1853/1853, operator 187/187, guard 40/40, frontend **156/156**
> (155 + 1 dr-1 pin) + build. NOT yet independently re-reviewed, NOT integrated,
> NOT pushed; integration readiness is NOT claimed until the DR-1 delta review
> passes. 2-ii-c unauthorized; 2-iii defined/unauthorized; WI-0037 in-progress.

> Slice 2-ii-b correction state (2026-07-26, superseding "IMPLEMENTED LOCAL, review
> pending"): the independent adversarial review of the 2-ii-b chain returned
> **WI0037_SLICE2IIB_REVIEW_CORRECTIONS_REQUIRED** (F-A High dev-artifact-review
> date:home:away identity aliasing; F-B Medium pair-path filtering before
> provider-integrity normalization; F-C Medium silent StartUtc whole-second
> truncation; F-D Medium byte-identical labels for identical-instant events; F-E Low
> year-one sentinel from missing commence; F-F Low overstated "end to end" identity
> wording; F-G Low weak duplicate-batch assertion). All seven were corrected
> RED-first with NEW commits on the same local branch (5a11a2c never amended):
> dai `5a11a2c` -> `c5ab834` (backend: normalize-before-filter on both paths;
> group-level malformed-record policy -- a valid member of a malformed same-id group
> never survives; STARTUTC_FIXED_WIDTH_UTC_100NS fixed-width
> yyyy-MM-ddTHH:mm:ss.fffffffZ with typed-instant ordering and excess-precision
> fail-closed; typed duplicate_gamepk_batch_input assertion) -> `c545117`
> (frontend: providerEventId is the identity for every dev-artifact-review
> selection/tracking/count/filter/cap surface -- one selected id = one game = one
> paid execution entry; shared deterministic occurrence-label policy in
> core/matchup-event-labels.ts consumed by analyzer and both dev-artifact-review
> lists, Game N ordinals for label-identical events; identity-boundary wording
> corrected). RED: 18/20 backend + 9 frontend failures pre-fix. Suites after:
> .NET **1853/1853**, operator **187/187**, guard **40/40**, frontend **155/155**
> + build. Durable residual:
> SELECTED_EVENT_IDENTITY_PROPAGATION_REQUIRED_BEFORE_WI0037_CLOSE (the analysis
> request still carries date+teams only -- see Slice 2-iii below). NOT
> delta-reviewed, NOT integrated, NOT pushed. Evidence:
> [[wi-0037-slice-2-ii-b-adversarial-review-and-corrections-2026-07-26-v1]].

> Slice 2-ii-b state (2026-07-26, superseding "2-ii-b unauthorized"): authorized and
> implemented RED-first on local branches
> `wi/0037-game-state-correctness-slice-2-ii-b` (dai `5a11a2c` from base `841ae26`;
> vault docs from base `664cd4a`): provider event id is the authoritative
> odds-surface identity through a shared NormalizeEvents pipeline (doubleheaders
> survive both discovery paths; same-id conflicts and blank ids fail closed with
> structured logs; deterministic Date/StartUtc/id ordering); MatchupEventDto gains
> StartUtc + ProviderEventId additively (eastern Date semantics unchanged); TS
> mirror/stub/analyzer updated (pills track providerEventId, start-time labels on
> shared dates, selection keyed by id, one-per-day assumption removed); batch
> duplicate gamePk input rejected atomically before any work
> (duplicate_gamepk_batch_input). RED: 11 C# failures pre-fix + frontend static
> analysis. Suites: .NET 1831/1831, operator 187/187, guard 40/40, frontend 136/136
> + build. NOT reviewed, NOT integrated, NOT pushed. Evidence:
> [[wi-0037-slice-2-ii-b-discovery-identity-2026-07-26-v1]]. Slice 2-ii-c (F3, F4,
> requireStatus hardening, normalizer consolidation, parity vectors) remains
> unauthorized -- the final slice before WI-0037 closure; WI-0037 stays
> `in-progress`.

> Slice 2-ii-a closeout (2026-07-26, superseding "IMPLEMENTED LOCAL; review
> pending"): implemented RED-first (`f8c0962`), INDEPENDENTLY REVIEWED (verdict
> WI0037_SLICE2IIA_CORRECTIONS_REQUIRED: F-A Critical terminal-duplicate, F-B High
> production payload boundary, F-C Medium refusal observability -- all other claims
> confirmed), CORRECTED (`841ae26`, RED-first: conflicting-team duplicates now
> terminally ambiguous with the full duplicate set; production-owned
> `GameStatusPayloadReader` shared by both starter paths AND the corpus runner,
> invalid JSON keeps pre-existing per-path source-failure semantics; every typed
> refusal logged with gamePk/bracket/path/context/duplicate correlation),
> DELTA-REVIEWED (PASS), INTEGRATED and PUBLISHED by coordinated fast-forward:
> dai main `dd760f9` -> `841ae26a70cfe53465b68a937ce714fa9ec418ca` (remote
> verified), vault main `59f32e4` -> `9f2a938`/`61877e9cc3120f14371ac731b3e74f9b4e
> 5fd1cf` (remote verified). Post-publication on published dai main: .NET
> **1817/1817**, operator **187/187**, guard **40/40**; contract 1.1; 25 fixtures;
> six-reason vocabulary. Slice 2-ii-a is CLOSED. Carried 2-ii-c obligations
> (mandatory): requireStatus type-level guarantee hardening; normalizer
> consolidation (resolver/adapter alias sets currently identical, machine-diffed).
> Slice 2-ii-b (discovery: OddsScheduleClient id-keyed dedup, additive
> MatchupEventDto, batch duplicate rejection, sports-app verify incl.
> agent-run.model.ts mirror) remains NEXT and unauthorized; 2-ii-c unauthorized;
> WI-0037 stays `in-progress`.

> Slice 2-ii-a state (2026-07-26, superseding "architecture bound, NOT authorized"
> for the 2-ii-a portion only): authorized and implemented RED-first on local
> branches `wi/0037-game-state-correctness-slice-2-ii-a` (dai `f8c0962` from base
> `dd760f9`; vault docs from base `59f32e4`): contract
> `game-status-resolution/1.1` (F2 resolved normatively; 25 fixtures incl. gsr-25;
> `csharp_resolver` tags), xunit corpus runner (linked single-source corpus,
> version-gated, generic tag selection with a no-skip proof), typed
> `GameStatusResolver` (staged stages 1-5, closed refusal enum, consumer-scoped
> stage-5 `requireStatus`), structural bracket routing in `MlbStarterClient`
> (flattens removed; duplicate in-bracket pk fails closed; `MlbEventResolver`
> unchanged as matchup validator; caller signatures unchanged), and typed frozen
> `ScheduleState` slate validation ("candidate schedule state is required"; empty
> is now request-level invalid input -- one Slice-1 matrix row migrated to a
> non-empty unknown alias, documented). RED A-D preserved; suites: .NET 1811/1811,
> operator 187/187, guard 40/40. NOT reviewed, NOT integrated, NOT pushed.
> Evidence: [[wi-0037-slice-2-ii-a-contract-conformance-2026-07-26-v1]].
> Slices 2-ii-b (discovery) and 2-ii-c (parity) remain unauthorized; WI-0037 stays
> `in-progress`.

> Slice 2-ii architecture (2026-07-26, superseding "defined but unauthorized" with a
> bound implementation plan; still NOT authorized, NOT implemented): see
> [[wi-0037-slice-2-ii-architecture-review-2026-07-26-v1]]. Bound decisions:
> contract bump to `game-status-resolution/1.1` REQUIRED (absent/null/non-array
> `dates` becomes normatively `bucket_malformed`; +1 fixture gsr-25; additive
> `csharp_resolver` consumer tags; resolves F2); resolver enforcement = Option B, a
> new typed `GameStatusResolver` over the existing bucketed `MlbScheduleResponse`
> with the starter-client flattens removed (`MlbEventResolver` unchanged); frozen
> `ScheduleState` = typed slate-validation rejection (caller-state contract, not the
> gsr vocabulary); discovery identity = provider event id within provider scope
> (gamePk unavailable on the odds surface; team+date rejected), with an additive
> `MatchupEventDto` shape (StartUtc + ProviderEventId) so doubleheaders survive
> end-to-end; batch boundary = duplicate input gamePks rejected fail-closed;
> F3 = null-safe harness detail strings (2-ii-c); F4 = AUTHORITY_PLUS_CONTEXT
> (CLI live URL drops `date=`, parity with the guard's accepted broad fetch; no
> live call anywhere in 2-ii). DECOMPOSITION SELECTED: three sub-slices --
> 2-ii-a contract conformance (xunit corpus runner + 1.1 + GameStatusResolver +
> frozen-state validation + narrow PS CLI conformance), 2-ii-b discovery
> correctness (OddsScheduleClient id-keyed dedup + matchup shape + batch guard +
> first client tests), 2-ii-c operator/harness parity (F3 + F4 + parity vectors +
> doctrine sync); dependency order a -> b -> c (b and c both depend only on a).
> WI-0037 CLOSES when a+b+c complete; reconciliation CLI/admin UI are explicitly
> not prerequisites. First implementation authorization = Slice 2-ii-a.

> Slice 2-i closeout (2026-07-26, superseding "IMPLEMENTED LOCAL; review pending"):
> implemented RED-first, INDEPENDENTLY REVIEWED (verdict
> WI0037_SLICE2I_CORRECTIONS_REQUIRED with exactly one docs-only finding F1),
> CORRECTED (`87c0d06` compatibility-wording commit, delta-review PASS), INTEGRATED
> and PUBLISHED by coordinated fast-forward: dai main `0a9129b` ->
> `dd760f92e2d7b204283ffea6c31910252f1ea6e1` (remote verified), vault main `5577f55`
> -> `f1b66e7` -> `87c0d067874c40de3d6a227d0872b03ee4276935` (remote verified).
> Post-publication offline verification on published dai main: guard harness 40/40,
> operator harness 181/181; 24 fixtures; closed six-reason refusal vocabulary;
> gsr-23 resolves final with one reschedule-context record; gsr-24 remains a
> fail-closed duplicate DEFECT; zero live calls; zero C# changes. Slice 2-i is
> CLOSED. Review follow-ups carried to Slice 2-ii (recorded, NOT implemented):
> F2 scalar `dates` refuses bracket_missing vs stricter bucket_malformed
> (cross-runtime classification convergence); F3 guard-harness eager detail
> interpolation can crash the RED tally path (fix in next harness-touching slice);
> F4 CLI live mode (date=+gamePks=) reports no reschedule context vs the guard's
> broad fetch (authority-only vs authority-plus-context live semantics is a 2-ii
> architecture decision; no live call until reviewed). Slice 2-ii remains
> unauthorized and unimplemented; WI-0037 stays `in-progress`.

> Slice 2-i state (2026-07-26, superseding "NOT authorized, NOT implemented" for the
> 2-i half only): authorized and implemented RED-first on local branches
> `wi/0037-game-state-correctness-slice-2-i` (dai `dd760f9` from base `0a9129b`; vault
> docs from base `5577f55`): contract `game-status-resolution/1.0` + canonical
> 24-fixture corpus + finals-guard staged-bracket correction (optional `-BracketDate`;
> legit postponed+makeup two-bucket pairs no longer false-DEFECT; in-bracket
> duplicates stay DEFECT; exit contract unchanged) + canonical operator
> `check-game-status.ps1` + corpus conformance harness + ACTIVE doctrine
> [[game-status-recheck-discipline-v1]]. RED 6 failures preserved pre-fix; guard
> 40/40 and operator 181/181 after; zero C# changes. NOT reviewed, NOT integrated,
> NOT pushed. Evidence: [[wi-0037-slice-2-i-status-resolution-2026-07-26-v1]].
> Slice 2-ii (C# conformance: resolver seam, OddsScheduleClient DH-safe dedup,
> batch-boundary tests, null-ScheduleState rider, xunit corpus runner) remains
> unauthorized and unimplemented; WI-0037 stays `in-progress`.

> Design state (2026-07-24, superseding "defined"): full read-only surface mapping and
> design completed -- see
> [[wi-0037-slice-2-status-resolution-design-2026-07-24-v1]]. Verified conclusions:
> the C# identity core is fail-closed and pinned (resolver date scope is an implicit
> caller invariant, both callers provably date-scoped); CONFIRMED defects are (D1) the
> finals guard's bare-`gamePks=` fetch + all-bucket flatten false-DEFECTing the
> legitimate postponed+makeup two-bucket class (untested), and (D2) `OddsScheduleClient`
> `DistinctBy(Date)` collapsing legitimate same-date doubleheaders on reference
> surfaces (zero tests); the realized July 23 cost was the OPERATOR gap (no canonical
> date-bracketed status script; discipline prose-only). "Exactly one match" is a STAGED
> policy: bracket-filter first, exact gamePk within bracket, uniqueness within bracket;
> same-pk entries outside the bracket are reschedule context, not duplicates.
> Architecture: Option B -- shared versioned contract `game-status-resolution/1.0` +
> one canonical fixture corpus (24 vectors) consumed by both xunit and the PowerShell
> harness. Null frozen `ScheduleState` -> typed validation rejection (rider).
> Decomposition DECIDED: two implementation slices -- 2-i (contract + fixtures +
> finals-guard staged-bracket correction + canonical operator script + doctrine),
> then 2-ii (C# conformance: resolver seam, DH-safe dedup, batch-boundary tests,
> null-state rider, xunit corpus runner). Neither is authorized; the r7a26 hygiene
> candidate stays outside WI-0037.

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

### Slice 2-iii -- selected-event identity continuity (2-iii-c + c1 INDEPENDENTLY REVIEWED + INTEGRATED INTO LOCAL BASELINE 2026-07-27, local-only/unpublished/undeployed; 2-iii-b2 integrated; activation DEFAULT-OFF; 2-ii-c NOT authorized)

> 2-iii-c local-integration closeout (2026-07-27, superseding "2-iii-c + c1
> IMPLEMENTED LOCAL, review-required"): the complete selected-event frontend
> continuity package (2-iii-c propagation + 2-iii-c1 comment/evidence hardening) was
> INDEPENDENTLY REVIEWED and INTEGRATED INTO THE LOCAL ENGINEERING BASELINE. The
> independent adversarial review (recorded in the external prompt ledger, slice
> 2-iii-c2) returned PASS with no findings over the complete ranges
> aee1ade..baf5e90 (source) and e217a3f..c497262 (records). The reviewed source was
> integrated by a local fast-forward: dai local `main` advanced aee1ade -> **baf5e90**
> (two commits: d5d093c propagation, baf5e90 hardening); dai `origin/main` remains
> aee1ade. This vault records branch is advanced into local vault `main` by an
> atomic local ref update; vault `origin/main` remains e217a3f. NOTHING was pushed
> to any remote and NOTHING was deployed.
>
> **Engineering-completion doctrine (correcting the earlier assumption).** Remote
> publication is NOT required for engineering completion during this hardening
> phase. Engineering completion is established by independent review, deterministic
> local verification, and integration into the designated LOCAL baseline. Remote
> publication and product deployment belong to a future release program and are not
> prerequisites for ordinary local slice completion. The earlier 2-iii-c/c1
> "corrected publication sequence" (source-first remote push) is superseded for the
> purpose of resolving the propagation residual; it may be revisited if and when a
> release program is authorized.
>
> **Residual disposition.** SELECTED_EVENT_IDENTITY_PROPAGATION_REQUIRED_BEFORE_
> WI0037_CLOSE = **RESOLVED 2026-07-27 (independently reviewed and integrated into
> the local WI-0037 baseline; remote publication and deployment deferred)**. The
> client identity-loss line is closed in the integrated baseline: both sports-app
> run-creation paths carry the operator's selected providerEventId + startUtc as the
> additive null-suppressed selectedEvent intent, reaching the server-owned
> selected-event authority. MULTI_INSTANCE_SELECTED_EVENT_ATOMICITY_REQUIRED_BEFORE_
> SCALE_OUT stays **blocking before scale-out**;
> DURABLE_PREEXECUTION_SELECTION_DECISION_LEDGER_DEFERRED stays
> **deferred/nonblocking**.
>
> **Release state (tracked separately and truthfully).** LOCAL-ONLY; UNPUBLISHED;
> UNDEPLOYED; selected-event activation DISABLED (appsettings Enabled=false, blank
> DeploymentId/EvidencePath); migration 20260727133845_AddAgentRunDomainExecution
> Provenance UNAPPLIED. No product release, production readiness, deployment
> readiness, feature activation, or live selected-event execution is claimed.
>
> **Verification at integration.** Focused frontend 50/50; full frontend 207/207 (24
> files) + build clean; full .NET 2025/2025, 0 skipped; source-range diff --check
> clean; branch-delta comment audit clean (zero labels/uppercase/non-ascii in added
> comments). Warning inventory (accurate, pre-existing, not introduced by this
> frontend-only package): NU1903 high-severity advisories for Microsoft.OpenApi
> 2.0.0 and System.Security.Cryptography.Xml 10.0.7, plus pre-existing
> compiler/nullability/xUnit analyzer warnings; the earlier "no unresolved warnings"
> phrasing is superseded by this inventory. Not remediated in this slice.
>
> **Status.** Slice 2-ii-c (F3, F4, requireStatus hardening, normalizer
> consolidation, parity vectors) remains UNAUTHORIZED and is the last slice before
> WI-0037 closure. WI-0037 remains **in-progress**. Next governed action = a separate
> authorization for Slice 2-ii-c or an operator-selected evidence/capture slice, based
> on the integrated baseline (dai main baf5e90, vault main = this closeout). Evidence:
> [[wi-0037-slice-2-iii-c-frontend-consumer-continuity-2026-07-27-v1]]
> (local integration closeout section).

> 2-iii-c1 correction state (2026-07-27, on top of 2-iii-c): a narrow
> correction/evidence-hardening pass, IMPLEMENTED LOCAL and review-required, on
> the same branches (dai `wi/0037-selected-event-frontend-continuity` correction
> commit `baf5e9088aff2a23453dbdc4f3a33d5ced2b9188` on top of d5d093c; vault records
> `wi/0037-selected-event-frontend-continuity-records` new commit on top of 9cee53b). Disclosed defect (not erased): the 2-iii-c commit d5d093c added
> production and test comments carrying work-item/slice/finding labels and
> uppercase symbol names in prose, violating the lowercase-ascii code-comment
> standard (the same low-severity comment-hygiene class the 2-iii-b2 final review
> flagged). Correction: every comment added by the branch delta was rewritten to
> lowercase ascii prose with no labels and no uppercase type/symbol names;
> pre-existing comments outside the delta were untouched; an end-of-slice audit
> confirms zero labels and zero uppercase or non-ascii characters in added
> comments; no runtime semantics changed. Added evidence: consumer-boundary
> analyzer refusal tests (identity_not_active safe message; server detail never
> surfaced; exactly one selected request never retried; generic fallback for a
> non-selection error; short run id on a post-create refusal), rendered dev
> identity-panel tests (complete/partial/not-recorded; label stays "server-recorded
> game identity", never selected-event provenance; the panel never renders an
> out-of-contract raw provenance value), and non-baseball (nba) propagation on both
> paths (exact provider event id and start utc still emitted; intent not dropped by
> sport; unsupported competition refuses with no retry). Operational envelope stated
> clearly: activation off means selected requests refuse with the inactive contract
> and no legacy substitution; after activation, competitions outside the staged
> baseball-only backend still refuse as unsupported; this deliberate fail-closed
> posture requires deployment/release review before publication and is not
> authorization to add fallback or broaden backend support. Suites: focused frontend
> 50/50; full frontend 207/207 (24 files) + build clean; full .NET 2025/2025, 0
> skipped (no backend diff); git diff --check clean; comment audit clean. NO C#
> change; NO backend contract change; activation DISABLED; migration UNAPPLIED; NOT
> reviewed, NOT integrated, NOT pushed. Corrected publication sequence recorded: (1)
> independent review; (2) publish/verify source first on pass; (3) NEW docs-only
> vault closeout commit atop the records branch reflecting the verified published
> source; (4) publish/verify vault second; (5) move the propagation residual only to
> an existing canonical resolved state -- do not invent a new state name. Residual
> SELECTED_EVENT_IDENTITY_PROPAGATION_REQUIRED_BEFORE_WI0037_CLOSE stays
> RESOLUTION-PENDING-VERIFICATION; scale-out blocking; decision ledger deferred;
> 2-ii-c unauthorized; WI-0037 in-progress. Evidence:
> [[wi-0037-slice-2-iii-c-frontend-consumer-continuity-2026-07-27-v1]]
> (slice 2-iii-c1 correction section).

> 2-iii-c state (2026-07-27, superseding "2-iii-c NOT authorized"): the operator
> authorized frontend/consumer selected-event continuity and it is IMPLEMENTED
> LOCAL, review-required, on branch `wi/0037-selected-event-frontend-continuity`
> (dai, from published `aee1ade`; vault records from published `e217a3f`).
> Terminal state **WI0037_SLICE2IIIC_IMPLEMENTED_LOCAL_REVIEW_REQUIRED**. Both
> sports-app run-creation paths (`AnalyzerComponent.analyze`,
> `DevArtifactReviewComponent.runSelectedGames`) now carry the operator's selected
> `MatchupEventDto.providerEventId` + `.startUtc` as the additive null-suppressed
> `selectedEvent` intent on `CompetitionMatchupInput` (legacy requests without it
> stay byte-identical); one shared typed refusal parser/presenter
> (`core/selected-event-refusal.ts`) maps the published closed 16-code
> selection_* vocabulary to safe operator messages (phase derived from agentRunId
> presence; server detail never surfaced; unknown selection_* preserved, never a
> silent success/legacy retry); the analyzer template renders the mapped message;
> the batch retains per-entry refusal code/message and the returned agentRunId on a
> post-create refusal, never retrying; and the internal dev-artifact-review surface
> gains a read-only "Server-recorded game identity" panel over the six existing
> AgentRunArtifactDto identity fields (complete/partial/not-recorded; never labelled
> as selected-event provenance; raw provenance envelope never parsed in TS). Frontend
> stays intent-only -- no verification, normalization, case-folding, or client gamePk
> derivation. RED-first: 7 failing desired assertions pre-implementation (selectedEvent
> absent on both paths; refusal fields/agentRunId not retained). Suites: focused
> frontend **38/38** (5 new specs); full frontend **195/195** (22 files) + build clean;
> focused backend regression **56/56** (SelectedEventCreationTests +
> OutcomeReconciliationMatcherTests, no backend diff); full .NET **2025/2025, 0
> skipped** (exact published baseline); solution build 0 errors; strict snapshot
> against published vault main **26 items / 0 warnings**. NO C# production or test
> change; NO backend contract change; activation remains DISABLED; migration
> 20260727133845 remains UNAPPLIED; NOT reviewed, NOT integrated, NOT pushed. Code
> review: 0 blocking; recorded note FC-1 (by-design, not a defect) -- once integrated,
> both paths always send selectedEvent, so an activation-disabled environment (the
> default) refuses with `selection_identity_not_active` and no legacy fallback; this is
> the authorized fail-closed behavior, surfaced truthfully, and enabling it for buyers
> is a separate activation + integration decision. Residuals:
> SELECTED_EVENT_IDENTITY_PROPAGATION_REQUIRED_BEFORE_WI0037_CLOSE is now
> **RESOLUTION-PENDING-VERIFICATION** (client identity-loss line closed locally;
> resolution requires independent review, integration, publication, and verification --
> NOT resolved); MULTI_INSTANCE_SELECTED_EVENT_ATOMICITY_REQUIRED_BEFORE_SCALE_OUT
> unchanged/blocking; DURABLE_PREEXECUTION_SELECTION_DECISION_LEDGER_DEFERRED
> unchanged/nonblocking. The operator-owned WI-level glossary disposition remains
> mandatory at WI-0037 completion and is NOT performed here. Next: a separately
> authorized independent adversarial review + integrate-on-PASS of the complete
> 2-iii-c source+records package; then 2-ii-c; WI-0037 stays `in-progress`. Evidence:
> [[wi-0037-slice-2-iii-c-frontend-consumer-continuity-2026-07-27-v1]].

> 2-iii-b2 closeout state (2026-07-27, superseding "corrected local incl.
> f-b2-8; final re-review required"): the final independent review of the
> complete six-commit package PASSED --
> **WI0037_SLICE2III_B2_FINAL_REREVIEW_PASSED_INTEGRATED_PUBLISHED** -- with
> all sixteen mandatory attacks holding: package integrity (six linear text
> commits, no merges, all platform/dotnet), the F-B2-8 provider-qualified
> comparison matrix reproduced, caller ownership (all five
> ProviderGameIdentity construction sites blank-guarded and server-prepared;
> the guard code parse-free), both arrival orders and the concurrency
> matrix, F-B2-7 provider-scoped discovery, F-B2-1 row/provenance
> agreement, F-B2-2 server-bound pre-model execution, F-B2-3 run-bound
> gate-2 authority, F-B2-4 bounded activation evidence, F-B2-5 atomic-save
> fault injection, F-B2-6 dual-binding conflict, input/refusal/lifecycle
> contracts, platform/domain boundary, and the operating envelope. Full
> .NET reproduced **2025/2025, 0 skipped**; solution build 0 errors. ONE
> finding: LOW (record accuracy, non-behavioral) -- earlier report claims
> that changed production comments were uniformly lowercase and label-free
> were inaccurate: seventeen comment lines across three production files
> carry wi/finding labels and some comments use uppercase emphasis; this
> conceals no authority and violates no enforced check; disposition =
> recorded here accurately, hygiene cleanup deferred to a future slice.
> INTEGRATED + PUBLISHED: dai main fast-forwarded 1311137 ->
> **aee1ade8a27da45d845510baffaabca7068be974** (all six commits atomically;
> plain non-force main-only push; direct remote verified); vault main
> advanced over the four records commits plus the documentation-only
> closeout commit containing this update (full SHA recorded in the operator
> report and external ledger; the vault publication is claimed only after
> direct remote verification). Production selected-event activation remains
> DISABLED everywhere; migration 20260727133845 remains UNAPPLIED
> (deployment separately gated); ops deliberately NOT realigned this slice.
> Residuals exact: SELECTED_EVENT_IDENTITY_PROPAGATION_REQUIRED_BEFORE_
> WI0037_CLOSE stays OPEN/blocking until 2-iii-c completes and verifies;
> MULTI_INSTANCE_SELECTED_EVENT_ATOMICITY_REQUIRED_BEFORE_SCALE_OUT stays
> blocking; DURABLE_PREEXECUTION_SELECTION_DECISION_LEDGER_DEFERRED stays
> deferred/nonblocking. The operator-owned glossary disposition for all
> WI-local terms remains mandatory at WI-0037 completion. Next: a
> separately authorized Slice 2-iii-c (frontend/consumer continuity); then
> 2-ii-c; WI-0037 stays `in-progress`.

> F-B2-8 correction state (2026-07-27, superseding "corrected incl. f-b2-7;
> final re-review required" for the duplicate-comparison contract only):
> staff review found F-B2-8 High -- discovery became provider-scoped but
> DuplicateRunGuard still compared external ids namespace-blind and parsed
> any numeric ExternalGameId, so inside normal scope a same-number
> different-provider row falsely blocked (and disclosed its AgentRunId) and
> different-number different-provider rows skipped the canonical-team
> fallback and ADMITTED a duplicate of the same physical game. CORRECTED by
> dai `aee1ade8a27da45d845510baffaabca7068be974` ("fix(agent-runs): qualify
> duplicate identity by provider"): typed
> ProviderGameIdentity(SourceProvider, ExternalGameId) carried by
> DuplicateRunIdentity and DuplicateRunCandidate; KnownGamePk and ALL
> numeric parsing removed from the guard (opaque ids are valid identities;
> shape carries no authority); bound decision table -- same source: id
> equality decides and different ids stay distinct with NO team fallback
> (doubleheader doctrine); different sources or incomplete pair: ids
> incomparable, canonical unordered-pair fallback fail-closed; exact
> ordinal comparisons over server-prepared values. Caller preparation
> sports-owned: selected sides use gate-1/classified agreed pairs; legacy
> uses persisted (SourceProvider, ExternalGameId) when both exist, else
> pending GamePk = mlb_statsapi ONLY under the existing WI-0009 mlb
> contract, else unknown -> pair fallback; both arrival orders pinned.
> F-B2-7 query rule preserved verbatim. RED-first: three genuine in-scope
> failures pre-fix (false cross-provider block + disclosure; admitted
> cross-provider duplicate; unknown-provider handling). Suites: full .NET
> **2025/2025, 0 skipped** (2017 + 8); build clean; guard parse scans
> clean; activation disabled; migration unapplied. Semantic disposition:
> "provider-qualified duplicate identity" MERGED with "provider-scoped
> candidate identity" as one WI-local concept over the governing pair,
> retained for the WI-completion glossary pass. NOT re-reviewed, NOT
> integrated, NOT pushed; the six-commit package 1311137..aee1ade is ONE
> atomic integration unit. Evidence:
> [[wi-0037-slice-2-iii-b2-backend-activation-2026-07-27-v1]] (f-b2-8
> section). Residuals unchanged. Next: final independent b2 re-review over
> 1311137..aee1ade, integrate-on-PASS; WI-0037 stays `in-progress`.

> F-B2-7 correction state (2026-07-27, superseding "corrections applied;
> delta re-review required" for the candidate-scope rule only): staff review
> found F-B2-7 Medium -- the f-b2-1 candidate-query widening keyed on
> ExternalGameId ALONE (although the governing game/settlement identity is
> the pair (SourceProvider, ExternalGameId)) and fell back to the CLIENT
> legacy GamePk, so a same-tenant row from another competition/provider with
> a colliding numeric id could be pulled in, falsely block creation, and
> disclose its AgentRunId, and a client value could steer the widening.
> CORRECTED by dai `b6aae1c84726995f19751d7b888e5b446bb49ea0`
> ("fix(agent-runs): scope duplicate candidates by provider identity"):
> widening exists ONLY for a selected request after gate 1 and its key is
> the EXACT verified (SourceProvider, ExternalGameId) pair in both the
> query predicate and the competition-filter bypass; a client legacy GamePk
> never widens discovery (legacy scope and policy unchanged); the selected
> path uses selectedResolution.GameDate/.Competition as the authoritative
> values; every completed b2 behavior preserved and pinned. Honest
> discoverability boundary bound: normal tenant/competition/date scope OR
> the exact verified pair -- a row corrupted in BOTH is not discoverable
> here (atomic insert prevents that state in normal creation;
> historical/manual repair out of scope). RED-first: three genuine
> failures pre-fix (cross-provider and cross-competition numeric collisions
> blocked/disclosed; client-GamePk widening). Suites: full .NET
> **2017/2017, 0 skipped** (2011 + 6); build clean; activation disabled;
> migration unapplied. NOT re-reviewed, NOT integrated, NOT pushed; the
> five-commit package 1311137..b6aae1c is ONE atomic integration unit.
> Evidence: [[wi-0037-slice-2-iii-b2-backend-activation-2026-07-27-v1]]
> (f-b2-7 section). Residuals unchanged. Next: final independent b2
> re-review over 1311137..b6aae1c, integrate-on-PASS; WI-0037 stays
> `in-progress`.

> 2-iii-b2 correction state (2026-07-27, superseding "implemented local;
> independent review required"): the final independent review returned
> **WI0037_SLICE2III_B2_REVIEW_CORRECTIONS_REQUIRED** (F-B2-1 High row/
> provenance agreement unchecked and classifier output ignored; F-B2-2 High
> post-gate-2 retrieval grounded via client team names with only a post-model
> ExternalGameId check; F-B2-3 Med providerEventId-only execution-seam check;
> F-B2-4 Med unbounded/semantically-unused evidence freshness; F-B2-5 Low
> save-failure seam untested; F-B2-6 Low dual selected/binding divergence
> unchecked; passed dispositions PROVIDER_NAMESPACE_DURABLE and
> CLIENT_DATE_CROSSCHECK_ONLY not reopened). ALL SIX CORRECTED with two new
> commits on the same branch (nothing amended): Commit C
> `8d2d0642cfd78cc5040c555fe000de195935d40d` -- ClassifyCandidateRow complete
> row/document agreement with bound internal detail
> row_provenance_identity_disagreement, pk-widened tenant candidate query,
> duplicate path consumes the AGREED classifier identity, gate-1 freezes the
> server's own canonical binding wire (ServerBindingWire + payload
> fingerprint), selected retrieval runs on a separate authoritative internal
> input (observed names/canonical competition/verified date+pk/server wire;
> client descriptions never select evidence; InputJson untouched), and the
> LAST pre-model gate compares the retrieved identity to the complete frozen
> bundle before any analyzer call (SelectedExecutionIntegrityException ->
> truthful failed run, zero model calls), dual selected/binding event
> divergence refuses selection_binding_conflict; Commit D
> `0b523a562d5c1f1896ed613d2b4071176815d4e6` -- gate 2 MINTS the run-bound
> SelectedExecutionAuthority consumed by the execution seam with full
> request/authority coherence before retrieval (run id, event id, start,
> canonical competition, date, client pk), activation evidence freshness
> bounded (5-min skew, 24h max age and validity window, expiry strictly after
> observation, structured {artifact, reference} citations; runbook updated),
> deterministic SaveChanges failure injection proves the atomic insert
> (no row, no execution, gate released), and selected persistence uses the
> server-confirmed resolution GameDate. RED-first per finding (the two High
> admission paths captured failing pre-fix). Suites: full .NET **2011/2011,
> 0 skipped** (1977 + 34); build clean; activation disabled; migration
> unapplied. NOT re-reviewed, NOT integrated, NOT pushed; the four-commit
> package 1311137..0b523a5 remains ONE atomic integration unit. Evidence:
> [[wi-0037-slice-2-iii-b2-backend-activation-2026-07-27-v1]] (corrections
> section). Residuals unchanged. Next: delta re-review of b4734aa..0b523a5
> plus the full-chain re-run, then integrate-on-PASS; WI-0037 stays
> `in-progress`.

> 2-iii-b2 state (2026-07-27, superseding "foundation integrated; b2 not
> authorized" for the b2 portion only): the operator authorized the verified
> selected-event backend activation batch and it is IMPLEMENTED LOCAL on
> branch `wi/0037-selected-event-backend-activation` (dai, from published
> `1311137`), two commits forming one atomic publication unit: Commit A
> `9f12d2dce012bc5dc44ffa1c7751372a0e527954` ("feat(sports): add verified
> selected event resolution") -- server-owned provider observation seam
> (full-bracket, catalog-derived namespace, opaque ordinal event id, typed
> missing/malformed/conflict/transport outcomes over one shared
> normalization core), canonical translation through ProviderEventQualifier,
> staged GameStatusResolver verification, immutable
> VerifiedSelectedEventResolution (complete six-field bundle; selection
> namespace never conflated with identity SourceProvider), sports-owned
> validated provenance builder proving every document through the generic
> TryRead gate (SELECTED_RUN_PROVENANCE_VALIDATED_WRITER_CONTRACT_V1
> fulfilled), and the gate-2 compare-not-replace seam; Commit B
> `b4734aa10cd631bcf905c678fcb5028c09f1d654` ("feat(agent-runs): enforce
> verified selected event creation") -- nullable null-suppressed
> SelectedEvent intent on CompetitionMatchupInput (legacy byte-identical,
> malformed never falls to legacy), typed DEFAULT-OFF activation gate
> requiring current external deployment evidence with ten individually
> required topology assertions ([[selected-event-activation-evidence-v1]]),
> gate-1 fully before the shared process-local tenant+competition creation
> gate (SHARED_TENANT_COMPETITION_CREATION_GATE_V1; no client field in the
> key; no network under the gate), part-8 four-arm candidate classification
> at the creation boundary (database NULL sole legacy route; active
> malformed candidates never skipped; typed 409
> duplicate_candidate_identity_invalid, no mutation/no new run),
> CROSS_PATH_CANONICAL_DUPLICATE_IDENTITY_V1 (verified-pk-first,
> NormalizeTeamRef-single-sourced canonical pair, selected candidates read
> authoritative persisted refs), ATOMIC_VERIFIED_GAME_IDENTITY_BUNDLE_V1
> insert (six fields + gamePk + provenance in ONE SaveChanges under the
> gate; AssignDomainExecutionProvenance exactly once), gate-2 before any
> model call with truthful failed-run outcomes, the ApplyGameIdentity split
> (selected runs never overwrite creation-time identity), and the
> IVerifiedSelectedExecution seam (the ordinary execution path throws on
> selected requests -- direct callers cannot bypass verification). Suites:
> full .NET **1977/1977, 0 skipped** (baseline 1896 + 81 new incl. the
> mandatory concurrency/branch matrix); solution build clean; no frontend/
> migration/dependency change; the foundation migration remains UNAPPLIED;
> production activation remains DISABLED everywhere. Code review: zero
> blocking; two recorded notes (atomicity fault-injection limitation;
> guard normalization migrated to the canonical authority). NOT reviewed,
> NOT integrated, NOT pushed; the A+B package is the atomic b2 publication
> unit. Evidence: [[wi-0037-slice-2-iii-b2-backend-activation-2026-07-27-v1]].
> Residuals unchanged (propagation OPEN/blocking -- b2 alone does not close
> it, 2-iii-c frontend propagation remains; scale-out blocking; decision
> ledger deferred). Next: independent review-and-integrate-on-PASS of the
> b2 batch; then 2-iii-c as the next separately authorized slice; WI-0037
> stays `in-progress`.

> Foundation review/integration state (2026-07-27, superseding "implemented
> local; independent review required"): the independent adversarial review of
> Commit A, Commit B, and the combined delta PASSED with the mandatory staff
> attacks resolved: RED independently reproduced at base (exactly 7 ownership
> failures / 15); wire emitters proven byte-untouched across the whole chain;
> matcher attack verdict **MATCHER_EVIDENCE_LOCATION_ACCEPTABLE_WITH_BOUND_NOTE**;
> provenance write-seam verdict **RAW_STORAGE_AND_VALIDATED_DOMAIN_WRITER_BOUND**;
> JsonElement.WriteTo empirically proven whitespace-normalizing (semantic
> preservation only), so the preauthorized comment-correction commit dai
> `13111375b9257c1542eb2861df9c08c02163f983` ("docs(sports): correct
> extraction and payload comments", comments-only, 2 files) fixed the "moved
> verbatim" and payload-"verbatim" wording; full .NET 1896/1896 and clean
> solution build re-verified with visible counts on the corrected tip. The
> batch is INTEGRATED by fast-forward: dai main `af59853` -> `1311137`
> (19fbc77 matcher, 63c7009 inert provenance, 1311137 comments) and vault
> main advanced over the records commit `416ab74` plus the documentation-only
> integration-closeout commit containing this update (full SHA recorded in
> the operator report and external prompt ledger; publication is claimed only
> after direct remote verification of both mains, plain non-force, main-only).
>
> **Bound note (matcher evidence location).** `ProviderEventBinding` is shared
> sports evidence co-located with its frozen wire in the agent-runs boundary
> for byte-compatibility; the canonical matching decision executes zero
> market-contrast, gate, run-creation, filesystem, or model code (the record's
> wire serialization helper is a lazily invoked pure string escaper and a
> candidate for relocation to a neutral home in an optional future hygiene
> slice -- not required). Selected-event translation (b2/c) must depend only
> on the canonical matcher, the evidence record, and the strict wire
> validator -- never on gate/adapter/orchestration types; b2 review re-checks
> this boundary.
>
> **Bound: SELECTED_RUN_PROVENANCE_VALIDATED_WRITER_CONTRACT_V1 (2-iii-b2
> obligation).** The entity stores the assigned document raw and
> byte-verbatim; write-side integrity is owned by the sports builder: a newly
> created selected run may persist a domain-execution provenance document
> ONLY when (1) generic structural TryRead accepts the envelope; (2) the
> sports-owned selected-event payload validation accepts the content
> (complete authoritative bundle, known schemaVersion); (3) the document was
> assembled by the sports-owned builder -- generic platform code never
> assembles one; (4) any failure refuses fail-closed with no run created and
> no document persisted. Mandatory b2 RED contracts: malformed json refused;
> non-object envelope refused; missing generic metadata refused; duplicate or
> extra generic members refused; unvalidated sports payload refused;
> generic-assembly impossible by construction. Malformed HISTORICAL rows
> continue to materialize for audit and part-8 fail-closed classification --
> never repaired. Until b2, no production writer exists (grep-proven inert).
>
> Suites at integration: full .NET **1896/1896, 0 skipped**; solution build
> 0 errors; frontend not applicable by proof. Residuals unchanged
> (propagation OPEN/blocking until b2 + c complete and verify; scale-out
> blocking; decision ledger deferred). 2-iii-b2 (atomic activation: shared
> creation gate + four-arm classification + gate-2 + validated writer) is the
> NEXT separately authorized implementation batch; 2-iii-c and 2-ii-c remain
> unauthorized; WI-0037 stays `in-progress`. Evidence:
> [[wi-0037-slice-2-iii-foundation-implementation-2026-07-27-v1]] (review and
> integration closeout section).

> Foundation-batch state (2026-07-27, superseding "architecture published;
> implementation not authorized" for the 2-iii-a and 2-iii-b1 portions only):
> the operator authorized and this batch implemented BOTH foundation slices
> RED-first on local branch `wi/0037-selected-event-foundation` (dai, from
> published `af59853`): Commit A `19fbc77c4db9884609ae752041f4a11d99d3dc85`
> ("refactor(sports): centralize provider event game matching") moves the
> WI-0035 binding rule family to the sports domain as the canonical matcher
> (`Sports/ProviderEventQualifier.cs` -- the ADR role
> "ProviderEventGameBindingMatcher" realized by the retained established
> production type; single-definition predicates now shared by the decision
> and the wire content contract; wire emitters byte-untouched; RED = 7
> ownership failures at base); Commit B
> `63c70099a095383d96df9cbbbf3d6632ba145ed8` ("feat(agent-runs): add inert
> domain provenance storage") adds nullable single-assignment
> `AgentRun.DomainExecutionProvenanceJson` (private setter; second
> assignment fails deterministically preserving original bytes), the opaque
> generic `DomainExecutionProvenanceEnvelope` {domain, type, schemaVersion,
> payload}, and the additive UNAPPLIED migration
> `20260727133845_AddAgentRunDomainExecutionProvenance` (nvarchar(max)
> nullable, precedent-identical; snapshot consistent via
> has-pending-model-changes; no database accessed). Nothing writes or reads
> the field in production; no SelectedEventIntent; no controller/frontend/
> guard/dependency change. Suites: full .NET **1896/1896, 0 skipped**
> (baseline 1853 + 43 new); solution build clean; frontend proven not
> applicable (zero non-platform paths in the delta). Code review: zero
> blocking findings; one recorded comment-precision note ("moved verbatim"
> vs predicate extraction) carried to the independent review. NOT reviewed,
> NOT integrated, NOT pushed; integration readiness NOT claimed until the
> independent review-and-integrate-on-PASS authorization passes. Semantic
> dispositions recorded (domain-execution-provenance promoted to
> [[glossary]]; matcher role/type distinguished). Evidence:
> [[wi-0037-slice-2-iii-foundation-implementation-2026-07-27-v1]].
> Residuals unchanged (propagation OPEN/blocking -- the foundation batch
> does NOT resolve it; scale-out blocking; decision ledger deferred).
> Next: independent review; then 2-iii-b2 as the next separately
> authorized implementation batch; WI-0037 stays `in-progress`.

> Publication state (2026-07-27, superseding "ADR part 8 bound local;
> closing delta review required"): the final independent closing review
> returned **WI0037_SLICE2III_ADR8_CLOSING_REVIEW_PASSED_PUBLICATION_READY**
> with no findings, authorizing publication of the complete reviewed
> nine-commit architecture package 234d3f0d25ac462f4acdd3e62842995d7f72aae0
> .. `d78545c9f23aabcc4f161fef542f25d1a31e8dad` (eight
> architecture/ADR commits through part 8 plus one
> record-conformance commit). The package tip was integrated into vault main
> by verified fast-forward only and the ARCHITECTURE IS PUBLISHED by the
> documentation-only publication-closeout commit containing this update
> (its full SHA is recorded in the post-commit operator report and the
> external prompt ledger; publication is claimed only after direct remote
> verification). Publication publishes obligations, not activation: no dai
> source changed (dai main remains `af59853`), implementation remains NOT
> authorized, 2-ii-c remains unauthorized, and every implementation slice
> (2-iii-a matcher, 2-iii-b1 inert provenance migration, 2-iii-b2 atomic
> activation, 2-iii-c frontend/consumers) is separately gated. Publication
> does NOT resolve the propagation residual:
> SELECTED_EVENT_IDENTITY_PROPAGATION_REQUIRED_BEFORE_WI0037_CLOSE stays
> OPEN and blocking until full 2-iii implementation and verification;
> MULTI_INSTANCE_SELECTED_EVENT_ATOMICITY_REQUIRED_BEFORE_SCALE_OUT stays
> blocking before any scale-out authorization;
> DURABLE_PREEXECUTION_SELECTION_DECISION_LEDGER_DEFERRED stays deferred and
> nonblocking. Published architecture evidence:
> [[wi-0037-slice-2-iii-architecture-review-2026-07-26-v1]] (publication
> closeout section). Next governed action = a separate operator
> authorization for Slice 2-iii-a; WI-0037 stays `in-progress`.

> Part-8 state (2026-07-27, superseding "part 7 bound"): closing review found
> CR-1 Med -- part 7 section 7.1's third arm ("without selected-event
> provenance -> legacy fallback") was ambiguous for non-null envelopes that
> cannot be recognized as the authorized sports selected-event document. ADR
> part 8 binds the exhaustive FOUR-ARM total classification: (a) database
> NULL = the ONLY legacy route (empty/whitespace/json-null/empty-object/
> malformed/missing-metadata are NOT legacy; no IsNullOrWhiteSpace
> semantics); (b) recognized authorized selected document (exact/ordinal
> domain=sports + type=selected_event_binding, known schemaVersion, complete
> bundle) -> typed authoritative candidate; (c) recognized selected document
> with invalid content -> 409 duplicate_candidate_identity_invalid (existing
> internal detail reasons); (d) EVERY other non-null value incl. well-formed
> foreign domain/type or unrecognized identifiers -> 409 with internal
> `unrecognized_provenance_document`, never legacy. Eligibility-before-
> classification, part-7 refusal/durability/disclosure rules, and all part-6
> identity/deployment/lifecycle/residual decisions preserved; eleven-item b2
> RED contract extended per part 8.4; forward-compatibility fail-closed
> consequence recorded; report OKF front matter aligned (evidence-report /
> in-progress / repos.dai-vault docs-only; date, filename, title preserved);
> prompt-ledger pre-execution record created at the resolved
> <OBSIDIAN_PROMPT_LEDGER_ROOT>. NOT published; implementation NOT
> authorized; 2-ii-c unauthorized; WI-0037 in-progress. Evidence:
> [[wi-0037-slice-2-iii-architecture-review-2026-07-26-v1]] part 8.

> Part-7 state (2026-07-27, superseding "part 6 bound"): staff review found
> FR-9 Med -- part 6 section 6.5's "durably explain"/failure-mechanism wording
> conflicted with the part-3 pre-create durability bound, and eligibility
> ordering vs the guard's exclusion/failed doctrine was unbound. ADR part 7
> binds **PRECREATE_DUPLICATE_CANDIDATE_INTEGRITY_REFUSAL_V1**: eligibility
> BEFORE identity parsing (excluded/failed candidates nonblocking by status
> doctrine, never by tolerating malformed identity); every potentially
> blocking candidate safely classified before any duplicate verdict; an
> active malformed selected candidate is NEVER skipped; typed 409
> `duplicate_candidate_identity_invalid` with NO incoming run, NO provenance
> for the refused attempt, no candidate mutation, no external/paid work,
> gate always released; internal detail reasons (incomplete_identity_bundle /
> malformed_provenance_envelope / unknown_provenance_schema_version) in
> correlation-linked observability only; durability = response + log exactly
> per PRECREATE_REFUSAL_DURABILITY_LIMITED_TO_RESPONSE_AND_OBSERVABILITY
> (part 6's "durably explain" wording superseded); eleven mandatory b2 RED
> scenarios. All part-6 identity decisions and all three residual states
> preserved. NOT published; implementation NOT authorized; 2-ii-c
> unauthorized; WI-0037 in-progress. Evidence:
> [[wi-0037-slice-2-iii-architecture-review-2026-07-26-v1]] part 7.

> Part-6 state (2026-07-26, superseding "part 5 bound"): staff review found
> FR-6 High (part 5 persisted ExternalGameId + team refs WITHOUT
> SourceProvider/ScheduledStartUtc/Season -- a partial settlement identity
> against the (SourceProvider, ExternalGameId) key and the six-field
> GameIdentityContext bundle), FR-7 Med (conversion semantics unpinned across
> the two existing normalizers; selected-candidate classification
> underspecified), FR-8 Med (the propagation residual is NOT resolved by
> architecture publication). ADR part 6 binds:
> **ATOMIC_VERIFIED_GAME_IDENTITY_BUNDLE_V1** -- Gate 1 constructs the
> complete immutable six-field bundle (+ Competition + GameDate) before run
> creation, persisted all-or-none in the initial atomic insert; settlement
> key stays exactly (SourceProvider, ExternalGameId); selection
> providerNamespace (odds) never conflated with game SourceProvider
> (mlb_statsapi); Gate 2 = COMPARE-NOT-REPLACE (typed failed-run outcome on
> any authoritative mismatch, before model; ApplyGameIdentity mutation seam
> split for selected runs); canonical conversion authority pinned =
> GameIdentityDerivation.NormalizeTeamRef semantics (single implementation
> consumed by binding AND duplicate preparation; screened-workflow scope;
> characterization over every supported name before retiring the old
> normalizer; no alias guarantees); selected-candidate classification via
> envelope metadata in sports-owned builder code (fail-closed on incomplete
> bundles or unknown schema versions; malformed candidates never skipped);
> part-5 truth table preserved; residuals corrected --
> SELECTED_EVENT_IDENTITY_PROPAGATION_REQUIRED_BEFORE_WI0037_CLOSE remains
> OPEN and blocking until full 2-iii implementation/verification; scale-out
> residual retained blocking; decision-ledger residual retained nonblocking.
> NOT published; implementation NOT authorized; 2-ii-c unauthorized; WI-0037
> in-progress. Evidence:
> [[wi-0037-slice-2-iii-architecture-review-2026-07-26-v1]] part 6.

> Part-5 state (2026-07-26, superseding "part 4 bound"): staff review found
> FR-4 High (the shared gate ORDERS checks but does not equalize duplicate
> IDENTITY -- the guard's fallback compares InputJson-sourced team pairs,
> DuplicateRunGuard.cs:36-43/84-89, which a selected run does not
> authoritatively provide) and FR-5 Medium (the two-spelling RED scenario was
> overbroad vs the divergence policy). ADR part 5 binds
> **CROSS_PATH_CANONICAL_DUPLICATE_IDENTITY_V1**: a typed sports-prepared
> duplicate identity (known-pk equality first; otherwise ONE canonical
> unordered team-reference pair single-sourced across guard and provider
> normalization, screened-workflow scope, no alias resolution); selected-run
> atomic insert persists verified gamePk + canonical competition/date +
> server-derived team refs into the EXISTING authoritative fields
> (ExternalGameId, HomeTeamRef/AwayTeamRef) so candidate queries read
> authoritative rows, never selected InputJson teams and never parsed
> provenance; candidate-source precedence bound; the full selected/legacy/
> doubleheader concurrency truth table replaces part 4's unqualified claim
> (fail-closed on unknown-pk DH identity; same-verified-pk condition on the
> two-spelling scenario; ten corrected RED scenarios). All other part-4
> decisions preserved (shared gate key, default-off activation evidence gate,
> extended residual, single-assignment provenance, two-gate freshness,
> resubmission-only, four slices). NOT published; implementation NOT
> authorized; 2-ii-c unauthorized; WI-0037 in-progress. Evidence:
> [[wi-0037-slice-2-iii-architecture-review-2026-07-26-v1]] part 5.

> Part-4 state (2026-07-26, superseding "constraints bound local"): the final
> delta review returned CORRECTIONS_REQUIRED (FR-1 High: the creation gate is
> keyed from raw client team strings, AgentRunsController.cs:129-131, so
> client-varied spellings could defeat serialization; FR-2 Med activation
> enforceability; FR-3 Med provenance cardinality). ADR part 4 (current
> authority) binds: **SHARED_TENANT_COMPETITION_CREATION_GATE_V1** -- ONE
> shared process-local gate for legacy AND selected paths keyed by
> server-authenticated tenant + server-canonical competition (no client
> fields; split-key strategy explicitly rejected because it would not
> serialize a legacy/selected race on the same game; only the db
> read/check/insert boundary is held, no network I/O under the gate; selected
> path dedups with the VERIFIED gamePk; seven named 2-iii-b2 RED scenarios);
> **enforceable activation gate** -- 2-iii-b2 DEFAULT-OFF until a deployment
> evidence record proves exactly one active run-creating process (Dockerfile
> is one-process-per-container but topology is otherwise unproven;
> compose.smoke.yaml non-production); architecture publication = publishing
> an obligation, not activation; residual
> MULTI_INSTANCE_SELECTED_EVENT_ATOMICITY_REQUIRED_BEFORE_SCALE_OUT extended
> to same-host multi-process, deployment overlap, multi-container, and future
> workers; **ONE_IMMUTABLE_DOMAIN_DOCUMENT_PER_RUN** -- zero-or-one
> single-assignment document, complete before insert, never merged/appended/
> replaced; future evidence types = new-run aggregates or a separately
> designed append-only surface, never in-place mutation. NOT published,
> implementation NOT authorized; 2-ii-c unauthorized; WI-0037 in-progress.
> Evidence: [[wi-0037-slice-2-iii-architecture-review-2026-07-26-v1]] part 4.

> Part-3 state (2026-07-26, superseding "architecture corrected local"): the
> delta review of part 2 returned CORRECTIONS_REQUIRED with seven bindable
> findings (DF-1..DF-7). ADR part 3 (current authority) binds:
> CREATION_GATE_PROCESS_LOCAL_SINGLE_INSTANCE_CONSTRAINT with named residual
> **MULTI_INSTANCE_SELECTED_EVENT_ATOMICITY_REQUIRED_BEFORE_SCALE_OUT** (no
> durable owner yet; assignment required before any scale-out authorization);
> SINGLE_CUTOVER_DEPLOYMENT_ASSUMED with four phases and rollback rules;
> VERIFIED_CANDIDATE_STALENESS_ACCEPTED_WITH_TWO_GATE_VERIFICATION (Gate 2 =
> retrieve-time staged re-verification; failures fail the run, never execute
> another game); CONCURRENT_SELECTION_DIVERGENCE_VERSIONED_WITHIN_SINGLE_HOST_
> BOUND (execution dedup = verified gamePk; selection-level idempotency NOT
> claimed); persistence form DECIDED =
> GENERIC_OPAQUE_DOMAIN_PROVENANCE_EXTENSION_SELECTED (generic nullable
> `DomainExecutionProvenanceJson` envelope {domain, type, schemaVersion,
> payload}; platform stores opaquely, sports owns the payload; one additive
> migration; sports-column-on-generic-row and child-record rejected); evidence
> schemaVersion/evolution/immutability bound (unknown versions fail closed for
> authoritative reuse); lifecycle = RESUBMISSION_ONLY_CURRENTLY (no same-run
> retry path exists; retry semantics are future-proofing only);
> PRECREATE_REFUSAL_DURABILITY_LIMITED_TO_RESPONSE_AND_OBSERVABILITY with
> nonblocking residual **DURABLE_PREEXECUTION_SELECTION_DECISION_LEDGER_
> DEFERRED**; final decomposition = 2-iii-a matcher / 2-iii-b1 inert
> provenance migration / 2-iii-b2 atomic activation / 2-iii-c consumers. NOT
> published, NOT approved, implementation NOT authorized, scale-out NOT safe,
> pre-run refusal learning NOT complete; 2-ii-c unauthorized; WI-0037
> in-progress. Evidence:
> [[wi-0037-slice-2-iii-architecture-review-2026-07-26-v1]] part 3.

> Architecture-correction state (2026-07-26, superseding "architecture reviewed
> local"): the independent adversarial architecture review returned
> **WI0037_SLICE2III_ARCHITECTURE_REVIEW_CORRECTIONS_REQUIRED** (AF-1 High:
> translation placed after DuplicateRunGuard + run creation would 409 the second
> doubleheader selection under the guard's fail-closed matchup rule; AF-2/3/4/5
> Medium: durable verified-evidence home unspecified, provider namespace not
> persisted, activation gap, replay classes conflated; L-1 Low serialization
> pin). Corrected in ADR part 2 (part 1 preserved as history): canonical design
> is now **E-PRIME-PRECREATE** -- SERVER_TRANSLATION_BEFORE_DUPLICATE_GUARD
> (validate -> observe -> translate via the canonical
> ProviderEventGameBindingMatcher -> staged verification -> creation gate ->
> pk-conflict check -> guard with the VERIFIED pk -> persist evidence -> create
> run -> execute; translation I/O outside the gate); durable evidence home
> DECIDED = new nullable run-row column (sketch `SelectedEventBindingJson`,
> PromptRouteProvenanceJson migration precedent) --
> **NEW_DURABLE_FIELD_AND_MIGRATION_REQUIRED**, "zero schema change" withdrawn;
> server-derived provider namespace frozen per run; three replay classes bound
> (same-run retry reuses frozen evidence; resubmission re-resolves; audit replay
> never contacts providers; cross-run rebinding = permit-with-provenance);
> internal-first decomposition (2-iii-a translator, 2-iii-b atomic contract
> activation + enforcement + evidence column, 2-iii-c frontend/consumers) with
> mandatory pre-activation refusal `selection_identity_not_active` and
> all-or-none intent semantics (`selection_intent_malformed`). NOT
> delta-reviewed, NOT published, implementation NOT authorized; 2-ii-c
> unauthorized; WI-0037 in-progress. Evidence:
> [[wi-0037-slice-2-iii-architecture-review-2026-07-26-v1]] part 2.

> Architecture-review state (2026-07-26, superseding "defined, unauthorized"):
> the architecture review is COMPLETE LOCALLY on branch
> `wi/0037-selected-event-identity-continuity-architecture` (base `234d3f0`).
> Key finding: the platform already owns every load-bearing mechanism -- the
> request contract's optional WI-0009 `GamePk` (server-verified via the 2-ii-a
> staged resolver with terminal DH ambiguity), the gamePk-aware
> DuplicateRunGuard, the run row's canonical settlement identity
> (SourceProvider, ExternalGameId), and the WI-0035/36 frozen binding wire +
> trust boundary. The sole gap is frontend selection -> request (identity-loss
> line `analyzer.component.ts:649` / `dev-artifact-review.component.ts:463`).
> RECOMMENDED (Candidate E-prime): reuse the existing gamePk execution
> authority; add additive null-suppressed SelectedEvent intent fields
> (providerEventId + StartUtc); translate intent -> gamePk SERVER-SIDE during
> retrieval using the WI-0035 rule family; new closed selection-refusal
> vocabulary (never overloading game-status-resolution codes); zero migrations;
> legacy clients keep WI-0006 fail-closed DH ambiguity (never guess).
> Decomposition: 2-iii-a contract+trust boundary, 2-iii-b server translation,
> 2-iii-c frontend propagation + provenance surfacing. Candidates B (selection
> token) and C (binding registry) rejected -- no registry exists and no
> correctness gain; D (new command) duplicates the pipeline. Evidence:
> [[wi-0037-slice-2-iii-architecture-review-2026-07-26-v1]]. NOT independently
> reviewed, NOT published, implementation NOT authorized; WI-0037 in-progress.

> Defined during the 2-ii-b correction pass (F-F); architecture work, NOT
> implementation-ready. Purpose: bind the provider event the operator selected to
> the analysis execution request and durable run provenance without duplicating or
> bypassing the existing provider-event binding architecture (WI-0035/WI-0036).
> Motivating residual: SELECTED_EVENT_IDENTITY_PROPAGATION_REQUIRED_BEFORE_WI0037_CLOSE
> -- the analysis-request payload carries date and teams only, so two distinct
> provider selections (a doubleheader pair) may still serialize to equivalent
> analysis requests; count parity does not prove semantic run identity. Changing
> that payload may affect WI-0035/WI-0036 provider binding, stored runs, API
> compatibility, reconciliation provenance, and other callers, so it requires its
> own architecture-binding review before any implementation. Slice 2-iii must be
> reviewed and completed before WI-0037 closes, and is deliberately NOT folded into
> Slice 2-ii-c, which retains its existing obligations (F3, F4, requireStatus
> hardening, normalization consolidation, cross-runtime parity).

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

Parent-close criteria: **SATISFIED 2026-07-28.** Both slices (and every sub-slice) were
independently implemented, reviewed, and integrated under their own authorizations with the
proofs above; the final Slice 2-ii-c + c1 package passed independent adversarial review
(verdict WI0037_SLICE2IIC_C1_INDEPENDENT_REVIEW_PASSED_LOCAL_INTEGRATION_READY, zero
findings) and was integrated by pure local fast-forward to dai main
`4c6cd985662caebec686d4b96a12a4104440e857`; the mandatory glossary disposition pass is
recorded in the WI-0037 CLOSEOUT block above; the selected-event propagation residual is
resolved; scale-out atomicity and the durable ledger remain visible with their exact
meanings; the state stays local-only, unpublished, undeployed, activation-disabled, and
migration-unapplied.

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

- work item: WI-0037 (ADO: AB#- when wired; no ADO item created; local-spine mode per
  [[work-item-traceability]])
- branch: planning `wi/0037-game-state-correctness`; Slice 1
  `wi/0037-game-state-correctness-slice-1`; 2-i `...-slice-2-i`; 2-ii-a `...-slice-2-ii-a`;
  2-ii-b `...-slice-2-ii-b`; 2-iii `wi/0037-selected-event-foundation`,
  `wi/0037-selected-event-backend-activation`, `wi/0037-selected-event-frontend-continuity`;
  2-ii-c `wi/0037-game-state-correctness-slice-2-ii-c` (dai) and
  `wi/0037-game-state-correctness-slice-2-ii-c-records` (vault records); all retained
- pr: no GitHub PR. Published slices (through 2-iii-b2) merged direct by coordinated
  fast-forward + plain non-force push: dai `origin/main` = `aee1ade`, vault `origin/main` =
  `e217a3f`. Local-only slices merged direct by fast-forward, UNPUSHED: 2-iii-c dai
  `aee1ade..baf5e90`; **this closeout: 2-ii-c + c1 dai `baf5e90..4c6cd98`** (source local
  main fast-forward) and the vault records branch advanced into local vault `main` by an
  atomic ref update ending at the closeout commit below
- commits: Slice 1 dai `0a9129b`; 2-i dai `dd760f9`; 2-ii-a dai `841ae26`; 2-ii-b dai
  `af59853`; 2-iii foundation/b2 dai `1311137`->`aee1ade`; 2-iii-c dai `baf5e90`; 2-ii-c
  dai `80e8d1a` (operator/harness + status parity) -> `4c6cd98` (blank-state + comment
  correction) = local main tip. Vault records for 2-ii-c: `d98d1a5` (report/WI/current-slice/
  doctrine) -> `462d7a4` (c1 correction) -> the closeout commit containing this report (SHA
  in the closeout report and the prompt ledger). Full per-slice SHA history:
  [[wi-0037-local-integration-and-governance-closeout-2026-07-28-v1]]
- tests: `platform/dotnet/DevCore.Api.Tests/Sports/GameStatusResolutionCorpusTests.cs`;
  `.../AgentRuns/MarketContrastSourceAdapterTests.cs`; `.../Sports/StarterClientBracketAuthorityTests.cs`;
  `.../Integration/SelectedEventResolutionTests.cs`, `.../SelectedEventCreationTests.cs`;
  `scripts/dev/sports/test-check-game-status.ps1`, `.../test-check-settlement-finals.ps1`.
  Post-integration on dai main `4c6cd98`: PS 195/0 + 40/0 + probe exit 1 by design; focused
  corpus 63; combined focused 189; full `DevCore.Api.Tests` **2059/2059, 0 skipped**; build 0
  errors
- verification notes: independent adversarial review of the 2-ii-c + c1 package returned
  **WI0037_SLICE2IIC_C1_INDEPENDENT_REVIEW_PASSED_LOCAL_INTEGRATION_READY** (zero findings);
  local fast-forward re-verified all opening/preservation gates, both range allowlists (source
  11 paths 329+/62-; vault 4 paths 440+/4-), corpus byte-identity `539f137`, comment audit
  69/0/0, protected WI-0035 fingerprint `86aa8b74` unchanged before and after; full evidence
  matrix in [[wi-0037-local-integration-and-governance-closeout-2026-07-28-v1]]
- docs updated: this WI; MOC (completion state); `06 Execution/handoffs/current-slice.md`
  (closeout entry + Slice Synopsis);
  [[wi-0037-slice-2-ii-c-operator-harness-parity-2026-07-27-v1]] (dated closeout section);
  [[wi-0037-local-integration-and-governance-closeout-2026-07-28-v1]] (new). `glossary.md`
  unchanged (no term promoted); `sports-v1-backlog.md` unchanged (WI-0037 not registered
  there and doctrine does not require adding completed items -- not required)
- lessons: characterization-matrix-first made the one-line predicate change reviewable;
  driving every normalization vector through STAGED resolution (not just direct normalization)
  is what caught the cross-runtime whitespace divergence the direct-only tests hid; a warning
  in a file outside the changed-path set cannot have been introduced by the change; governed
  local fast-forward + atomic `update-ref` compare-and-swap advances main without checking out
  or disturbing a protected dirty worktree

## final handoff requirements

Closeout handoff appended to `06 Execution/handoffs/current-slice.md` with the mandatory
Slice Synopsis; status is `complete` (all slices integrated into the local baseline). Next
governed action = an operator-selected next slice against the integrated local baseline (dai
main `4c6cd98`, vault main = this closeout commit): a candidate evidence/capture slice, or a
future release program that would take remote publication + deployment + activation review
(none of which are authorized or performed here).
