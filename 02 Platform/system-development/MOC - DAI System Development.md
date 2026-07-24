---
title: "MOC - DAI System Development"
type: "moc"
date: "2026-07-09"
status: "in-progress"
project: "DAI"
slice: "DAI System Development v1"
repos:
  dai: "docs-only"
  dai-vault: "docs-only"
tags:
  - system-development
  - moc
  - okf
related:
  - "02 Platform/system-development/operating-model.md"
  - "06 Execution/patterns/okf-yaml-front-matter-pattern-v1.md"
---

# MOC - DAI System Development

Hub for the DAI development operating model: how work is specified, traced, implemented,
tested, and turned into reusable knowledge. Execution mechanics stay in the slice doctrine
(`06 Execution/agent-slice-workflow-doctrine-v1.md`); this subtree governs intent and
traceability around it.

## front matter placement decision (recorded here on purpose)

The canonical 9-field OKF schema (`06 Execution/patterns/okf-yaml-front-matter-pattern-v1.md`)
is documented for `06 Execution` docs. This subtree deliberately extends the same schema —
same nine fields, same allowed values — to `02 Platform/system-development/`. One new `type`
value is introduced: `moc` (no existing value fits a map-of-content). This is a placement
extension, not a second dialect. A future hygiene audit should read this as intent, not drift.

## operating model

- [[operating-model]] — the spine (work item / spec / code), meaningful-change threshold, six lenses, learning loop
- [[work-item-traceability]] — IDs, naming, link topology, local-spine mode until Azure DevOps is wired
- [[implementation-lifecycle]] — intake → spec → gate → branch → implement → verify → review → close

## implementation doctrine

- [[frontend-implementation]] — Angular/sports-app repo patterns (seeded, in-progress)
- [[backend-implementation]] — platform API repo patterns (seeded, in-progress)
- [[architecture-contracts]] — truth hierarchy, buyer boundary, correlation, error semantics
- [[testing-strategy]] — what must be tested per change class

## design system

- [[component-rules]] — chip primitive, long-token treatment, status color tokens
- [[interaction-states]] — the shipped status/feedback system (StatusDescriptor, queue states)
- [[visual-qa-checklist]] — pre-completion visual gate

## work items

- [[_template]] — tiered work-item spec template (lite core + feature-class sections)
- [[WI-0001-chip-primitive-and-long-token-treatment]] — chip primitive, long-token treatment,
  status tokens (**complete** 2026-07-09; first proving run; integrated to dai main
  `bb10c3c` and pushed, branch deleted)
- [[WI-0002-artifact-chip-primitive-alignment]] — evaluate aligning the squared
  `.artifact-chip` tags with the chip system (**BACKLOG, not authorized**; disposition
  open, activation gate in spec)
- [[WI-0003-shared-chip-and-long-token-module-promotion]] — promote `.chip` +
  `app-long-token` to a shared boundary (**BACKLOG, not authorized**; gated on a concrete
  second consumer)
- [[WI-0004-platform-api-shutdown-process-match]] — Truthful Platform API Shutdown v1:
  `stop-platform-api.ps1` now verifies port release + process exit before reporting success,
  targets the real listener owner, and refuses to kill an unrelated port owner (**complete +
  integrated** 2026-07-11; dai `e8050a9` fast-forwarded to main and pushed, origin synced;
  unit 15/15 + live scenarios A-D; branch pushed + retained)
- [[WI-0005-starter-retrieval-caches-transport-failures]] — Identity-Safe Starter Cache v1:
  `MlbStarterClient` no longer caches transport failures as "no starters announced"; cache admits
  only fully-grounded results, identity-safe key, configurable TTL, cancellation propagated
  (**complete + integrated** 2026-07-11; dai `4693b9d` fast-forwarded to main and pushed,
  origin synced; 1092/1092 tests; branch pushed + retained)
- [[WI-0006-identity-safe-mlb-doubleheader-resolution]] — Identity-Safe MLB Doubleheader
  Resolution v1: a matchup is not an event identity. `/source-readiness` takes an optional
  `gamePk`; an ambiguous same-day matchup now fails closed with explicit candidates instead of
  `FirstOrDefault`-ing whichever game the provider listed first; starter cache re-keyed to
  `starters:v2:statsapi:mlb:{gamePk}` so game 1 and game 2 cannot collide (**complete +
  integrated** 2026-07-13; dai `4f8f381` fast-forwarded to main and pushed, origin synced;
  1120/1120 tests; live-verified on the real 2026-07-11 MIL@PIT split DH incl. adversarial
  cache ordering both directions; branch pushed + retained)
