---
title: "WI-0036 Wildcard Evidence Discovery Loop v1"
type: "plan"
date: "2026-07-21"
status: "in-progress"
project: "DAI"
slice: "WI-0036 Slice 2 + minimum Slice-3 seam: integrated 2026-07-22 (Slice-3 remainder and Slices 4-6 deferred)"
repos:
  dai: "code+tests integrated to main ce34a9e7 (wildcard flight-plan core/CLI + provenance seam)"
  dai-vault: "docs integrated to main 2cdb275b"
tags:
  - system-development
  - work-item
  - evidence-operations
  - cognitive-factory
related:
  - "02 Platform/decisions/0011-orchestrated-interrogate-perceive-refresh-loop-v1.md"
  - "02 Platform/architecture/cognitive-factory/wildcard-evidence-discovery-loop-v1.md"
  - "04 Products/sports-v1/daily-evidence-acquisition-orchestrator-v1.md"
  - "02 Platform/system-development/work-items/WI-0034-daily-evidence-planner-stage-0.md"
  - "02 Platform/system-development/work-items/WI-0035-market-contrast-candidate-screen.md"
---

# WI-0036 wildcard evidence discovery loop v1

Governing parent for the Wildcard Evidence Discovery Loop: a governed vertical discovery
loop that widens safe evidence acquisition through a bounded wildcard candidate lane,
carries wildcard provenance through production artifacts, turns artifact interrogation
into proposal-only signal-need inputs, and (later, separately gated) closes the
orchestrator-mediated `Question -> Probe -> authorized retrieval -> Perceive refresh ->
Verify -> Discern` loop with a callable protocol-service seam.

**Current disposition (2026-07-22): Slice 1 documentation COMPLETE and integrated; Slice 2
plus the MINIMUM Slice-3 provenance seam IMPLEMENTED, corrected, reviewed, and INTEGRATED
to both published mains.** WI-0036 itself remains `in-progress`: the Slice-3 remainder and
Slices 4-6 are still deferred and each requires its own operator authorization. Integrated
tips: dai main `ce34a9e74659b42c71317267c64901a24ceb7091` and dai-vault main
`2cdb275b85229c6ee11a7b2930dc50d847ae8240`, each equal to its `origin/main`. Live
contracts: request/plan/planner/realization/CLI `1.1`;
`flight-selection-provenance/1.0` and `wi0036-candidate-lane/1.0` unchanged. Reverified on
the published mains 2026-07-22: agent-service pytest **617/617**, DevCore.Api.Tests
**1530/1530**. The integrated path is default off with all-false authority ledgers; it
executed no source, model, gateway, or database call and cost **$0**. Repository
publication granted NO runtime or commercial authority: the next governed action for this
code is the separately authorized Slice-3 remainder, NOT activation. The 2026-07-22
events-gate observation and any PAID wildcard flight remain separately governed and
unauthorized.

*(Historical: the original Slice-1 disposition below read "documentation/contract Slice 1
COMPLETE; runtime implementation NOT started", and implementation was sequenced not before
the July 22 observation plus a separate authorization. That sequencing was superseded by
the 2026-07-21 operator override recorded immediately below; the dated blocks that follow
are preserved as the historical record of local-only and integration-pending states.)*

