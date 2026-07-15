---
title: "Protocol Hardening Candidate Specifications v1 (WI-0021, 2026-07-15)"
type: "plan"
date: "2026-07-15"
status: "active"
project: "DAI"
slice: "WI-0021 AI Engineering Protocol Hardening Candidate Specifications v1"
repos:
  dai: "unchanged (read-only inspection at 85a8831)"
  dai-vault: "docs-only (WI-0021 branch)"
tags:
  - planning
  - hardening
  - protocol
  - specifications
related:
  - "06 Execution/plans/ai-engineering-hardening-catalog-v1.md"
  - "06 Execution/plans/hardening-ready-queue-v1.md"
  - "06 Execution/reports/protocol-coverage-and-maturity-matrix-v1.md"
  - "02 Platform/system-development/work-items/WI-0021-protocol-hardening-candidate-specifications.md"
---

# protocol hardening candidate specifications v1

Authoritative branch-ready specifications for the six operator-approved hardening
candidates PH-01..PH-06. This document specifies FUTURE work; it implements nothing,
mints no implementation WI, creates no implementation branch, and authorizes no
integration. Every implementation branch slug uses `<next-id>` (assigned only at
pull). dai remains frozen at `85a8831`; RC Gate 1 (2026-07-17) is the next
operational event. Documenting a candidate does not authorize either integration path
(section 0.4).

## 0. common doctrine (binding for all six candidates)

### 0.1 protocol doctrine
Sequence: Perceive -> Interrogate -> Discern -> Decide -> Synthesize. Canonical
micro-actions: Interrogate(Question, Probe, Verify); Discern(Weigh, Contrast,
Stress); Decide(Resolve, Position, Justify); Synthesize(Integrate, Compose, Deliver).
Perceive is the intake layer with NO canonical micro-actions; operational intake
guidance may use Observe / Frame / Bound ONLY when clearly labeled NONCANONICAL.
No candidate renames protocol vocabulary or activates a dormant station
(DiscernStationRunner, ProtocolNodeRunner non-probe stations, Probe Refresh chain
remain doctrine anti-goals). The G-01 vocabulary-count question remains open and is
unaffected by this document.

### 0.2 evidence doctrine (WI-0020 v1.1 taxonomy)
Classes: contract-represented / fixture-proven / integration-proven / paid-run
observed / production-observed / operationally proven / commercially validated --
dimension-specific, non-interchangeable, not a ladder. Never infer: semantic
correctness from test volume; validation from paid execution; operational proof from
internal test delivery; commercial validation from buyer-shaped projections;
calibration from numeric confidence; market superiority from observed grading.
Each candidate states the evidence class its acceptance criteria CAN establish and
what remains unproven.

### 0.3 work-package pattern (branch-execution phases -- these do NOT replace the
canonical cognitive micro-actions)
- **Inspect**: locate existing behavior; identify current contracts and residues;
  inventory current fixtures/tests; classify observed vs preventive gaps; NO behavior
  change.
- **Prove**: deterministic fixtures or diagnostics; expose the missing invariant or
  characterize current behavior; no paid calls, no provider dependencies; failing or
  characterization test BEFORE any runtime change.
- **Guard**: smallest justified invariant; regression resistance; explicit failure
  semantics preserved; authoritative docs and evidence records updated.

### 0.4 release-candidate impact policy
- **RC-neutral**: vault docs; isolated fixtures; tests excluded from production
  artifacts; standalone static analysis. Eligible for pre-RC-completion integration
  ONLY after explicit review proving the RC artifact and drill assumptions are
  byte/behavior-unchanged.
- **RC-affecting**: runtime behavior; contracts Gate 1 uses; configuration;
  prompt/recipe resolution; decision semantics; projections; persistence; delivery;
  startup/shutdown. Integration requires explicit operator authorization,
  supersession of the old RC when applicable, affected Gate 0 checks rerun, and the
  Friday authorization updated to the new commit.
Documenting a candidate authorizes NEITHER path.

### 0.5 definition of ready (all candidates)
Ready to pull ONLY when: concrete problem statement; observed vs preventive
separated; implementation locus verified; scope + exclusions explicit; lane assigned;
branch slug defined; falsifiable acceptance criteria; required fixtures identified;
expected affected tests named; spend/write boundaries explicit; RC impact classified;
dependencies known; no hidden blocking architectural question; operator authorization
requirements named.

### 0.6 definition of done (all candidates)
Complete ONLY when: Inspect evidence recorded; Prove fixtures/characterization tests
exist; the narrow Guard implemented; EVERY acceptance criterion evaluated; tests
pass; unresolved criteria explicitly marked; achieved evidence class stated
accurately (never a higher class inferred); documentation updated; branch has a final
handoff; review complete; integration separately authorized; runtime returned to the
expected state; spend/write ledger complete. "Code complete" alone is NOT done.

### 0.7 shared boundaries
WIP = 1 active implementation branch. Zero paid calls and zero external writes during
implementation (fixtures use persisted runs). No schema change without moving the
implementation into a separately gated change class. Buyer identifiers/receipts never
committed. Tenant scoping preserved on every new surface.

---

## PH-01 — Representative Protocol Failure Corpus v1

1. **id** PH-01 | 2. **title** Representative Protocol Failure Corpus v1
3. **problem**: DAI's failure knowledge is scattered across suites and history (4
   production lean-mismatch exclusions, 823357 postponement, 822877 classifier
   ambiguity, v7c provenance-thin burst, R1-R14 recovery classes) with no single
   deterministic corpus that replays every known failure class through the projection
   and validation layers; regressions in failure handling are currently discoverable
   only piecemeal.
