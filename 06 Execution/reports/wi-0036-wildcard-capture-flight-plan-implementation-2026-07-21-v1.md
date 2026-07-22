---
title: "WI-0036 Wildcard Capture Flight Plan Implementation v1"
type: "evidence-report"
date: "2026-07-21"
status: "complete (implementation on local branches; review + integration pending)"
project: "DAI"
slice: "WI-0036 Slice 2 + minimum Slice-3 provenance seam"
repos:
  dai: "code+tests (offline deterministic flight-plan core/CLI + additive provenance seam; branch wi/0036-wildcard-capture-flight-plan; local only)"
  dai-vault: "docs (WI, architecture, orchestrator, cohort doctrine, WI-0034 seam, MOC, timeline, this report, current-slice; branch wi/0036-wildcard-capture-flight-plan; local only)"
tags:
  - system-development
  - evidence-operations
  - cognitive-factory
  - implementation
related:
  - "02 Platform/system-development/work-items/WI-0036-wildcard-evidence-discovery-loop-v1.md"
  - "02 Platform/architecture/cognitive-factory/wildcard-evidence-discovery-loop-v1.md"
  - "04 Products/sports-v1/daily-evidence-acquisition-orchestrator-v1.md"
  - "06 Execution/reports/wi-0036-wildcard-evidence-discovery-loop-planning-2026-07-21-v1.md"
---

# wi-0036 wildcard capture flight plan implementation v1

## purpose

Closeout for the operator-authorized WI-0036 Slice 2 vertical: the smallest complete
implementation that lets a FUTURE separately authorized paid flight include wildcard
captures safely and preserve their identity through the production artifact. "Allow
wildcard capture" means the software can deterministically plan, freeze, export, and
carry provenance -- it does NOT authorize capture or spend; every ledger is all-false and
the path is default off.

## context

Operator sequencing override of 2026-07-21 (after WI-0036 Slice 1 integration at vault
main `b04e6421...`): proceed with the offline/default-off implementation now, superseding
only the earlier July-22 gating of implementation. Opening state verified live: dai main
`8369d64a...` == origin, vault main `b04e6421...` == origin, protected sets byte-hashed.
Coordinated local branches `wi/0036-wildcard-capture-flight-plan` created in both repos.

## scope

Delivered: (1) WI-0036-owned deterministic wildcard flight-plan core + portable CLI in
`services/agent-service`; (2) the minimum Slice-3 provenance seam in the platform.
Excluded (verified non-actions below): the July 22 observation, any paid/source/model/DB
call, any real AgentRun/capture, scheduling, endpoints, migrations, SignalNeedProposal,
the refresh loop, callable services, dormant-chain activation, recapture, promotion,
buyer/commercial changes.

## gap audit findings and ownership decision

- WI-0034 owns `daily-evidence-board/2.2` / planner 2.2 / request 2.1 / cli 2.5 with
  primary/reserve pools only -- confirmed in source; candidate records conflate objective
  eligibility with allocation; no wildcard lane, no coverage/novelty concept.
- No canonical signal-combination or novelty-dimension registry exists anywhere in the
  platform (repo-wide audit) -- the WI-0036-owned closed registries
  `wi0036-signal-combinations/1.0` (combo.starter_only / combo.starter_market /
  combo.starter_market_movement / combo.market_only; grounded in the canonical
  source-signal vocabulary) and `wi0036-novelty-dimensions/1.0` (regime / recipe_version /
  signal_combination) are net-new; request vocabularies may only narrow them.
- Recipe/version/regime authority: the prompting manifest
  (`app/prompting/templates/manifest.json`) + `app/prompting/dataregime.py`. The core
  stays pure, so the recognized vocabulary is an EXPLICIT request input validated against
  the wi-0036 registries; nothing is discovered ambiently.
- Production run path already carries exact game identity (WI-0009 GamePk, null-suppressed)
  and observed prompt-route provenance (`PromptRouteProvenance`, run-row column), but no
  frozen selection lane/role/hypothesis/coverage snapshot -- the seam this slice adds.
- **Ownership decision:** the flight-plan contract is WI-0036-OWNED and CONSUMES the
  unchanged WI-0034 board as upstream input (embedded, strictly validated: closed keys,
  cohort-proposed outcome, all-false ledger, matching target date). WI-0034 versions and
  primary/reserve semantics untouched; the market-screen tier is carried as a separate
  fact and never overloaded as a lane.

## delivered contracts and versions

- request `wildcard-flight-plan-request/1.0`; plan `wildcard-flight-plan/1.0`; planner
  `wildcard-flight-planner/1.0`; realization `wildcard-flight-realization/1.0`;
  provenance `flight-selection-provenance/1.0`; lane vocabulary
  `wi0036-candidate-lane/1.0`; registries `wi0036-signal-combinations/1.0` +
  `wi0036-novelty-dimensions/1.0`; CLI `wildcard-flight-plan-cli/1.0`.
