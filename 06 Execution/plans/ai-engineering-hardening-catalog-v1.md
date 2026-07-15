---
title: "AI Engineering Hardening Catalog v1.1 (2026-07-15; WI-0020 corrections)"
type: "plan"
date: "2026-07-15"
status: "active"
project: "DAI"
slice: "DAI AI Engineering Hardening Catalog and Protocol Ready Queue v1"
repos:
  dai: "unchanged (evidence at 85a8831)"
  dai-vault: "docs-only"
tags:
  - planning
  - hardening
  - ai-engineering
  - protocol
related:
  - "06 Execution/reports/protocol-coverage-and-maturity-matrix-v1.md"
  - "06 Execution/plans/hardening-ready-queue-v1.md"
  - "06 Execution/plans/ai-engineering-fitness-checks-v1.md"
  - "06 Execution/plans/v1-to-v2-release-sequence-v1.md"
---

# ai engineering hardening catalog v1.1

Evidence language in this catalog uses the canonical evidence taxonomy defined ONCE in
`protocol-coverage-and-maturity-matrix-v1.md` (contract-represented / fixture-proven /
integration-proven / paid-run observed / production-observed / operationally proven /
commercially validated); the classes are not interchangeable, and maturity is
dimension-specific (never inferred from test volume alone).

Catalog of branch-ready hardening features organized by the canonical protocol
(operator doctrine 2026-07-15: Perceive intake -> Interrogate -> Discern -> Decide ->
Synthesize; 12 micro-actions per the authorization; doctrine-count discrepancy with
protocol-vocabulary-map.md recorded in the maturity matrix section 0 and queued as
G-01). Nothing here is authorized work; nothing integrates before the RC verdict;
commercial posture (outreach deferred) is unchanged. Evidence loci live in the
maturity matrix -- this catalog does not repeat file:line citations already there.

Catalog defaults unless a card says otherwise: repo = dai; paid calls = 0; external
writes = 0; db writes = 0 (test databases only); reversibility = easy; release
relevance = strengthens RC confidence but NOT required before the drill; cloud/
multisport relevance = none. Labels: [observed] = evidence of the problem exists;
[preventive] = no observed defect, guarding an invariant. Sizes XS/S/M/L/XL.
Change classes: docs | tests | observability | runtime | prompt | schema | paid
operation. Every feature decomposes into Inspect -> Prove -> Guard (ready-queue doc
section 2). Branch slugs use wi/<next-id>-... -- numeric ids assigned only at mint.

## 1. feature catalog by protocol element

### Perceive (intake layer -- operational guidance, no canonical micro-actions invented)

- **CAT-PER-1 intake identity + regime contract test pack** | discipline: evidence/data
  quality | class: tests | [preventive] extends production-observed controls | problem: intake
  invariants (explicit gamePk on DH dates, regime classification, eligibility fail-closed)
  are tested piecewise; no single contract suite states the intake promise | locus:
  SourceReadiness.cs, DuplicateRunGuard, gamePk plausibility check | change: one intake
  contract test suite naming the invariants | value: operator trust in screening |
  exclusions: no readiness semantics change | deps: none | slug:
  wi/<next-id>-perceive-intake-contract-pack | size S | tests: C# contract suite |
  acceptance: suite green, invariants enumerated | artifact: suite + doc note |
  priority P2 | confidence high
- **CAT-PER-2 detect/frame/aim fixture corpus** | discipline: evaluation | class: tests |
  [preventive] | problem: perceive prose parsed + projected but content never asserted
  against staged evidence | locus: sports_analyzer parse, projection tests | change:
  persisted-run fixtures asserting frame mentions staged facts (containment style, like
  lean containment) | value: catches silent intake-prose drift | exclusions: no prompt
  change | deps: none | slug: wi/<next-id>-perceive-intake-fixture-corpus | size S |
  tests: python fixtures | acceptance: corpus green on persisted runs | priority P3 |
  confidence med
