---
title: "Hardening Ready Queue v1 (2026-07-15)"
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
  - queue
  - workflow
related:
  - "06 Execution/plans/ai-engineering-hardening-catalog-v1.md"
  - "06 Execution/reports/protocol-coverage-and-maturity-matrix-v1.md"
  - "06 Execution/plans/ai-engineering-fitness-checks-v1.md"
---

# hardening ready queue v1

Branch-ready queue over the hardening catalog. NOTHING here is minted or authorized;
branches are created only when a card is pulled; numeric WI ids are assigned only at
mint time (slugs use <next-id>). Release boundary: dai main is FROZEN as the RC commit
`85a8831` until the final RC verdict -- no dai-touching card may integrate before it,
and no card may delay RC Gate 1 (2026-07-17). A passing RC verdict does NOT authorize
this queue; pulling any card is its own operator decision.

## 1. work lanes (authoritative)

### Green -- idle-safe
- characteristics: docs, tests, fixtures, static checks, read-only analysis; no paid
  calls; no external writes; no db writes (test dbs only); no schema change; no
  locked-layer behavior change; easily reversible; does not alter the active RC
- entry criteria: repo state verified; card unblocked; no operational event in progress
  that needs attention
- authorization: standing for vault-docs-only cards; dai-touching Green cards need a
  minted WI + branch but NO extra spend authority
- branch: vault-docs cards commit direct to vault main; dai cards get wi/<next-id>-...
  branch, never integrate pre-verdict
- tests: suites relevant to touched area green; docs cards = snapshot 0 warnings
- integration: ff-only after review; pre-verdict dai integration PROHIBITED
- prohibited: paid calls, external writes, db writes, schema, prompt/model/scoring
  changes, touching locked layers, touching the RC commit

### Amber -- bounded implementation
- characteristics: may change runtime or observability code; formal WI + branch
  required; no paid calls or external writes during implementation; separate review +
  integration boundary; must not silently alter prompts, scoring, or buyer semantics
- entry criteria: WI minted under the WI-0007 gate; evidence cited; WIP slot free
- authorization: implementation authorization per WI; integration separately
- branch: wi/<next-id>-<protocol>-<micro-action>-<feature>
- tests: red-first; full relevant suites green; byte-identical checks where applicable
- integration: ff-only, post-RC-verdict only, own authorization
- prohibited: paid calls, external writes, schema migrations, prompt/model/scoring/
  buyer-semantics changes, /metrics tracked-number changes

### Red -- explicit operator gate
- includes: paid model calls, captures, reconciliation, external writes, schema
  migrations, prompt or scoring changes, model changes, cloud deployment, tenant-data
  changes, multi-instance concurrency changes
- entry criteria: named gate open (see release-sequence gates) + explicit operator
  authorization naming caps/candidates
- authorization: per-event, never standing; branch + WI mandatory; staged spend
- tests: full gate structure unchanged (red-first, suites, drill-style evidence)
- integration: own authorization; evidence artifact mandatory
- prohibited: borrowing a prior authorization; inferring authority from a passing
  verdict or an adjacent gate

## 2. three-step branch execution pattern (work-packaging convention)

This is a WORK-PACKAGING convention for every card. It is NOT a replacement for, or a
mapping of, the cognitive protocol micro-actions -- do not conflate them.

1. **Inspect** -- locate the existing behavior; establish the contract as written;
   identify evidence and gaps; make NO behavior change (read-only; produces notes or a
   failing-test sketch).
2. **Prove** -- add a deterministic fixture, test, replay, or diagnostic that reproduces
   the gap or pins the current invariant; avoid paid calls (persisted runs are the
   corpus); red-first where a defect is claimed.
3. **Guard** -- add the smallest protection, invariant, alert, or documentation
   correction; prove regression resistance (the Prove artifact now passes and stays);
   preserve explicit boundaries (no scope growth inside the branch).