- Formulas implemented exactly: `wildcard_scheduled_max = floor(total_scheduled_runs/4)`;
  `minimum_executed_core_runs = 1`; `wildcard_substitution_reserve_max =
  max(0, scheduled_core_runs - 1)`; closed `wildcard_plan_role = scheduled |
  substitution_reserve`; reserve-first precedence (core-qualified reserve -> frozen
  wildcard substitution reserve by strongest novelty over eligible wildcards only ->
  fail-closed non-execution); one-for-one substitution never raising scheduled count or
  paid ceiling; zero-core realization = hard-stop error.
- Strongest-novelty ordering (closed lexicographic): settled count for the exact
  recipe/version/regime triple asc; min settled count across recognized signal
  combinations asc; distinct recognized novelty dimensions desc; scheduled start UTC asc;
  provider-scoped identity asc. Free text never ranks; captured-but-unsettled never
  counts as settled.
- Freeze: canonical json (sorted keys, ascii, single line) with a sha-256
  `freeze_fingerprint` over the fingerprint-nulled plan; realization and provenance
  export verify it and reject any tamper (`PLAN_INTEGRITY_VIOLATION`).
- Qualification gates (all fail-closed): board-ledger-derived safety (identity valid,
  pregame, no safety-class exclusion; ONLY market-absence codes tolerated and only when
  the hypothesis expects a recognized market-missing regime -- market-missing reachable,
  unknown never favorable); measurable settled-coverage gap vs the explicit
  `underrepresentation_max_settled`; written frozen hypothesis with recognized
  recipe/version/regime/combinations/dimensions; historical recapture rejected by
  default with no override field; `wildcard_mode` defaults disabled and no mode grants
  capture authority.
- CLI operations: `plan`, `realize`, `export-provenance`, `validate`
  (request/plan/realization), `version` -- strict closed-key json boundary and the
  race-safe atomic no-overwrite publication REUSED by import from the wi-0034 planner cli
  (single authority, not copied). Exit-code classes reused (3 bad input / 4 stale /
  5 version / 7 inconsistent-or-integrity / 8 publication).

## the minimum provenance seam (platform)

`FlightSelectionProvenance` (`DevCore.Api/AgentRuns/FlightSelectionProvenance.cs`):
additive default-null null-suppressed `CompetitionMatchupInput.FlightSelection` (the
WI-0009 GamePk pattern) -> fail-closed trust-boundary validation in the create action
BEFORE the run row is written (schema, closed lanes/roles, required identity fields,
target-date == GameDate, gamePk identity consistency, mandatory all-false authority
attestation) -> carried on `SportsRunArtifact.Input` -> copied into
`AgentRunExecutionResult.FlightSelection` in BOTH `Compose` and `ComposeFailedRun` (the
SourceDepth pattern) -> persisted via existing InputJson/OutputJson (no migration, no
column, no endpoint) -> projected read-only on `AgentRunArtifactDto.FlightSelection`
beside the observed `PromptRouteProvenance` (expected vs actual at the internal
inspection boundary) -> excluded from every buyer surface (buyer-projection sentinel
test). Compatibility pinned by tests: legacy InputJson byte-identity when absent; old
rows deserialize null; FastAPI request bytes provably unchanged (the analyzer hand-copies
only competition/home/away/gameDate; structural + wire assertions); presence changes no
retrieval/model-call/decision/settlement behavior. The deterministic mapping from a
frozen plan candidate to this block is the CLI `export-provenance` operation -- a future
paid-slice prompt never hand-composes it.

## verification

- RED evidence: pytest collection error `ModuleNotFoundError: app.services.wildcard_flight_plan`
  (then `_cli`) against the integrated baseline; C# build CS0246 `FlightSelectionProvenance`
  not found (4 sites) before implementation.
- GREEN: new suites 56 (core) + 14 (cli) + 12 (xunit seam incl. buyer sentinel) = 82 new
  tests. Full gates: agent-service pytest **634/634**; DevCore.Api.Tests **1516/1516**
  (baseline 1504); targeted composer/gamepk/buyer suites green; zero warnings referencing
  changed files.
- Desk-scenario fixtures executable: C1-C8 exactly as written in the Slice-1 correction
  addendum, plus cap arithmetic (1,3,4,7,8 -> 0,0,1,1,2), tie-breakers individually and
  permutation invariance (byte-identical plans), settled-vs-unsettled separation,
  market-missing reachability, safety gates (rank never rescues), default-disabled,
  recapture rejection, malformed/unknown/duplicate/version/stale fail-closed, canonical
  json + fingerprint byte determinism, authority-tamper rejection.
