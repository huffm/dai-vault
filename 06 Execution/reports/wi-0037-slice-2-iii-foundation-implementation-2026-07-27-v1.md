---
title: "WI-0037 Slice 2-iii Foundation Implementation: Canonical Matcher + Inert Provenance 2026-07-27 v1"
type: "evidence-report"
date: "2026-07-27"
status: "complete (implementation) -- independent review pending"
project: "DAI"
slice: "WI-0037 Slice 2-iii foundation batch: 2-iii-a matcher + 2-iii-b1 inert provenance"
repos:
  dai: "code (local branch wi/0037-selected-event-foundation, af59853 -> 19fbc77 -> 63c7009; NOT integrated, NOT pushed)"
  dai-vault: "docs-only"
tags:
  - system-development
  - sports-v1
  - identity
  - persistence
related:
  - "02 Platform/system-development/work-items/WI-0037-game-state-correctness-v1.md"
  - "06 Execution/reports/wi-0037-slice-2-iii-architecture-review-2026-07-26-v1.md"
---

# wi-0037 slice 2-iii foundation implementation: canonical matcher + inert provenance

## purpose

Implement the two foundation slices of the published Slice 2-iii architecture as one
continuous batch with two independently reviewable source commits: 2-iii-a (the
sports-domain-owned canonical provider-event game-binding matcher) and 2-iii-b1 (inert
generic domain-execution provenance persistence). No activation, no public contract
change, no migration execution. Implementation is local only and awaits an independent
review-and-integrate-on-PASS authorization.

## context

Authorized by the operator foundation-batch prompt of 2026-07-27 against published
architecture (vault main 0e191e0, reviewed package tip d78545c) and published source
main af59853. Prompt-ledger pre-execution record:
`<OBSIDIAN_PROMPT_LEDGER_ROOT>/dai/prompts/2026/07/2026-07-27-wi-0037-slice-2-iii-foundation-batch.md`.

## 1. phase a -- 2-iii-a canonical matcher (commit 19fbc77)

**Commit A:** dai `19fbc77c4db9884609ae752041f4a11d99d3dc85`
"refactor(sports): centralize provider event game matching" (3 files).

**Purpose.** Ownership correction, not a new algorithm: the existing WI-0035 binding
rule family moved from the agent-run workflow layer into the sports domain as the one
canonical matching authority both existing consumers and the future selected-event
translation consume.

**Design.**
- NEW `platform/dotnet/DevCore.Api/Sports/ProviderEventQualifier.cs` (namespace
  `DevCore.Api.Sports`) now owns the complete rule family: `ProviderEventBindingPolicy`
  (schema/policy versions, inclusive +/-60s window), `ProviderEventMatchMethod`,
  `ProviderEventQualificationStatus`, `ProviderEventCandidate`, `ProviderEventBracket`,
  `ProviderEventQualification`, and the `ProviderEventQualifier.Qualify` decision.
- Single-definition predicates added to the policy and consumed by BOTH the decision
  and the wire content contract: `IsWithinAdmittedWindow`, `MatchMethodFor`,
  `IsWholeSecondInstant`, `SignedStartDeltaSeconds`, plus `ProviderEventBracket.Contains`
  (half-open membership). The wire validator's private copies were removed.
- `AgentRuns/ProviderEventBinding.cs` keeps ONLY the frozen binding evidence record,
  canonical wire emission, the strict content-integrity validator, the verification
  vocabulary, and the integrity exception. The canonical json emitters and fingerprint
  logic have ZERO changed lines in Commit A (wire byte-compatibility preserved by
  construction).
- No forwarding shim was needed: both consumers already call the single authority and
  resolve it via existing `using` directives.

**Naming decision (explicit).** The published ADR names the architectural role
"ProviderEventGameBindingMatcher". The established production type realizing that role
is `ProviderEventQualifier`, whose qualifier/qualification vocabulary is pinned by the
frozen binding wire (`qualification status` strings) and the existing test surface. The
batch's shared rules forbid renaming established production concepts solely to match
ADR prose, so the production name is retained and the file lives at
`Sports/ProviderEventQualifier.cs`; the role-to-type mapping is recorded here and in
the WI (semantic disposition below).

