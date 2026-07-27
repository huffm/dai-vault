---
title: "WI-0037 Slice 2-iii-b2 Backend Activation: Verified Selected-Event Creation 2026-07-27 v1"
type: "evidence-report"
date: "2026-07-27"
status: "complete (implementation) -- independent review pending"
project: "DAI"
slice: "WI-0037 Slice 2-iii-b2: verified selected-event backend activation"
repos:
  dai: "code (local branch wi/0037-selected-event-backend-activation, 1311137 -> 9f12d2d -> b4734aa; NOT integrated, NOT pushed)"
  dai-vault: "docs-only"
tags:
  - system-development
  - sports-v1
  - identity
  - persistence
related:
  - "02 Platform/system-development/work-items/WI-0037-game-state-correctness-v1.md"
  - "06 Execution/reports/wi-0037-slice-2-iii-architecture-review-2026-07-26-v1.md"
  - "06 Execution/reports/wi-0037-slice-2-iii-foundation-implementation-2026-07-27-v1.md"
  - "06 Execution/patterns/selected-event-activation-evidence-v1.md"
---

# wi-0037 slice 2-iii-b2 backend activation: verified selected-event creation

## purpose

Implement the complete fail-closed selected-event backend path bound by the published
Part-8 architecture, as one atomic two-commit batch: Commit A = the internal
selected-event authority (server-owned provider observation, canonical matcher
translation, staged statsapi verification, immutable verified resolution, validated
sports provenance builder); Commit B = atomic request activation (public intent
contract, default-off activation gate, gate-1 placement, shared creation gate,
four-arm candidate classification, cross-path duplicate identity, atomic persistence,
gate-2 compare-not-replace, refusal matrix, concurrency proofs). Production activation
is DEFAULT-OFF; neither commit may integrate independently.

## user and system goal

The operator's selected provider event -- the thing they clicked -- and the verified
gamePk, persisted game identity, model execution identity, reconciliation identity, and
settlement identity always refer to the same physical game. Client data is intent or
cross-check input only; no client field authorizes execution.

## 1. commit a -- internal selected-event authority

**Commit A:** dai `9f12d2dce012bc5dc44ffa1c7751372a0e527954`
"feat(sports): add verified selected event resolution" (5 files, 980/14).

- **Provider observation** (`OddsScheduleClient.ObserveSelectedEventAsync`): namespace
  and sport key derive from the server-owned CompetitionCatalog registration, never the
  client; the FULL eastern-day-bracket observation is fetched (no client team filters);
  the selected id is opaque -- exact ordinal match, never case-folded, blank/oversized
  rejected, bounded-safe logging; outcomes distinguish event_missing,
  response_malformed, identity_conflict (integrity-excluded id), and source_failure.
  The existing normalization pipeline was refactored into one shared typed core
  (`NormalizeCore`) so the observation seam and the discovery projection consume the
  SAME identity/integrity rules -- no second copy.
- **Canonical translation** (`SelectedEventResolutionService.ResolveAsync`): for every
  describable in-bracket statsapi game, `ProviderEventQualifier` (the one authority)
  decides unique binding to the selected event; zero binders refuse identity-mismatch
  (or ambiguity when the event is admissible elsewhere); multiple binders refuse
  ambiguity. Depends only on the matcher, the binding evidence, and sports-owned typed
  inputs -- no gate, adapter, filesystem, capture, model, or run dependency (pinned by
  reflection + namespace tests).
- **Staged verification**: `MlbStarterClient.FetchVerificationScheduleAsync` (uncached,
  evidence-grade) + `GameStatusResolver.Resolve` (explicit bracket, exact pk,
  uniqueness, required status, reschedule context retained). Client GamePk is
  cross-check only -> `selection_gamepk_conflict` before any run.
- **`VerifiedSelectedEventResolution`** (immutable): selection namespace (odds_api),
  observed event facts + observation instant, verified gamePk, operational GameDate,
  server-canonical competition, the COMPLETE six-field `GameIdentityContext`, matcher
  policy/schema versions, bracket, verified status, reschedule-context count, frozen
  timestamp. Selection namespace and identity SourceProvider are asserted distinct.
- **Provenance builder** (`SelectedEventProvenance.BuildEnvelopeJson`): sports-owned;
  validates its complete payload (all-or-none identity), emits the generic envelope
  (domain `sports`, type `selected_event_binding`, schemaVersion `1.0`), and proves it
  through `DomainExecutionProvenanceEnvelope.TryRead` before returning -- generic code
  can never author a sports payload (no such API exists).