> **Operator sequencing override (2026-07-21, after Slice 1 integration).** A new operator
> decision authorized proceeding with the offline/default-off implementation NOW: Slice 2
> plus the minimum Slice-3 provenance seam. This supersedes ONLY the earlier statement
> that WI-0036 implementation must wait for the July 22 events-gate observation; that
> observation remains a separate, still-unexecuted, separately governed action and is no
> longer a precondition for this offline implementation. Paid use of the new capability
> still requires a future explicit flight authorization.
>
> **Disposition after that slice (2026-07-21):** Slice 2 (deterministic wildcard
> flight-plan core + portable CLI) and the MINIMUM Slice-3 provenance seam are
> IMPLEMENTED on local review branches `wi/0036-wildcard-capture-flight-plan` (dai +
> dai-vault, matching), NOT pushed / NOT merged / NOT integrated; default off; zero
> granted authority. Delivered contracts: request `wildcard-flight-plan-request/1.0`,
> plan `wildcard-flight-plan/1.0`, planner `wildcard-flight-planner/1.0`, realization
> `wildcard-flight-realization/1.0`, provenance `flight-selection-provenance/1.0`, lane
> vocabulary `wi0036-candidate-lane/1.0`, registries `wi0036-signal-combinations/1.0` +
> `wi0036-novelty-dimensions/1.0`, CLI `wildcard-flight-plan-cli/1.0`. Files: dai
> `services/agent-service/app/services/wildcard_flight_plan.py` + `wildcard_flight_plan_cli.py`
> (+ two pytest suites, 70 tests) and the platform seam
> `platform/dotnet/DevCore.Api/AgentRuns/FlightSelectionProvenance.cs` + additive optional
> `CompetitionMatchupInput.FlightSelection` / `AgentRunExecutionResult.FlightSelection` /
> `AgentRunArtifactDto.FlightSelection` threading (+12 xunit tests incl. buyer-boundary
> sentinel). The REST of Slice 3 and Slices 4-6 remain deferred and unauthorized. Next
> governed action for this code: independent review + coordinated integration of the two
> local branches -- NOT the July 22 observation and NOT a paid wildcard run.
>
> **Pre-integration contract correction (2026-07-21, same day).** An independent Codex
> review found the first implementation pair (dai `e64baab` / vault `8d98da9`) NOT
> integration-ready: seven material defects (A: python snake_case export did not bind to
> the .NET record and authority defaulted true on omission; B: core-qualified reserves
> mislabeled lane `core`; C: caller-supplied vocabulary could invent recognized
> recipes/regimes/taxonomy; D: a rehashed content fingerprint laundered escalated plan
> authority through realization; E: provenance export trusted a tampered/forged
> realization artifact; F: the .NET boundary tolerated omitted authority and lacked the
> full validation matrix; G: a caller-edited board interior was treated as
> producer-certified). All seven were reproduced RED-first and corrected in new commits
> ON TOP (nothing amended): ONE canonical camelCase wire contract with an exact eight-key
> all-false `authorityLedger` and provider-scoped `substitutedFor` (update-together
> cross-runtime vector embedded verbatim in both suites); selection lane `reserve`
> preserved end-to-end with `realized_via` recording the slot fill and the core minimum
> counting reserve-via-core without relabeling; the recognized registry now DERIVED from
> the canonical prompt manifest (hash-verified, digest-pinned
> `prompt-manifest/2+sha256:...`) + the wi-0036 closed registries, caller
> narrowing-only; strict closed-contract plan/realization validators independent of the
> content fingerprint (a sha-256 is content identity, never authority); provenance
> export re-derives realization internally from plan + availability (hand-authored
> realization artifacts are never inputs); the .NET Validate matrix enforces
> versions/ledger/fingerprint/timestamp/conditional-wildcard/substitution-consistency/
> registered-values with a controller-host proof that malformed provenance returns 400
> before the stub service and before any run row; and the WI-0036 CLI now REPRODUCES the
> board through the real WI-0034 planner (the request supplies the upstream planner
> request; any claimed board must be byte-identical to the reproduction). Suites after
> correction: agent-service pytest 653/653 (89 wi-0036 tests), DevCore.Api.Tests
> 1528/1528. Branch state: base commits preserved + one correction commit per repo;
> still NOT pushed / NOT merged / NOT integrated; next = independent review of the
> complete two-commit chains in both repos.
>
> **Semantic-integrity correction (2026-07-21, same day, second independent review).**
> The A-G correction still left five semantic gaps (findings H-L, all reproduced
> RED-first against dai `b0ff396`): H -- a core-qualified reserve was exported/validated
> substitution-INELIGIBLE (the formula ignored the reserve lane); I -- a rehashed
> wildcard block with a fake recipe/version/regime, blanked hypothesis fields, invented
> combinations/dimensions, or a non-deterministic novelty rank passed plan validation
> and exported; J -- producer references (board schema/planner versions, manifest
> version/digest) were caller-rewriteable after a rehash (shape/64-hex is not producer
> verification); K -- a realization missing a scheduled slot from the realized+unfilled
> partition, or carrying a forged unfilled identity/reason, passed; L -- membership plus
> a self-consistent graph could not prove reserve-first precedence, so a hand-authored
> wildcard substitution over an available core reserve passed. Corrections (new commits
> on top; nothing amended): an explicit `TrustedValidationContext` (wi-0034 producer
> version constants + the manifest-derived canonical vocabulary, built ONLY by the cli's
> `build_trusted_context`) is now REQUIRED by validate/realize/export -- no context-free
> path remains; plan validation became semantic (recognized recipe/version/regime,
> non-blank hypothesis/selection fields, registered duplicate-free sorted
> combinations/dimensions, threshold adherence, exact-typed novelty rank equal to the
> deterministic formula, strongest-novelty ordering across scheduled + reserve pools,
> closed reason shapes, producer references verified against the context) with the
> sha-256 checked LAST as a content fingerprint only, and malformed nested content
> returns the closed error contract; FULL realization validity is now
> availability-derived -- valid iff canonically identical to
> `realize_flight(plan, availability, context)` (cli `validate --realization` requires
> `--plan` AND `--availability`; omission is a fail-closed usage error); and the
> substitution-eligibility truth table was corrected in python + .NET (core false,
> reserve TRUE, scheduled wildcard false, substitution-reserve wildcard true;
> eligibility is a frozen plan fact, never proof of substitution, never authority) with
> a second update-together reserve-substitution cross-runtime vector and a
> controller-host end-to-end proof. Suites after this correction: agent-service pytest
> 664/664 (wi-0036 76 core + 25 cli), DevCore.Api.Tests 1530/1530. Chains = three
> commits per repo (base + A-G + H-L), still NOT pushed / NOT merged / NOT integrated;
> next = independent review of the complete three-commit chains.