**Tests (RED-first).** New
`DevCore.Api.Tests/Sports/ProviderEventQualifierOwnershipTests.cs` (26 test cases):
sports-namespace ownership pins for all seven rule types (the meaningful RED -- 7
failures at base because the family lived in `DevCore.Api.AgentRuns`), matcher
purity/statelessness, deterministic policy versions, decision-vs-wire window agreement
at -61/-60/0/+60/+61, empty-set refusal, and the shared-predicate matrix (inclusive
bounds, exact-only-at-zero method, whole-second rule, signed-delta definition,
half-open bracket). The behavior matrix (orientation, brackets, identity, ambiguity,
accounting, doubleheaders) remains pinned by the existing provider-event suites and
was NOT duplicated.

**Behavior preservation proof.** RED = exactly the 7 ownership failures; every
pre-existing suite green after extraction: provider-event + market-join family 240/240,
adjacent consumers (MarketContrast*, SportsRetriever, PlannerEnvelope) 210/210, full
suite 1896/1896. Repository search proves single definitions: one
`MaxAbsStartDeltaSeconds`, one whole-second rule, one `ProviderEventQualifier`, no
inline window comparison left in production.

## 2. phase b -- 2-iii-b1 inert provenance (commit 63c7009)

**Commit B:** dai `63c70099a095383d96df9cbbbf3d6632ba145ed8`
"feat(agent-runs): add inert domain provenance storage" (6 files).

**Purpose.** The published architecture's generic opaque domain-provenance persistence,
added inert: storage exists, nothing writes it, nothing reads it in production, no
request path changes.

**Schema.** `AgentRun.DomainExecutionProvenanceJson`: nullable string, private setter,
convention-mapped exactly like the `PromptRouteProvenanceJson` precedent (nullable
`nvarchar(max)`, no default, no backfill, no index -- no read path exists yet to
justify one). Single-assignment seam `AssignDomainExecutionProvenance(string)`: throws
deterministically on a second assignment leaving the original bytes untouched, refuses
blank input, stores the supplied document byte-verbatim; EF materializes through the
private setter and ordinary updates to other members never rewrite the field. No
merge/append/replacement API exists.

**Migration proof.** `20260727133845_AddAgentRunDomainExecutionProvenance`: Up = exactly
one `AddColumn<string>` (`AgentRuns.DomainExecutionProvenanceJson`, `nvarchar(max)`,
nullable, no default); Down = exactly one `DropColumn`. Snapshot delta = 3 lines (the
one property). `dotnet ef migrations has-pending-model-changes` -> "No changes have
been made to the model since the last migration." Generated and inspected only; NOT
applied; no database contacted (design-time tooling only).

**Generic envelope.** NEW `DevCore.Domain/Agentic/DomainExecutionProvenanceEnvelope.cs`:
the four-member shape `{domain, type, schemaVersion, payload}` with a strict structural
`TryRead` (exactly four members; three non-empty string metadata; payload any json
value; precise errors) and deterministic camel-case `ToJson` that writes the payload
through verbatim. No domain/type/schema value is recognized, preferred, or rejected;
`selected_event_binding` is never referenced by platform code. No platform
envelopeVersion in v1.

**Inertness proof.** Repository search: no production caller invokes
`AssignDomainExecutionProvenance` or reads the column (sole references = the entity
itself, tests, migration artifacts); no `SelectedEventIntent` anywhere; no
controller/DTO/request/serialization file in the diff; combined delta touches only
`platform/dotnet` (9 files, zero frontend paths).

**Tests (RED-first).** New
`DevCore.Api.Tests/AgentRuns/DomainExecutionProvenanceTests.cs` (17 test cases). RED =
compile-absence of the new persistence surface (captured before implementation). Pins:
null-by-default + single assignment; deterministic second-assignment refusal preserving
original bytes; blank refusal; historical rows materialize null; round-trip survives
unrelated updates; EF metadata (nullable, unbounded, string, no default); the generated
migration's exact Up/Down operations (asserted from the migration class's own
operations); envelope round-trip without payload interpretation; foreign
domain/type/schema opacity; structural envelope failures.

## 3. platform/domain ownership

The generic platform owns storage, retrieval, tenant/run association, the generic
envelope shape, atomic persistence capability, zero-or-one cardinality, and single
assignment -- nothing else. Sports classification (the part-8 four-arm matrix) is NOT
implemented in this batch and remains sports-owned future work (2-iii-b2). One
deliberate, documented dependency: the sports matcher file constructs the frozen
binding evidence record that lives with its wire in the agent-runs boundary
(`using DevCore.Api.AgentRuns` on a pure data record); it depends on none of the
forbidden surfaces (events gate, workflow state, filesystem publication, capture
state, model state).

## 4. rollback and compatibility