- **Gate-2 seam** (`VerifyForExecutionAsync`): independent staged re-resolution, fresh
  six-field bundle COMPARED to the frozen bundle memberwise -- compare, never replace.

## 2. commit b -- atomic contract activation and enforcement

**Commit B:** dai `b4734aa10cd631bcf905c678fcb5028c09f1d654`
"feat(agent-runs): enforce verified selected event creation" (15 files, 1543/100).

- **Request contract**: nullable null-suppressed `CompetitionMatchupInput.SelectedEvent`
  (`SelectedEventIntent { providerEventId, startUtc }`, both nullable strings so a
  partial/blank/malformed member is precisely detectable -- never a silently accepted
  default instant). Absent or json-null block -> unchanged legacy path; present block
  validates strictly (canonical explicit-utc instant forms) -> 400
  `selection_intent_malformed`; malformed intent NEVER falls through to legacy. Legacy
  serialization byte-identical (pinned by exact-string test); historical InputJson
  deserializes with null.
- **Default-off activation** (`SelectedEventActivation*`): typed options
  (`Enabled=false` shipped in configuration with blank identity/path), pure evidence
  evaluator requiring a current external deployment-evidence record -- deployment
  identity match, observation + expiry instants, configuration and process citations,
  and TEN individually-required topology assertions (single run-creating process, one
  worker, no web garden, no overlapping recycle, no revision overlap, no mixed pool,
  no second manual process, no alternate run creator, old-process-stopped-first,
  migration present). Anything missing/false/expired/mismatched keeps activation off.
  Application code claims nothing about topology; the evidence is operator-supplied
  (see [[selected-event-activation-evidence-v1]]). An inactive gate refuses
  `selection_identity_not_active` BEFORE any provider or database work.
- **Gate-1 placement**: intent validation -> activation -> full resolution +
  provenance assembly complete BEFORE the creation gate; no run exists yet.
- **Shared creation gate**: `DuplicateRunGuard.GateFor(tenantKey, canonicalCompetition)`
  -- one process-local namespace for legacy AND selected paths, keyed by
  server-authenticated tenant + server-canonical competition (CompetitionCatalog code);
  no client team/date/eventId/start/gamePk in the key. Held only across candidate read
  -> classification -> duplicate verdict -> run construction -> atomic insert ->
  SaveChanges; no network under the gate; explicitly documented process-local-only.
- **Eligibility + four-arm classification**: excluded and failed rows are nonblocking
  by status doctrine BEFORE provenance inspection; every remaining potentially blocking
  candidate is classified by the sports-owned
  `SelectedEventProvenance.ClassifyCandidate`: (a) database NULL -> legacy fallback
  (the ONLY legacy route); (b) recognized exact-ordinal sports/selected_event_binding,
  known schemaVersion, complete bundle -> typed selected candidate; (c) recognized but
  invalid -> 409 `duplicate_candidate_identity_invalid` (internal details
  incomplete_identity_bundle / malformed_provenance_envelope /
  unknown_provenance_schema_version); (d) EVERY other non-null value (empty,
  whitespace, json null, empty object, malformed, missing metadata, extra members,
  wrong-case, foreign domain/type) -> 409 with internal detail
  `unrecognized_provenance_document`, never legacy. An active malformed candidate is
  never skipped; the refusal mutates nothing, creates no run/provenance, and returns
  only the tenant-safe existing AgentRunId.