- **CAT-PER-3 readiness-vs-run regime consistency telemetry** | discipline:
  observability | class: observability | [preventive] | problem: screening regime and
  generation-time observedDataRegime can drift; consistency not surfaced | locus:
  /source-readiness, /rows observedDataRegime | change: read-side consistency flag on
  /rows | value: operator sees when screen != capture | exclusions: no behavior change |
  deps: none | slug: wi/<next-id>-perceive-regime-consistency-telemetry | size S |
  class amber | priority P3 | confidence med

### Interrogate.Question

- **CAT-INT-Q-1 question presence contract on directional reads** | discipline:
  evidence/data quality | class: tests | [preventive] | problem: question is null-safe
  only; a directional read with no named counter-question is a silent quality drop
  (counter_case falls back to question today) | locus: seed parse, quality checker |
  change: contract test: directional artifact => question non-null OR quality warning
  present | value: honest counter-case coverage | exclusions: no fail-closed change to
  compose | deps: none | slug: wi/<next-id>-interrogate-question-presence-contract |
  size XS | acceptance: test states the invariant, green on corpus | priority P2 |
  confidence high
- **CAT-INT-Q-2 question/verify semantic fixture suite** | discipline: evaluation |
  class: tests | [observed: matrix records structural-only coverage for
  question/verify/justify/resolve] | problem: prose micro-actions have zero
  content-level evaluation | change: persisted-run fixtures asserting question names a
  concrete uncertainty and verify references staged evidence (containment heuristics,
  no model call) | slug: wi/<next-id>-interrogate-question-semantic-fixtures | size S |
  priority P1 (top evaluation gap) | confidence med
- **CAT-INT-Q-3 protocol completeness metric (question field)** | discipline:
  observability | class: observability | [observed: no operator surface shows per-field
  protocol completion] | change: per-run protocol field null-map on prompt-trace (part
  of trace-completeness slice A-02) | slug: wi/<next-id>-interrogate-question-completeness-metric |
  size S (shared) | priority P1 | confidence high

### Interrogate.Probe

- **CAT-INT-P-1 probe template registry contract** | discipline: prompt/recipe
  governance (deterministic templates) | class: tests+docs | [observed: unknown signals
  silently dropped; line_movement permanently not_implemented but present in
  FollowUpSignals] | change: contract test enumerating templated vs deliberately
  untemplated signals; doc table of drop reasons | slug:
  wi/<next-id>-interrogate-probe-template-contract | size XS | priority P2 |
  confidence high
- **CAT-INT-P-2 probe determinism replay corpus** | discipline: evaluation | class:
  tests | [preventive] | change: golden fixtures from persisted runs re-running
  BuildProbe; byte-stable output | slug: wi/<next-id>-interrogate-probe-replay-corpus |
  size S | priority P3 (already 35 tests) | confidence high
- **CAT-INT-P-3 probe population telemetry** | discipline: observability | class:
  observability | [preventive] | change: probe null-rate per regime on /rows |
  slug: wi/<next-id>-interrogate-probe-population-telemetry | size S | priority P3 |
  confidence med

### Interrogate.Verify

- **CAT-INT-V-1 verify grounding cross-check** | discipline: evidence/data quality |
  class: tests | [preventive] | problem: QualityChecker rule 4 validates signals_used
  against grounded set, but verify prose can reference ungrounded evidence unchecked |
  change: containment check: verify references only staged evidence terms | slug:
  wi/<next-id>-interrogate-verify-grounding-check | size M | priority P2 |
  confidence med
- **CAT-INT-V-2 semantic fixtures** | shared with CAT-INT-Q-2 | slug:
  wi/<next-id>-interrogate-verify-semantic-fixtures | size (shared)
- **CAT-INT-V-3 quality-warning operator surfacing** | discipline: observability |
  class: observability | [OBSERVED: SportsQualityChecker warnings persist in OutputJson
  only, deliberately unsurfaced (:8-9) -- operator cannot see them anywhere] | change:
  read-side exposure of ArtifactQualityWarnings on /artifact response (dev page
  section) | value: closes the biggest protocol observability gap | exclusions: buyer
  surfaces untouched; checker stays fail-open | slug:
  wi/<next-id>-interrogate-verify-warning-surfacing | size S | priority P1 |
  confidence high