4. **phase** Discern | 5. **micro-action** Stress (primary)
6. **supporting**: Interrogate.Verify, Decide.Resolve, Synthesize.Deliver
7. **discipline**: evaluation engineering
8. **loci** (verified at 85a8831): SportsQualityChecker.cs (6 fail-open rules);
   ArtifactDirectionConsistencyEvaluator (compose) + settlement 422 integrity guard
   (AgentRunsController.cs:893-1115); SourceReadiness.cs regimes + fail-closed
   eligibility; DuplicateRunGuard.cs; SettlementProvenance.cs (residue 422);
   model_metering.py named unpriced states; ToolGateway.cs fail-closed registry/node
   checks; BuyerCopySafety.SuppressUnsafe; brief/recap determinism suites; runbook
   R1-R14.
9. **observed evidence**: production-observed failures exist for lean/prose
   contradiction (4 excluded runs), postponement (823357), attribution ambiguity
   (822877), provenance-thin residue (v7c, 15 rows); fixture-proven guards exist
   piecemeal (guard 23 tests, claim-safety 48, readiness 10+).
10. **preventive concerns**: stale evidence, contradictory evidence, unauthorized tool
    action, delivery-without-entitlement, numeric-confidence leakage have no observed
    production instance -- fixtures for them are PREVENTIVE and must be labeled so.
11. **scope**: one deterministic corpus (fixtures + runner inside the existing test
    suites) covering AT MINIMUM the 15 authorized failure classes: missing required
    evidence; stale evidence; unavailable provider; contradictory evidence;
    requested/resolved identity mismatch; incomplete provenance; unsupported
    directional position; unresolved decision contradiction; unknown model pricing;
    unauthorized tool action; duplicate active identity; buyer projection from
    incomplete residue; numeric-confidence leakage; unsupported profitability/
    superiority claim; delivery without valid entitlement.
12. **exclusions**: no runtime behavior change (fixes discovered = separate
    candidates/WIs); no prompt/model/scoring change; no schema; no new endpoints; no
    activation of dormant stations; entitlement fixtures assert against the DOCUMENTED
    ledger/runbook contract (no delivery runtime exists to change -- see PH-05).
13. **contract introduced**: "every known failure class has a deterministic, labeled,
    replayable fixture with an expected machine-readable classification."
