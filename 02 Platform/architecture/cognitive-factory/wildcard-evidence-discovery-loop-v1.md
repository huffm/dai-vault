# Wildcard Evidence Discovery Loop v1

**date:** 2026-07-21
**status:** active doctrine for contracts and boundaries; every runtime capability described here is TARGET architecture, deferred and separately gated (WI-0036 Slices 2+). No runtime behavior exists or changed with this record.
**scope:** the governed vertical discovery loop that widens safe evidence acquisition (wildcard candidate lane in preflight), preserves selection provenance through production artifacts, and turns artifact interrogation into proposal-only signal-need inputs for later retrieval work.

## purpose

Define the contracts of WI-0036 (Wildcard Evidence Discovery Loop v1) in one focused record:
the candidate-lane semantics and wildcard scheduling rules, the artifact/interrogation
signal-need proposal contract, and the orchestrator-mediated feedback loop and future
callable protocol-service target. Decision authority for the loop is
`02 Platform/decisions/0011-orchestrated-interrogate-perceive-refresh-loop-v1.md`; work-item
state and decomposition live in
`02 Platform/system-development/work-items/WI-0036-wildcard-evidence-discovery-loop-v1.md`.

## problem it solves

Three currently separate concerns, none of which closes the learning loop alone:

1. Deterministic evidence planning (WI-0034 planner, WI-0035 screen) optimizes the dominant
   known objective. Underrepresented recognized recipes/versions, data regimes, and signal
   combinations can be missed repeatedly, so settled coverage narrows instead of widening.
   The regime-discovery precedent (`06 Execution/reports/regime-discovery-candidate-selection-v1.md`)
   proved thin-regime candidates exist and are reachable, but no governed lane schedules them.
2. Artifact production already has Interrogate Question/Probe/Verify semantics and a typed
   execution-safe Probe handoff (`ProbeRequest`), but discovery of novel retrieval needs is
   narrower than the desired loop: the Probe template set is deliberately closed (four
   signals), and consequential questions the artifact raises do not become reviewable
   retrieval proposals.
3. The dormant probe-refresh machinery does not yet embody the complete orchestrator-mediated
   `Question -> Probe -> authorized retrieval -> Perceive refresh -> Verify -> Discern` loop
   (the shipped chain re-enters Discern without an `Interrogate.Verify` step) or a remotely
   callable protocol-service boundary.

## strategic fit

The objective is a governed vertical discovery loop, not "run more games." It widens safe
evidence acquisition, preserves why each wildcard was selected, surfaces which additional
signals might have resolved the artifact's consequential questions, and turns those findings
into proposal-only inputs for later, separately reviewed retrieval-source work items.
Commercially, this is part of building the governed system and charging for access; this
record changes no pricing, Stripe, entitlement, buyer claim, or delivery authorization.

## mental model

```text
coverage gaps (settled evidence, by exact recipe/version/regime/signal-combination)
-> wildcard hypotheses (written, frozen)
-> preflight (Perceive-side, target): discover and bind available evidence + candidate
   conditions; schedule core + reserve + capped wildcard lanes; freeze
-> operator authorization -> capture -> artifact production carries wildcard provenance
-> Interrogate raises consequential questions and signal-need proposals (proposal-only)
-> settlement/reconciliation reads core and wildcard strata separately
-> reconciliation-backed review: map / open retrieval WI / reject / defer; promotion of a
   wildcard target into ordinary selection is a later reconciliation-owned decision
```

## what it is

- A closed candidate-lane vocabulary and a bounded wildcard scheduling policy for future
  paid-flight preflight.
- A provenance contract that keeps wildcard identity, hypothesis, and expected-vs-actual
  recipe/regime/signal facts attached to the produced artifact and readable at
  settlement/reconciliation as a separate stratum.
- A proposal-only interrogation output contract (`SignalNeedProposal`, name tentative) that
  can never grant retrieval, tool, mutation, or decision authority.
- A statement of the target orchestrated refresh loop and the future callable
  protocol-service shape, bounded by decision 0011 and the Tool Gateway doctrine.

## what it is not