### Discern.Weigh

- **CAT-DIS-W-1 signal-quality matrix lock** | discipline: evidence/data quality |
  class: tests | [preventive] | problem: SignalQualityEvaluator couplings (sharp_public
  grounding gates market quality; block_aggressive_posture) are behavior-critical and
  tested piecewise | change: single table-driven matrix test locking the full
  quality/decision-use/confidence-effect mapping | slug:
  wi/<next-id>-discern-weigh-matrix-lock | size XS | priority P2 | confidence high
- **CAT-DIS-W-2 weigh prose vs deterministic surface containment** | discipline:
  evaluation | class: tests | [preventive] | change: fixture check that weigh prose
  does not contradict SignalAvailability quality grades (containment heuristic) |
  slug: wi/<next-id>-discern-weigh-prose-containment | size M | priority P3 |
  confidence med
- **CAT-DIS-W-3 anti-hype gate telemetry** | discipline: observability | class:
  observability | [preventive] | change: count + surface block_aggressive_posture
  activations per cohort on /rows | slug: wi/<next-id>-discern-weigh-antihype-telemetry |
  size XS | priority P3 | confidence high

### Discern.Contrast

- **CAT-DIS-C-1 market-attribution fidelity extension** | discipline: evidence/data
  quality | class: tests | [preventive; 15 tests exist] | change: extend fidelity states
  to cover all divergence wording branches incl. thin-market states | slug:
  wi/<next-id>-discern-contrast-fidelity-extension | size XS | priority P3 |
  confidence high
- **CAT-DIS-C-2 contrast vs persisted marketAgreement replay** | discipline: evaluation |
  class: tests | [preventive] | change: settled-run fixtures asserting contrast direction
  consistent with persisted market fields | slug:
  wi/<next-id>-discern-contrast-agreement-replay | size S | priority P2 | confidence med
- **CAT-DIS-C-3 contrast population rate on /rows** | discipline: observability |
  class: observability | [preventive] | shared with completeness metric | slug:
  wi/<next-id>-discern-contrast-completeness | size (shared) | priority P2

### Discern.Stress

- **CAT-DIS-S-1 single-source stress invariant guard** | discipline: prompt/recipe
  governance | class: tests | [preventive; retired names documented] | change: static
  vocabulary test banning interrogate.stress / discern.test / decide.choose from
  runtime code (fitness check FC-1) | slug: wi/<next-id>-discern-stress-vocabulary-guard |
  size XS | priority P2 | confidence high
- **CAT-DIS-S-2 stress -> watchFor derivation corpus** | discipline: evaluation |
  class: tests | [preventive; one fallback test exists] | change: fixtures through brief
  AND recap watchFor paths | slug: wi/<next-id>-discern-stress-watchfor-corpus |
  size XS | priority P3 | confidence high
- **CAT-DIS-S-3 stress completeness telemetry** | shared completeness metric | slug:
  wi/<next-id>-discern-stress-completeness

### Decide.Resolve

- **CAT-DEC-R-1 direction-consistency failure corpus** | discipline: evaluation |
  class: tests | [OBSERVED: 4 real lean-vs-prose mismatch runs were excluded in
  production (lean-encoding-integrity history); 823357 postponement precedent] |
  change: encode those runs as permanent regression fixtures through the consistency
  evaluator + settlement 422 guard | value: the control that fired stays fired |
  slug: wi/<next-id>-decide-resolve-failure-corpus | size S | priority P1 |
  confidence high
- **CAT-DEC-R-2 resolve prose containment** | discipline: evidence/data quality |
  class: tests | [preventive] | change: resolve prose containment vs structured lean
  (extends the 11 lean-containment tests) | slug:
  wi/<next-id>-decide-resolve-prose-containment | size S | priority P2 | confidence high
- **CAT-DEC-R-3 consistency-refusal telemetry** | discipline: observability | class:
  observability | [preventive] | change: surface 422 integrity-refusal counts on the
  calibration read path | slug: wi/<next-id>-decide-resolve-refusal-telemetry |
  size XS | priority P3 | confidence med

### Decide.Position