- Strict planning snapshot (session scratch, outside both repos): exit 0, 0 warnings,
  25 work items, 6 timeline entries, authorization posture unchanged (no-spend,
  fail-closed).
- Hygiene: `git diff --check` clean in both repos; scans over added lines clean (no
  machine paths, secrets, authority grants, or live-source invocations; the only
  network-adjacent term is in a comment listing what the cli does NOT do); no-network
  proof: the new python modules import only json/hashlib/dataclasses/argparse/os/sys +
  in-repo modules -- no socket/http/requests/urllib.
- Protected state byte-identical open -> close (dai csproj 63EF2488...; vault
  graph/CLAUDE/manifest/synopsis hashes unchanged; Welcome.md still deleted).

## safety / non-actions

External calls: model 0; StatsAPI 0; Odds /events 0; Odds /odds 0; database 0; Tool
Gateway 0; generation/AgentRun/capture/screening/settlement/reconciliation/scheduling 0;
services started 0; paid cost $0. No endpoint, migration, schema, flag activation, prompt
recipe/routing change, second model call, or confidence/posture/lean/decision/evaluator/
buyer/pricing/Stripe/entitlement/tenancy change. `ProbeRequest`, station permissions, and
Tool Gateway permissions unwidened; the dormant probe-refresh chain untouched. The July 22
observation neither executed nor re-authorized. Wildcard capture is NOT active, NOT
scheduled, NOT authorized, NOT production enabled: implementation present on local review
branches, default off/no authority, available to a future separately authorized
paid-flight prompt only after independent review and integration.

## pre-integration correction addendum (2026-07-21, same day -- correction commits on top)

The completion claim above described the FIRST implementation pair (dai `e64baab`, vault
`8d98da9`). An independent Codex review then found that pair NOT integration-ready; this
addendum records the correction honestly rather than rewriting the original claim.

Findings (all reproduced RED-first against `e64baab` before any fix; probe evidence:
snake_case export keys with no camel `flightId` and a summary authority boolean; core
reserve lane `core`; `fake.recipe@v999` scheduled from caller-authored vocabulary; a
ledger-tampered rehashed plan realized and its output ledger laundered all-false; a
forged `forged/ghost` substitution exported; C# authority acquired by omission via the
`AuthorityLedgerAllFalse = true` default; a caller-edited board interior accepted):

- **A** python export was not the .NET wire contract -> ONE canonical camelCase wire
  shape (exact eight-key all-false `authorityLedger`; provider-scoped `substitutedFor`;
  `flightPlanSchemaVersion`/`laneVocabularyVersion`/`realizedVia` carried), pinned by an
  update-together cross-runtime vector embedded verbatim in BOTH suites and regenerated
  byte-for-byte from the real exporter on the python side.
- **B** reserve lane collapsed -> selection lane `reserve` preserved in plan, realization,
  and provenance; `realized_via` records the filled slot; the core minimum counts a
  realized reserve-via-core without relabeling.
- **C** caller-invented recognition -> the CLI derives the recognized registry from the
  canonical prompt manifest (hash-verified; taxonomy pinned as
  `prompt-manifest/2+sha256:<digest>`); caller vocabulary is narrowing-only; invented
  recipe/version/regime/market-missing/taxonomy values fail closed; happy-path fixtures
  now use the real manifest recipes (e.g.
  `mlb.pregame.analysis.starter_enriched_market_missing.v1 @ v1`).
- **D** fingerprint-laundered authority -> `validate_flight_plan` checks the full closed
  contract (ledger, formulas, lanes, identities, positions, references) INDEPENDENT of
  the fingerprint; the sha-256 is documented as a content fingerprint, never
  authenticity/authority; a rehashed tampered plan still fails.
- **E** trusted forged realization -> `export_run_provenance` re-runs deterministic
  realization internally from plan + availability (the CLI export op takes
  `--availability`; a realization artifact is no longer an input); a strict
  `validate_realization` recomputes membership/substitution graph/counts/share and adds a
  realization content fingerprint for the validate op.
- **F** permissive .NET boundary -> `FlightSelectionProvenance` requires the exact ledger
  (omission/extra/true/missing keys -> 400), validates versions, 64-lowercase-hex
  fingerprint, explicit-UTC timestamp, positive positions, non-negative counts,
  conditional wildcard fields, registered combination/dimension values, and
  lane/role/realizedVia/substitution consistency; a controller-host test posts the exact
  python wire body (200 + exact provenance at the stub service) and seven malformed
  variants (400 before the stub and before any run row).
