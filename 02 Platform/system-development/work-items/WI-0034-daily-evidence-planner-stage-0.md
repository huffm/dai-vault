---
title: "WI-0034 Daily Evidence Planner Stage 0"
type: "plan"
date: "2026-07-19"
status: "in-progress"
project: "DAI"
slice: "WI-0034 Daily Evidence Planner Stage 0 (Slice 1)"
repos:
  dai: "code+tests (offline deterministic planner core; additive; branch wi/0034-daily-evidence-planner-stage-0)"
  dai-vault: "docs-only"
tags:
  - system-development
  - work-item
  - evidence-operations
  - sports-v1
related:
  - "04 Products/sports-v1/daily-evidence-acquisition-orchestrator-v1.md"
  - "02 Platform/system-development/dai-knowledge-architecture-and-writing-standard-v1.md"
  - "06 Execution/patterns/documentation-slice-impact-declaration-v1.md"
  - "02 Platform/system-development/work-items/WI-0031-model-assisted-capability-recommendation-and-tool-selection-v1.md"
---

# WI-0034 daily evidence planner stage 0

Governing parent for the Daily Evidence Planner, Stage 0 of the Daily Evidence Acquisition
Orchestrator. A niche/domain (sports evidence operations) decision system whose canonical result
is the Daily Evidence Board. This work item authorizes Slice 1 (offline deterministic core) only;
Slices 2-4 are recorded and deferred. Status stays `in-progress`.

## problem  <!-- LITE -->

Evidence-acquisition decisions (is more evidence needed; what shape; is it worth pursuing; can the
supplied inputs address it; which candidates are eligible; what pool to propose) have been
re-derived manually per session (2026-07-17 fallback scan; 2026-07-18 cohort assessment), with the
key failure modes documented live: schedule presence mistaken for cohort worthiness, "no eligible
candidates" claimed without an evaluated slate, and deficits (e.g. market disagreement) pursued
with cohorts that cannot close them. No deterministic, testable planner encodes these decisions.

## desired behavior

A pure offline planner core turns a canonical evidence-policy verdict + normalized inputs into
exactly one mutually-exclusive Daily Evidence Board outcome with stable reasons, deterministic
serialization, and zero granted authority (screening/capture/execution always false).

## affected surfaces

- dai: `services/agent-service/app/services/daily_evidence_planner.py` (new),
  `services/agent-service/tests/test_daily_evidence_planner.py` (new). Nothing else.
- vault: this WI; `04 Products/sports-v1/daily-evidence-acquisition-orchestrator-v1.md` (new
  architecture/authority record); MOC registration; Slice-1 closeout; current-slice append.

## non-goals

No CLI/process contract, artifact/file publication, live schedule or source access, PowerShell,
skills, model calls (or fake production wiring), Tool Gateway calls, `CandidateAccessFacts`,
concrete tool selection, executable recipe compilation, WI-0031 runtime integration, services,
endpoints, schedulers, persistence, schema, migrations, UI, paid screening, capture, settlement,
tuning, production policy changes, application-data writes, Azure DevOps mutation, push, or merge.

## architecture and policy-authority decision (evidence-backed)

**Language: Python, in `services/agent-service`.** The canonical evidence-sufficiency authority is
`app/services/pooled_calibration.py` (`pooled_calibration_summary` -> `conclusionsAllowed` +
`failingReasons`, criterion `discrimination_hybrid_v1`; pure, thresholds internal). The planner
**consumes that authority's verdict as normalized input** (`EvidencePolicyVerdict`) and never
recomputes sufficiency or copies thresholds -- one policy authority. Niche logic stays out of the
C# platform core (`DevCore.Domain` remains generic capability selection). Python also matches the
service's pure-module + pytest conventions and the future CLI location. No exposure refactor of
pooled_calibration was needed (its public function already returns the consumed fields).

## planner / WI-0031 separation

The planner is niche/domain; WI-0031 is the generic platform facility. Slice 1 emits
implementation-independent **missing-capability records** (capability id = required input type,
blocked objective, reasons, provenance, blocked transition; never a tool id). A later, separately
authorized WI-0031 pilot/consumer boundary may map those records to capability recommendations.
No fifth planner slice and no adapter work item are allocated now.

## decision flow (binding, implemented)