- **CAT-DEC-P-1 posture vocabulary lockstep contract** | discipline: prompt/recipe
  governance | class: tests | [preventive] | problem: posture enum lives in python
  validation, C# passthrough, Angular rendering, buyer stance mapping -- drift across
  stacks would be silent | change: shared vocabulary contract test in each stack from
  one fixture list | slug: wi/<next-id>-decide-position-vocabulary-lockstep | size S |
  priority P2 | confidence high
- **CAT-DEC-P-2 no-position matrix extension** | discipline: evaluation | class: tests |
  [preventive; WI-0011 no-position rendering fixture-proven + production-observed] | change: full posture x lean-null
  matrix through brief + recap | slug: wi/<next-id>-decide-position-noposition-matrix |
  size XS | priority P3 | confidence high
- **CAT-DEC-P-3 posture distribution telemetry** | discipline: observability | class:
  observability | [preventive] | change: posture distribution per cohort on /rows |
  slug: wi/<next-id>-decide-position-distribution-telemetry | size S | priority P3 |
  confidence med

### Decide.Justify

- **CAT-DEC-J-1 justify claim-safety contract** | discipline: evidence/data quality |
  class: tests | [preventive] | problem: justify is dev-only today, but if ever surfaced
  it must pass buyer-copy banned-term rules; no test asserts that | change: claim-safety
  check over justify text in the suppression-list style | slug:
  wi/<next-id>-decide-justify-claim-safety | size XS | priority P3 | confidence high
- **CAT-DEC-J-2 justify presence fixture on directional reads** | discipline:
  evaluation | class: tests | [preventive] | slug:
  wi/<next-id>-decide-justify-presence-fixture | size XS | priority P3 | confidence high
- **CAT-DEC-J-3 completeness telemetry (justify field)** | shared completeness metric

### Synthesize.Integrate

- **CAT-SYN-I-1 protocol completion invariant contract** | discipline: evidence/data
  quality | class: tests | [preventive; partially in composer tests] | change: contract
  test: successful compose => CognitiveProtocol non-null with seed passthrough +
  deterministic probe + trio; ComposeFailedRun => null | slug:
  wi/<next-id>-synthesize-integrate-completion-invariant | size XS | priority P2 |
  confidence high
- **CAT-SYN-I-2 deterministic run replay harness (seed)** | discipline: evaluation |
  class: tests+observability | [OBSERVED gap: shadow soak replays prompt assembly only;
  no persisted-OutputJson -> projection replay exists] | change: replay persisted runs
  through ProtocolView + brief + recap + /rows row builders, byte-stable, zero model
  calls | slug: wi/<next-id>-synthesize-integrate-run-replay | size M | priority P1 |
  confidence high | post-RC candidate 1
- **CAT-SYN-I-3 compose failure-mode telemetry** | discipline: observability |
  class: observability | [preventive] | change: ComposeFailedRun rate + reasons
  surfaced operator-side | slug: wi/<next-id>-synthesize-integrate-failure-telemetry |
  size XS | priority P3 | confidence med

### Synthesize.Compose

- **CAT-SYN-C-1 export determinism edge-state corpus** | discipline: evidence/data
  quality | class: tests | [preventive; 25+23 determinism tests exist for the main
  states] | change: extend byte-determinism corpus to excluded / no_position /
  not_settled / settled_not_evaluated / inconsistent recap states | slug:
  wi/<next-id>-synthesize-compose-edge-determinism | size XS | priority P2 |
  confidence high
- **CAT-SYN-C-2 recap state-machine coverage matrix** | discipline: evaluation |
  class: tests | [preventive] | change: one matrix test enumerating every recapState
  with its rendering contract | slug: wi/<next-id>-synthesize-compose-recap-matrix |
  size S | priority P2 | confidence high
- **CAT-SYN-C-3 markdown double-fetch hash discipline** | discipline: observability /
  human operation | class: docs | [preventive; runbook step 4.2 already requires it] |
  change: record hashes on the ledger row (docs-only amendment) | slug:
  wi/<next-id>-synthesize-compose-hash-ledger | size XS | priority P3 | confidence high

### Synthesize.Deliver