> **Producer-replay correction (2026-07-22, third independent review; findings M-S).**
> The H-L correction remained intact but its root contract was still wrong: full plan
> validity was INFERRED from a context holding producer version strings and a canonical
> vocabulary, not PROVEN by re-production. All findings were reproduced RED-first against
> dai `994b0c1`: M -- a `board_reference.board_digest` rewritten to 64 `b` characters
> validated after re-fingerprinting, because the trusted context never carried the
> reproduced board/request digest; N -- `wildcard_mode=disabled` with a live wildcard
> lane, a blank core-reserve identity, a selected/non-selected overlap with empty
> reasons, and a boolean position (`True == 1` in python) all validated after
> re-fingerprinting; O -- a safe market-missing wildcard rewritten to the canonical
> market-backed recipe/version/regime with `market_screen_tier = null` validated AND
> exported, because recognition alone cannot re-prove the producer's market-safety
> decision; P -- the registry stored recipe versions and data regimes as INDEPENDENT
> sets, so individually canonical components cross-producted into manifest tuples that
> do not exist (existing ranking fixtures relied on exactly such a pairing); Q -- an
> availability row whose `source_provider` was a list raised
> `TypeError: unhashable type: 'list'` out of the pure core, escaping the closed
> structured-error contract; R -- an unfilled row replaced the validated source reason
> `postponed` with the disposition code `scheduled_wildcard_unavailable`, contradicting
> the H-L claim that the source reason is preserved; S -- the verification accounting
> said `76 + 25` when fresh collection proved `76 core + 24 cli = 100`.
>
> **Root correction: exact producer re-production is now the full-validity rule.** A
> candidate plan is fully valid IFF `plan_canonical_json(plan)` equals
> `plan_canonical_json(build_flight_plan(verified_request))` -- the same governing
> pattern already adopted for realization. Duplicated semantic field-checks over a
> mutable serialized artifact cannot prove producer decisions; only re-production can,
> and it proves the allocation, safety, ordering, and cap decisions together instead of
> approximating them one field at a time. `validate`, `realize`, and `export-provenance`
> all REQUIRE the verified request; omission is a fail-closed usage error and no
> context-free or version-only path can report full validity. A narrow
> `validate_flight_plan_structure` remains as defense in depth for malformed artifacts
> (strict int typing so booleans fail, blank/overlapping identities, closed non-selected
> reason codes, disabled-mode-with-wildcards) and is explicitly documented as never
> conferring producer validity.
>
> The recognition registry now preserves the manifest's EXACT
> `(recipe_id, recipe_version, data_regime)` relation plus the `market_state` DERIVED
> from that same regime; caller narrowing selects a subset of real tuples and can never
> author a market state. Availability rows are type-validated BEFORE any tuple/dict
> keying, so malformed input returns `INVALID_REQUEST` rather than an unhandled
> exception. Unfilled realization rows now carry the exact validated
> `unavailability_reason` AND a separate closed `unfilled_reason_code`
> (`scheduled_wildcard_unavailable | no_eligible_substitute`), keeping source
> observation and orchestration disposition distinct. Because the request/registry
> semantics and the unfilled-row shape changed, the still-unintegrated contracts bump
> together to `wildcard-flight-plan-request/1.1`, `wildcard-flight-plan/1.1`,
> `wildcard-flight-planner/1.1`, `wildcard-flight-realization/1.1`, and
> `wildcard-flight-plan-cli/1.1`; `flight-selection-provenance/1.0` and
> `wi0036-candidate-lane/1.0` are UNCHANGED. Both cross-runtime vectors were
> regenerated from the real exporter (fingerprints changed with the plan version) and
> embedded verbatim in the python and C# suites under the update-together rule.
>
> **Verification-accounting correction (finding S).** At dai `994b0c1` the true targeted
> counts were 76 core (72 defs + 4 parametrized) + 24 cli = **100**, not `76 + 25`; the
> earlier figure is corrected here and left in place above as history. After this
> correction the suites were consolidated -- producer re-production replaced many
> single-field semantic tests with mutation matrices -- so the targeted counts are now
> **39 core + 14 cli = 53**. A coverage-delta audit against `994b0c1` found four legacy
> scenarios silently dropped by that consolidation (the not-pregame safety gate and that
> novelty rank never rescues it; required/non-negative coverage facts per hypothesis
> triple and combination; the fully-available realization identity; strongest-novelty
> ordering among substitution-reserve members) and they were RESTORED as characterization
> tests in a second commit with production code untouched. Suites after the correction:
> agent-service pytest **617/617**, DevCore.Api.Tests **1530/1530**.

## problem  <!-- LITE -->

Three currently separate concerns do not close a learning loop:

1. **Deterministic evidence planning optimizes the dominant known objective.** The
   WI-0034 planner and WI-0035 screen deterministically pursue the standing deficit
   (market disagreement), so underrepresented recognized recipes/versions, data regimes,
   and signal combinations can be repeatedly missed. The regime-discovery report
   ([[regime-discovery-candidate-selection-v1]], 2026-06-30) proved thin-regime candidates
   exist and are reachable by lead-time-aware probing, but no governed lane schedules them.
2. **Artifact production has Interrogate Question/Probe/Verify semantics and a typed Probe
   handoff, but discovery of novel retrieval needs is narrower than the desired loop.**
   Verified in source 2026-07-21: `Interrogate.Probe` is deterministic at compose time
   from `SignalFollowUps`; `ProbeRequest` is a proposal/handoff contract with a closed
   four-signal template set (unknown signals dropped, never granted authority). The
   artifact's consequential questions do not become reviewable retrieval proposals.