- **G** caller-edited board -> the flight request supplies the strict wi-0034 planner
  request; the CLI re-runs the REAL wi-0034 planner and requires any claimed board to be
  byte-identical (`BOARD_NOT_REPRODUCIBLE` otherwise); the mutated-interior fixture is a
  permanent regression.

Corrected verification: RED evidence recorded per finding; suites after correction --
agent-service pytest **653/653** (wi-0036: 66 core + 23 cli), DevCore.Api.Tests
**1528/1528** (wi-0036 seam: 13 unit + 9 controller-host); zero warnings from changed
files; strict planning snapshot re-run 25 work items / 6 timeline entries / 0 warnings;
diff --check + machine-path/secret/authority scans clean; protected state byte-identical.
External calls/spend remain zero (the manifest load is a packaged repo file, hash-verified,
no network).

## semantic-integrity correction addendum (2026-07-21, same day -- third commits on top)

A second independent adversarial review of the complete two-commit chains found the A-G
correction still not integration-ready: five semantic gaps (findings H-L), all reproduced
RED-first against dai `b0ff396` (probe evidence: reserve substitution exported
`substitutionEligible:false`; a rehashed fake recipe/version/regime validated AND
exported; blanked hypothesis + invented combos/dims + string rank validated; forged
producer references validated; a partition-hole realization covering 3 of 4 slots
validated; a forged unfilled row validated; a hand-authored wildcard substitution over an
available core reserve validated):

- **H** substitution eligibility now follows the frozen-lane truth table in python AND
  .NET: core -> false; reserve -> TRUE; wildcard scheduled -> false; wildcard
  substitution_reserve -> true. Eligibility is a frozen plan fact -- never proof of
  substitution (realizedVia/substitutedFor carry that) and never authorization. A second
  update-together reserve-substitution cross-runtime vector is embedded verbatim in both
  suites and proven end-to-end through the controller host (lane stays reserve,
  realizedVia core_reserve, eligibility true, substitutedFor intact, buyer boundary
  untouched).
- **I** plan validation became SEMANTIC: recognized recipe/version/regime against the
  trusted context; non-blank hypothesis id/text/selection reason; registered,
  duplicate-free, sorted combinations/dimensions; frozen counts within the recorded
  underrepresentation threshold; the novelty rank exact-typed and EQUAL to the
  deterministic formula from the entry's own facts; strongest-novelty ordering enforced
  across scheduled wildcards and the substitution reserve (scheduled precedes reserve).
- **J** producer references are verified against an explicit `TrustedValidationContext`
  (the real wi-0034 core's board schema/planner version constants + the hash-verified
  manifest-derived vocabulary, constructed only by the cli's `build_trusted_context`).
  Every serialized-plan consumer (validate/realize/export) REQUIRES the context; no
  context-free validation path remains; shape/64-hex syntax is documented as never being
  producer verification.
- **K/L** FULL realization validity is availability-derived: a candidate realization is
  valid iff canonically byte-identical to `realize_flight(plan, availability, context)`
  -- the only rule that proves the complete scheduled-slot partition, exact unfilled
  identities/reasons bound to availability, that unavailable members never realize, and
  reserve-first precedence with the deterministic frozen substitute order. The cli
  `validate --realization` requires `--plan` AND `--availability`; omission is a
  fail-closed usage error, never a full-valid result. Membership-plus-graph
  self-consistency is explicitly documented as insufficient.
- Malformed nested plan/realization content now returns the closed structured error
  contract (never an unhandled KeyError/TypeError/comparison exception).
- Narrowed claim recorded honestly: frozen coverage counts are validated for internal
  consistency, threshold adherence, and rank equality; their equality with the external
  coverage snapshot was enforced at plan time and is bound by the content fingerprint
  (the coverage reference carries no digest) -- not re-proven at validation time.

Verification after this correction: RED probe evidence per finding; agent-service pytest
**664/664** (wi-0036: 76 core + 25 cli); DevCore.Api.Tests **1530/1530** (seam 14 unit +
10 controller-host); zero warnings from changed files; strict snapshot 25 work items /
6 timeline entries / 0 warnings; diff --check + scans clean; protected state
byte-identical; zero external calls; $0.

## next step

Independent review of the COMPLETE three-commit chains in both repos
(`wi/0036-wildcard-capture-flight-plan`: base implementation + A-G correction + H-L
semantic-integrity correction), then coordinated integration under a later explicit
prompt -- NOT the July 22 observation and NOT a paid wildcard run. A recommendation is
not an authorization.

## related

- [[WI-0036-wildcard-evidence-discovery-loop-v1]]
- [[wildcard-evidence-discovery-loop-v1]]
- [[daily-evidence-acquisition-orchestrator-v1]]
- [[cohort-selection-and-run-discipline-v1]]
- [[WI-0034-daily-evidence-planner-stage-0]]