- [[WI-0007-mandatory-work-items-and-slice-synopsis-workflow]] — Mandatory Structured Work
  Items and Slice Synopsis Workflow v1: qualification gate owned by `dai-slice-runner`
  (qualifying slices need a WI before execution), canonical Slice Synopsis owned by
  `dai-agent-handoff` as every handoff's mandatory final section (**complete + integrated**
  2026-07-13; dai `41e0a46` fast-forwarded to main and pushed, origin synced; validation =
  single-owner greps + 6 desk-check scenarios; branch pushed + retained)
- [[WI-0008-evidence-grounded-next-slice-planner]] — Evidence-Grounded Next-Slice Planner v1:
  read-only deterministic planning snapshot (`scripts/dev/planning/`) + `dai-next-slice-planner`
  skill + operator-owned [[platform-delivery-timeline-v1]]; planner recommends one bounded next
  slice with provenance, never creates/authorizes/executes (**complete + integrated** 2026-07-13; dai `88c9f09` fast-forwarded to main and pushed, origin synced; 45/45 fixture asserts + real-state run + planner desk-run; branch pushed + retained)
- [[WI-0009-gamepk-propagation-through-competition-matchup-input]] — Propagate gamePk Through
  CompetitionMatchupInput v1: optional exact event identity on the initiating generation
  request so doubleheaders become safely capturable instead of only safely rejected; completes
  the WI-0006 seam (**complete, local only** 2026-07-13; authorized via the WI-0008 planner
  decision gate 12/12; 1127/1127 tests incl. 7 new + live non-paid regression; **complete +
  integrated** 2026-07-13, dai `d493f84` on main and pushed, origin synced; branch pushed +
  retained)
- [[WI-0010-planner-evidence-fidelity]] — Planner Evidence Fidelity v1.1: MOC integration
  fallback (WI-0001 false-unknown), delivered-candidate exclusion + explicit-alias dedup,
  structured reconciliation planning readout; four defects reproduced by the first
  post-WI-0009 planning run (**complete + integrated** 2026-07-14, dai `e64567f` on main and
  pushed, origin synced; 73/73 fixture asserts re-verified on main 2026-07-14; strict
  real-state run zero warnings; branch deleted local + remote — deviation from the
  branch-retained convention of WI-0004..0009)
- [[WI-0011-buyer-decision-brief-contract]] — Buyer Decision Brief Contract v1: canonical
  server-owned buyer pregame brief (GET /brief + deterministic markdown export at
  /brief/markdown); numeric confidence and threshold labels removed from every buyer
  surface (panel, buyer artifact wire, history, landing); persisted identity incl. gamePk;
  evidence-gated band as sole strength language; deterministic market-context copy via the
  fidelity guard; first V1 critical-path item (**complete + integrated** 2026-07-14, dai
  `140b5a2` fast-forwarded to main and pushed, origin synced; 1176/1176 C# + 134/134
  vitest re-verified pre-integration; review 10 findings fixed; live-verified on
  823845/822882; branch pushed + retained)
- [[WI-0012-settled-outcome-recap]] — Settled Outcome Recap v1: canonical buyer postgame
  recap (GET /recap + deterministic markdown at /recap/markdown) embedding the WI-0011
  brief verbatim; closed fail-closed state vocabulary incl. honest `no_result` for
  non-final settlements; persisted evaluation is the sole correctness source; no-position
  never scored; excluded renders exact non-evaluation copy; per-read only, no aggregates;
  all four buyer surfaces consolidated onto one loader + suppression tripwire; second V1
  critical-path item (**complete + integrated** 2026-07-14, dai `7152818`
  fast-forwarded to main and pushed, origin synced -- pure ff, tree-identical;
  1212/1212 C# + 134/134 vitest re-verified pre-integration; review 9 fixed / 1
  refuted; live-verified on 823845/823357/no-position/unsettled; branch pushed +
  retained)