3. **The dormant probe-refresh machinery does not yet embody the complete loop or a
   callable protocol-service boundary.** Verified in source 2026-07-21:
   `ProbeRefreshChainAssembly` proceeds from Perceive intake directly into
   Discern/Decide/Synthesize preview with no `Interrogate.Verify` re-entry step, has no
   production pipeline consumer, and every mutation/gateway flag defaults false.

## desired behavior

A governed vertical discovery loop -- not "run more games": future paid-flight preflight
can schedule a bounded wildcard lane (cap `floor(total_scheduled_runs / 4)`, one-core
minimum, frozen provenance, authorized substitution with strongest-novelty ordering);
wildcard selection reasons and expected-vs-actual recipe/regime/signal facts survive
through capture, settlement, and reconciliation as a separate evidence stratum; the
artifact's Interrogate protocol yields proposal-only `SignalNeedProposal` records
(execution/retrieval/mutation/confidence/posture/lean authority all false) that a later
reconciliation-backed review maps, opens as retrieval-source WIs, rejects, or defers; and
the orchestrated refresh loop plus callable protocol-service seam close under decision
0011 without ever letting Interrogate execute or a refresh mutate protected fields.

## affected surfaces

- Slice 1 (this slice): dai-vault only -- this WI; decision 0011; the architecture record
  [[wildcard-evidence-discovery-loop-v1]]; MOC registration; reconciliation edits to the
  orchestrator record, cohort doctrine, perceive/interrogate phase docs, probe-refresh
  readiness, recipe architecture, station blueprint, deferred-decisions ledger, delivery
  timeline, WI-0034/WI-0035 seam links; closeout report; current-slice append.
- Slices 2+ (deferred): named per slice below; nothing in `<DAI_REPO_ROOT>` changes until
  a slice is separately authorized.

## non-goals

No runtime code, tests, endpoints, config flags, migrations, schemas, services, or tools;
no model/StatsAPI/Odds/database/Tool Gateway/capture/generation/settlement/reconciliation/
scheduling call; no activation or authorization change to the July 22 events gate; no
retrieval source added or enabled; no widening of `ProbeRequest`, station permissions, or
Tool Gateway permissions; no splitting of the shared analyze model call; no activation or
mutation of the dormant probe-refresh chain; no prompt/routing/confidence/posture/lean/
buyer-copy/pricing/Stripe/entitlement/tenant change; no historical-identity recapture (a
future governed recapture capability is a separate design); no claim that any target
loop, proposal type, wildcard planner, or service is implemented.

## binding operator decisions (2026-07-21, not re-decidable by executors)

1. A wildcard is a safe candidate targeting an underrepresented or unusual **recognized**
   prompt recipe/version, data regime, or signal combination.
2. Wildcard selection widens evidence capture; the artifact's Interrogate protocol then
   exposes consequential questions and signal gaps that can become governed retrieval
   proposals.
3. A wildcard requires both a measurable evidence-coverage gap and a written novelty
   hypothesis.
4. Preflight may propose a wildcard lane whenever qualified candidates exist; use remains
   explicitly operator-approved.
5. Wildcards rank below core candidates in selection priority.
6. A frozen, flight-authorized wildcard may substitute for an unavailable scheduled core
   candidate; it stays labeled `wildcard`; the substitution and missing-core reason are
   recorded; no new candidate enters after freeze; at least one core run must remain, else
   hard stop for a new operator decision.
7. Initial scheduled wildcard capacity: 25 percent via `floor(total_scheduled_runs / 4)`;
   flights smaller than four schedule no wildcard absent a future explicit policy change.
8. The cap governs the initially scheduled flight; realized share may exceed 25 percent
   only via authorized substitutions with the one-core minimum intact.
9. Substitution selects the **strongest novelty** (deterministic, evidence-backed
   ordering; never unconstrained model ranking), not the candidate closest to the missing
   core objective.
10. Historical-identity recapture is prohibited by default.
11. Underrepresentation is measured primarily by settled evidence count for the exact
    recipe/version and regime; captured-but-unsettled evidence is visible but separate and
    never settled coverage.
12. Market-missing recipes are valid wildcard targets when market missing is their
    expected recognized data shape.
13. Promotion from wildcard/discovery status into ordinary selection is owned by a later
    reconciliation-backed decision; one wildcard result never promotes itself.
14. Preflight is conceptually part of Perceive (target doctrine, distinguished from
    current implementation).
15. Interrogate's canonical micro-actions are Question, Probe, Verify; "Question subphase"
    means `Interrogate.Question`, not a separate standalone prompt phase.
16. The coherent governed refresh loop is `Question -> Probe -> authorized retrieval ->
    Perceive refresh -> Verify -> Discern`.
17. Interrogate is a requester; an orchestrator mediates authorization and routes
    retrieval through platform-owned boundaries; direct Interrogate-to-Perceive
    self-invocation remains forbidden.
18. Interrogate proposes signal needs; it does not add sources.
19. Protocols/stations target eventual invocation through the full orchestrated pipeline
    or as independently callable governed services -- target architecture only, no
    endpoint or model-call split authorized here.
20. Commercial direction: build the governed system and charge for access; this WI records
    the direction and changes no pricing/Stripe/entitlement/buyer claim or delivery
    authorization.