## 3. ready queue (17 cards)

Format: id | title | slug | lane | size | dep | repo | likely files | I/P/G | tests |
auth | integration | stop | done. Defaults: paid 0 / external writes 0 / db writes 0.

### Green

- **G-01 | Protocol Doctrine Count Reconciliation v1** |
  wi/<next-id>-perceive-doctrine-reconciliation | Green | XS | none | dai-vault |
  02 Platform/architecture/cognitive-factory/protocol-vocabulary-map.md |
  I: present both countings (map 2026-05-14 vs authorization 2026-07-15) with evidence;
  P: side-by-side table, impact of each on catalog organization;
  G: operator decides the canonical counting; map updated with a decision record |
  tests: none (docs) | auth: operator doctrine decision REQUIRED | integration: vault
  commit | stop: any temptation to rename runtime fields | done: one canonical counting
  recorded, no runtime change
- **G-02 | Perceive Intake Checklist v1 (operational guidance)** |
  wi/<next-id>-perceive-intake-checklist | Green | XS | G-01 helpful not required |
  dai-vault | runbook section 2 + matrix section 4 | I: confirm checklist matches code;
  P: walk one persisted run against it; G: label as operational guidance, link from
  runbook | done: checklist published, explicitly non-canonical
- **G-03 | Prose Micro-action Semantic Fixture Suite v1** (CAT-INT-Q-2/V-2) |
  wi/<next-id>-interrogate-question-semantic-fixtures | Green | S | none | dai |
  services/agent-service/tests/ + persisted-run fixtures | I: pick 10-15 persisted runs
  across regimes; P: containment heuristics for question/verify/justify/resolve;
  G: suite green + documented heuristic limits | tests: python | auth: WI (no spend) |
  integration: post-verdict | stop: heuristics require a model call -> STOP, redesign |
  done: 4 prose fields have content-level assertions
- **G-04 | Direction-Consistency Failure Corpus v1** (CAT-DEC-R-1) |
  wi/<next-id>-decide-resolve-failure-corpus | Green | S | none | dai |
  DevCore.Api.Tests fixtures | I: extract the 4 excluded mismatch runs + 823357 residue;
  P: fixtures reproduce the 422/exclusion behavior; G: permanent regression corpus |
  auth: WI (no spend; fixtures from persisted data) | stop: fixture requires live db ->
  use exported residue only | done: real production failures pinned forever