- [[WI-0013-pilot-ops-hardening]] — Pilot Operations Hardening v1: fail-closed 409
  duplicate active-run creation guard (identity precedence gamePk-equality over
  orientation-insensitive matchup; per-identity in-process creation gate; excluded/failed
  never block; distinct doubleheader gamePks independently creatable; tenant-scoped,
  zero-spend/zero-row rejection); metering price coverage for every configurable model
  (gpt-4.1-mini added) with explicit pricingStatus + loud unpriced warning; V1 concierge
  operations runbook + delivery-ledger/operator-time templates + RC drill package
  (prepared, NOT executed; four separate authorization gates); third and final V1
  critical-path item (**complete + integrated** 2026-07-14, dai `85a8831`
  fast-forwarded to main and pushed, origin synced -- pure ff, tree-identical;
  1235/1235 C# + 453/453 pytest + 134/134 vitest re-verified pre-integration; review
  12 fixed / 2 refuted; branch pushed + retained; RC drill NOT executed, separately
  gated; single-operator-host constraint standing)
- [[WI-0020-platform-hardening-catalog-corrections]] — AI Engineering Hardening
  Catalog v1.1 Corrections: canonical seven-class evidence taxonomy applied across the
  four hardening docs; one branch policy (every pulled queue card branches, vault-docs
  included, no direct-to-main, ff-only separately authorized integration); G-10 recast
  as credential exposure classification + preparation with R-05-only remediation and
  the corrected exposure facts (Development.json untracked/never committed; sa
  password historically exposed in tracked appsettings.json until `ded9969`); fitness
  count fixed at fourteen adopted = eleven existing + three added with FC-C1..C3
  conditional and excluded (**complete + integrated** 2026-07-15, docs-only, dai-vault
  `5c2500b` fast-forwarded to main -- pure ff, tree-identical; branch pushed +
  retained; zero spend, zero writes, zero credential access; G-10 unexecuted, R-05
  gated, G-01 unresolved)
- [[WI-0021-protocol-hardening-candidate-specifications]] — AI Engineering Protocol
  Hardening Candidate Specifications v1: exactly six branch-ready implementation
  specifications (PH-01 failure corpus, PH-02 trace completeness, PH-03 abstention
  invariants, PH-04 artifact contract invariants, PH-05 delivery/entitlement guard,
  PH-06 tool authorization fitness) with 40 fields each, falsifiable acceptance
  criteria, Inspect/Prove/Guard packaging, WI-0020 evidence taxonomy, RC-neutral vs
  RC-affecting policy, readiness verdicts (3 READY / 2 open-question / PH-05 NOT
  READY); WIP-policy correction 361a2e6 (pulled PH-06 counts as active implementation
  branch) (**complete + integrated** 2026-07-15, docs-only, dai-vault e3a5eb3+361a2e6
  fast-forwarded to main -- pure ff, tree-identical; branch pushed + retained; zero
  spend, zero writes; NO candidate activated or pulled)
