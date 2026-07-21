---
title: "WI-0036 Wildcard Evidence Discovery Loop Planning v1"
type: "evidence-report"
date: "2026-07-21"
status: "complete"
project: "DAI"
slice: "WI-0036 Slice 1: documentation and contract baseline"
repos:
  dai: "unchanged (read-only audit at 8369d64)"
  dai-vault: "docs-only (branch wi/0036-wildcard-evidence-discovery-loop)"
tags:
  - system-development
  - evidence-operations
  - cognitive-factory
  - planning
related:
  - "02 Platform/system-development/work-items/WI-0036-wildcard-evidence-discovery-loop-v1.md"
  - "02 Platform/decisions/0011-orchestrated-interrogate-perceive-refresh-loop-v1.md"
  - "02 Platform/architecture/cognitive-factory/wildcard-evidence-discovery-loop-v1.md"
  - "04 Products/sports-v1/daily-evidence-acquisition-orchestrator-v1.md"
---

# wi-0036 wildcard evidence discovery loop planning v1

## purpose

Closeout for WI-0036 Slice 1: the operator-authorized documentation/architecture/planning
slice that created the Wildcard Evidence Discovery Loop work item, decision 0011, and the
focused doctrine record, and reconciled affected current-state doctrine and planning
records. Docs only; zero runtime change; zero spend; one local vault commit.

## context

Operator-sent authorization prompt of 2026-07-21 (prompt ledger record
`<OBSIDIAN_PROMPT_LEDGER_ROOT>/dai/prompts/2026/07/2026-07-21-wi-0036-wildcard-evidence-discovery-loop-documentation.md`,
prompt sha-256 `7365B678DA4C20E0543EEB815E092432BDF3D87B2E990CE3228418548D9448D9`).
Opening state matched the prompt's expectations exactly: dai `main` @
`8369d64a2b4ed29ab1c6297de81270d2f9dd8a46` (0/0 vs origin), dai-vault `main` @
`a7481f88c3ad49e7c67187f64845de07ad7ecef5` (0/0 vs origin), with the known protected
dirty/untracked set present in both repos.

## scope

Included: parent WI-0036 with a six-slice decomposition; decision 0011 (the orchestrated
Question -> Probe -> authorized retrieval -> Perceive refresh -> Verify -> Discern loop +
callable protocol-service target); the wildcard-evidence-discovery-loop doctrine record
(lanes, cap, substitution, strongest-novelty ordering, provenance fields,
`SignalNeedProposal`); MOC registration; reconciliation edits (orchestrator record, cohort
doctrine, perceive/interrogate phases, probe-refresh readiness, recipe architecture,
station blueprint, deferred-decisions ledger entries 1 + new 27, delivery timeline,
WI-0034/WI-0035 seam links); this report; current-slice append.

Excluded (verified non-actions below): every runtime, retrieval, capture, settlement,
commercial, and remote-mutation action; the July 22 events-gate action and its
authorization, which remain separately governed and untouched.

Allowlisted files deliberately NOT edited, with reasons: `protocol-node-specs.md` (its
Probe facets already match verified source: deterministic `BuildProbe`, closed template
set, unknown signals dropped); `cognitive-protocol-runtime.md` (no statement becomes
false; linked via decision 0011 and the phase docs); `v1-to-v2-release-sequence-v1.md`
(WI-0036 changes no release-ladder dependency; commercial deferral and Stripe-as-truth
unaffected); `06 Execution/roadmap/platform-execution-sequence.md` and
`06 Execution/roadmap/sports-v1-roadmap.md` (2026-04-era records orthogonal to this
scope); `04 Products/sports-v1/market-contrast-candidate-screen-v1.md` (nothing in it
becomes false; the lane-vs-tier distinction is recorded in the orchestrator record,
WI-0035 link, and the doctrine record).

## key decisions / findings

1. **Decision 0011 accepted (docs only).** The governed refresh loop and its authority
   rules are decision-bound: Interrogate requests, the orchestrator authorizes/routes/
   re-enters, direct Interrogate-to-Perceive self-invocation stays forbidden, refreshed
   evidence must re-enter `Interrogate.Verify` before Discern, preflight is Perceive-side
   target doctrine, and stations target dual (orchestrated + independently callable)
   invocation under one contract. It conflicts with no current runtime truth because every
   current-implementation claim was verified in source first (below) and every target
   capability is labeled target/deferred.
2. **Source-verified current truth (2026-07-21).** `Interrogate.Probe` is deterministic at
   compose time (`CognitiveProtocolBuilder.BuildProbe` / `BuildProbeRequest` over
   `SignalFollowUps`); `ProbeRequest` fetches nothing, calls no gateway, invokes no
   Perceive, mutates nothing, and its template set is closed to `sharp_public`, `market`,
   `rest_schedule`, `starting_pitching` with unknown signals dropped; `Question`/`Verify`
   are model-emitted in the single shared analyze call; the Stage-2 dev endpoint executes
   only the inert `interrogate.probe` audit-only path under the four-condition config gate
   and env IsDevelopment gate, with hard-coded all-false side-effect flags; the
   probe-refresh chain is dormant with no production consumer and all mutation/gateway
   flags default false; and the chain's sequence after Perceive intake goes directly to
   Discern/Decide/Synthesize preview -- **no `Interrogate.Verify` re-entry step exists**.
   That gap is now named (ledger entry 27) instead of papered over.