calibration verdict -> evidence need -> (objective selection; narrow operator decision when
several compete) -> cohort worthiness -> required input capabilities -> input addressability ->
slate evaluation only when inputs support it -> identity + schedule-state validation ->
eligibility -> lexicographic ranking of eligible only -> deterministic primary/reserve allocation
-> Daily Evidence Board. Schedule presence never implies need, worthiness, addressability,
eligibility, market availability, or permission.

## board outcomes (closed, mutually exclusive)

`COHORT_PROPOSED_FOR_OPERATOR_REVIEW` | `NO_ADDITIONAL_EVIDENCE_WARRANTED` |
`EVIDENCE_NEEDED_INPUT_TYPES_NOT_ADDRESSABLE` | `VALIDATED_SLATE_NO_ELIGIBLE_CANDIDATES` |
`DIAGNOSTIC_REQUIRED_BEFORE_TRUSTWORTHY_DECISION` | `NARROW_OPERATOR_DECISION_REQUIRED` (valid
only with a typed decision object). Execution/contract failures are separate structured
`PlannerError` objects (MISSING_REQUIRED_INPUT, UNSUPPORTED/POLICY_VERSION_MISMATCH,
STALE_EVIDENCE_STATE, INCONSISTENT_NORMALIZED_STATE, UNRESOLVED_IDENTITY, ...).

## invariants

Pure functions; time is an explicit input. Eligibility precedes ranking; rank cannot rescue
exclusion; unknown never favorable. `slate_evaluated == false` => slate status NOT_EVALUATED and
eligible count null (never zero). "No eligible candidates" only after an evaluated slate. Pools
disjoint, all members eligible, limits respected, allocation deterministic. Candidate-level
identity failure excludes the candidate; terminal UNRESOLVED_IDENTITY only when no trustworthy
candidate remains. `market_availability: unknown_until_paid_screening` when no market input.
Every board: screening/capture/execution/settlement/tuning/scheduling/mutation false and no
concrete tool selection. Canonical JSON byte-identical for identical inputs; Markdown is a pure
projection.

## acceptance criteria  <!-- LITE -->

- all 14 required fixtures pass (see test plan) plus the invariant tests;
- identical inputs (any candidate order) produce byte-identical canonical JSON (sha-256 equal);
- no test performs network/model/gateway/db/filesystem access;
- full agent-service suite passes with the planner added;
- vault records validate (profiles, links, MOC) and current-slice stays append-only.

## test plan (written before code)  <!-- LITE -->

Fixtures 1-14 of the authorization mapped to named tests in
`tests/test_daily_evidence_planner.py`: no-evidence-needed; needed+addressable; needed but input
types not addressable (missing-capability record, fixture 13 merged); schedule-cannot-infer-shape;
validated slate zero eligible; single invalid identity excluded; terminal unresolved identity;
market unknown; deterministic allocation/disjoint/overflow; repeated byte-identical JSON hash;
policy-version mismatch; fixture-scoped policy behavior change; no-authority ledger (fixture 14);
plus diagnostic (inversion + incomplete readout), narrow operator decision, terminal errors,
reason rendering + JSON/Markdown equivalence, rank-cannot-rescue/unknown-not-favorable.

## verification commands  <!-- LITE -->

- `./.venv/Scripts/python.exe -m pytest tests/test_daily_evidence_planner.py -q` (targeted)
- `./.venv/Scripts/python.exe -m pytest -q` (full agent-service suite)
- double-run canonical JSON sha-256 comparison (in-test)
- strict planning snapshot `-Strict` exit 0 / 0 warnings; `git diff --check`; staged allowlist;
  protected-hash comparison; machine-path/secret scans

## decomposition

- **Slice 1 (delivered + reviewed + integrated 2026-07-19): offline deterministic planner core**
  -- contracts, decision flow, board, errors, canonical JSON, Markdown projection; 25 tests after
  the independent review corrections; dai main `e3ef9a5`, vault main `6e5c99d` (ff-only, pushed).
- **Slice 2 (delivered + contract-integrity reviewed 2026-07-19): portable CLI + atomic
  artifact publication** -- `plan`/`validate`/`version` on branch
  `wi/0034-daily-evidence-planner-cli`; strict closed-schema request boundary, stable
  exit-code classes, writer-unique staged+fsync+os.replace publication with atomic
  no-overwrite admission and canonical json as the commit marker. The independent review
  reproduced and corrected an input-authority BLOCKER (asserted capability strings could
  turn a schedule-only slate into a market-objective cohort): capability availability now
  derives only from typed input-evidence envelopes; `input.market_divergence_screen`
  retired for `input.market_contrast_screen`; request/board/planner/cli contracts bumped
  to 2.0 (no silent migration). Core 35 + cli 43 tests; suite 553 green.