- **G-05 | Protocol Vocabulary Stability Check v1** (CAT-DIS-S-1, FC-1) |
  wi/<next-id>-discern-stress-vocabulary-guard | Green | XS | none | dai | one static
  test (C# or script) | I: enumerate retired names; P: failing test on a seeded
  violation; G: static check in the suite | done: retired vocabulary cannot silently
  return
- **G-06 | Provider Degradation Contract Design Doc v1** |
  wi/<next-id>-perceive-provider-degradation-design | Green | S | none | dai-vault |
  02 Platform/architecture/ | I: inventory per-client fail-soft behaviors; P: map to
  unavailable/thin/stale/contradictory/failed states against the existing regime
  vocabulary; G: design doc, implementation explicitly multisport-gated | done: contract
  designed, nothing activated
- **G-07 | Competition Capability Descriptor Design Doc v1** (WI-0016 design-only,
  allowed early by the gates table) | wi/<next-id>-synthesize-deliver-capability-descriptor-design |
  Green | M | none | dai-vault | 02 Platform/architecture/ + capability matrix doc |
  I: seam classification from the matrix; P: descriptor schema draft against MLB as
  reference adapter; G: design doc; NO code, NO WI-0016 promotion | done: descriptor
  design exists for the second-sport gate to consume
- **G-08 | Angular Protocol View Spec Coverage v1** |
  wi/<next-id>-synthesize-compose-devpage-specs | Green | S | none | dai |
  apps/sports-app dev-artifact-review specs | I: current rendering contract; P: vitest
  specs for ProtocolView rendering incl. null fields; G: specs in suite | done: dev page
  protocol rendering no longer untested
- **G-09 | Tool Authorization Classification Audit v1** |
  wi/<next-id>-interrogate-probe-tool-authorization-audit | Green | S | none | dai +
  dai-vault | Tools/ToolRegistry.cs + static test + doc table | I: enumerate 10 tools'
  CostClass/AllowedProtocolNodes + all controller endpoints' auth classes; P: static
  test asserting every tool declares both; doc table of endpoint -> class; G: test in
  suite + table in vault | done: every tool/endpoint has a declared class; enforcement
  gaps explicitly listed (cost-class enforcement stays Red)
- **G-10 | Secrets Hygiene Record + Rotation Checklist v1** |
  wi/<next-id>-perceive-secrets-hygiene-checklist | Green | XS | none | dai-vault |
  runbook amendment | I: record that OddsApi:ApiKey + Dev:ProvisionKey sit in committed
  appsettings.Development.json (observed this slice); P: rotation checklist (odds key,
  sa password, provision key) with verification steps; G: runbook note; ROTATION ITSELF
  = operator action (R-05) | done: hygiene state recorded, rotation ready to execute

### Amber

- **A-01 | Quality-Warning Operator Surfacing v1** (CAT-INT-V-3) |
  wi/<next-id>-interrogate-verify-warning-surfacing | Amber | S | none | dai |
  AgentRunContracts.cs, AgentRunsController /artifact, dev page | I: warning storage +
  read path; P: failing test: warnings present in OutputJson invisible on /artifact;
  G: read-side field + dev-page section; checker stays fail-open, buyer untouched |
  tests: C# + vitest | stop: any pressure to make warnings fail-closed in compose |
  done: operator sees every quality warning
- **A-02 | Protocol Trace Completeness v1** (post-RC candidate 2) |
  wi/<next-id>-interrogate-question-trace-completeness | Amber | S-M | none | dai |
  PromptTrace.cs + tests | I: current trace fields; P: failing test for protocol
  field null-map absence; G: completion status (per-field present/null + probe/
  attribution status) on prompt-trace | stop: temptation to add model-quality scoring |
  done: one surface answers "which protocol fields did this run actually fill"
- **A-03 | Deterministic Run Replay Harness v1** (post-RC candidate 1; CAT-SYN-I-2 +
  G-04 corpus) | wi/<next-id>-synthesize-integrate-run-replay | Amber | M | G-04
  helpful | dai | test harness project/scripts | I: enumerate projection surfaces
  (ProtocolView, brief, recap, /rows row); P: replay 15+ persisted runs byte-stable,
  zero model calls; G: harness in suite + failure corpus wired in | stop: any network
  call in the harness | done: full-run replay exists; failure corpus replayable
- **A-04 | Model Configuration Single Source v1** |
  wi/<next-id>-decide-position-model-config-single-source | Amber | S | none | dai |
  sports_analyzer.py:657, AgentProfile provisioning | I: document both sources and
  which calls use which; P: test pinning current effective behavior; G: one config
  source read by both paths WITHOUT changing any effective model per path (any change
  to an effective model = Red, separate gate) | stop: effective model would change ->
  STOP | done: one source of truth, byte-identical behavior
- **A-05 | Cognitive Protocol Fitness Harness v1** |
  wi/<next-id>-synthesize-integrate-fitness-harness | Amber | M | A-03 | dai |
  cross-boundary test suite | I: boundary inventory (seed -> completed -> persisted ->
  projected); P: roundtrip invariants as one suite; G: harness green in CI-less local
  run script | done: cross-boundary invariants named and locked
- **A-06 | Readiness Probe Split v1** | wi/<next-id>-perceive-readiness-probe | Amber |
  S | none | dai | Program.cs health mapping | I: single /health today; P: failing
  readiness expectation (db reachable); G: /health/live + /health/ready split, runbook
  updated | cloud relevance HIGH (stage-2 prereq) | done: readiness distinct from
  liveness
- **A-07 | Per-Run Cost Rollup on /rows v1** |
  wi/<next-id>-synthesize-deliver-cost-rollup | Amber | S | none | dai |
  calibration export + cost log correlation | I: cost lines vs run ids; P: failing test
  for rollup absence; G: measured cost per run surfaced read-side | stop: no new
  persistence schema (read-side correlation only) or -> Red | done: operator sees cost
  per run where they already look

### Red (retained, gated -- NOT pull-able from this queue)

- **R-01** /metrics denominator exclusion filtering (changes tracked numbers; own
  approval; post-V1 triage standing)
- **R-02** DB-coordinated duplicate enforcement = WI-0015 (schema migration;
  multi-instance gate)
- **R-03** any prompt/recipe change + its impact harness (locked layer; harness builds
  only when a change WI exists)
- **R-04** AI cost/latency budget ENFORCEMENT (gateway cost-class enforcement, spend
  caps in code; triggers: unpriced event, cap near-miss, automation or cloud gate)
- **R-05** secrets ROTATION execution (odds key, sa password, provision key -- operator
  action; G-10 prepares it)
- **R-06** EF migration path verification on managed SQL (WI-0014, cloud gate)
- **R-07** NBA settlement-grade identity provider (WI-0017 ladder stage 1)

### Deferred (not premature in principle, wrong time)
- Prompt & Recipe Change Impact Harness (until a prompt-change WI exists)
- current-slice.md compaction / single-evidence-record adoption (process doctrine
  decision from the release review, operator-owned)
- AgentRunsController split (~1,400 lines; split when it next changes materially)

### Rejected as premature abstraction
- activating DiscernStationRunner / ProtocolNodeRunner stations to make runtime match
  the conceptual diagram (explicit doctrine anti-goal, recorded in diagnostics)
- unified multi-provider state-machine IMPLEMENTATION before a second sport exists
- mutation testing, dependency-graph linting, runtime architecture monitors (already
  rejected in the runway fitness checks; risk unchanged)
- distributed starter cache / multi-instance concurrency work before WI-0015's gate
- Prometheus /metrics endpoint (no consumer exists; single-host pilot)

## 4. prioritized ranking (criteria: observed pain > release/tenant risk > feedback
quality > cross-niche reuse > cost > reversibility > evidence > distraction risk)