Old application code tolerates the additive nullable column (never selected); new code
tolerates historical NULL rows (pinned by test). Application rollback = retain the
inert column and its contents. Running the migration Down after provenance is
populated DROPS the column and its data -- no data-preserving Down is claimed. b1
changes no accepted input, response shape, duplicate behavior, execution behavior,
model path, settlement path, or frontend behavior.

## 5. current limitations and explicitly unauthorized behavior

- No production writer or reader of the provenance column exists; activation,
  the shared creation gate, selected-candidate classification, and duplicate-guard
  consumption are 2-iii-b2 (unauthorized).
- No public SelectedEventIntent contract; no controller/request change (2-iii-b2/c).
- No frontend propagation (2-iii-c, unauthorized).
- 2-ii-c obligations untouched.
- The migration is unapplied everywhere; no database was accessed.
- Implementation is NOT integrated, NOT pushed, and NOT claimed integration-ready
  until the independent review passes.

## 6. review findings and dispositions

`dai-code-reviewer` run over Commit A, Commit B, and the combined delta. Blocking:
none. Non-blocking (1): the Sports matcher header comment says the rule family was
"moved verbatim" while four shared predicates were in fact extracted and consumed by
both authorities -- comment precision only, zero behavior claim; recorded here for the
independent review's correction pass rather than breaking the authorized
exactly-two-commit batch shape. Naming, comments (lowercase ascii), and test
discipline otherwise conform.

## 7. evidence -- live verification commands and counts

- RED A: focused ownership run -- 7 failed / 8 passed (namespace pins failing at base).
- GREEN A: provider-event + market-join filter 240/240; adjacent consumers 210/210.
- RED B: compile-absence of the b1 surface (CS1061 on the new members), captured.
- GREEN B: `FullyQualifiedName~DomainExecutionProvenanceTests` 17/17.
- Cumulative: full `dotnet test DevCore.Api.Tests` **1896/1896, 0 skipped**
  (published baseline 1853 + 43 new); full solution `dotnet build` 0 errors (6
  pre-existing NU1903 package advisories, untouched -- no dependency change).
- `dotnet ef migrations has-pending-model-changes`: no pending changes.
- `git diff --check` clean per phase; combined delta = 9 files, all `platform/dotnet`.
- Frontend: NOT run -- proven not applicable (zero frontend/shared-contract paths in
  the combined delta).

## safety / non-actions

No push; no remote branch; no integration; no migration executed; no database access;
no live Odds API/StatsAPI/model/paid/capture/reconciliation/settlement call; no
dependency change; no frontend change; no DuplicateRunGuard change; preserved WI-0035
worktree untouched; ops branch untouched.

## 8. semantic dispositions

- **domain execution provenance** (generic opaque envelope + single-assignment column):
  PROMOTED to the operator-owned dictionary `01 Operating System/glossary.md` (one new
  entry, this slice).
- **provider-event game-binding matcher** (ADR role) vs **ProviderEventQualifier**
  (production type): DISTINGUISHED -- the ADR role name maps to the retained
  established production type; recorded in WI-0037 and this report; no production
  rename.
- shared predicate names (`IsWithinAdmittedWindow`, `MatchMethodFor`,
  `IsWholeSecondInstant`, `SignedStartDeltaSeconds`, `Contains`): RETAINED WI-local
  (implementation vocabulary, not product doctrine).
- `DomainExecutionProvenanceEnvelope` members (`domain`, `type`, `schemaVersion`,
  `payload`): MERGED into the promoted glossary entry.

## 9. residual states

Unchanged by this batch, exactly as published:
- SELECTED_EVENT_IDENTITY_PROPAGATION_REQUIRED_BEFORE_WI0037_CLOSE -- OPEN/blocking
  (this foundation batch does NOT resolve it; propagation needs b2 + c).
- MULTI_INSTANCE_SELECTED_EVENT_ATOMICITY_REQUIRED_BEFORE_SCALE_OUT -- blocking before
  any scale-out authorization.
- DURABLE_PREEXECUTION_SELECTION_DECISION_LEDGER_DEFERRED -- deferred/nonblocking.

## next step

Exactly one: operator issues the independent review-and-integrate-on-PASS
authorization for the foundation batch (attack Commit A, Commit B, and the combined
delta; integrate and publish only on PASS; 2-iii-b2 stays unauthorized and is the next
separately authorized implementation batch).

## related

- [[WI-0037-game-state-correctness-v1]] -- governing work item.
- [[wi-0037-slice-2-iii-architecture-review-2026-07-26-v1]] -- the published
  architecture this batch implements.