- **CAT-SYN-D-1 protocol-leak sentinel on buyer payloads** | discipline: security /
  evidence quality | class: tests | [preventive; suppression list exists fail-closed] |
  change: sentinel test asserting no protocol vocabulary (probe, sharp_public, regime
  names, fallback...) reaches brief/recap JSON or markdown | slug:
  wi/<next-id>-synthesize-deliver-protocol-leak-sentinel | size S | priority P1 |
  confidence high
- **CAT-SYN-D-2 settled-pair delivery e2e fixture** | discipline: evaluation | class:
  tests | [preventive] | change: 823845 brief+recap pair as an end-to-end fixture
  (persisted -> both exports -> claim-safety -> byte-stable) | slug:
  wi/<next-id>-synthesize-deliver-settled-pair-fixture | size XS | priority P2 |
  confidence high
- **CAT-SYN-D-3 delivery ledger linkage guidance** | discipline: human operation |
  class: docs | [preventive] | change: ledger row <-> run id <-> export hash linkage
  documented in the ledger template | slug: wi/<next-id>-synthesize-deliver-ledger-linkage |
  size XS | priority P3 | confidence high

## 2. cross-cutting candidate evaluations (suggested candidates, judged on evidence)

- **Cognitive Protocol Fitness Harness v1** -- worthwhile as ONE cross-boundary
  roundtrip suite (seed -> completed -> persisted -> projected); much is piecewise
  covered already. Verdict: Amber M (queue A-05); the static vocabulary check lands
  earlier as fitness check FC-1.
- **Deterministic Run Replay and Failure Corpus v1** -- strongest candidate; observed
  gap (assembly-only replay today) + real failure corpus available (4 mismatch runs,
  823357). Verdict: Amber M; POST-RC CANDIDATE 1 (CAT-SYN-I-2 + CAT-DEC-R-1).
- **Evidence and Decision Trace Completeness v1** -- observed gap (no protocol
  completion status on any surface; fallbackDetail only for assembly_error). Verdict:
  Amber S-M; POST-RC CANDIDATE 2 (queue A-02).
- **Tool Authorization Classification v1** -- registry already declares CostClass +
  AllowedProtocolNodes and the gateway enforces node allowlists fail-closed; cost-class
  is recorded NOT enforced; callers pass stage sentinels. Verdict: static
  classification audit = Green S (queue G-09); runtime enforcement = Red (spend
  behavior), gated.
- **Prompt and Recipe Change Impact Harness v1** -- equivalence + golden-hash + manifest
  integrity tests already fail-close the registry path, and NO prompt change is
  authorized or planned. Verdict: DEFERRED until a prompt change is actually proposed
  (evidence trigger: an authorized prompt-change WI exists). Building it now is
  speculative.
- **Provider Degradation Contract v1** -- observed: unified regime vocabulary exists
  only at the MLB readiness layer; raw clients fail-soft ad hoc. Verdict: design doc =
  Green S (queue G-06); implementation belongs to the WI-0016 capability-contract seam
  (multisport-gated).
- **Competition Capability Descriptor v1** -- already proposed as WI-0016 (gated); the
  gates table explicitly allows an early contract DESIGN doc. Verdict: design doc =
  Green M-docs (queue G-07); POST-RC CANDIDATE 3.
- **AI Cost and Latency Budget Guard v1** -- caps are procedural (runbook/drill);
  cost-class enforcement deferred by design; single-operator V1 risk is low and
  metering already names unpriced states loudly. Verdict: retained GATED (Red R-04);
  triggers: unpriced-warning event, cap near-miss, automation gate, or cloud gate.

## 3. ai engineering discipline map (controls at 85a8831)