**Top five Green:** 1) G-10 secrets hygiene (observed, security, XS) 2) G-01 doctrine
reconciliation (observed conflict, blocks clean catalog vocabulary) 3) G-04 failure
corpus (real production defects pinned) 4) G-03 semantic fixtures (largest evaluation
gap) 5) G-09 tool authorization audit (release+tenant safety visibility).

**Top five Amber:** 1) A-01 quality-warning surfacing (observed invisibility)
2) A-02 trace completeness 3) A-03 run replay harness 4) A-04 model config single
source (observed dual-source) 5) A-06 readiness probe (cheap, cloud-prep).

## 5. idle-window selector (deterministic)

Inputs: available time; lane clearance; repo state (must match approved hashes); active
release gate (pre-Gate-1 / between gates / post-verdict); risk; dependencies; WIP state.

Selection algorithm:
1. verify repo state (runbook 1.1); mismatch -> STOP, no selection.
2. if an operational event is today (drill day, settlement day): select NOTHING
   (docs-only planning is allowed only when it does not distract).
3. pre-RC-verdict: only vault-docs-only Green cards are eligible (G-01, G-02, G-06,
   G-07, G-10 doc part). dai-touching cards are NOT eligible before the verdict.
4. post-verdict: eligible = unblocked cards in lane order Green then Amber, ranked per
   section 4; Amber eligible only if the single WIP slot is free.
5. within eligible, pick by time window:

| window | pick | expectation | stop condition |
|---|---|---|---|
| 15 min | top vault-docs Green card, Inspect only | progress only | timebox ends -> note state |
| 30 min | vault-docs Green card XS (complete) or Inspect of a test card | XS completion | any surprise -> record, stop |
| 60 min | Green XS/S card full Inspect+Prove+Guard | completion | Prove fails to pin behavior -> stop after Prove, record |
| half day (240 min) | Green S complete, or Amber S Inspect+Prove (no integration) | Green: done; Amber: through Prove | red test cannot be written -> stop |
| full day | one Amber S/M card through Guard on its branch | branch ready for review; integration SEPARATE | suite regression -> stop, handoff |

6. output: one card + why it fits + exact I/P/G boundary for the window + completion
   vs progress expectation + stop condition.

The selector NEVER recommends: a second active implementation branch; anything
modifying the RC commit before the drill/verdict; a card whose evidence is missing;
speculative abstraction (rejected list); any paid or write-bearing action.

## 6. wip and branch lifecycle rules

- default WIP limit: **max ONE active implementation branch** across dai.
- docs-only planning may proceed in parallel ONLY when it does not distract from an
  authorized operational event (drill days: nothing else).
- NO precreated empty branches; a branch exists only when its card is pulled and its
  WI is minted (branch = wi/<next-id>-<protocol>-<micro-action>-<feature>).
- <next-id> is assigned at mint time from the WI registry; catalog/queue docs never
  fix numeric ids in advance (avoids collision with WI-0014..0019 proposals).
- an unfinished branch MUST carry a current handoff note + explicit next action before
  the operator walks away.
- integration NEVER happens merely because a branch exists: ff-only, reviewed, under
  its own authorization, post-RC-verdict for anything touching dai.
- retention (existing convention): integrated wi/* branches are retained at their
  integration hash; abandoned branches are deleted only after their evidence (if any)
  is recorded in the vault; no history rewrites.

## 7. first three post-RC hardening candidates (considered AFTER the final RC verdict;
NOT authorized, NOT minted by this document)

1. **Deterministic Run Replay and Failure Corpus v1** (A-03 + G-04) -- strengthens
   evaluation/replay. Why: the only replay today is prompt-assembly-side; projections
   and buyer exports have no persisted-run replay, and four real production failures
   plus the 823357 precedent exist as an unpinned corpus. Why not before the verdict:
   it touches dai and the RC commit is frozen; the drill needs zero of it. Trigger
   evidence: final RC verdict recorded AND the failure corpus enumerated from /rows.
   Branch: wi/<next-id>-synthesize-integrate-run-replay. Size M. Payment/repeat-use
   gated: NO (internal quality; zero spend).
2. **Protocol Trace Completeness v1** (A-02) -- strengthens protocol trace
   completeness. Why: no surface answers "which protocol fields did this run fill";
   quality warnings are invisible (pairs with A-01); operator debugging during the
   pilot depends on the trace. Why not before: prompt-trace is on the RC-frozen
   surface; drill evidence uses the existing trace shape. Trigger evidence: RC verdict
   + first pilot-week operational friction note referencing trace gaps (or operator
   election). Branch: wi/<next-id>-interrogate-question-trace-completeness. Size S-M.
   Payment-gated: NO.
3. **Competition Capability Descriptor Design Doc v1** (G-07) -- strengthens
   cross-sport portability/capability declaration. Why: the seam inventory shows the
   buyer-ready list hardcoded in the frontend and sport logic behind thin seams; the
   gates table explicitly permits early contract DESIGN. Why not before: zero release
   value pre-verdict; attention belongs to the drill. Trigger evidence: RC verdict +
   operator interest in the second-sport option (NBA window planning). Branch:
   vault-docs (design only; implementation stays WI-0016, second-sport-gated). Size M
   (docs). Payment/repeat-use gated: implementation YES (second-sport gate);
   design doc NO.