- Not implemented. No wildcard planner, no proposal type, no service seam, no loop runtime
  exists as of 2026-07-21; the current planner (WI-0034) has no wildcard lane and the
  current Probe path is the closed deterministic template set verified in source.
  *(current-state update, 2026-07-21, WI-0036 Slice 2 under the operator sequencing
  override -- supersedes the first sentence for the PLANNER and PROVENANCE layers only:
  the offline deterministic wildcard flight-plan core + CLI
  (`services/agent-service/app/services/wildcard_flight_plan.py` / `_cli.py`; request
  `wildcard-flight-plan-request/1.0`, plan `wildcard-flight-plan/1.0`, realization
  `wildcard-flight-realization/1.0`, CLI `wildcard-flight-plan-cli/1.0`) and the minimum
  run-provenance seam (`flight-selection-provenance/1.0`;
  `DevCore.Api/AgentRuns/FlightSelectionProvenance.cs` + optional
  `CompetitionMatchupInput.FlightSelection` threading into the artifact on success and
  failure paths, internal inspection only) are IMPLEMENTED on local review branches,
  default off, all-false ledgers, not integrated. The `SignalNeedProposal` type, the
  callable service seam, the refresh-loop runtime, and every activation remain
  NOT implemented and separately gated. Pre-integration correction, same day, per the
  independent review findings A-G: the provenance wire contract is ONE canonical
  camelCase shape carrying the exact eight-key all-false `authorityLedger` (data keys
  stay canonical snake_case) and a provider-scoped `substitutedFor` object; the
  selection lane `reserve` is preserved end-to-end (a core-qualified backup is never
  relabeled core; `realized_via = scheduled | core_reserve | wildcard_substitution`
  records the slot fill, and the one-core minimum counts reserve-via-core); the
  recognized recipe/version/regime registry is DERIVED from the canonical prompt
  manifest at the boundary (hash-verified load, digest-pinned taxonomy
  `prompt-manifest/<v>+sha256:<digest>`) and caller input is narrowing-only; frozen
  plans and realizations are validated by strict closed-contract validators INDEPENDENT
  of the sha-256 content fingerprint, which is content identity only and never
  authenticity or authority; and provenance export derives realization facts internally
  from plan + availability, never from a hand-authored realization artifact.)*
- Not an authorization: nothing here authorizes model calls, source retrieval, capture,
  settlement, spend, endpoint creation, or activation of the dormant probe-refresh chain.
- Not a relaxation of `ProbeRequest`, station permissions, or Tool Gateway permissions.
- Not historical-identity recapture, which stays prohibited by default (a governed recapture
  capability would be a separately designed future work item).
- Not a promotion mechanism: one wildcard result never promotes itself into ordinary
  selection; promotion is owned by a later reconciliation-backed decision.

## candidate-lane semantics (closed)

Selection lane is a flight-plan role. It is distinct from, and must never be collapsed
into, the market-screen classification tier (`primary | secondary | excluded | blocker` in
`market-contrast-screen/1.x`): tier is a screen fact about one candidate's market-evidence
value; lane is the planner's selection intent for the flight. A candidate carries both.

| lane | meaning |
|---|---|
| `core` | selected to serve the flight's primary evidence objective under standing doctrine; highest selection priority |
| `reserve` | eligible backup for a core slot under the existing deterministic primary/reserve allocation |
| `wildcard` | a safe candidate targeting an underrepresented or unusual **recognized** prompt recipe/version, data regime, or signal combination; requires a measurable evidence-coverage gap AND a written novelty hypothesis; below core in priority |
| `excluded` | validly screened out (out of filter); a valid decision, not a failure |
| `blocker` | trust failure (identity, readiness, duplication, contradictory input); never generate |

Safety is non-negotiable for wildcards: a wildcard passes the same identity, schedule-state,
readiness, and blocker gates as any other candidate. "Wildcard" widens the objective, never
the safety envelope. Market-missing recipes are valid wildcard targets when market missing
is their expected recognized data shape; a universal market-baseline requirement must not
make those recipes unreachable.

## wildcard scheduling policy (initial)

- **Cap.** Initially scheduled wildcard capacity is at most 25 percent of the flight:
  `wildcard_scheduled_max = floor(total_scheduled_runs / 4)`. A flight smaller than four
  scheduled runs schedules zero wildcards unless a future separate policy explicitly
  changes the rule. Desk arithmetic: 3 -> 0; 4 -> at most 1; 8 -> at most 2.
- **Priority.** Wildcards rank below core in selection. No qualified wildcard -> a
  core/reserve-only flight; wildcard slots are never force-filled and never force spend.
- **Freeze.** The flight (core, reserve where applicable, wildcard) is frozen with
  immutable provenance before any paid call, per
  `06 Execution/patterns/cohort-selection-and-run-discipline-v1.md`. No new candidate may
  be introduced after freeze.