| discipline | existing controls (strength) | gaps / weak / duplicated / expensive | phases served |
|---|---|---|---|
| evaluation engineering | deterministic testing and invariant coverage: MATURE (1141 C# + 399 py + 134 vitest; golden hashes; byte-determinism brief/recap; pooled_calibration Gate-4 sufficiency gates; shadow soak assembly replay). semantic evaluation and representative failure-corpus coverage: DEVELOPING -- test volume alone does not confer maturity on that dimension | GAP: no persisted-run replay through projections; prose micro-actions untested semantically; Angular protocol view untested | all |
| evidence/data quality | regime classifier (production-observed through 15 settlements); grounded-signal rule; sufficiency gate fail-closed; lean containment; direction consistency (production-observed: fired on 4 real runs) | WEAK: quality checker fails open AND invisible; verify prose ungrounded-checkable | Perceive, Interrogate, Decide |
| prompt/recipe governance | registry v2 + manifest hash verify + 57 governance tests; canary default-off; byte-identical promotion rule; fail-closed fallback with named reasons | none urgent; impact harness deferred until a change is proposed | Interrogate, Decide |
| model config/metering | PRICING covers both configured models; named unpriced states; cost log JSON lines; metering coverage test | OBSERVED: model name dual-sourced (analyzer hardcodes gpt-4o-mini :657; AgentProfile default gpt-4.1-mini) -- consistency risk, not a defect today | all |
| tool routing/permissions | ToolGateway fail-closed (registry + node allowlist); 10 tools with CostClass/idempotency/TTL metadata; per-invocation telemetry | cost-class NOT enforced (deferred by design); station-id policy branch dormant | Interrogate (probe seam), platform |
| provenance/traceability | settlement residue 422 (ADR 0006); X-Agent-Run-Id; route provenance + attributionStatus; gamePk propagation (WI-0009) | trace lacks protocol completion status; fallbackDetail narrow | Decide, Synthesize |
| reliability/recovery | duplicate 409 guard (23 tests); idempotent settlement 409; failed-never-blocks matrix; verified shutdown script; identity-safe starter cache | single-instance only (known, gated WI-0015) | intake, Synthesize |
| observability | structured logs; cost log; /rows rich export; prompt-trace; cognitive-factory diagnostics (read-only) | WEAK: single /health (no readiness); calibration metrics denominator includes excluded runs (documented, gated); quality warnings invisible | all |
| security/tenant isolation | fail-closed authorize policy + DevBypass double condition; tenant-scoped queries + 404 matrices; batch-runs double-gated fail-closed; buyer-copy suppression | OBSERVED (corrected v1.1, verified read-only 2026-07-15): appsettings.Development.json is GITIGNORED and was NEVER committed -- its OddsApi:ApiKey, Dev:ProvisionKey, and SQL password are local-file-only; the tracked appsettings.json DID historically carry the SQL connection string incl. password until `ded9969` replaced it with a placeholder, so HISTORICAL repository exposure exists for the sa password. Classification + escalation = G-10 (documentation, unexecuted); rotation/revocation/history response = R-05 (operator gate). G-10 completion is NOT remediation | all |
| cost/latency | per-call cost + latency_ms; 30s model timeout; 1500 completion-token cap | no per-run rollup persisted; no budget enforcement (procedural caps only) -- acceptable for V1, gated R-04 | Decide, Synthesize |
| deployment/reproducibility | Dockerfiles + compose.smoke; deterministic exports; snapshot tooling | GAP: CI empty despite 1600+ tests; 1 EF migration, schema drift out-of-band (cloud blockers, WI-0014) | platform |
| provider resilience | per-provider fail-soft-to-null; readiness regime vocabulary bridges to honest postures; ActionNetwork degradation observable | no unified cross-provider state machine (G-06 design doc; implementation multisport-gated) | Perceive |
| human operation | runbook (Gate 0 verified end-to-end); recovery table R1-R14; ledger templates; drill package | none blocking; hash-ledger linkage (CAT-SYN-C-3/D-3) is polish | all |
| commercial feedback | ledger + pilot metrics designed; Stripe-as-truth doctrine | loop has never carried a transaction -- DEFERRED by operator decision 2026-07-15; hardening work must NOT be reinterpreted as commercial validation | Deliver |

Duplicated controls: none found that warrant removal; probe re-derivation on
prompt-trace intentionally mirrors the builder (verification value).
Disproportionately expensive controls: none observed; the three-authorization docs
cadence for XS reviewed slices was already flagged in the release review (process
doctrine, operator decision).