- **Cross-path duplicate identity** (`DuplicateRunIdentity`): known-gamePk equality
  first (selected requests use the VERIFIED pk, never the client's); otherwise one
  canonical unordered team-reference pair, single-sourced through
  `GameIdentityDerivation.NormalizeTeamRef` and prepared by the caller -- selected
  candidates read authoritative persisted refs (never selected InputJson teams, never
  provenance parsed in the guard); legacy candidates convert their persisted request
  through the same authority. No alias resolution introduced; unknown-pk doubleheader
  identity fails closed; selection-level idempotency is NOT claimed.
- **Atomic insert**: one row, one SaveChanges inside the gate: tenant, run id, pending
  status, correlation, InputJson = original client intent, server-canonical
  Competition, verified GameDate, the complete six-field identity bundle (all or
  nothing), the authoritative gamePk in ExternalGameId, and
  `AssignDomainExecutionProvenance` exactly once (envelope + payload pre-validated).
  No model call can begin before this save succeeds. Legacy creation unchanged except
  the shared gate.
- **Gate-2 + execution split**: after persistence, before any model work, the frozen
  resolution re-verifies; any refusal leaves the run + input + provenance + frozen
  bundle untouched, sets truthful failed status with the typed reason on ErrorMessage,
  makes no model call, creates no outcome, cannot settle, and permits a later NEW
  resubmission. `ApplyGameIdentity` is split: legacy runs keep retrieve-time capture;
  selected runs NEVER overwrite creation-time identity -- a disagreeing statsapi
  execution identity fails the run truthfully. Model-capable execution for a selected
  run exists ONLY through the internal `IVerifiedSelectedExecution` seam carrying the
  frozen authority; the ordinary `IAgentRunService.ExecuteAsync` throws on a selected
  request, so direct service callers cannot bypass gate 1/2.

## 3. refusal matrix (pinned)

- 400 `selection_intent_malformed` (partial/blank/malformed/oversized block; no
  resolution call, no row);
- 422 `selection_identity_not_active` (before provider/database work);
- 422 gate-1 refusals: `selection_event_not_in_schedule`,
  `selection_provider_identity_conflict`, `selection_provider_source_failure`,
  `selection_start_mismatch`, `selection_identity_mismatch`,
  `selection_ambiguous_candidates`, `selection_gamepk_conflict`,
  `selection_game_status_refused` (staged reason carried),
  `selection_verification_source_failure`, `selection_competition_unsupported`,
  `selection_provenance_invalid` -- typed response + correlation-linked logs, no run,
  no provenance, no durable refusal ledger, no model;
- 409 duplicate-run refusal (existing shape) and 409
  `duplicate_candidate_identity_invalid` (four-arm integrity);
- 422 post-create gate-2 refusals (`selection_execution_identity_mismatch`,
  `selection_execution_verification_failed`): run + frozen evidence persist, failed
  status + typed reason, no model, no settlement.

## 4. test evidence (all offline; visible counts)

- Phase A RED: compile-absence of the authority captured; GREEN
  `SelectedEventResolutionTests` 27/27 (opaque/ordinal id, blank/oversized, absent
  event, conflicting records, transport/malformed source, stale start, doubleheader
  separation, no-bind, ambiguity, gamePk agree/conflict, staged refusals incl. missing
  status + malformed container, unsupported competition, gate-2 exact/drift/refusal,
  envelope build/refuse/round-trip, namespace separation, purity). Focused + adjacent
  480/480 (provider-event, market-join, odds schedule, starters, retriever,
  market-contrast).
- Phase B RED: guard signature migration compile break captured; GREEN new suites:
  `SelectedEventActivationTests` (default-off, per-assertion requirement matrix,
  expiry, revision mismatch, malformed), `SelectedCandidateClassificationTests` (all
  four arms incl. builder/classifier agreement, wrong-case, foreign, unknown schema,
  structural garbage never legacy), `SelectedEventCreationTests` (atomic bundle +
  provenance persisted together; gate-2 truthful failure with zero model calls; gate-1
  refusal no-row; intent matrix never falls to legacy; byte-identical legacy
  serialization; same-event duplicate; CONCURRENT two-spelling requests -> exactly one
  run; re-resolved pk = versioned decision, not idempotency; two events -> one pk
  duplicate; doubleheader two pks -> two runs; selected/legacy races in both orders;
  legacy known-pk blocks selected; tenant isolation; malformed active candidate refuses
  both paths and is never skipped; excluded/failed malformed nonblocking; gate releases
  after every refusal; ordinary-path selected refusal; mismatched-authority refusal).
- Cumulative: full `dotnet test DevCore.Api.Tests` **1977/1977, 0 skipped**
  (foundation baseline 1896 + 81 new); full solution build 0 errors; `git diff
  --check` clean per phase.

## 5. migration, rollback, and deployment

No new migration in b2; the foundation migration
`20260727133845_AddAgentRunDomainExecutionProvenance` remains REQUIRED and UNAPPLIED to
every database (deployment separately gated; the activation evidence asserts its
presence in the target database). Rollback: disable activation (config or evidence
removal, effective next request); the additive column and any persisted provenance
retain their bytes. b2 changes no legacy accepted input, response shape, duplicate
behavior for known-identity classes, execution behavior, model path, settlement path,
or frontend behavior.

## 6. platform/domain boundary

The guard consumes sports-PREPARED typed identity and never parses provenance or
InputJson; classification and provenance authoring are sports-owned
(`SelectedEventProvenance`); the platform transports the opaque envelope and enforces
storage/cardinality/single-assignment only; the matcher dependency bound note holds
(the authority depends on matcher + evidence + strict validator only -- pinned).

## 7. current limitations and explicitly unauthorized behavior

- Production activation remains DISABLED everywhere; no evidence record exists.
- Process-local gate only; scale-out blocked
  (MULTI_INSTANCE_SELECTED_EVENT_ATOMICITY_REQUIRED_BEFORE_SCALE_OUT).
- Atomic-insert failure atomicity is enforced by construction (single SaveChanges; all
  members set before Add) and pinned observationally; it is not fault-injected under
  the in-memory provider (documented limitation for the reviewer).
- Frontend propagation (2-iii-c) and 2-ii-c are unauthorized; the frontend cannot yet
  send SelectedEvent.
- Staged verification supports baseball (statsapi) competitions only; others refuse
  `selection_competition_unsupported`.
- Pre-create refusals are response + observability only
  (DURABLE_PREEXECUTION_SELECTION_DECISION_LEDGER_DEFERRED).

## 8. semantic dispositions

- **selected event intent** / **verified selected-event resolution** /
  **activation deployment evidence** / **validated writer**: RETAINED WI-local
  (implementation vocabulary; the operator-owned glossary already carries "domain
  execution provenance", which these implement; promotion deferred to WI-0037
  completion's mandatory glossary disposition pass).
- SELECTED_RUN_PROVENANCE_VALIDATED_WRITER_CONTRACT_V1: FULFILLED by
  `SelectedEventProvenance.BuildEnvelopeJson` + the pre-assignment validation path
  (its six RED contracts pinned across builder/classifier/endpoint tests).

## safety / non-actions

No push; no integration; no migration applied; no live Odds API, StatsAPI, model,
paid, capture, reconciliation, or settlement call (fake transports only); no database
beyond in-memory test stores; no dependency change; no frontend change; production
activation off; preserved WI-0035 worktree, ops branch, architecture branch, and
foundation branches untouched.

## review findings and dispositions

`dai-code-reviewer` over Commit A, Commit B, and the combined delta: zero blocking
findings. Notes recorded: (1) the atomicity fault-injection limitation above; (2) the
guard's legacy normalization migrated from its private normalizer to the single
canonical `NormalizeTeamRef` authority -- equivalence for the screened-workflow name
classes is pinned by the adapted guard suite and the full regression run.

## b2 authority corrections -- 2026-07-27 (f-b2-1 .. f-b2-6)

The final independent review returned CORRECTIONS_REQUIRED (two High, two Medium, two
Low). All six findings are corrected by two new commits on the same branch (nothing
amended): Commit C dai `8d2d0642cfd78cc5040c555fe000de195935d40d` ("fix(agent-runs):
enforce selected identity consistency") and Commit D dai
`0b523a562d5c1f1896ed613d2b4071176815d4e6` ("fix(agent-runs): harden selected
execution activation"). Finding-to-fix-to-test traceability:

- **F-B2-1 (High)** row/provenance agreement: candidate projection extended
  (SourceProvider, ScheduledStartUtc, Season, GameDate) with a pk-widened tenant query
  (rows persisting the request's known gamePk are seen even when their row GameDate or
  Competition contradicts provenance; cost = one extra OR on a tenant-scoped equality);
  `SelectedEventProvenance.ClassifyCandidateRow` compares the COMPLETE persisted row
  bundle against the frozen document under canonical representations and refuses 409
  `duplicate_candidate_identity_invalid` with bound internal detail
  `row_provenance_identity_disagreement` on any missing/partial/contradictory pair;
  the duplicate path consumes the classifier's AGREED identity (parsed authority is
  never discarded for raw row fields); payload internal consistency extended (gamePk ==
  externalGameId, readable start, competition/date/namespace/versions present).
  RED first: null-row-identity and contradictory-pk seeds ADMITTED duplicates before
  the fix (captured); now refused. Tests: SelectedEventCreationTests row-agreement
  block + SelectedCandidateClassificationTests.
- **F-B2-2 (High)** server-bound retrieval + last pre-model gate: gate 1 now freezes
  the server's own canonical binding wire (`VerifiedSelectedEventResolution.
  ServerBindingWire`, emitted through the complete content contract; fingerprint frozen
  in the provenance payload); selected execution retrieves with a SEPARATE internal
  authoritative input (observed provider names, canonical competition, verified date +
  gamePk, server wire -- client home/away never select evidence; InputJson untouched);
  after retrieval and BEFORE the analyzer the retrieved identity must exist and agree
  with the complete frozen six-field bundle or `SelectedExecutionIntegrityException`
  fails the run truthfully with zero model calls; bound-market verification is required
  for every selected execution; the post-model check remains defense in depth. RED
  first: client-name grounding and null-identity-reaches-analyzer captured failing.
  Tests: AgentRunServiceSelectedExecutionTests.
- **F-B2-3 (Medium)** run-bound execution authority: gate 2 now MINTS
  `SelectedExecutionAuthority` (internal constructor; run-id-bound) and the execution
  seam consumes it, refusing before retrieval on any incoherence: run identity,
  selected-block absence, provider event id, declared start, canonical competition,
  operational date, client gamePk. A gate-1 resolution alone cannot start execution.
- **F-B2-4 (Medium)** bounded evidence freshness: observedAtUtc semantically enforced
  (future beyond 5-minute skew refused; age > 24h refused), expiry strictly after
  observation, validity window capped at MaxEvidenceLifetime = 24h (a longer window was
  not needed for the single-operator deployment and is therefore not justified);
  citations are structured {artifact, reference} with bounded members; runbook updated
  ([[selected-event-activation-evidence-v1]]).
- **F-B2-5 (Low)** deterministic save-failure proof: an EF SaveChanges interceptor
  injects a first-save failure over the selected insert -- no persisted row, no
  independent provenance, zero executions, gate released, following request succeeds
  (SelectedEventInsertFailureTests). The production transaction boundary is unchanged.
- **F-B2-6 (Low)** dual selected/binding cross-check: a client flight binding naming a
  different provider event than the selected block refuses 422
  `selection_binding_conflict` at the trust boundary; execution always uses the server
  wire.
- Operational-date clarity: selected persistence uses `selectedResolution.GameDate`
  (server-confirmed; the observation seam additionally re-checks bracket membership of
  the selected event, and an adjacent-bracket client date is pinned refusing).

Suites after corrections: full .NET **2011/2011, 0 skipped** (b2 base 1977 + 34
correction tests); solution build 0 errors; activation remains disabled in tracked
configuration; the migration remains unapplied; no frontend/dependency change; the
accepted review dispositions (PROVIDER_NAMESPACE_DURABLE, CLIENT_DATE_CROSSCHECK_ONLY)
were not reopened.

## f-b2-7 provider-scoped candidate identity -- 2026-07-27

A staff review of the correction batch found F-B2-7 (Medium): the f-b2-1 candidate
query widened on `ExternalGameId` ALONE although the governing game and settlement
identity is the pair `(SourceProvider, ExternalGameId)`, and the widening key fell
back to the CLIENT-supplied legacy GamePk. Impact demonstrated RED-first: an active
same-tenant row for another competition/provider with a colliding numeric external id
was pulled into the candidate set, falsely blocked valid creation, and disclosed its
AgentRunId; a legacy client GamePk could deliberately steer the widening. Corrected by
dai `b6aae1c84726995f19751d7b888e5b446bb49ea0` ("fix(agent-runs): scope duplicate
candidates by provider identity"):

- normal candidate scope = authenticated tenant + operational date
  (selectedResolution.GameDate when available) + server-canonical competition,
  compared in memory;
- widening exists ONLY for a selected request after gate 1 and its key is the EXACT
  verified pair (SourceProvider AND ExternalGameId) in both the database predicate and
  the in-memory competition-filter bypass;
- a client-supplied legacy GamePk never widens discovery (legacy scope unchanged; no
  new legacy authority);
- classification, complete row/provenance agreement, agreed-identity consumption,
  status doctrine, doubleheader behavior, tenant isolation, and every completed b2
  behavior preserved (pinned).

Honest discoverability boundary (supersedes the earlier "cannot hide" wording in the
f-b2-1 entry above): a candidate is found through normal tenant/competition/date
scope, or -- for selected requests -- through the exact verified provider-identity
pair. A row whose normal-scope fields AND provider-identity pair have all been
corrupted is not discoverable here; the atomic selected insert prevents that state
during normal creation, and historical/manual data repair is outside this slice.

RED: three genuine failures pre-fix (cross-provider collision blocked+disclosed;
same-date cross-competition collision blocked; client-GamePk widening disclosed an
unrelated row); GREEN: those plus exact-pair discovery of a contradictory selected
row (still 409 row/provenance disagreement), same-pair other-tenant isolation, and
pair-matched excluded/failed nonblocking pins. Suites: full .NET **2017/2017, 0
skipped** (2011 + 6); solution build 0 errors; scan proves no client-GamePk query
widening remains. "provider-scoped candidate identity" is RETAINED WI-local pending
the WI-completion glossary pass.

## f-b2-8 provider-qualified duplicate identity -- 2026-07-27

Staff review found F-B2-8 (High): candidate DISCOVERY became provider-scoped at
b6aae1c, but DuplicateRunGuard still compared external ids without their namespace --
`DuplicateRunIdentity`/`DuplicateRunCandidate` carried bare ids, `KnownGamePk` parsed
ANY numeric ExternalGameId, and the F-B2-7 tests could not reach the gap because their
cross-provider rows were filtered before the guard ran. Genuine RED inside normal
scope: (1) a same-competition/date row from another provider with a colliding numeric
id FALSELY BLOCKED an unrelated game and disclosed its AgentRunId; (2) different
numeric ids from different providers were treated as "two known distinct games",
SKIPPING the canonical-team fallback and ADMITTING a duplicate of the same physical
game; (3) unknown-provider identity handling was untyped. Corrected by dai
`aee1ade8a27da45d845510baffaabca7068be974` ("fix(agent-runs): qualify duplicate
identity by provider"):

- **`ProviderGameIdentity(SourceProvider, ExternalGameId)`** -- the governing pair as
  one typed, sports-prepared, opaque identity; `DuplicateRunIdentity` and
  `DuplicateRunCandidate` now carry it; `KnownGamePk` and every numeric parse are gone
  from the guard (identifier shape carries no authority: opaque nonnumeric ids are
  valid identities).
- **Decision table (bound):** same SourceProvider + same ExternalGameId -> the same
  authoritative game, blocks under status doctrine; same SourceProvider + different
  ids -> two distinct games, independently creatable, canonical-team fallback NOT
  consulted (doubleheader doctrine preserved); different SourceProviders -> ids
  incomparable, accidental string/numeric equality ignored, canonical unordered-pair
  fallback; either side without a complete pair -> pair fallback, fail closed. All
  comparisons exact ordinal over server-prepared canonical values.
- **Caller preparation** (sports-owned, in the controller): selected requests use the
  gate-1 verified pair; selected candidates use the agreed `ClassifyCandidateRow`
  result (never unchecked row fields); legacy candidates use persisted
  (SourceProvider, ExternalGameId) when BOTH exist, else a pending row's own GamePk is
  an `mlb_statsapi` identity ONLY under the existing WI-0009 mlb contract, else
  identity stays unknown and the pair fallback applies. Both arrival orders pinned
  (selected-first/legacy-second and legacy-pending-first/selected-second refuse
  exactly one active run).
- **Scans:** the guard contains zero TryParse/numeric authority; the only remaining
  production ExternalGameId comparisons are the F-B2-7 provider-PAIRED widening
  predicate and the already-paired reconciliation read; the one retained
  `long.TryParse` is the statsapi-specific starter-cache admit (a validated
  StatsAPI-only operation outside this contract).

F-B2-7 preserved verbatim (query rule untouched; its suite green). Suites: full .NET
**2025/2025, 0 skipped** (2017 + 8: three RED-to-GREEN endpoint attacks, one endpoint
fallback pin, four guard decision-table pins incl. opaque-id and unknown-provider
rows); solution build 0 errors; activation disabled; migration unapplied.
"provider-qualified duplicate identity" is MERGED with the F-B2-7 "provider-scoped
candidate identity" vocabulary as one WI-local concept (discovery scope + comparison
contract over the same governing pair), retained for the WI-completion glossary pass.

## next step

Exactly one: operator issues the final independent b2 re-review over the complete
package `1311137..aee1ade` (reproducing the provider-qualified comparison matrix and
all F-B2-1..8 corrections) and integrates on PASS; production activation stays
disabled; 2-iii-c and 2-ii-c stay unauthorized.

## related

- [[WI-0037-game-state-correctness-v1]] -- governing work item.
- [[wi-0037-slice-2-iii-architecture-review-2026-07-26-v1]] -- the published
  architecture (parts 1-8) this batch implements.
- [[wi-0037-slice-2-iii-foundation-implementation-2026-07-27-v1]] -- the integrated
  foundation this builds on.
- [[selected-event-activation-evidence-v1]] -- the operator activation runbook.