- **Bounded substitution reserve (contract correction, 2026-07-21).** The frozen flight
  plan may carry a substitution-only wildcard pool, distinct from the scheduled lane
  slots:

  ```text
  wildcard_scheduled_max = floor(total_scheduled_runs / 4)
  minimum_executed_core_runs = 1
  wildcard_substitution_reserve_max =
    max(0, scheduled_core_runs - minimum_executed_core_runs)
  ```

  A closed flight-plan field distinguishes planned use:
  `wildcard_plan_role = scheduled | substitution_reserve`. A `substitution_reserve`
  candidate retains lane `wildcard` (never relabeled `reserve` or `core`); it does not
  occupy an initially scheduled run slot and therefore does not count against
  `wildcard_scheduled_max`; the pool is bounded by
  `wildcard_substitution_reserve_max` and is never an unlimited authorized set. Every
  substitution-reserve candidate must independently pass the wildcard qualification and
  safety gates, be selected by the closed strongest-novelty ordering, be present in the
  immutable frozen plan, and be explicitly covered by that flight's operator
  authorization before any paid execution. Too few qualified wildcards leave the pool
  partially or wholly empty; it is never force-filled and never forces spend. Because the
  scheduled cap and the substitution contingency are intentionally different controls, a
  flight smaller than four -- which initially schedules zero wildcards -- may still carry
  bounded substitution-reserve wildcards.
- **Substitution and precedence (contract correction, 2026-07-21).** When a scheduled
  core candidate becomes unavailable, the closed precedence is:

  ```text
  1. use the existing deterministic, eligible core-qualified reserve;
  2. only when no such reserve is eligible/available, use an eligible frozen wildcard
     substitution reserve;
  3. when multiple wildcard substitutes are eligible, choose strongest novelty using the
     lexicographic ordering below;
  4. if neither class can fill the slot, preserve fail-closed non-execution; never invent
     a candidate or perform a new retrieval.
  ```

  A substitution is one-for-one for the vacated scheduled core slot; it never increases
  `total_scheduled_runs` or the flight's maximum paid-run count. The substitute remains
  labeled `wildcard`; the substitution and the missing-core reason are recorded; no
  post-freeze candidate addition is permitted; at least **one scheduled core run** must
  remain executable and execute. If substitution would yield zero core runs, the flight
  hard-stops for a new operator decision rather than executing all-wildcard.
- **Realized share.** The 25 percent cap governs the initially scheduled flight. The
  realized wildcard percentage may exceed 25 percent only through eligible one-for-one
  substitutions from the frozen, explicitly authorized substitution reserve while the
  one-core minimum stays satisfied.
- **Operator approval.** Preflight may propose a wildcard lane whenever qualified
  candidates exist; use remains explicitly operator-approved. A proposed lane authorizes
  nothing.

## strongest-novelty substitution ordering (deterministic, falsifiable)

When more than one authorized wildcard can substitute, the strongest novelty is selected --
not the candidate closest to the missing core objective, and never by unconstrained model
ranking. The ordering compares **eligible wildcards only**: it never allows a wildcard to
outrank an eligible core-qualified reserve, which always fills a vacated core slot first
(see the substitution precedence above). The later implementation (WI-0036 Slice 2, the
named authority, fixture-proven) must realize this as a closed lexicographic ordering over
facts frozen at flight freeze:

1. ascending settled evidence count for the exact (expected recipe id, recipe version,
   expected data regime) triple;
2. ascending minimum settled evidence count over the candidate's recognized
   signal-combination ids;
3. descending count of distinct recognized novelty dimensions named by the frozen
   hypothesis (each dimension must map to the recognized taxonomy; free text never counts);
4. ascending scheduled start instant (UTC);
5. ascending stable candidate identity (provider-scoped external event id, ordinal).

Rules: all counts are computed from settled evidence only and frozen with the flight;
captured-but-unsettled evidence is visible but never counts as settled coverage; there is
no floating "novelty score" without a named authority and fixtures; ties resolve only by
the stable identity/time tie-breakers above.

## candidate/flight provenance contract (proposed fields, later implementation)

Every scheduled candidate and executed flight must preserve at minimum:

- stable candidate identity (provider-scoped) and target date;
- selection lane and lane version;
- `wildcard_plan_role` (`scheduled | substitution_reserve`) plus the flight's
  `wildcard_scheduled_max`, `wildcard_substitution_reserve_max`, and realized pool counts;
- scheduled position and realized position;
- wildcard hypothesis id and hypothesis text (wildcards only);
- novelty dimensions (recognized taxonomy values);
- expected recipe id/version and expected data regime;
- recognized signal-combination ids;
- settled evidence count by exact recipe/version/regime (at freeze);
- captured-but-unsettled count, carried separately and never merged into settled counts;
- wildcard reason and selection provenance;
- preauthorization/substitution eligibility;
- substituted-for candidate identity and substitution reason (when substitution occurred);
- actual recipe id/version and actual data regime after production;
- realized signals and source-depth facts;
- settlement/reconciliation linkage; and
- an authority ledger in which every execution permission is explicit and false rather
  than inferred (matching the events-gate/planner booleans-only precedent).