14. **runtime impact**: none (tests/fixtures only; any needed hook = stop condition).
15. **test impact**: new corpus suite (C# + python); existing suites untouched.
16. **docs impact**: corpus index doc + evidence artifact.  17. **persistence**: none
    (test dbs only). 18. **buyer-surface**: none. 19. **operator**: corpus results
    readable as one report. 20. **tenant-isolation**: fixtures include the cross-tenant
    404 class (existing matrices referenced, not duplicated). 21. **tools/permissions**:
    unauthorized-tool fixture exercises ToolGateway fail-closed paths (no registry
    change). 22. **spend/write class**: 0 paid calls / 0 external writes / 0 db writes.
23. **branch** wi/<next-id>-discern-stress-protocol-failure-corpus
24. **size** M | 25. **lane** Green (tests/fixtures; becomes Amber ONLY if a runtime
    hook proves necessary -- which is a stop condition, not a scope change)
26. **reversibility** easy | 27. **dependencies** none (feeds PH-03/PH-04)
28. **fixtures**: the 4 excluded mismatch runs (exported residue), 823357, 822877,
    v7c thin-residue rows [observed/reconstructed]; synthetic fixtures for the
    preventive classes [preventive].
29. **tests**: corpus runner asserting per-fixture expected state + classification;
    determinism double-run assertion.
30. **acceptance criteria** (binding, falsifiable): the 14 criteria of the WI-0021
    authorization section 10, verbatim: (1) every fixture carries stable ID, phase,
    failure class, expected state, expected machine-readable classification, expected
    buyer behavior, expected operator behavior; (2) fixtures run with no model calls,
    no external providers, no db mutation; (3) deterministic across repeated
    execution; (4) every failure labeled observed | reconstructed | preventive;
    (5) identity ambiguity fails closed; (6) unavailable evidence stays unavailable,
    never synthetic; (7) unresolved contradictions visible downstream; (8) unsupported
    directional outputs rejected or converted to explicit abstention; (9) numeric
    confidence never reaches buyer surfaces; (10) unsupported profitability/
    market-superiority language fails projection validation; (11) delivery without
    valid entitlement fails closed (asserted against the documented runbook/ledger
    contract until PH-05 exists); (12) failure output traceable to originating fixture
    + protocol boundary; (13) any runtime defect discovered outside approved scope
    becomes a separate candidate/WI -- never silent expansion; (14) results establish
    FIXTURE-PROVEN evidence only unless broader evidence is separately produced.
31. **evidence artifact**: protocol-failure-corpus-results-v1
32. **stop conditions**: a fixture cannot be made deterministic; a class requires a
    runtime hook; a discovered defect tempts in-branch fixing; paid call needed.
33. **integration implications**: RC-neutral (tests only) -- still requires explicit
    review proving RC artifact unchanged before any pre-RC-completion integration.
34. **RC impact**: RC-neutral. 35. **cloud**: none. 36. **multisport**: corpus pattern
    reusable per sport at ladder stage 2-3.
37. **idle-window tasks** (entry state = WI minted + branch pulled unless noted):
    - 15m: classify ONE failure class as observed/reconstructed/preventive with its
      evidence pointer (artifact: one corpus-index row; stop: class ambiguous;
      partial OK)
    - 30m: complete ONE fixture specification (ID, phase, class, expected states)
      (artifact: fixture spec; stop: expected state undecidable; partial OK)
    - 60m: Inspect complete for one guard component (e.g. QualityChecker) -- inventory
      of covered vs uncovered classes (artifact: inspect note; stop: none; complete)
    - 240m: one Inspect+Prove cycle: encode 3-5 fixtures incl. one observed
      production failure, corpus runner green (artifact: passing fixtures; stop: any
      nondeterminism; partial = fixtures without runner acceptable)
38. **definition of ready**: MET (section 23 verdict READY).
39. **definition of done**: section 0.6 + evidence artifact committed + corpus index
    lists all 15 classes with zero unlabeled fixtures.
40. **unresolved questions**: none blocking. Deferred note: whether the corpus should
    later gate CI (no CI exists; FC-C3 conditional).

**Evidence class achievable**: fixture-proven (for the encoded classes). Remains
unproven: integration-proven for classes whose guards live in endpoints (partially
coverable via existing integration test host), everything operational/commercial.

---

## PH-02 — Evidence and Decision Trace Completeness v1

1. **id** PH-02 | 2. **title** Evidence and Decision Trace Completeness v1
3. **problem**: no single surface answers "which evidence did this run stage, verify,
   reject, and use, and which protocol fields did it fill" -- prompt-trace covers
   route/market/probe/guard/reconciliation but has no protocol completion status, no
   freshness state, no rejected-evidence retention, and fallbackDetail populates only
   for assembly_error; SignalAvailability records availability but not per-decision
   usage.
4. **phase** Interrogate | 5. **micro-action** Verify (primary)
6. **supporting**: Perceive intake, Discern.Weigh, Decide.Justify, Synthesize.Integrate
7. **discipline**: provenance/traceability + observability
8. **loci**: PromptTrace.cs (route :40-55, probe re-derivation :240-278, market
   attribution guard); route_provenance.py + provenance_sink.py; SignalQualityEvaluator
   (Quality/DecisionUse/FollowUpSignals/ConfidenceEffect); SportsRetriever
   RecordRetrieve (grounded/missing signals, SourceEnvelopes, identity, snapshot);
   CognitiveProtocol persistence in OutputJson; /rows export fields.
9. **observed evidence**: v7c provenance-thin burst (production-observed) motivated the
   residue contract; fallbackDetail added for run 260018-class gaps; operator
   debugging during v7/v8 required manual OutputJson reads (recorded in handoffs).
10. **preventive**: freshness/staleness state, contradiction state, rejected-evidence
    retention have no observed failure yet -- preventive, labeled so.
11. **scope**: define ONE minimum trace contract (documented + machine-testable):
    material evidence entries declare source identity, provider, retrieval timestamp,
    event timestamp (where applicable), freshness state, authorization class, spend
    class, tenant scope, requested identity, resolved identity, provenance status,
    availability state, contradiction state, downstream usage; decision trace
    declares evidence considered/rejected (with reasons), unresolved assumptions,
    unresolved contradictions, selected posture, abstention reason (where
    applicable), justification references, prompt/recipe version, model identifier
    (where applicable), synthesized artifact version. Read-side derivation FIRST
    (prompt-trace extension over persisted residue); persistence changes only per
    criterion 14.
12. **exclusions**: no schema migration assumed (criterion 14); no prompt change; no
    new evidence sources; no buyer exposure of the trace; no backfill of legacy rows.
13. **contract introduced**: minimum trace contract above.
14. **runtime impact**: read-side projection extension (prompt-trace) -- RC-AFFECTING
    (prompt-trace is a Gate-1-verified surface). 15. **test impact**: trace contract
    suite (C#) + serialization determinism tests. 16. **docs**: trace contract doc.
17. **persistence**: none in the read-side-first scope; any persisted field = explicit
    implementation decision + separately gated. 18. **buyer-surface**: none (operator
    only). 19. **operator**: one trace answers evidence+decision completeness.
20. **tenant-isolation**: trace reads tenant-scoped; no cross-tenant existence leak
    (criterion 11). 21. **tools/permissions**: evidence entries carry authorization +
    spend class (read from ToolRegistry metadata). 22. **spend/write**: 0/0/0.
23. **branch** wi/<next-id>-interrogate-verify-evidence-trace
24. **size** M | 25. **lane** Amber | 26. **reversibility** easy-moderate (read-side)
27. **dependencies**: none hard; PH-01 fixtures make trace assertions cheaper (soft);
    PH-06 declarations feed authorization/spend classes (soft).
28. **fixtures**: persisted runs across eras (legacy null-provenance row, v1 registry
    row, v2 row, no-decision row, excluded row) -- legacy rows must classify as
    explicit incomplete/unavailable (criterion 13).
29. **tests**: per-criterion contract tests + deterministic serialization double-run.
30. **acceptance criteria** (binding): the 15 criteria of authorization section 11,
    verbatim: no verified-without-source-identity; explicit freshness; unavailable/
    stale/contradictory/unknown distinct; requested+resolved retained; rationale
    claims linked to evidence ids or labeled inference; rejected evidence retained
    with reason; downstream artifacts cannot silently omit a blocking contradiction;
    machine-testable completeness; optional-missing explicit; required-missing =
    blocking/abstention state; tenant-scoped reads leak nothing; deterministic
    serialization; legacy = explicit incomplete classification, never synthetic
    backfill; no schema migration without explicit decision; tests distinguish
    contract-represented from integration-proven.
31. **evidence artifact**: evidence-trace-completeness-report-v1
32. **stop conditions**: contract requires a schema change to be useful (stop,
    re-scope); trace surface would leak to buyers; serialization nondeterminism.
33. **integration implications**: RC-affecting -> post-RC-verdict integration under
    section 0.4. 34. **RC impact**: RC-affecting (Gate-1-used surface). 35. **cloud**:
    durable trace sink pairs with stage-3 observability. 36. **multisport**: trace
    contract is sport-agnostic; capability descriptor (WI-0016) would populate it per
    sport.
37. **idle-window tasks**:
    - 15m: map ONE trace field to its current source-of-truth locus (artifact: field
      map row; stop: field has no source; partial OK)
    - 30m: complete the field map for one group (evidence identity fields) (artifact:
      table; partial OK)
    - 60m: Inspect complete for PromptTrace.cs -- present/absent matrix vs the
      contract (artifact: matrix; complete expected)
    - 240m: Inspect+Prove -- characterization tests pinning today's trace shape on 3
      era fixtures (artifact: green characterization suite; stop: nondeterminism)
38. **ready**: READY WITH OPEN QUESTION (see 40). 39. **done**: 0.6 + report artifact.
40. **unresolved questions** (named, blocking-for-pull): (a) read-side derivation vs
    persisted trace fields -- criterion 14 requires an explicit decision; recommend
    read-side-first, persistence only on demonstrated insufficiency; (b) whether
    freshness state can be derived for legacy rows or is declared unavailable.

**Evidence class achievable**: contract-represented + fixture-proven; integration-
proven only for the read-side path exercised by the integration host. Unproven:
operational, commercial.

---

## PH-03 — Decision Abstention Invariants v1

1. **id** PH-03 | 2. **title** Decision Abstention Invariants v1
3. **problem**: abstention exists (lean-null no-position, posture validation,
   readiness fail-closed eligibility, 422 direction integrity) but is NOT a unified
   first-class policy: SportsQualityChecker warns without blocking (fail-open by
   design), abstention REASONS are not machine-readable, and no versioned policy
   states which conditions MUST prevent a directional position.
4. **phase** Decide | 5. **micro-action** Resolve (primary)
6. **supporting**: Decide.Position, Interrogate.Verify, Discern.Stress
7. **discipline**: evidence/data quality + decision policy
8. **loci**: sports_analyzer.py posture validation (:401-434) + lean parsing;
   SportsQualityChecker rules 1-6 (fail-open, :57-106); SourceReadiness eligibility
   (fail-closed, :122-156); ArtifactDirectionConsistencyEvaluator;
   DirectionIntegrityRefusalOrNull (settlement 422); no-position rendering (WI-0011);
   recap no_position state (WI-0012); model_metering unpriced states; DuplicateRunGuard.
9. **observed evidence**: production-observed abstentions work (18 no-decision rows
   settle honestly; readiness gate blocks ineligible games; 4 contradiction runs
   refused at settlement). Observed GAP: quality-rule violations produce invisible
   warnings, not abstention (fail-open recorded in WI-0013-era review).
10. **preventive**: stale-evidence, unauthorized-tool-dependency, unsupported-
    capability abstention rules have no observed failure -- preventive.
11. **scope**: define + implement the versioned deterministic abstention policy over
    the authorized condition list (identity ambiguity; missing required evidence;
    stale required evidence; unresolved blocking contradiction; unsupported
    competition capability; attribution failure; unauthorized tool dependency;
    incomplete required provenance; unpriced model execution where cost authorization
    required; explicit no-position model result; decision-artifact inconsistency),
    each with machine-readable reason preserved through synthesis, buyer-safe
    no-position copy, operator-technical reason.
12. **exclusions**: no prompt change (model prose CANNOT bypass deterministic gates --
    the gates are platform-side); no scoring/confidence-value change; no
    reclassification of legacy rows (criterion 12); no schema unless separately gated;
    no new evidence sources.
13. **contract**: versioned abstention policy (clamps versioned, criterion 13).
14. **runtime impact**: YES -- decision semantics (which artifacts carry directional
    positions). Strictly RC-AFFECTING; integration requires RC-impact review +
    supersession path (criterion 16). 15. **tests**: positive AND negative tests per
    rule (criterion 14); fixture + integration acceptance separated (criterion 17).
16. **docs**: policy doc versioned. 17. **persistence**: abstention reason lives in
    OutputJson artifact (no schema) unless an explicit decision says otherwise.
18. **buyer-surface**: no-position copy only (claim-safe, existing forms).
19. **operator**: technical reason on operator surfaces. 20. **tenant**: unchanged.
21. **tools**: unauthorized-tool-dependency rule consumes ToolGateway/PH-06 classes.
22. **spend/write**: 0/0/0 (fixtures from persisted runs). 23. **branch**
    wi/<next-id>-decide-resolve-abstention-invariants
24. **size** M-L | 25. **lane** Amber | 26. **reversibility** moderate (decision
    semantics) | 27. **dependencies**: PH-01 corpus (soft, provides the failing
    fixtures); PH-02 trace (soft, carries reasons); PH-06 (soft, authorization
    classes). Independent implementation possible but wasteful before PH-01.
28. **fixtures**: PH-01 corpus classes map 1:1 onto abstention conditions; plus
    existing no-decision rows (824993/824339/823526 residue) as positive abstention
    fixtures.
29. **tests**: per-rule positive/negative matrix; contradictory-rules fail-closed test
    (criterion 15); prose-bypass attempt test (criterion 11).
30. **acceptance criteria** (binding): the 17 criteria of authorization section 12,
    verbatim (blocking ambiguity/missing/stale conditions prevent directional
    positioning; no-position first-class; machine-readable + synthesis-preserved
    reason; buyer-safe language; operator technical reason; confidence never
    overrides; prose cannot bypass; legacy untouched; versioned clamps; positive+
    negative tests; contradictions fail closed; RC-impact review before integration;
    fixture vs integration acceptance separated).
31. **evidence artifact**: abstention-invariant-evaluation-v1
32. **stop conditions**: a rule requires prompt changes to enforce; a rule would
    retroactively reclassify settled rows; policy contradiction unresolvable ->
    fail-closed + operator decision; schema pressure.
33. **integration implications**: RC-affecting; MUST NOT integrate before the final
    RC verdict; integrating after requires new RC review per 0.4.
34. **RC impact**: RC-affecting (decision semantics). 35. **cloud**: none direct.
36. **multisport**: policy is the natural home for per-sport thin-data postures
    (capability descriptor consumes it).
37. **idle-window tasks**:
    - 15m: classify ONE condition as observed/preventive with evidence pointer
      (artifact: condition row; partial OK)
    - 30m: define ONE invariant table row (condition -> gate -> reason code -> buyer
      form -> operator form) (artifact: row; partial OK)
    - 60m: Inspect complete for the fail-open QualityChecker path -- what today
      warns vs what the policy would block (artifact: delta table)
    - 240m: Inspect+Prove -- encode 3 conditions as failing policy tests against
      current behavior (characterization of fail-open) (artifact: red tests +
      characterization; stop: nondeterminism or prompt-dependence)
38. **ready**: READY WITH OPEN QUESTION (see 40). 39. **done**: 0.6 + artifact.
40. **unresolved questions** (blocking-for-pull): (a) OPERATOR POLICY DECISION -- which
    of the 11 conditions become BLOCKING (abstention) vs remain warnings; converting
    QualityChecker warnings to blocks changes decision semantics and buyer-visible
    output rates; (b) whether unpriced-model execution blocks generation or blocks
    delivery only (cost-authorization posture).

**Evidence class achievable**: fixture-proven + integration-proven for the gates.
Unproven until real operation: production-observed for the new blocking behavior.

---

## PH-04 — Synthesized Artifact Contract Invariants v1

1. **id** PH-04 | 2. **title** Synthesized Artifact Contract Invariants v1
3. **problem**: artifact-level invariants (identity coherence, posture/lean/copy
   consistency, claim safety, determinism, persisted-residue-only recaps) are
   individually tested but not stated or enforced as ONE cross-projection contract;
   malformed upstream residue behavior is uncharacterized.
4. **phase** Synthesize | 5. **micro-action** Integrate (primary)
6. **supporting**: Decide.Position, Decide.Justify, Synthesize.Compose
7. **discipline**: contracts + evaluation engineering
8. **loci**: SportsComposer.Compose (+ ComposeFailedRun null-protocol contract);
   CognitiveProtocolBuilder completion; BuyerDecisionBrief.cs (+ BuyerCopySafety
   suppression) and BuyerSettledRecap.cs (persisted-evaluation-only); markdown export
   determinism suites (25+23); ProtocolVocabularyMapper; AgentRunArtifactDto;
   GamePkPropagationTests; tenant 404 matrices.
9. **observed evidence**: production-observed identity coherence through 15+18
   settlements; byte-determinism fixture-proven; claim-safety live-verified (WI-0011).
   Observed defect class: lean/prose contradiction (the 4 excluded runs) = exactly the
   invariant family this candidate locks at projection time.
10. **preventive**: malformed-residue projection behavior, timezone determinism,
    null-field consistency -- preventive.
11. **scope**: one invariant suite over the authorized field set (requested identity,
    resolved identity, source identity, posture, lean, abstention state,
    justification, prompt/recipe provenance, evaluation residue, brief, recap,
    markdown export) implementing the 19 authorized criteria; characterize + validate
    malformed-residue projection failure.
12. **exclusions**: no projection redesign; no buyer-copy voice changes; no new
    fields on buyer DTOs; no schema; no recomputation paths.
13. **contract**: cross-projection artifact invariant contract.
14. **runtime impact**: projection validation additions (fail-validation on malformed
    residue, criterion 15) -- RC-AFFECTING where enforcement lands in serving paths;
    the fixture/verification subset is RC-neutral. 15. **tests**: invariant fixtures
    (criterion 17) + integration tests over real serialization/projection paths
    (criterion 18). 16. **docs**: invariant table; identification of which invariant
    changes would REDEFINE the RC (criterion 19). 17. **persistence**: none.
18. **buyer-surface**: stricter validation only (no copy change). 19. **operator**:
    malformed residue becomes a named failure instead of partial output.
20. **tenant**: criterion 16 (retrieval leak) re-asserted in the suite.
21. **tools**: none. 22. **spend/write**: 0/0/0.
23. **branch** wi/<next-id>-synthesize-integrate-artifact-invariants
24. **size** M | 25. **lane** Amber | 26. **reversibility** easy-moderate
27. **dependencies**: PH-01 (soft: malformed-residue fixtures); PH-03 (soft:
    abstention state names used by invariants 2/3). Can start independent.
28. **fixtures**: settled pair 823845 (brief+recap), no-position run, excluded run
    (823357), a malformed-residue synthetic fixture [preventive], DH pair fixtures.
29. **tests**: 19-criterion matrix; double-fetch byte-equality on all edge recap
    states; timezone/date determinism.
30. **acceptance criteria** (binding): the 19 criteria of authorization section 13,
    verbatim (identity conflict never silent; no-position never directional copy;
    buyer result matches persisted posture; justification cites only trace-present
    evidence; no numeric confidence; no profitability/certainty/superiority claims;
    same canonical identity across brief+recap; recap from persisted residue only;
    byte-stable markdown; consistent null projection; explicit artifact version;
    provenance operator-available not buyer-leaked; separate buyer/operator
    contracts; deterministic timezone/date; malformed residue fails validation;
    tenant retrieval leak-free; deterministic fixtures for all; real-path integration
    tests; RC-redefining invariants identified).
31. **evidence artifact**: artifact-contract-invariant-report-v1
32. **stop conditions**: an invariant requires buyer-copy semantic changes; residue
    validation would block currently-valid production rows; determinism unachievable
    for an edge state.
33. **integration implications**: verification subset RC-neutral; enforcement subset
    RC-affecting -> post-verdict. 34. **RC impact**: split (documented per criterion
    19). 35. **cloud**: none direct. 36. **multisport**: invariant table is the
    portable buyer-contract core for future sports.
37. **idle-window tasks**:
    - 15m: verify ONE existing invariant's test coverage (artifact: coverage row)
    - 30m: define one invariant table group (identity invariants) (artifact: table)
    - 60m: Inspect complete for BuyerSettledRecap invariants vs criteria (artifact:
      present/absent matrix)
    - 240m: Inspect+Prove -- malformed-residue characterization fixtures (artifact:
      characterization tests; stop: a fixture mutates real data)
38. **ready**: READY. 39. **done**: 0.6 + artifact. 40. **unresolved**: none blocking.
    Deferred note: whether export etags belong here or in ops docs (CAT-SYN-C-3).

**Evidence class achievable**: fixture-proven + integration-proven. Unproven:
operationally proven (needs real pilot use), commercially validated.

---

## PH-05 — Delivery Idempotency and Entitlement Guard v1

1. **id** PH-05 | 2. **title** Delivery Idempotency and Entitlement Guard v1
3. **problem**: delivery today is ENTIRELY manual/procedural (operator emails a
   rendered export; entitlement = Stripe receipt + ledger row per the freeze doc;
   runbook section 7 withholds on unclear entitlement). No runtime concept of
   entitlement state, delivery attempt, idempotency, or receipt exists -- correct for
   V1 concierge, but the boundary is enforced only by operator discipline.
4. **phase** Synthesize | 5. **micro-action** Deliver (primary)
6. **supporting**: Synthesize.Compose; platform authorization + billing boundaries
7. **discipline**: reliability + security/tenancy + commercial feedback
8. **loci** (verified): brief/recap markdown endpoints (the only artifact surfaces);
   delivery-ledger template (entitlement_status paid|test|withheld); runbook section
   7 (withhold rule, test-mode dry-run, alias privacy); NO payment/entitlement code
   anywhere (verified broad search, critical-path audit); Stripe = truth doctrine.
9. **observed evidence**: none adverse (no delivery has ever occurred -- deferred
   posture). The candidate is ENTIRELY PREVENTIVE and must be labeled so.
10. **preventive concerns**: double-delivery, delivery-without-payment, test/real
    confusion, cross-tenant delivery, receipt duplication.
11. **scope**: define delivery as an explicitly authorized operation with the
    authorized minimum model (entitlement state, delivery target, artifact identity,
    delivery attempt identity, delivery status, timestamp, retry state, receipt or
    local delivery proof, tenant scope, redelivery policy) and implement the guard
    per the 22 authorized criteria.
12. **exclusions**: NO real Stripe transaction in implementation testing (criterion
    19); no email automation (delivery execution stays manual/concierge); no billing
    integration; no checkout code; real payment + real buyer delivery remain under
    SEPARATE operational authorization (criterion 20) -- currently DEFERRED by the
    2026-07-15 commercial posture.
13. **contract**: delivery authorization contract (entitlement checked immediately
    before real delivery; simulated distinguishable from real; idempotent attempts).
14. **runtime impact**: YES -- new runtime concept; 17. **persistence impact**: the
    minimum model needs durable state; ledger-file vs database is an EXPLICIT
    implementation decision (criterion 21); ANY schema change moves the
    implementation into a separately gated change class (criterion 22) = Red lane.
15. **tests**: entitlement matrix (missing/ambiguous/expired/revoked/test/paid),
    idempotency/retry, tenant isolation, alias privacy. 16. **docs**: runbook section
    7 alignment. 18. **buyer-surface**: none new (validation before delivery,
    criterion 15). 19. **operator**: delivery status observable (criterion 14).
20. **tenant**: criterion 11. 21. **tools**: delivery becomes a declared capability
    class under PH-06 (delivery class). 22. **spend/write class**: implementation 0
    paid calls / 0 external writes; db writes ONLY if the persistence decision says
    db (then separately gated).
23. **branch** wi/<next-id>-synthesize-deliver-entitlement-idempotency
24. **size** L | 25. **lane** Amber; Red for any external-delivery exercise or schema
    change | 26. **reversibility** moderate-difficult (new persistent state)
27. **dependencies**: HARD: persistence-model decision + commercial activation
    posture (real delivery is deferred; building the guard before any delivery
    channel exists risks anticipatory design). SOFT: PH-04 (artifact validation
    precedes delivery, criterion 15); PH-06 (delivery declared as a capability).
28. **fixtures**: entitlement-state matrix fixtures; simulated-delivery records from
    the RC drill (operator-address only). 29. **tests**: 22-criterion matrix.
30. **acceptance criteria** (binding): the 22 criteria of authorization section 14,
    verbatim (missing/ambiguous/expired entitlement blocks; test != real; simulated
    distinguishable; idempotent repeats; no duplicate receipts; stable attempt
    identity; artifact survives entitlement failure; retry only under explicit
    policy; tenant isolation; no buyer destinations in repo; alias privacy;
    operator-observable status; buyer-safe validation before delivery; entitlement
    checked immediately before real delivery; no payment inference from artifact
    existence; no entitlement inference from test-mode records; no real Stripe
    transaction in testing; real payment/delivery separately authorized; persistence
    decision explicit; schema change = separately gated class).
31. **evidence artifact**: delivery-entitlement-idempotency-report-v1
32. **stop conditions**: schema pressure without the gated decision; any real
    payment/delivery temptation; email automation temptation; buyer PII near the repo.
33. **integration implications**: RC-affecting (delivery + persistence); MUST NOT
    integrate before the final RC verdict; ALSO gated on the commercial activation
    decision (a delivery guard without authorized delivery is shelf inventory).
34. **RC impact**: RC-affecting. 35. **cloud**: entitlement state is a prerequisite
    for any hosted delivery (stage 3+). 36. **multisport**: sport-agnostic.
37. **idle-window tasks**:
    - 15m: inspect the ledger template's entitlement fields vs the minimum model
      (artifact: field-delta row)
    - 30m: document one declaration group (delivery attempt identity model options)
    - 60m: Inspect complete: everything the runbook section 7 promises, mapped to
      enforceable vs procedural (artifact: enforcement map)
    - 240m: Prove subset -- entitlement matrix as pure-logic fixtures with NO
      persistence (artifact: fixture suite; stop: fixtures require storage decisions)
38. **ready**: NOT READY (see 40). 39. **done**: 0.6 + artifact.
40. **unresolved questions** (exact missing decisions): (a) persistence model --
    ledger-file-backed vs database-backed entitlement/attempt state (criterion 21;
    db => schema => Red-gated); (b) commercial activation posture -- real delivery is
    deferred (2026-07-15 operator decision), so WHEN this guard becomes buildable at
    all is an operator call; (c) delivery-target handling that keeps buyer
    destinations out of the repo while remaining testable.

**Evidence class achievable**: fixture-proven (+ integration-proven for guard paths).
Structurally CANNOT establish operationally proven or commercially validated until
real, separately-authorized deliveries occur.

---

## PH-06 — Tool Authorization Fitness v1

1. **id** PH-06 | 2. **title** Tool Authorization Fitness v1
3. **problem**: ToolGateway enforces registry + protocol-node allowlists fail-closed
   and ToolRegistry declares CostClass/idempotency/TTL for 10 tools -- but cost class
   is recorded, NOT enforced; callers pass stage sentinels (station-id branch
   dormant); controller endpoints have no unified authorization-class inventory; and
   nothing detects an undeclared new tool or capability class drift.
4. **phase** Interrogate | 5. **micro-action** Probe (primary -- the tool-invoking
   station seam)
6. **supporting**: tenant permissions, cost control, observability, provider resilience
7. **discipline**: tool routing/permissions + security
8. **loci**: Tools/ToolGateway.cs (:42-119 fail-closed checks + telemetry incl.
   costClass), Tools/ToolRegistry.cs (:36-136, 10 tools), ToolDefinition.cs (deferred
   enforcement note), IToolGateway.cs, ProtocolToolAccessPolicy (+ dormant station-id
   branch), AgentRunsController route surface, DevBypass (fail-closed double
   condition), Dev endpoints double-gates (enableDevTools/enableDevBatchRuns).
9. **observed evidence**: fixture-proven gateway fail-closed behavior (6 test files,
   ~50 tests); production-observed telemetry lines. Observed GAP (documented in
   code): cost-class enforcement deferred; declaration completeness unchecked.
10. **preventive**: undeclared-tool drift, adapter permission self-expansion --
    preventive.
11. **scope**: (a) declaration contract for every tool, provider integration, and
    externally effective endpoint using the authorized class set (read-only local/
    read-only external/paid external/local write/database write/external write/
    reconciliation/credential operation/delivery/deployment/administrative), each
    declaring capability name, owning service, tenant scope, permission class, spend
    class, write class, retry policy, timeout policy, rate limit, idempotency
    requirement, observability requirement, secret requirement, failure behavior,
    operator authorization requirement; (b) machine-checkable validation (static or
    startup, criterion 14) + fitness check detecting undeclared additions (criterion
    15); (c) the MINIMUM enforcement consistent with criterion 19 -- explicitly NOT a
    generalized permission platform.
12. **exclusions**: no rate-limiting infrastructure build-out beyond declarations +
    bounded-retry validation; no tenant-tier billing; no new tools; no sports logic
    in the platform layer (criterion 18); credential operations remain Red-gated
    (criterion 6); reconciliation tools remain separately authorized (criterion 7).
13. **contract**: tool/capability declaration contract + fitness check.
14. **runtime impact**: declaration inventory + static check = NONE (Green);
    startup validation / unknown-class fail-closed extension = runtime (Amber).
15. **tests**: declaration completeness static test; per-criterion checks; denial vs
    provider-failure distinguishability test (criterion 11). 16. **docs**: capability
    table in vault. 17. **persistence**: none. 18. **buyer**: none. 19. **operator**:
    one authoritative capability/permission table. 20. **tenant**: declarations carry
    tenant boundary (criterion 5). 21. **tools**: this IS the tool contract.
22. **spend/write**: 0/0/0.
23. **branch** wi/<next-id>-interrogate-probe-tool-authorization
24. **size** M (S for the Green inventory subset) | 25. **lane** Green (declaration
    inventory + static check) then Amber (startup validation/enforcement)
26. **reversibility** easy | 27. **dependencies**: none. Feeds PH-02 (authorization/
    spend classes on trace) and PH-03 (unauthorized-tool abstention rule).
28. **fixtures**: registry snapshot fixture; seeded undeclared-tool violation fixture.
29. **tests**: criteria 1-20 checks incl. dev-bypass-cannot-become-production test
    (criterion 16, extends existing DevBypass tests).
30. **acceptance criteria** (binding): the 20 criteria of authorization section 15,
    verbatim (every tool declared; unknown fails closed; paid tools declare spend
    class; external writes declare idempotency; tenant boundaries declared;
    credential ops Red-gated; reconciliation separately authorized; bounded declared
    retries; retries cannot silently multiply paid calls/writes; explicit timeouts;
    denial != provider failure; traceable observability without secret leakage;
    machine-checkable; static-or-startup validated; undeclared-tool detection;
    dev bypass never production authorization; adapters cannot self-expand;
    no sports logic in platform layer; minimum-viable over generalized platform;
    consistent with: agents fallible-but-powerful, capabilities explicitly granted,
    permissions tenant-scoped, actions rate-limited and observable).
31. **evidence artifact**: tool-authorization-fitness-report-v1
32. **stop conditions**: enforcement pressure toward a generalized permission
    platform; any credential-operation implementation temptation (Red); registry
    changes beyond declarations.
33. **integration implications**: Green subset RC-neutral (static checks/docs);
    enforcement subset RC-affecting -> post-verdict. 34. **RC impact**: split.
35. **cloud**: declarations are stage-2 secrets/permissions groundwork.
36. **multisport**: capability classes are sport-agnostic; new sport adapters arrive
    pre-classified.
37. **idle-window tasks**:
    - 15m: classify ONE registry tool against the declaration contract (artifact:
      one table row)
    - 30m: document one declaration group (the four market/schedule PaidExternal
      tools) (artifact: table section)
    - 60m: implement the static declaration-completeness check on the candidate
      branch (artifact: green static test; stop: requires runtime change)
    - 240m: complete Inspect+Prove: full 10-tool + endpoint inventory with violation
      fixture proving the check fails on an undeclared tool (artifact: inventory +
      red/green test pair)
38. **ready**: READY. 39. **done**: 0.6 + artifact. 40. **unresolved**: none blocking
    for the Green subset. Deferred note: cost-class ENFORCEMENT remains R-04-gated
    (budget guard) -- not part of this candidate's minimum.

**Evidence class achievable**: contract-represented -> fixture-proven (declaration
checks); integration-proven for gateway behavior already exercised. Unproven:
operational (enforcement under real load).

---

## 8. lane and change classification (evidence-based; matches expected except noted)

| candidate | implementation lane | testing lane | change class | integration risk | operational authorization |
|---|---|---|---|---|---|
| PH-01 | Green (tests/fixtures; runtime hook = stop) | Green | tests+fixtures | low (RC-neutral) | WI mint only |
| PH-02 | Amber | Green fixtures + Amber integration | contracts+observability (read-side; persistence only by explicit decision) | medium (Gate-1 surface) | WI + post-verdict integration |
| PH-03 | Amber | Green fixtures + Amber integration | runtime decision policy | HIGH (decision semantics; RC redefinition on integrate) | WI + operator policy decision + RC-impact review |
| PH-04 | Amber (verification subset Green) | Green + Amber | contracts+projection+tests | medium | WI + post-verdict |
| PH-05 | Amber; RED if external delivery exercised or schema | Green fixtures | runtime+persistence+operational | HIGH | WI + persistence decision + commercial activation gate |
| PH-06 | Green (inventory/static) then Amber (enforcement) | Green | docs+static then runtime | low then medium | WI; enforcement post-verdict |

## 9. dependency analysis (repository-evidenced)

- HARD dependencies: PH-05 -> persistence-model decision + commercial-activation
  posture (external, operator-owned). No hard inter-candidate dependency exists.
- SOFT dependencies: PH-03 and PH-04 consume PH-01 fixtures (cheaper, not required);
  PH-02 consumes PH-06 authorization/spend classes; PH-03 consumes PH-02 reasons and
  PH-06 classes; PH-04 consumes PH-03 abstention-state names.
- PARALLEL opportunities (WIP-policy-corrected): PH-06's Green inventory and
  static-validation subset can be independently bounded, but a pulled PH-06 card
  still counts as an ACTIVE IMPLEMENTATION BRANCH under the default
  one-active-implementation-branch limit. It may begin after the current
  implementation branch closes, unless the operator explicitly authorizes a
  temporary WIP-limit exception. Documentation planning does not count as an
  implementation branch, and read-only inspection performed inside an already
  authorized planning WI does not pull PH-06 -- but creating PH-06's branch marks the
  card as pulled and active. Green classification does not exempt any card from
  branch or WIP rules; no implicit WIP exception exists.
- Keep INDEPENDENT: PH-01 (corpus must not absorb runtime fixes); PH-06 declarations
  (must not grow into a permission platform).
- COUPLING RISK: building PH-05 before commercial activation = anticipatory design
  against the restraint doctrine; PH-03+PH-04 in one branch would couple decision
  policy to projection enforcement -- keep separate.
- MINIMUM USEFUL ORDER (differs from the provisional sequence by swapping PH-06
  earlier): PH-01 -> PH-06(Green subset) -> PH-02 -> PH-03 -> PH-04 -> PH-05.
  Rationale: PH-06's Green subset is idle-safe, zero-risk, and feeds PH-02/PH-03;
  PH-02 before PH-03 so abstention reasons have a trace home; PH-04 after PH-03 for
  stable state names; PH-05 last and gated. The provisional sequence's PH-02-second
  is acceptable if the operator prefers trace work before any static tooling; the
  swap is recommended, not required.

## 10. candidate readiness verdicts (READY does not authorize implementation)

| candidate | verdict | missing decision/evidence if not READY |
|---|---|---|
| PH-01 | READY | — |
| PH-02 | READY WITH OPEN QUESTION | read-side vs persisted trace fields (criterion 14 decision); legacy freshness derivability |
| PH-03 | READY WITH OPEN QUESTION | operator policy: which conditions BLOCK vs warn; unpriced-execution posture |
| PH-04 | READY | — |
| PH-05 | NOT READY | persistence model (ledger-file vs db/schema, criterion 21); commercial-activation timing (real delivery deferred); repo-safe delivery-target handling |
| PH-06 | READY | — (cost-class ENFORCEMENT explicitly excluded, R-04-gated) |

## 11. recommended implementation order (priority factors applied)

1. **First pull: PH-01** -- observed production failures exist unpinned (factor 1),
   Green lane, zero RC risk, feeds three other candidates, cheapest reversible.
2. **Second pull: PH-06 Green subset** -- tenant/authorization risk visibility
   (factor 2), idle-safe, dependency-reducing for PH-02/PH-03.
3. Then PH-02 -> PH-03 -> PH-04 (all post-RC-verdict integration; PH-03 additionally
   needs the operator blocking-policy decision before pull).
4. **Defer: PH-05** until the persistence decision AND commercial activation exist
   (building a delivery guard with delivery deferred is premature inventory).
5. **Must NOT integrate before the final RC verdict**: every dai-touching change from
   any candidate (dai main is frozen at 85a8831); categorically PH-02/PH-03/PH-04/
   PH-05 and PH-06's enforcement subset (all RC-affecting).
6. **Tests-only subsets independently useful**: PH-01 (entirely), PH-04's invariant
   fixture set, PH-06's declaration inventory + static check, PH-03's
   characterization-of-fail-open tests.

## 12. deferred notes (NOT candidates under WI-0021)

Export etag/hash ledger discipline (ops docs); cost-class budget ENFORCEMENT (R-04);
CI hosting of the corpus (FC-C3, cloud-gated); trace durable sink (stage-3);
per-sport corpus replication (qualification ladder stage 2-3).