### contract correction (2026-07-21, operator; binding -- narrows decisions 6-9)

- **Bounded substitution reserve.** `wildcard_scheduled_max = floor(total_scheduled_runs / 4)`
  continues to govern only initially scheduled run slots. The frozen plan may additionally
  carry a substitution-only wildcard pool bounded by
  `wildcard_substitution_reserve_max = max(0, scheduled_core_runs - minimum_executed_core_runs)`
  with `minimum_executed_core_runs = 1`. A closed field
  `wildcard_plan_role = scheduled | substitution_reserve` distinguishes planned use; role is
  not a lane change -- every such candidate stays lane `wildcard`, passes the full wildcard
  qualification/safety gates, is selected by the closed strongest-novelty ordering, is in the
  immutable frozen plan, and is explicitly covered by the flight's operator authorization.
  Substitution reserves do not count against `wildcard_scheduled_max`; the pool may be
  partially or wholly empty (never force-filled, never forced spend); flights smaller than
  four may carry substitution reserves even though they schedule zero wildcards.
- **Reserve-first precedence.** For an unavailable scheduled core candidate: (1) the
  existing deterministic eligible core-qualified reserve fills the slot; (2) only when none
  is eligible/available, an eligible frozen wildcard substitution reserve; (3) among
  multiple eligible wildcards, strongest novelty (ordering compares eligible wildcards
  only -- a wildcard never outranks an eligible core-qualified reserve); (4) otherwise
  fail-closed non-execution -- never invent a candidate or perform a new retrieval.
- **Spend invariants.** Substitution is one-for-one for the vacated scheduled core slot and
  never increases `total_scheduled_runs` or the flight's maximum paid-run count; no
  post-freeze addition; the one-core hard minimum stands (zero core runs -> hard stop).
  Realized wildcard share may exceed 25 percent only through eligible one-for-one
  substitutions from this frozen, explicitly authorized reserve.

## architecture

The full contracts live in the two canonical records this slice created (single-writer:
this WI does not restate them):

- **Decision:** [[0011-orchestrated-interrogate-perceive-refresh-loop-v1]] -- the loop
  order, orchestrator authority, Verify re-entry requirement (and the named gap in the
  dormant chain), preflight-as-Perceive target placement, and the callable
  protocol-service target contract.
- **Doctrine:** [[wildcard-evidence-discovery-loop-v1]] -- closed candidate-lane
  vocabulary (core/reserve/wildcard/excluded/blocker, distinct from market-screen tier),
  cap arithmetic and substitution invariants, the closed lexicographic strongest-novelty
  ordering, the candidate/flight provenance field contract, the `SignalNeedProposal`
  proposal-only contract, and the current-vs-target truth ledger verified against source.

## decomposition (dependency-ordered; every slice separately authorized)

Common to every slice: purpose, input/output contract, explicit non-goals, authority
level, activation rung, dependencies, tests/evidence, rollback boundary, and the separate
authorization each needs are stated inline. No slice inherits authority from this WI.

### Slice 1 -- documentation and contract baseline (THIS SLICE, complete 2026-07-21)

- **Purpose:** durable WI definition, decision 0011, focused architecture record,
  MOC registration, current-state reconciliation across affected doctrine/planning.
- **Input/output:** audited vault + verified source truth in; the docs listed under
  affected surfaces out.
- **Non-goals:** everything under non-goals above; zero runtime behavior change.
- **Authority/activation:** none / activation stage `none`.
- **Dependencies:** none (documentation only).
- **Tests/evidence:** strict planning snapshot zero warnings; desk scenarios (closeout
  report); source verification citations; OKF review; vault grill.
- **Rollback:** revert the single vault docs commit.
- **Separate authorization:** the operator-sent WI-0036 documentation prompt (prompt
  ledger record 2026-07-21).

### Slice 2 -- deterministic wildcard preflight and flight-plan allocation (INTEGRATED 2026-07-22)

> **Post-integration disposition (2026-07-22).** Slice 2 and the minimum Slice-3
> provenance seam are INTEGRATED and published: dai main
> `ce34a9e74659b42c71317267c64901a24ceb7091`, dai-vault main
> `2cdb275b85229c6ee11a7b2930dc50d847ae8240`, each equal to its `origin/main` after a
> fast-forward-only coordinated publication (no amend, squash, rebase, force-push, or
> merge commit; every prior commit preserved). Reverified on the published mains:
> agent-service pytest **617/617** and DevCore.Api.Tests **1530/1530**, with the
> adversarial producer-replay probes reproducing zero leaks. Live contracts are
> request/plan/planner/realization/CLI `1.1`, with
> `flight-selection-provenance/1.0` and `wi0036-candidate-lane/1.0` unchanged. The path
> stays default off with all-false authority ledgers; integration executed no source,
> model, gateway, or database call and cost **$0**, and granted no runtime or commercial
> authority. Next governed action for this code: the separately authorized Slice-3
> remainder. The acceptance evidence quoted below is the ORIGINAL 2026-07-21 delivery
> record and is retained as history; the correction blocks near the top of this document
> supersede its contract versions and test counts.