3. **Two stale current-truth claims reconciled by superseding notes (history preserved):**
   `phases/interrogate.md` ("Probe has no current field") and
   `perceive-interrogate-recipe-architecture-v1.md` ("question+probe live in the analyze
   call" -- the split is question+verify model-emitted, probe deterministic).
4. **Wildcard contracts frozen as doctrine:** closed lane vocabulary distinct from screen
   tiers; cap `floor(total_scheduled_runs / 4)`; one-core hard minimum; freeze
   immutability; substitution only by frozen-and-authorized wildcards, label preserved,
   reasons recorded; realized share may exceed 25 percent only via such substitutions;
   closed lexicographic strongest-novelty ordering with stable tie-breakers;
   settled-vs-captured-unsettled separation; market-missing recipes reachable;
   historical recapture prohibited by default; promotion reconciliation-owned.
5. **`SignalNeedProposal` defined proposal-only** (name tentative; no existing canonical
   type covers it -- `ProbeRequest` is execution-adjacent/closed and the WI-0034
   missing-capability records are planner-side), with every execution/retrieval/mutation/
   confidence/posture/lean authority false by contract.
6. **Sequencing recorded without inventing dates:** timeline initiative
   `wi-0036-wildcard-evidence-discovery-loop` carries the operator decision that
   implementation is not before the separately governed 2026-07-22T12:00:00Z events-gate
   observation plus a new implementation authorization; `desired_by`/`due_by` left empty;
   the system suggestion lives only in `proposed_by_system`; the authorization posture
   block (paid model calls / sports capture / reconciliation writes not-authorized,
   posture no-spend) is unchanged.

## desk-scenario verification -- wildcard arithmetic and substitution

| # | scenario | expected | verdict against the written contracts |
|---|---|---|---|
| 1 | 3 scheduled runs | `floor(3/4) = 0` wildcards | PASS -- flights under four schedule none |
| 2 | 4 scheduled runs | at most `floor(4/4) = 1` wildcard | PASS -- >= 3 core/reserve remain scheduled |
| 3 | 8 scheduled runs | at most `floor(8/4) = 2` wildcards | PASS |
| 4 | one core drops; an authorized wildcard substitutes | substitution allowed; label stays `wildcard`; substitution + missing-core reason recorded; no post-freeze addition | PASS -- only a wildcard already frozen AND authorized for that flight is eligible |
| 5 | substitutions raise realized share above 25 percent | permitted iff every substitute was pre-authorized and >= 1 core run remains (e.g. scheduled 4 = 3 core + 1 wildcard; one core drops and a second frozen-authorized wildcard substitutes -> executed 2 core + 2 wildcard = 50 percent realized) | PASS -- cap governs the initially scheduled flight only |
| 6 | all core candidates drop | hard stop for a new operator decision; no all-wildcard flight executes | PASS -- one-core minimum is a hard invariant |
| 7 | no qualified wildcard exists | core/reserve-only flight; wildcard slots stay empty; no forced spend | PASS -- wildcards are never force-filled |
| 8 | market-missing recognized recipe as wildcard target | reachable when market missing is its expected recognized data shape; no universal market-baseline gate blocks it | PASS -- explicit in the lane contract |
| 9 | historical-identity recapture proposed as a wildcard | rejected by default; a governed recapture capability is a separately designed future WI | PASS |

## desk-scenario verification -- loop authority

| # | scenario | verdict against decision 0011 + the proposal contract |
|---|---|---|
| 1 | Question names a consequential gap | PASS -- `interrogate.question` output; grants nothing |
| 2 | Probe proposes a known signal | PASS -- `SignalNeedProposal` mapping status `known`; proposal only; `ProbeRequest` closed set untouched |
| 3 | Probe proposes an unmapped signal | PASS -- `unmapped_candidate`; retrieval-research candidate only; never a tool id |
| 4 | unauthorized retrieval attempted | PASS -- fail-closed: no orchestrator authorization means no call and no Perceive refresh (gateway fails closed; chain gateway flags default false) |
| 5 | authorized retrieval | PASS -- orchestrator-owned Perceive refresh through platform retrieval (`platform.retrieve` constraint preserved) |
| 6 | refreshed evidence before Discern | PASS in target doctrine -- must re-enter `Interrogate.Verify`; current dormant chain lacks the step (ledger entry 27; WI-0036 Slice 5 owns closure) |
| 7 | direct Interrogate self-invocation | PASS -- forbidden (decision 0011; ledger entry 1; tool-gateway doctrine) |
| 8 | remote station invocation | PASS -- same artifact/version/tenant/idempotency/permission contract as the orchestrated path; never bypasses the Tool Gateway |
| 9 | proposal attempts to change confidence/posture/lean/history | PASS -- impossible by contract: proposal authority fields all false; protected-field guards remain authoritative |

## evidence

- Strict planning snapshot (read-only, output to session scratchpad outside both repos):
  exit 0, **warnings 0**, work items 24 -> **25** (WI-0036 parsed: status `in-progress`,
  precedence 2, confidence high), timeline entries 5 -> **6**
  (`wi-0036-wildcard-evidence-discovery-loop` parsed with empty operator dates and the
  not_before sequencing text), authorization posture unchanged
  (`paid_model_calls/sports_capture/reconciliation_writes: not-authorized`, posture
  `no-spend`).
- Source verification: full reads of `ProbeRequest.cs`, `CognitiveProtocolBuilder.cs`,
  `PromptTrace.cs`, `SignalQualityEvaluator.cs`, `SignalFollowUpEvaluator.cs`,
  `SportsRunArtifact.cs`, `SportsComposer.cs`, `ProtocolNodeRunner.cs`,
  `ProbeRefreshChainAssembly.cs`, `ProbeRefreshPerceiveIntake.cs`,
  `ProbeRefreshDiscernReweigh.cs`, `CognitiveFactoryExecutionController.cs`,
  `CognitiveFactoryExecutionGate.cs`, plus their direct tests (`ProbeRequestTests`,
  `CognitiveProtocolBuilderTests`, `ProtocolNodeExecuteTests`,
  `CognitiveFactoryExecutionGateTests`, `ProbeRefreshChainAssemblyTests`). Key line-level
  facts recorded in [[wildcard-evidence-discovery-loop-v1]] (current-truth section) and
  decision 0011.
- Repo/branch state: dai untouched on `main` @ `8369d64a...` (no new diff, no commit);
  vault work on branch `wi/0036-wildcard-evidence-discovery-loop` from `a7481f88...`;
  one local docs commit at close (hash recorded in the current-slice handoff and the
  prompt-ledger outcome).
- Protected-state proof (hashes open == close): dai
  `platform/dotnet/DevCore.Data/DevCore.Data.csproj` sha-256 `63EF2488...1D16A8F`
  (1099 bytes); vault `.obsidian/graph.json` `B3D68588...2FD0B4A` (511 B), `CLAUDE.md`
  `9127E464...39AAA4C0` (870 B), untracked
  `06 Execution/reports/preflight-settlement-manifest-2026-07-06-v1.json`
  `68948EBD...D118066B` (6428 B) and `06 Execution/system-state-synopsis-v1.md`
  `25835E6C...447C692F` (19164 B); `Welcome.md` remains deleted. None staged, restored,
  normalized, or incorporated.
- Hygiene: `git diff --check` clean; scans over the changed set for machine-specific
  paths, secrets, `authorized: true`, and accidental runtime claims clean; OKF review
  checklist applied to this report and every new/edited record; vault grill
  (dai-grill-with-vault) run against the final docs with no blocking contradiction.

## safety / non-actions

External calls: model 0; StatsAPI 0; Odds `/events` 0; Odds `/odds` 0; database 0; Tool
Gateway 0. Generation/capture/screening/settlement/reconciliation/scheduling: 0. Paid
cost: $0. No service started. No dai file edited; no runtime code, test, endpoint, config
flag, schema, migration, prompt, routing, confidence, posture, lean, buyer-copy, pricing,
Stripe, entitlement, or tenant change. `ProbeRequest`, station permissions, and Tool
Gateway permissions unwidened. The dormant probe-refresh chain untouched and inactive.
The July 22 events-gate authorization neither executed nor altered. No push, merge, PR,
or remote mutation; no history rewrite; no persistent assistant memory created. The
25-percent wildcard policy, the loop, the proposal type, the wildcard planner, and the
service seam are DOCUMENTED TARGETS -- none is implemented.

## next step

Operator review of this slice. The next time-sensitive action remains the separately
governed one (unchanged by this slice): no earlier than 2026-07-22T12:00:00Z, one
refreshed free preflight plus at most one zero-quota `/events` observation; no `/odds`.
Only after that gate completes AND a new operator implementation authorization is issued:
propose **WI-0036 Slice 2 -- deterministic wildcard preflight and flight-plan allocation**
(offline, fixture-only) per the decomposition in
[[WI-0036-wildcard-evidence-discovery-loop-v1]]. A recommendation is not an authorization.

## related

- [[WI-0036-wildcard-evidence-discovery-loop-v1]]
- [[0011-orchestrated-interrogate-perceive-refresh-loop-v1]]
- [[wildcard-evidence-discovery-loop-v1]]
- [[daily-evidence-acquisition-orchestrator-v1]]
- [[cohort-selection-and-run-discipline-v1]]
- [[regime-discovery-candidate-selection-v1]]