- [[WI-0022-representative-protocol-failure-corpus]] — Representative Protocol
  Failure Corpus v1 (PH-01 Green subset, first hardening pull): deterministic
  15-class failure corpus (schema 1.0) + integrity harness + real-seam
  characterization tests (+27 C# / +3 python; suites 1262/456 green);
  review-corrected classifications 9 guard_verified / 3 behavior_characterized /
  1 guard_missing / 1 policy_blocked / 1 not_applicable, every gap owned by
  PH-02..PH-06; evidence class contract-represented + fixture-proven only
  (**complete + reviewed + integrated** 2026-07-15: dai 85a8831 -> `a0ca54d`
  tests-only ff with RC-equivalence record + runtime smoke; vault `b7c6842`;
  branches pushed + retained; PH-01 CLOSED; WIP freed; zero spend/writes)

- [[WI-0023-tool-authorization-fitness]] — Tool Authorization Fitness v1 (PH-06 Green
  subset, second hardening pull): deterministic 41-capability inventory (10 registered
  tools + 8 provider integrations + 16 externally effective routes + 7 operational
  procedures) + declaration contract + static drift/invalid-combination harness +
  real-seam tests (+17 C# / +3 python; suites 1279/459 green); enforcement 6 enforced /
  8 partially_enforced / 8 procedural / 2 ABSENT / 3 n/a; the two ABSENT findings
  (anonymous paid reference route + unauthenticated paid model service) dispositioned
  **conditional_rc_risk** — loopback-safe in V1, Gate-1 bind verification required;
  tests-and-declarations only, no runtime enforcement added (**complete + integrated**
  2026-07-15: dai a0ca54d -> `3f244c8` tests-only ff with RC-equivalence + risk-
  disposition records; vault `6036897`; branches pushed + retained; PH-06 Green subset
  CLOSED, Amber unpulled; WIP freed; zero spend/writes)

- [[WI-0031-model-assisted-capability-recommendation-and-tool-selection-v1]] — Model-Assisted
  Capability Recommendation and Tool Selection v1 (platform architecture + orchestration
  contract + observability planning): governing parent for a model-assisted capability
  recommendation and deterministic tool-selection feedback loop. Normative standard
  ([[capability-recommendation-and-tool-selection-standard-v1]]) plus a six-slice
  implementation plan; the model is a semantic recommender, the Tool Gateway remains the
  execution authority, and relevant-but-inaccessible recommendations are retained as
  capability-gap telemetry. **in-progress** (Slices 1-4 delivered and integrated -- dai main
  contains the implementation through Slice-4 `f926484`+`209d485`, verified 2026-07-24
  audit; Slices 5 telemetry + 6 pilot remain deferred). Reuses the Tool
  Gateway doctrine and the prompt-selection provenance dialect (ADR 0007) rather than forking.

- [[WI-0032-dai-knowledge-architecture-and-writing-standard-v1]] — DAI Knowledge Architecture and
  Writing Standard v1 (knowledge-system architecture + documentation governance + vault writing
  standard): normative standard ([[dai-knowledge-architecture-and-writing-standard-v1]]) defining
  areas / knowledge modules / typed records, the physical/logical/navigational separation, the
  record-profile table (no forced universal front-matter dialect), the authority hierarchy (links
  never grant authority), naming/relationship/inheritance rules, the optional-manifest trigger
  policy, OKF compatibility without bundle terminology, the mandatory documentation-slice impact
  declaration, supersession/validation/migration posture, plus an embedded WI-0031 pilot
  assessment and a prospective adoption plan. **in-progress** (planning + documentation only
  2026-07-18; prospective adoption; no broad migration; no tooling; local vault-only branch +
  commits; not integrated).

- [[WI-0033-azure-devops-publication-contract-v1]] — Azure DevOps Publication Contract v1
  (workflow-governance, docs-only): binding create-only contract (ADR 0010) for projecting
  approved vault work items into Azure DevOps org `jera-technologies` / project `dai` (Basic
  process; Epic/Issue/Task). Vault stays canonical for definition/approval; Azure DevOps owns
  post-publication operational state; identity via normalized `vault-wi-####` tag with WIQL
  pre-check fail-closed on collision; routing to area `dai` + backlog root (never sprints);
  governed description sections carry acceptance criteria (`vault-managed` provenance tag);
  explicit bounded batches, dry-run vs single-Issue canary distinction, and a closed
  prohibited-behavior list. Publisher, adapter selection, update workflow, and process
  migration all deferred. **complete** (2026-07-19; docs-only; local vault-only branch +
  commit; not integrated; zero Azure DevOps mutations).

- [[WI-0034-daily-evidence-planner-stage-0]] — Daily Evidence Planner Stage 0 (niche/domain
  evidence-operations decision system; Stage 0 of the Daily Evidence Acquisition Orchestrator,
  architecture record [[daily-evidence-acquisition-orchestrator-v1]] in 04 Products/sports-v1):
  Slice 1 delivered the offline deterministic planner core in `services/agent-service`
  (consumes the canonical `pooled_calibration` sufficiency verdict, never recomputes policy;
  closed six-outcome Daily Evidence Board; missing-capability records for the future WI-0031
  seam; 25 fixture/invariant tests at Slice-1 close, 35 after the Slice-2 review; canonical
  byte-deterministic JSON + Markdown projection). Slice 2 delivered the portable CLI.
  **in-progress** (Slices 1-2 integrated on published dai main -- `e3ef9a5`, `3ab9568`,
  `9147549` verified ancestors 2026-07-24 audit; Slices 3 schedule adapter + 4 operating/skill
  integration remain deferred; zero screening/capture/execution authority).

- [[WI-0035-market-contrast-candidate-screen]] — Market-Contrast Candidate Screen v1 (the
  niche/domain producer for the planner capability `input.market_contrast_screen`;
  architecture record [[market-contrast-candidate-screen-v1]] in 04 Products/sports-v1):
  Slice 1 delivered the offline deterministic C# classifier core beside the existing market
  authorities (policy profile `market-contrast-screen/1.0` from the cohort-selection
  doctrine; blocker-vs-exclusion semantics; single de-vig authority `MarketDepth.DevigPair`;
  additive `H2hBookCount`; canonical envelope projection `input-evidence-envelope/1.1`;
  planner tier-aware ordering at contracts 2.1; cross-language vectors in both suites).
  Slices 2-3 (bounded source adapter, one governed live screen) were delivered and
  integrated; operating integration was delivered by the provider-event binding vertical
  A-D (`2e24782`..`85af96d`, published dai main tip) and operationally proven 2026-07-23.
  **complete** (2026-07-24 completion audit; all integration SHAs verified ancestors of
  the published mains; result authorizes nothing; known corrective candidate
  `caller_state_mismatch` over-constraint recorded in the spec's final disposition;
  binding residuals F1/F2/F3/F5 recorded as accepted follow-ups).

- [[WI-0036-wildcard-evidence-discovery-loop-v1]] — Wildcard Evidence Discovery Loop v1
  (evidence-operations + cognitive-factory architecture; parent for the bounded wildcard
  candidate lane, artifact wildcard provenance, proposal-only Interrogate signal needs, and
  the orchestrator-mediated Question -> Probe -> authorized retrieval -> Perceive refresh ->
  Verify -> Discern loop with a future callable protocol-service seam): Slice 1 delivered the
  documentation/contract baseline — decision [[0011-orchestrated-interrogate-perceive-refresh-loop-v1]],
  architecture record [[wildcard-evidence-discovery-loop-v1]] (closed candidate-lane
  vocabulary, 25-percent `floor(n/4)` cap, one-core minimum, strongest-novelty substitution
  ordering, `SignalNeedProposal` proposal-only contract), and current-state reconciliation
  incl. the named dormant-chain Verify re-entry gap (ledger entry 27). Slices 2-6
  (deterministic wildcard preflight, artifact provenance, proposal projection, callable
  service seam + re-entry, limited activation) deferred and each separately gated;
  implementation not before the separately governed 2026-07-22 events-gate observation plus a
  new operator authorization. **in-progress** (Slice 1 documentation complete 2026-07-21;
  local vault-only branch + commit; not pushed; not integrated; zero runtime change, zero
  spend, zero authority granted). Slice 1 (+ substitution-contract correction + provenance
  fix) INTEGRATED to vault main `b04e6421` and pushed 2026-07-21. Then, under a same-day
  operator sequencing override (offline implementation no longer gated on the July 22
  observation), Slice 2 (deterministic wildcard flight-plan core + CLI,
  `wildcard-flight-plan/1.0`, WI-0036-owned, consuming the unchanged WI-0034 board 2.2)
  and the MINIMUM Slice-3 provenance seam (`flight-selection-provenance/1.0`, additive
  null-suppressed `CompetitionMatchupInput.FlightSelection` -> artifact -> internal
  inspection only) were implemented TDD (70 pytest + 12 xunit new; suites 634/634 +
  1516/1516) on matching local branches `wi/0036-wildcard-capture-flight-plan`; NOT
  pushed / NOT merged / NOT integrated; default off; zero authority; paid use separately
  gated. Slices 4-6 remain deferred. Same-day independent Codex review: NOT
  integration-ready (findings A-G: wire-contract binding, reserve-lane collapse,
  caller-invented vocabulary, fingerprint-laundered authority, trusted forged
  realization, permissive .NET boundary, caller-edited board). Corrected RED-first in
  new commits on top of the preserved pair: canonical camelCase cross-runtime wire +
  exact all-false authorityLedger, reserve lane preserved with realized_via, canonical
  manifest-derived recognized registry (digest-pinned, caller narrowing-only), strict
  fingerprint-independent plan/realization validators, export-side internal
  re-realization, full .NET validation matrix + controller-host 400 proof, and
  producer-verified board reproduction through the real WI-0034 planner. Suites after
  correction: pytest 653/653, DevCore.Api.Tests 1528/1528. Chains (2 commits per repo)
  still local-only; next = independent review of the complete chains. A second
  independent review then found five semantic gaps (H-L: reserve substitution
  eligibility, rehashed wildcard/producer-reference forgeries, incomplete realization
  partition, unprovable precedence); corrected same-day RED-first in third commits per
  repo: required TrustedValidationContext for every plan consumer, semantic plan
  validation, availability-derived full realization validity, and the corrected
  eligibility truth table with a second cross-runtime vector + controller-host proof
  (pytest 664/664, DevCore.Api.Tests 1530/1530). Chains = 3 commits per repo,
  local-only; next = independent review of the complete three-commit chains. A third
  independent review (2026-07-22, findings M-S) then found the ROOT contract still
  wrong: full plan validity was inferred from a version/vocabulary context rather than
  proven, so a forged reproduced-board digest, a disabled mode with a live wildcard lane,
  a blank identity, a pool overlap, a boolean position, and a market-missing candidate
  rewritten to a canonical market-backed tuple all passed after re-fingerprinting; the
  registry cross-producted recipe versions against data regimes; malformed availability
  escaped as an unhandled `TypeError`; unfilled rows dropped the validated source reason;
  and the accounting said 76+25 where fresh collection proved 76+24=100. Corrected
  RED-first in new commits on top of the preserved chain: **exact producer re-production
  is now the full-validity rule** (a plan is valid iff canonically identical to
  `build_flight_plan(verified_request)`), the verified request is REQUIRED by validate/
  realize/export, the registry preserves EXACT manifest `(recipe_id, version, regime)`
  tuples with a DERIVED market state, availability is type-validated before keying, and
  unfilled rows separate `unavailability_reason` from `unfilled_reason_code`. Contracts
  bumped to request/plan/planner/realization/cli `1.1` (`flight-selection-provenance/1.0`
  and `wi0036-candidate-lane/1.0` unchanged); both cross-runtime vectors regenerated. A
  coverage-delta audit caught four legacy scenarios dropped by the suite consolidation and
  restored them. Suites: pytest **617/617**, DevCore.Api.Tests **1530/1530** (targeted
  wi-0036 39 core + 14 cli). Slice 2 + minimum Slice-3 seam are review-complete and
  authorized for coordinated fast-forward integration by the 2026-07-22 prompt; the
  executed publication result (integration tips, push proof) is recorded in the external
  prompt-ledger record and the current-slice handoff. Slice 3 remainder and Slices 4-6
  stay deferred and separately gated; no runtime activation, capture, spend, or paid
  flight is authorized. **INTEGRATION EXECUTED 2026-07-22** (coordinated, fast-forward
  only, ordinary non-force pushes; every prior commit preserved, no amend/squash/rebase/
  force-push/merge commit): dai main `8369d64 -> ce34a9e74659b42c71317267c64901a24ceb7091`
  and dai-vault main `b04e6421 -> 2cdb275b85229c6ee11a7b2930dc50d847ae8240`, each now
  EQUAL to its `origin/main` and to the corrected branch tip. Reverified on the published
  mains: agent-service pytest **617/617**, DevCore.Api.Tests **1530/1530**, adversarial
  producer-replay probes zero leaks, protected state byte-identical. Repository
  publication granted **no runtime or commercial authority** -- the integrated path stays
  default off with all-false ledgers, executed no source/model/gateway/database call, and
  cost $0. WI-0036 remains **in-progress**: next is the separately authorized **Slice-3
  remainder** (settlement/reconciliation stratum reads + realized-position writeback);
  Slices 4-6 remain deferred; the 2026-07-22 events-gate observation and any paid wildcard
  flight remain separately governed.

  **Slice-3 remainder VERIFIED AND INTEGRATED 2026-07-22.** On coordinated branches
  `wi/0036-wildcard-settlement-strata-writeback` (dai base `ce34a9e7`, vault base
  `6e667b5c`), nullable run-row writeback records the validated frozen selection's
  flight/fingerprint/lane/role, scheduled and realized positions, realization mode, and
  optional substituted-for provider identity. The existing settlement-joined calibration
  row read adds `evidenceStratum` plus these facts: core/reserve -> `core`, wildcard ->
  `wildcard`, absent/unknown -> `unclassified`. Aggregate metrics and reconciliation
  matching are unchanged; provenance and lane contracts remain 1.0; both producer vectors
  are unchanged; the migration is generated but **NOT applied**. RED-first targeted tests
  37/37, full pytest 617/617, full DevCore.Api.Tests 1536/1536. An **independent
  verification against live repository state PASSED** with no correction commit required
  (ancestry, protected hashes, validation-before-persistence ordering, fail-closed strata,
  buyer-boundary absence, migration symmetry, untouched provenance/planner, and
  cross-runtime consumer tolerance all re-derived from source). **INTEGRATION EXECUTED**
  by coordinated fast-forward with ordinary non-force pushes: dai main
  `ce34a9e7 -> 48a29313988beac34cb8dad388371c472ff21d73` and dai-vault main
  `6e667b5c -> fa31a3e91a09753602e34ca8cee3d38e059d9dd0`. dai main remains `48a2931` ==
  `origin/main`; the vault implementation commit `fa31a3e` is an ANCESTOR of vault main,
  which has since advanced with post-integration reconciliation/correction commits whose
  final tip is recorded in the external prompt-ledger outcome. Two
  operator-accepted semantics are binding and closed: an AgentRun IS an executed
  realization (so run-creation provenance must carry `realizedPosition`/`realizedVia`; an
  endpoint-context requirement, not a 1.0 contract change), and scheduled-mode projection
  coverage is sufficient. No source/model/gateway/database/run/capture/settlement/
  scheduling/activation action occurred; cost $0; no runtime or commercial authority
  granted. WI-0036 stays **in-progress** with Slices 4-6 deferred.

- [[WI-0037-game-state-correctness-v1]] — Game-State Correctness v1 (corrective;
  sports-v1 evidence operations): parent invariant = deterministic, test-pinned agreement
  with authoritative MLB game-state truth across planning, screening, capture, finality,
  and settlement boundaries. Created 2026-07-24 from the completion audit's two verified
  corrective findings (July 23 operational evidence): Slice 1 = monotonic caller-state
  progression (the `caller_state_mismatch` over-constraint at
  `MarketContrastSourceAdapter.PreEliminationReason`, twice-demonstrated paid cost, zero
  coverage) and Slice 2 = canonical date-bracketed status resolution (frozen Eastern
  bracket + exact gamePk + exactly-one-match across all consumers; doubleheader/makeup/
  duplicate handling; no positional bucket selection). Discovered by WI-0035 operational
  validation; corrects behavior exercised through WI-0034 and WI-0035; related to
  WI-0036 flight/provenance consumers. WI-0035 is NOT reopened. **in-progress**
  (planning only 2026-07-24; NEITHER slice authorized; no implementation).
  **Backlog sequencing (2026-07-24 briefing + audit):** WI-0037 Slice 1 is the
  recommended next implementation; Slice 2 follows Slice 1; WI-0034 Slice 4
  (operating/skill integration; its precondition is met) should follow the correctness
  slices; WI-0034 Slice 3 remains deferred; WI-0031 Slices 5-6 remain deferred; WI-0032
  Slices 3-4 remain deferred and Slice 5 activation-gated; WI-0036 Slices 4-5 remain
  deferred and Slice 6 activation-gated; WI-0002 and WI-0003 remain valid conditional
  backlog. No status or supersession change to any of those items.

## scope boundary of this registry

`WI-####` items cover qualifying slices per the WI-0007 qualification rule (canonical
categories in the `dai-system-development` skill): implementation, investigation/discovery,
integration and closeout, operational **remediation**, migrations, workflow-governance,
doctrine or skill changes, measurement/validation, and documentation that changes canonical
operating behavior. A slice does not stop qualifying because it is small, docs-only,
script-only, single-session, or ends blocked/discovery-only.

Routine governed operational **cadence** (capture, settlement, calibration readouts executed
under an existing OKF plan) remains outside this taxonomy per [[operating-model]] and is
governed by OKF records under `06 Execution/`. Precedent preserved:
`06 Execution/plans/v2-day1-settlement-day2-capture-slice-2026-07-10-v1.md` is a governed
operational slice that deliberately did **not** mint a WI, while the code defect it
discovered did (WI-0004). Genuinely clerical edits (typo, broken link, formatting-only,
hash update in an open WI) proceed without a WI but record the explicit non-qualifying
reason (WI-0007).

## skill layer

- `dai/.claude/skills/dai-system-development/SKILL.md` — the workflow router (coordinates
  existing skills; never replaces them). Tracked in
  `06 Execution/skills/dai-skills-inventory-v1.md`.