> Delivered under the operator sequencing override as a WI-0036-OWNED contract that
> CONSUMES the WI-0034 board (`daily-evidence-board/2.2` embedded and strictly validated:
> closed keys, cohort-proposed outcome, all-false ledger, matching target date) and
> reuses the board ledger's safety evaluations rather than re-implementing them. WI-0034
> board/request/planner versions are UNCHANGED (2.2/2.1/2.2). Acceptance evidence: 56
> core fixtures + 14 CLI fixtures GREEN (RED-first), full agent-service pytest 634/634;
> cap arithmetic 1/3/4/7/8, C1-C8, tie-breakers + permutation invariance,
> settled-vs-unsettled separation, market-missing reachability without unknown-favorable,
> recapture rejection, default-disabled mode, tamper/fingerprint rejection, byte
> determinism, always-false ledgers. Rollback: revert the dai commit (pure additive
> module + CLI + tests; nothing else references them).

- **Purpose:** offline functional core first: a versioned planner/board contract with a
  distinct wildcard pool/lane; the 25 percent cap with floor rounding; one-core minimum;
  safe-candidate gates identical to core; the closed strongest-novelty ordering;
  deterministic allocation; immutable freeze; the bounded substitution-reserve pool
  (`wildcard_plan_role`, `wildcard_substitution_reserve_max`) and reserve-first
  substitution precedence per the 2026-07-21 contract correction.
- **Input/output:** typed coverage-gap facts (settled counts by exact
  recipe/version/regime and signal-combination), frozen hypotheses, and the existing
  planner inputs in; a versioned flight plan (core/reserve/wildcard, provenance,
  authority ledger all false) out.
- **Non-goals:** no live source, model, DB, capture, endpoint, scheduler, or spend;
  fixture-only.
- **Authority/activation:** none / `none` (pure offline core).
- **Dependencies:** Slice 1 reviewed; the 2026-07-22 events-gate observation complete;
  fresh live-state verification.
- **Ownership note (resolve at implementation):** determine whether this is a new WI-0036
  contract consumed by WI-0034's planner rather than silently changing WI-0034 ownership.
- **Tests/evidence:** fixtures for every cap/substitution/ordering invariant, including
  the desk-scenario set from the Slice-1 closeout, plus byte-determinism.
- **Rollback:** revert the slice's commits; no state exists outside code+fixtures.
- **Separate authorization:** a new operator implementation authorization (explicitly not
  granted by Slice 1).

### Slice 3 -- wildcard provenance through production artifacts (MINIMUM SEAM IMPLEMENTED 2026-07-21; remainder deferred)

> The minimum seam shipped with Slice 2: additive default-null null-suppressed
> `CompetitionMatchupInput.FlightSelection` (`flight-selection-provenance/1.0`; the
> WI-0009 GamePk pattern), fail-closed trust-boundary validation (schema, closed
> lanes/roles, target-date + gamePk identity consistency, mandatory all-false authority
> attestation) BEFORE the run row is written, threading through `SportsRunArtifact` into
> `AgentRunExecutionResult` on success AND analyze-failure composition (the SourceDepth
> pattern), persistence via existing InputJson/OutputJson (no migration, no column, no
> endpoint), read-only projection on `AgentRunArtifactDto` beside the observed
> `PromptRouteProvenance` (expected-vs-actual at the inspection boundary), and buyer
> exclusion guarded by the buyer-projection sentinel test. Legacy byte-identity and
> FastAPI request-byte invariance are test-pinned (12 xunit tests; full suite
> 1516/1516). Everything else in this slice (settlement/reconciliation stratum reads,
> realized-position writeback) remains deferred.

- **Purpose:** carry selection lane, hypothesis, expected/actual recipe/regime/signals,
  substitution facts, and evidence counts through the existing artifact without altering
  decision semantics; preserve core/wildcard strata for settlement/reconciliation.
- **Input/output:** frozen flight plan facts in; artifact provenance fields out.
- **Non-goals:** no retrieval activation, no source addition, no
  confidence/posture/lean/prompt change, no new model call.
- **Authority/activation:** none / `none` (provenance plumbing only).
- **Dependencies:** Slice 2.
- **Tests/evidence:** artifact round-trip fixtures; stratification preserved through the
  settlement/reconciliation read path; no-decision-drift regression.
- **Rollback:** revert; provenance fields additive.
- **Separate authorization:** required.

### Slice 4 -- Interrogate signal-need proposal projection (deferred)

- **Purpose:** the proposal-only typed contract (`SignalNeedProposal`, name reviewable)
  plus deterministic/read-side projections to inspect consequential questions, gaps,
  known/unmapped signal concepts, and Verify disposition.
- **Input/output:** existing artifact interrogation/probe/follow-up facts (and Slice-3
  provenance) in; proposal records with all authority fields false out.
- **Non-goals:** `ProbeRequest` stays execution-safe and closed; no Tool Gateway
  execution; no artifact mutation; no extra model call unless a later separately approved
  design proves one necessary.
- **Authority/activation:** none / `none` (read-side).
- **Dependencies:** Slice 3 (hypothesis linkage), Slice 1 contracts.
- **Tests/evidence:** projection fixtures incl. unknown-signal -> `unmapped_candidate`,
  grounded -> `already_grounded`, unsupported and not-evaluable paths; authority-ledger
  always-false guard tests.