- **Slice 3 (deferred, not authorized): bounded free schedule adapter + optional Windows wrapper.**
- **Slice 4 (deferred, not authorized): operating/skill integration** -- only after a stable CLI
  and one clean manual operator use.
Each slice records objective/inputs/outputs/authority/dependencies/invariants/verification/
completion in this WI at its authorization time; independently reviewable (core vs interfaces vs
adapters vs operating integration). No file-layer splitting.

## risks

Domain-mapping drift (failing-reason -> objective table must follow the canonical criterion's
vocabulary; covered by POLICY_VERSION_MISMATCH fail-closed); outcome overloading (guarded by the
closed six-outcome contract and tests); future CLI temptation to embed policy (bounded by Slice-2
boundary).

## links  <!-- LITE -->

- work item: WI-0034 (ADO: AB#- when wired; no ADO item created)
- branch: `wi/0034-daily-evidence-planner-stage-0` (dai + dai-vault, matching, from ac634b5 /
  e5d90e9; reviewed, pushed, ff-integrated 2026-07-19 -> dai main `e3ef9a5`, vault main `6e5c99d`);
  `wi/0034-daily-evidence-planner-cli` (dai + dai-vault, matching, from e3ef9a5 / 6e5c99d;
  local only, not pushed / not merged)
- pr: - (Slice 2 not pushed / not merged this slice)
- commits: recorded in the Slice-1 closeout at close
- tests: `services/agent-service/tests/test_daily_evidence_planner.py` (35 after the
  slice-2 contract review); `services/agent-service/tests/test_daily_evidence_planner_cli.py`
  (43, Slice 2 + review)
- verification notes: Slice-1 closeout (incl. independent review corrections section) +
  current-slice handoff
- docs updated: this WI; orchestrator architecture record; MOC; closeout; current-slice
- producer dependency: `input.market_contrast_screen` evidence is produced by WI-0035
  (market-contrast candidate screen); planner contracts moved to 2.1 with envelope 1.1
  (tier + priority facts) in the WI-0035 Slice-1 change
- wildcard seam (2026-07-21, docs-only link): WI-0036 defines a bounded wildcard candidate
  lane and flight-plan contract as target architecture
  ([[WI-0036-wildcard-evidence-discovery-loop-v1]]). Nothing in this planner implements a
  wildcard lane today; whether the WI-0036 Slice-2 contract is WI-0036-owned and consumed
  by this planner (vs changing WI-0034 ownership) is resolved at that slice's
  authorization, not here.
  RESOLVED 2026-07-21 (WI-0036 Slice 2 implementation): the flight-plan contract is
  WI-0036-OWNED (`app/services/wildcard_flight_plan.py`) and CONSUMES this planner's
  canonical board (`daily-evidence-board/2.2`, strictly validated, embedded) as upstream
  input. This planner's board/request/planner/cli versions and primary/reserve semantics
  are UNCHANGED; ownership did not transfer; the market-screen tier was not overloaded as
  a flight lane. Boundary tightened by the same-day pre-integration correction (finding
  G): the WI-0036 CLI now REPRODUCES the board by parsing the upstream wi-0034 planner
  request with THIS planner's real parser and re-running THIS planner's real core; a
  claimed board must be byte-identical to that reproduction, so a caller-edited board
  interior can never pass as producer-certified. Still consumption-only; nothing in this
  planner changed
- lessons: consume the canonical policy verdict instead of recomputing it; encode the
  slate-not-evaluated vs zero-eligible distinction in the type contract; independent review
  corrections (2026-07-19): unrecognized failing-reason codes route to a diagnostic (never a
  generic objective); identity is provider-scoped and an all-broken-identity slate is
  terminal; missing start time / team identity excludes (unknown never ranks favorably);
  negative pool limits fail validation; operator actions are outcome-specific and
  authority-free

## final handoff requirements

Slice handoff appended to `06 Execution/handoffs/current-slice.md`; status stays `in-progress`
(Slices 2-4 deferred); disposition: Slice 1 implementation complete / merge ready / not
integrated; next governed action = independent review + integration of the local branches.