Core and wildcard evidence stay stratified through capture, settlement, and reconciliation
so calibration reads can separate them; pooling the strata silently is a defect.

## signal-need proposal contract (`SignalNeedProposal`, name tentative)

The execution-safe `ProbeRequest` is NOT widened to arbitrary model-proposed tools; its
closed template set stays closed. A separate proposal-only concept carries what Interrogate
discovers. If documentation review later finds an existing canonical type that already
covers this, adopt that name; as of this record no existing type does (`ProbeRequest` is
execution-adjacent and closed; the WI-0034 planner's missing-capability records are
planner-side input-capability facts, not artifact-production interrogation output).

Minimum fields:

- proposal identity/version and artifact/run/correlation linkage;
- originating protocol/station (`interrogate.question`, `interrogate.probe`, or
  `interrogate.verify` as appropriate);
- wildcard hypothesis linkage when present;
- the consequential question;
- signal concept and nullable canonical signal id;
- mapping status: `known | unmapped_candidate | already_grounded | unsupported |
  not_evaluable`;
- why the signal would be decision-relevant;
- expected evidence class and candidate source classes (never an authorized tool);
- observed artifact gap and source-depth context;
- Verify disposition;
- execution, retrieval, mutation, confidence, posture, and lean authority all false; and
- later review disposition: map to an existing capability, open a retrieval-source WI,
  reject, or defer.

Free text may explain a hypothesis but must not control deterministic eligibility, ranking,
tool choice, or execution authority. Interrogate proposes signal needs; it does not add
sources. Source/retrieval additions are later, separately reviewed work items.

## feedback loop and callable-service target (per decision 0011)

The coherent governed refresh loop is:

`Question -> Probe -> authorized retrieval -> Perceive refresh -> Verify -> Discern`

- Interrogate remains a requester, never an execution authority. An orchestrator mediates
  authorization and routes retrieval through platform-owned retrieval/Tool Gateway
  boundaries. Direct Interrogate-to-Perceive self-invocation remains forbidden.
- The orchestrator, not Interrogate, owns re-entry, idempotency, authorization, Tool
  Gateway routing, audit, cost, and termination.
- Preflight is conceptually part of **Perceive**: it discovers and binds what evidence and
  candidate conditions are available before paid execution. This is target doctrine; the
  current implementation of preflight is the offline WI-0034/WI-0035 planner/screen path,
  and the current runtime Perceive is the sports retrieval path.
- Protocols and stations should eventually be invocable through the full orchestrated
  pipeline OR as independently callable governed services by other system services:
  shared versioned decision artifact as the unit that moves; bounded typed inputs and
  outputs; explicit tenant, run, artifact version, correlation, idempotency, station-card
  version, authority, cost, trace, and termination state; identical contracts on both
  invocation paths; remote callability never bypassing the Tool Gateway or station
  permissions. This is a target; this record creates no endpoint and splits no model call.
- A refresh cannot overwrite protected artifact fields or silently mutate confidence,
  posture, lean, identity, or history (the probe-refresh protected-field guards remain
  authoritative).
- Implementation must resolve how refreshed Perceive evidence re-enters
  `Interrogate.Verify` before Discern -- the shipped dormant chain lacks that re-entry
  (verified in source 2026-07-21; see current-vs-target below).

## current implemented truth (verified against source, 2026-07-21)

- The current sports path performs deterministic retrieval before one shared analyze model
  call and deterministic composition afterward (`SportsRunArtifact`, `SportsComposer`).
- `Interrogate.Question` and `Interrogate.Verify` are model-emitted within the shared
  analyze call; `Interrogate.Probe` is deterministic at compose time, derived from
  `SignalFollowUps` (`CognitiveProtocolBuilder.BuildProbe`); it is not a model call.
- `ProbeRequest` is a proposal/handoff contract: it fetches nothing, invokes no Tool
  Gateway, invokes no Perceive step, and mutates no artifact
  (`DevCore.Api/AgentRuns/ProbeRequest.cs`). Probe templates cover only the four
  doctrinally characterized signals (`sharp_public`, `market`, `rest_schedule`,
  `starting_pitching`); unknown signals are dropped, never granted retrieval authority.