- **Rollback:** revert; read-side only.
- **Separate authorization:** required.

### Slice 5 -- callable protocol-service seam and re-entry contract (deferred)

- **Purpose:** design then implement the smallest non-network service boundary over the
  existing protocol contracts before any remote endpoint; prove orchestrated and
  independently invoked paths share station cards, authority, idempotency, trace, and
  output contracts; explicitly close the `Perceive refresh -> Interrogate.Verify ->
  Discern` gap named by decision 0011 (ledger entry 27).
- **Input/output:** existing protocol/station contracts in; an in-process governed
  invocation seam with explicit tenant/run/artifact-version/correlation/idempotency/
  authority/cost/trace/termination state out.
- **Non-goals:** no HTTP/remote endpoint (any endpoint is its own runtime-surface slice
  under TDD and the activation ladder); no analyze-call split; no chain activation.
- **Authority/activation:** none beyond existing dormant seams / activation rung not
  advanced.
- **Dependencies:** Slices 1-4; decision 0011.
- **Tests/evidence:** contract-equality tests across both invocation paths; Verify
  re-entry covered by fixtures; protected-field guards regression.
- **Rollback:** revert; seam is dormant until Slice 6.
- **Separate authorization:** required.

### Slice 6 -- limited orchestrated probe-refresh activation (deferred, own gate)

- **Purpose:** first bounded activation of the orchestrated loop, only after earlier
  slices prove the contracts: development/local first, tenant-scoped, operator-reviewed,
  audited, default off; authorized retrieval only through platform retrieval/Tool Gateway
  (`platform.retrieve`); reuse the existing dormant chain where valid rather than
  building a second authority.
- **Non-goals:** no automatic artifact/decision/confidence/posture/lean mutation; no
  production or public exposure; no scheduler.
- **Authority/activation:** explicit feature-flag design per the probe-refresh readiness
  review; activation-ladder discipline (stop at the audited non-mutating rung absent a
  further gate).
- **Dependencies:** Slices 2-5; the standing probe-refresh activation blockers resolved;
  a dedicated operator activation authorization.
- **Tests/evidence:** diagnostics capture before/after; audit-ledger-only persistence
  proof; protected-field mutation impossibility proof.
- **Rollback:** single-flag disable; revert commits.
- **Separate authorization:** required (activation-class).

### Later governed slices (recorded, not scheduled)

Remote service exposure/deployment; signal-to-capability mapping and retrieval-source WIs;
governed historical-recapture capability; reconciliation-driven wildcard promotion;
commercial packaging/Stripe entitlement/pricing/hosted access.

## acceptance criteria  <!-- LITE -->

Slice 1 (checked at close, evidence in the closeout report):

- WI-0036 exists, is MOC-registered, status `in-progress`, and states documentation
  Slice 1 complete / implementation not started;
- the 20 binding operator decisions are represented without semantic drift;
- core/wildcard scheduling, the 25 percent cap, the bounded substitution reserve,
  reserve-first precedence, strongest-novelty priority, and the one-core hard minimum are
  unambiguous and desk-tested (3->0, 4->1, 8->2; bounded reserve on small flights;
  core-qualified reserve outranks wildcard; substitution above 25 percent realized
  one-for-one from the frozen reserve; all-core-drop hard stop; no-qualified-wildcard
  no-forced-spend; post-freeze candidates never substitute; substitution never raises the
  scheduled run count or spend ceiling; market-missing reachable; recapture rejected);
- wildcard/core evidence stays stratified through capture and reconciliation in every
  contract statement;
- preflight is placed as Perceive-side target architecture without misreporting current
  code;
- the canonical loop is documented as Question -> Probe -> authorized retrieval ->
  Perceive refresh -> Verify -> Discern and the dormant-chain Verify re-entry gap is
  named honestly;
- `SignalNeedProposal` is proposal-only and cannot grant retrieval/tool/mutation/decision
  authority;
- callable protocol services are documented as a later governed target, not built;
- every slice description carries boundaries, contracts, gates, verification, and
  rollback;
- affected planning/timeline/doctrine records agree on sequencing and preserve the
  separate July 22 authorization;
- strict planning snapshot reports zero warnings;
- protected pre-existing dirty/untracked state is byte-identical; dai is unchanged;
- exactly one local vault docs commit exists; no runtime action, source call, spend, or
  remote mutation occurred.

Slices 2-6: each slice's acceptance criteria are finalized in this WI at its
authorization time against the contracts in the architecture record.

## test plan  <!-- LITE, written before implementation -->

- Slice 1: desk-scenario verification (wildcard arithmetic + loop authority, recorded in
  [[wi-0036-wildcard-evidence-discovery-loop-planning-2026-07-21-v1]]), strict planning
  snapshot, OKF review checklist, link/front-matter checks, vault grill.
- Slice 2: fixture tests for every invariant above (cap, floor, one-core, freeze
  immutability, substitution eligibility/labeling/recording, lexicographic ordering incl.
  tie-breakers, settled-vs-unsettled separation, market-missing reachability,
  recapture-rejected, all-false authority ledger), plus canonical-JSON byte determinism.
- Slice 3: artifact provenance round-trip + stratification + decision-semantics
  non-change regression.
- Slice 4: proposal projection fixtures per mapping status + always-false authority
  guards.
- Slice 5: dual-invocation contract equality + Verify re-entry + protected-field guards.
- Slice 6: activation gate tests, diagnostics capture, audit-only persistence proof.

## implementation notes

Language/placement decisions (planner-side python vs platform-side C#, and whether the
Slice-2 contract is WI-0036-owned and WI-0034-consumed) are resolved at Slice-2
authorization with live verification, not pre-decided here. The dormant probe-refresh
chain is reused where valid; a second refresh authority is a rejected design.

## docs to update

Done in Slice 1: MOC; [[daily-evidence-acquisition-orchestrator-v1]] (wildcard lane +
preflight placement + proposal-only signal needs + promotion); cohort doctrine (bounded
wildcard lane + evidence separation); perceive/interrogate phase docs, probe-refresh
readiness, recipe architecture, station blueprint (current-vs-target reconciliation +
loop links); deferred-decisions ledger (entry 1 note, new entry 27);
[[platform-delivery-timeline-v1]] (wi-0036 initiative, proposed_by_system only); WI-0034 /
WI-0035 narrow seam links. Later slices name their own doc updates at authorization.

## verification commands  <!-- LITE -->

- `pwsh <DAI_REPO_ROOT>/scripts/dev/planning/build-next-slice-snapshot.ps1 -OutputPath <scratch>/next-slice-snapshot.json -Strict` (exit 0, zero warnings)
- `git -C <DAI_VAULT_ROOT> diff --check` and staged-allowlist review
- protected-state hash comparison (dai csproj; vault graph.json/CLAUDE.md/untracked pair; Welcome.md still deleted)
- machine-path / secret / `authorized: true` / runtime-claim scans over the changed files
- Slices 2+: the dotnet/pytest suites named per slice at authorization time.

## risks

Lane/tier conflation; novelty-ordering drift into an unfixtured score or model ranking;
settled-vs-unsettled count corruption; proposal-scope creep into `ProbeRequest` or
station/gateway permissions; premature activation of the dormant chain before Slices 2-5
prove the contracts; documentation drift claiming target capabilities as implemented
(guarded by the current-vs-target ledger in the architecture record).

## links  <!-- LITE; all 8 required at close, per work-item-traceability -->

- work item: WI-0036 (ADO: AB#- when wired; no ADO item created)
- branch: `wi/0036-wildcard-evidence-discovery-loop` (dai-vault only; dai remained
  read-only on `main` at `8369d64a2b4ed29ab1c6297de81270d2f9dd8a46` for Slice 1;
  INTEGRATED to vault main `b04e6421` 2026-07-21); implementation branches
  `wi/0036-wildcard-capture-flight-plan` (dai + dai-vault, matching, from
  8369d64 / b04e6421) -- INTEGRATED 2026-07-22 by coordinated fast-forward: dai main
  `ce34a9e74659b42c71317267c64901a24ceb7091`, dai-vault main
  `2cdb275b85229c6ee11a7b2930dc50d847ae8240`, both == `origin/main`
- pr: - (no PR; direct fast-forward publication of both mains 2026-07-22)
- commits: Slice-1 docs `b04e6421` (vault); Slice-2 chain dai
  `e64baab -> b0ff396 -> 994b0c1 -> 1c556f4 -> ce34a9e` and vault
  `8d98da9 -> 967170d -> 8780e92 -> 2cdb275`
- tests: Slice 1 docs-only; Slice 2 as integrated -- agent-service pytest 617/617
  (wi-0036 targeted 39 core + 14 cli) and DevCore.Api.Tests 1530/1530 (16 provenance
  unit + 10 controller-host), reverified on the published mains 2026-07-22
- verification notes: [[wi-0036-wildcard-evidence-discovery-loop-planning-2026-07-21-v1]]
  (desk scenarios + snapshot + scans) and the current-slice handoff
- docs updated: see "docs to update" (Slice 1 list)
- lessons: define the loop's decision authority (0011) before any activation slice so
  re-entry cannot be skipped by construction; verify current-vs-target against source
  before writing doctrine (the dormant chain's missing Verify re-entry was confirmed in
  source, not assumed)

## final handoff requirements

Slice handoff appended to `06 Execution/handoffs/current-slice.md`; WI status stays
`in-progress`.

*(Historical, Slice 1: disposition was documentation complete / local commit only / not
pushed / not merged, with the next governed action being operator review, then the July 22
events-gate observation, then Slice 2 under a new implementation authorization. Slice 1
was integrated to vault main `b04e6421` on 2026-07-21 and the sequencing was superseded
the same day.)*

**Current (2026-07-22):** Slice 1, Slice 2, and the minimum Slice-3 provenance seam are
INTEGRATED and published (dai `ce34a9e7`, dai-vault `2cdb275b`, both == `origin/main`).
The Slice-3 remainder and Slices 4-6 remain deferred and separately gated. Next governed
action = the separately authorized Slice-3 remainder (settlement/reconciliation stratum
reads plus realized-position writeback). The 2026-07-22 events-gate observation and any
paid wildcard flight remain separately governed; nothing here authorizes activation,
capture, retrieval, or spend.