- The Stage-2 development endpoint executes only the inert deterministic
  `interrogate.probe` audit-only path under strict config gates (Enabled, AllowExecution,
  AuditOnly true; AllowArtifactMutation false); it makes no model, gateway, external, or
  DB call and mutates nothing (`CognitiveFactoryExecutionController`,
  `CognitiveFactoryExecutionGate`).
- The probe-refresh value-object/audit chain is dormant with no production pipeline
  consumer or endpoint; its mutation and gateway flags default false
  (`ProbeRefreshChainAssembly`).
- The dormant chain's sequence after Perceive intake proceeds directly into
  Discern/Decide/Synthesize preview with **no explicit `Interrogate.Verify` re-entry
  step**. This is recorded as a target-contract gap (decision 0011; deferred-decisions
  ledger entry 27), not claimed to exist.

## truth hierarchy

1. Runtime source and tests (`DevCore.Api` AgentRuns/Protocols/Controllers.Dev and their
   suites) for every current-implementation claim above.
2. Decision 0011 and this record for the target loop, lane, and proposal contracts.
3. WI-0036 for state, decomposition, and gates.
4. Handoffs and navigation records.

## source or vault references to verify

- `platform/dotnet/DevCore.Api/AgentRuns/ProbeRequest.cs`,
  `CognitiveProtocolBuilder.cs`, `SportsComposer.cs`, `SignalFollowUpEvaluator.cs`
- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshChainAssembly.cs`,
  `ProbeRefreshPerceiveIntake.cs`, `ProbeRefreshDiscernReweigh.cs`,
  `ProtocolNodeRunner.cs`
- `platform/dotnet/DevCore.Api/Controllers/Dev/CognitiveFactoryExecutionController.cs`,
  `Diagnostics/CognitiveFactoryExecutionGate.cs`
- `02 Platform/architecture/tool-gateway-and-agent-permissions-doctrine-v1.md`
- `02 Platform/architecture/cognitive-factory/cognitive-factory-runtime-activation-readiness-v1.md`

## acceptance criteria (for this doctrine record)

- Lane vocabulary, cap arithmetic, substitution invariants, and the strongest-novelty
  ordering are unambiguous enough that a deterministic implementation can be tested
  against them with fixtures.
- The proposal contract cannot be read as granting any execution, retrieval, mutation, or
  decision authority.
- Current-vs-target is separated in every section; no deferred capability is described as
  implemented.

## risks and failure modes

- Lane/tier conflation: treating a market-screen `secondary` as a wildcard, or relabeling
  a wildcard as core after substitution. Both are contract violations.
- Novelty drift: replacing the closed lexicographic ordering with an unconstrained score
  or model ranking.
- Coverage-count corruption: letting captured-but-unsettled evidence inflate settled
  coverage, which would silently retire hypotheses that were never tested.
- Proposal-scope creep: widening `ProbeRequest` or granting `SignalNeedProposal` any
  authority "for convenience."
- Premature activation: wiring the dormant probe-refresh chain or adding endpoints before
  WI-0036 Slices 2-5 prove the contracts and a separate activation authorization exists.

## deferred decisions

- Remote service exposure/deployment of protocol stations (own slice + activation ladder).
- Signal-to-capability mapping and any retrieval-source addition (own WIs).
- Governed historical-identity recapture (prohibited by default until separately designed).
- Reconciliation-driven wildcard promotion policy.
- Commercial packaging, Stripe entitlement, pricing, and hosted access for the governed
  system (recorded direction: build the governed system and charge for access; activation
  is a separate operator decision).
- Whether `SignalNeedProposal` keeps its tentative name or adopts an existing canonical
  type at implementation review.

## related docs

- `02 Platform/decisions/0011-orchestrated-interrogate-perceive-refresh-loop-v1.md`
- `02 Platform/system-development/work-items/WI-0036-wildcard-evidence-discovery-loop-v1.md`
- `04 Products/sports-v1/daily-evidence-acquisition-orchestrator-v1.md`
- `06 Execution/patterns/cohort-selection-and-run-discipline-v1.md`
- `06 Execution/reports/regime-discovery-candidate-selection-v1.md`
- `02 Platform/architecture/cognitive-factory/probe-refresh-chain-activation-readiness-v1.md`
- `02 Platform/architecture/cognitive-factory/protocol-station-blueprint-v1.md`
- `02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md`

## recommended next slice

Superseded 2026-07-21: the operator sequencing override authorized and delivered Slice 2
plus the minimum Slice-3 seam offline/default-off (local branches
`wi/0036-wildcard-capture-flight-plan`). The next governed action is independent review +
coordinated integration of those branches; the separately governed July 22 events-gate
observation stays its own action; any paid wildcard flight requires a future explicit
flight authorization. Slices 4-6 remain deferred.
