---
title: "WI-0037 Slice 2-iii Architecture Review: Selected-Event Identity Continuity 2026-07-26 v1"
type: "architecture-review"
date: "2026-07-26"
status: "architecture review complete LOCAL -- independent architecture review required; implementation NOT authorized"
project: "DAI"
slice: "WI-0037 Slice 2-iii: selected-event identity continuity"
repos:
  dai: "read-only inspection of published main af59853; ZERO source changes"
  dai-vault: "this record on local branch wi/0037-selected-event-identity-continuity-architecture (base 234d3f0); NOT pushed"
tags:
  - system-development
  - sports-v1
  - architecture
  - identity
related:
  - "02 Platform/system-development/work-items/WI-0037-game-state-correctness-v1.md"
  - "06 Execution/reports/wi-0037-slice-2-ii-b-adversarial-review-and-corrections-2026-07-26-v1.md"
---

# wi-0037 slice 2-iii architecture review: selected-event identity continuity

## 1. executive decision summary

The platform ALREADY owns every load-bearing mechanism this residual needs. The
analysis request contract carries an optional, null-suppressed, server-verified
`GamePk` (WI-0009, `AgentRunContracts.cs:154-168`); retrieval already resolves a
supplied gamePk through the staged bracket authority with terminal doubleheader
ambiguity (WI-0037 2-ii-a, `MlbStarterClient.cs:306-353`); the duplicate-run
guard already keys on gamePk (`DuplicateRunGuard.cs:64-80`); the run row already
persists the canonical settlement identity `(SourceProvider, ExternalGameId =
gamePk)` plus start/season/team refs (`AgentRun.cs:58-82`); and WI-0035/36
already define the frozen provider-event binding wire, its fail-closed trust
boundary, and the bound-run verification seam (`AgentRunContracts.cs:169-178`,
`AgentRunService.cs:47-55`, `ProviderEventBinding.cs`). What is missing is ONE
thing: the operator's selected odds-surface event (providerEventId) is never
translated into that existing gamePk authority -- the frontend request drops it
(`analyzer.component.ts:642-650`; TS `CompetitionMatchupInput` has no gamePk,
`agent-run.model.ts:39-44`).

**Recommendation (Candidate E-prime): reuse the existing WI-0009 gamePk
execution authority; add additive selected-event INTENT fields to the request;
translate intent -> gamePk SERVER-SIDE inside the existing retrieval step using
WI-0035-style binding rules; persist selection provenance additively.** No new
identity object, no token service, no new authority, no migration.

## 2. motivation and published residual

SELECTED_EVENT_IDENTITY_PROPAGATION_REQUIRED_BEFORE_WI0037_CLOSE (published in
the 2-ii-b closeout): the request serializes competition/homeTeam/awayTeam/
gameDate only, so two distinct same-day provider events serialize into
equivalent analysis requests; count parity is not semantic execution identity.

## 3. canonical identity glossary

- **providerEventId** -- odds-provider namespace (`OddsApiEvent.id`). Owner: the
  odds provider. Uniqueness: provider scope; opaque; forgeable by any client;
  authoritative ONLY on the odds reference surface (2-ii-b contract,
  `OddsScheduleClient.cs`). Never execution authority.
- **StartUtc** -- STARTUTC_FIXED_WIDTH_UTC_100NS provider commence instant.
  Descriptive on the discovery surface; used as a binding VERIFICATION input
  (delta rules), never as identity by itself. Schedule changes make it stale
  without changing event identity.
- **gamePk (statsapi)** -- the authoritative MLB game identity; doubleheader-
  unique; owner: StatsAPI; verified server-side via the staged
  game-status-resolution contract 1.1 (bracket, in-bracket uniqueness,
  identity_mismatch).
- **Provider-event binding identity** -- WI-0035/36 shape: a FROZEN binding
  evidence bundle carried VERBATIM as raw json wire
  (`FlightSelection.ProviderEventBinding`, `AgentRunService.cs:47-55` "GetRawText,
  never a reserialization"), serialized with `provider_commence_time_utc` etc.
  (`ProviderEventBinding.cs:216`), validated fail-closed
  (`FlightSelectionProvenance.Validate`, `FlightSelectionProvenance.cs:96`),
  re-verified at the bound-run trust boundary (`MarketBindingVerification`,
  `AgentRunService.cs:88-95`). It is request-scoped frozen evidence, NOT a
  durable server-side reference registry -- there is no binding-id lookup table.
- **Selection identity** -- what the operator clicked: providerEventId held in
  `AnalyzerComponent.selectedEvent` (`analyzer.component.ts:54,115-125`) and
  dev-artifact-review `selectedGameKeys` (`dev-artifact-review.component.ts:420-443`).
  Today it exists ONLY in component state: declarative intent, zero trust.
- **Analysis-request identity** -- `CompetitionMatchupInput(Competition, HomeTeam,
  AwayTeam, GameDate, GamePk?, FlightSelection?)`; without GamePk two distinct
  DH selections are semantically identical requests.
- **Run identity** -- `AgentRunId` (Guid) + `CorrelationId` (X-Agent-Run-Id
  anchor); batch entries correlate 1:1 by construction (2-ii-b F-A).
- **Reconciliation/settlement identity** -- "the canonical outcome-reconciliation
  match key is (SourceProvider, ExternalGameId)" (`AgentRun.cs:58-68`);
  settlement writes are identity-driven with provenance-completeness refusal
  (`AgentRunsController.cs:995-1044`); finals guard operates on gamePks.

## 4. authority lattice

| fact | client intent | server-observed | server-authoritative | durable provenance |
|---|---|---|---|---|
| providerEventId | YES (selection) | discovery response | never | InputJson (proposed additive) |
| StartUtc | YES (display/verify) | discovery + binding | never (verification input) | InputJson (proposed) / binding wire |
| home/away teams | YES | schedule + odds | statsapi record wins (identity_mismatch) | run row TeamRefs |
| operational date | YES | eastern bracket | bracket authority (contract 1.1) | run row GameDate |
| gamePk | optional (WI-0009) | schedule | ALWAYS server-verified via staged resolver | run row ExternalGameId |
| provider-event binding | frozen wire (WI-0036 seam) | retrieve-time verification | server verification decides | InputJson wire + MarketBindingVerification |
| selected-event reference | n/a today | n/a | n/a | MISSING -- the residual |
| run id | no | server-minted | server | run row |

Bound invariants (all already enforced or preserved by the recommendation):
client fields are intent; client providerEventId never directly authorizes; a
client gamePk is ALSO only intent until the staged resolver verifies it
in-bracket with matching identity (2-ii-a); gamePk stays authoritative;
provider-to-game translation is server-owned; frozen binding immutable per
execution (verbatim wire); retries deduplicate via DuplicateRunGuard's
gamePk-aware comparison; DH events never collapse to date/team identity
(WI-0006 fail-closed ambiguity preserved for legacy requests); stale/mismatched
selection fails closed with typed reasons; durable records distinguish selected
(InputJson) / observed (binding verification) / authorized (resolver outcome) /
executed (run row identity); generic orchestration stays sport-agnostic.

## 5. platform-versus-domain boundary

Generic platform: AgentRunId, CorrelationId, TenantKey, InputJson/OutputJson
envelopes, duplicate-run guard SHAPE, refusal/audit envelope. Sports domain:
providerEventId, StartUtc, team normalization, DH interpretation, gamePk,
provider-event binding, game-status refusal vocabulary. The recommendation adds
NOTHING to generic orchestration: the new intent fields live on
`CompetitionMatchupInput` (already the sports-owned input shape,
`AgentRunContracts.cs:143-145` notes the future generic envelope), and the
translation lives in the sports retriever. No generic provenance envelope is
required for this slice.

## 6. current-state identity-flow map (file:line)

odds json -> `OddsApiEvent` -> `NormalizeEvents` (id-authoritative, fail-closed
integrity; `OddsScheduleClient.cs:244+`) -> `MatchupEventDto(Date, Home, Away,
StartUtc, ProviderEventId)` -> `GetMatchupDates`/`GetUpcoming` passthrough
(`SportsReferenceController.cs:108-205`) -> TS `MatchupEventDto` (required
fields) -> analyzer `selectedEvent` / dev-review `selectedGameKeys`
(id-keyed, 2-ii-b) -> **analysis request construction: identity DROPPED**
(`analyzer.component.ts:642-650`; `dev-artifact-review.component.ts:459-464`;
TS input type has no slot, `agent-run.model.ts:39-44`) -> HTTP ->
`CreateAgentRunRequest`/`CompetitionMatchupInput` (GamePk slot EXISTS, arrives
null) -> `AgentRunsController` -> `AgentRunService.RunSportsMatchupPipelineAsync`
(`AgentRunService.cs:39-55`: gamePk + binding wire onto the artifact) ->
`SportsRetriever.RetrieveAsync` -> starter path: with pk -> staged resolver +
terminal DH ambiguity (`MlbStarterClient.cs:306-353`); without pk ->
`MlbEventResolver` matchup resolution, DH -> Ambiguous fail-closed (WI-0006) ->
run persisted with Stable Game Identity (`AgentRun.cs:58-82`) -> reconciliation/
settlement keyed on (SourceProvider, ExternalGameId).

**Exact identity-loss point: `analyzer.component.ts:649` (`gameDate: ev.date` --
the last line of the payload; `ev.providerEventId` is in scope and unused) and
`dev-artifact-review.component.ts:463` for the batch path.**

## 7. caller and consumer inventory

- analyzer (`createMatchupAnalysis`, :634): no providerEventId sent; DH days
  currently refuse server-side (WI-0006 Ambiguous).
- dev-artifact-review batch (:456-464): same payload; knows providerEventId.
- sports-api.service: passthrough; stubs generate ids (2-ii-b).
- `/source-readiness` (`AgentRunsController.cs:1240-1268`): ALREADY takes an
  explicit gamePk parameter -- precedent for pk-carrying callers.
- market-contrast screen/batch (`MarketContrastSourceAdapter.cs:173`): already
  gamePk-native (candidates carry pks).
- tests/fixtures: `CompetitionMatchupInput` literals (GamePk null-suppressed
  keeps them byte-identical).
- No alternate/white-label frontend exists; PowerShell operator scripts do not
  create analysis runs.
- Legacy/persisted replays: rows without GamePk deserialize with null (WI-0009
  guarantee) -- ambiguous DH replays refuse rather than guess.

## 8. wi-0035/wi-0036 binding archaeology

The binding is created in the market-contrast/capture flow (WI-0035 Slices A-D):
candidate matching binds an odds event to a schedule candidate using orientation
+ bracket + commence-delta rules with sub-second guards
(`ProviderEventBinding.cs:640-700`: `sameOrientation`, `HasSubSecond`, delta
seconds vs `ScheduledStartUtc`, `InBracket` rule 4, unique-match requirement ->
`ProviderCommenceTimeUtc` frozen). Output: a frozen evidence bundle serialized
to wire (`:216`), carried VERBATIM into runs via
`FlightSelection.ProviderEventBinding` (WI-0036 Slice-3 seam), validated
fail-closed (`FlightSelectionProvenance.Validate`) and re-verified at retrieve
("market retrieval executes against EXACTLY the bound provider event"). Binding
inputs: provider event (id/teams/commence), bracket date, candidate gamePks.
There is NO durable binding-id registry: the binding is frozen EVIDENCE attached
to a request, not a referenceable server object. Lifecycle: request-local,
immutable once attached. Conclusion: Slice 2-iii can REUSE the binding RULES and
the wire seam, but a "binding reference id" (Candidate C) would require building
a registry that does not exist today.

## 9. persistence and provenance inventory

Run row (`AgentRun.cs`): InputJson (full request incl. GamePk/FlightSelection
when present), OutputJson, CorrelationId, Competition, GameDate, LeanSide,
SourceProvider, ExternalGameId (gamePk), ScheduledStartUtc, Season,
HomeTeamRef/AwayTeamRef, ExclusionReason, SupersededByAgentRunId,
PromptRouteProvenanceJson. Missing today: selected providerEventId/StartUtc
(would live in InputJson via the additive block -- no schema change), and an
explicit provenance-quality marker for legacy rows (derivable: absence of the
block IS the marker; document rather than migrate).

## 10-13. idempotency, replay, cache

Duplicate submission: `DuplicateRunGuard` compares by known gamePk (request
GamePk, resolved ExternalGameId, or pending candidate's persisted InputJson pk;
`DuplicateRunGuard.cs:12-14,64-80`) -- with the recommendation, two DH games
produce two DISTINCT pks -> two runs (correct); resubmitting the same selection
hits the guard -> one run. Sameness identity = gamePk (unchanged). Replay after
schedule change: gamePk is stable across reschedules (statsapi identity), and
the staged resolver re-verifies bracket/status at run time -- a moved game
resolves via reschedule-context rules or refuses; it can never silently become
the OTHER DH game (in-bracket uniqueness + identity_mismatch). Selection
references do not expire because none are minted (E-prime). Caches: schedule
caches are date/pair-keyed reference data (unchanged); run dedup is
guard-driven, not cache-keyed; no cache-identity change.

## 14. reconciliation and settlement

Both paths key on (SourceProvider, ExternalGameId) with provenance-completeness
refusals (`AgentRunsController.cs:995-1085`); the finals guard consumes gamePks.
E-prime introduces NO second authority: it makes the selection->gamePk->
settlement chain continuous. Historical runs remain truthfully reconcilable
(their identity was captured at generation time); rows lacking selection
provenance are distinguishable by the absent InputJson block.

## 15. traceability and learning-loop requirements

Selection evidence: selected providerEventId + StartUtc + teams/date + client
surface (analyzer|dev-review) -> additive InputJson block. Binding evidence:
resolver outcome (resolved pk, bracket, refusal reason, reschedule context) --
already logged structured (2-ii-a F-C) and persisted via run-row identity;
delta-vs-selection verification result added to the same structured log family.
Execution evidence: run id, correlation id, InputJson snapshot, route
provenance -- exists. Outcome evidence: reconciliation/settlement rows +
ExclusionReason -- exists. No internal reasoning is recorded.

## 16. candidate architectures

**A -- additive client fields (providerEventId + StartUtc), server re-resolves.**
Simple; forgery-safe ONLY if the server independently re-derives gamePk and
verifies identity; TOCTOU handled by run-time resolution. Weakness as stated:
leaves translation design open. (Subsumed by E-prime, which is A with the
translation pinned to existing machinery.)

**B -- server-minted selection token at discovery.** Requires a token
mint/store/expiry/tamper surface that does not exist; discovery becomes
stateful; token expiry adds refusals with no correctness gain over re-derivation;
duplicates binding responsibilities. WEAK (operational complexity,
platform purity), no correctness advantage: DISQUALIFYING on complexity/benefit.

**C -- binding-first durable reference.** Requires a binding registry (none
exists -- section 8); binding at discovery adds statsapi calls to a pure-odds
reference surface and creates stale-binding lifecycle management. WEAK
(coupling, latency, new persistence); rejected for this slice; the frozen-wire
seam already covers flight-capture needs.

**D -- analyze-by-selection command.** New endpoint that resolves + binds +
executes atomically. Correct but duplicates the existing create-run pipeline
(retrieval ALREADY resolves and verifies inside the run); versioning burden for
all callers. ACCEPTABLE but strictly more surface than E-prime for the same
guarantees.

**E-prime (RECOMMENDED) -- existing-authority reuse + additive intent +
server-owned translation.** Request gains an additive, null-suppressed
`SelectedEvent` block: `{ providerEventId, startUtc }` (WI-0009/WI-0036
serialization pattern -- legacy InputJson byte-identical). Frontend also
populates the EXISTING `gamePk` field ONLY when discovery supplies it (see
below) -- otherwise server-side translation runs: during retrieval, which
already fetches the date's schedule, the server matches the selected provider
event against bracket candidates using the WI-0035 rule family (orientation,
bracket, commence-delta with sub-second guards) to derive the candidate gamePk;
the derived pk then flows through the EXISTING staged-resolver verification
(in-bracket uniqueness, identity_mismatch, status authority) exactly as a
WI-0009 request would. Ambiguity or mismatch -> typed refusal, zero model
calls. Discovery MAY later be extended to return a server-computed candidate
gamePk per event (making the frontend a pure passthrough of server-derived
identity), but that is an optimization decision for implementation review --
the contract works with translation at request time alone and keeps discovery
odds-pure.

## 17. candidate scoring (summary)

| criterion | A | B | C | D | E-prime |
|---|---|---|---|---|---|
| authoritative correctness | ACCEPTABLE | ACCEPTABLE | STRONG | STRONG | STRONG |
| doubleheader safety | ACCEPTABLE | STRONG | STRONG | STRONG | STRONG |
| trust boundary | ACCEPTABLE | STRONG | STRONG | STRONG | STRONG |
| wi-0035/36 compatibility | ACCEPTABLE | WEAK (duplicates) | WEAK (registry absent) | ACCEPTABLE | STRONG (reuses rules + seam) |
| replay determinism | ACCEPTABLE | WEAK (expiry) | ACCEPTABLE | STRONG | STRONG (gamePk-stable) |
| idempotency | ACCEPTABLE | ACCEPTABLE | ACCEPTABLE | ACCEPTABLE | STRONG (guard already pk-aware) |
| durable provenance | ACCEPTABLE | ACCEPTABLE | STRONG | STRONG | STRONG (InputJson + run row) |
| reconciliation/settlement | ACCEPTABLE | ACCEPTABLE | ACCEPTABLE | ACCEPTABLE | STRONG (no new authority) |
| api/persisted compatibility | STRONG | WEAK | WEAK | WEAK (new endpoint) | STRONG (null-suppressed additive) |
| operational complexity | STRONG | DISQUALIFYING | WEAK | ACCEPTABLE | STRONG |
| platform purity | ACCEPTABLE | WEAK | ACCEPTABLE | ACCEPTABLE | STRONG |
| implementation scope | STRONG | WEAK | WEAK | WEAK | STRONG |

## 18. recommended canonical architecture

E-prime, precisely: client request shape = existing `CompetitionMatchupInput` +
additive `SelectedEvent { providerEventId, startUtc }` (+ optionally populated
existing `gamePk`); authoritative identity = statsapi gamePk, always
server-verified by the staged resolver; selected intent = the SelectedEvent
block (declarative, persisted, never authoritative); binding = server-side
translation during retrieval using WI-0035 rule family; binding lifecycle =
request-scoped, immutable per execution (existing artifact semantics);
idempotency = DuplicateRunGuard on gamePk (unchanged); cache identity =
unchanged; replay = deterministic via gamePk stability + run-time re-resolution;
refusals = new selection-domain vocabulary (section 20) NOT overloaded onto
game-status-resolution codes; compatibility = legacy requests unchanged
(WI-0006 fail-closed DH ambiguity preserved -- never guess); migration = none
(additive, null-suppressed); placement = sports domain only. Rejected
alternatives per section 17: B (complexity without correctness gain), C (no
registry exists; discovery coupling), D (duplicates the pipeline), naive A
(under-specified translation).

## 19. compatibility and migration

New clients send SelectedEvent (+gamePk when known). Existing clients keep
today's behavior exactly: uniquely-resolvable matchups run; ambiguous DH days
refuse (WI-0006) -- no silent guessing, no version bump needed (additive
optional fields). Historical runs: no backfill possible or required; absence of
the SelectedEvent block IS the provenance-quality marker (document in the run
inspection surface). Stored contracts: request DTO additive; responses
unchanged; run row unchanged; fixtures additive.

## 20. refusal and failure semantics (selection domain, new closed vocabulary)

`selection_event_not_in_schedule` (selected provider event matches no bracket
candidate); `selection_ambiguous_candidates` (matches >1 candidate after delta
rules); `selection_identity_mismatch` (teams/date of the selected event
contradict the matched candidate); `selection_start_mismatch` (StartUtc delta
exceeds binding tolerance -- stale schedule; retryable after re-discovery);
`selection_gamepk_conflict` (request supplies BOTH gamePk and SelectedEvent and
they disagree -- fail closed, client-correctable); plus the existing WI-0006/
2-ii-a vocabulary for pk verification. All detected server-side in retrieval
BEFORE any model call; all client-correctable except transient schedule
staleness; all produce the standard failed-run refusal record; no partial state
beyond the standard failed-run row. Game-status-resolution codes are NOT
overloaded.

## 21. contract sketches -- ARCHITECTURE SKETCH, NOT IMPLEMENTED

request (additive):
`{ competition, homeTeam, awayTeam, gameDate, gamePk?: 745123, selectedEvent?: { providerEventId: "ab12...", startUtc: "2026-07-26T17:05:00.0000000Z" } }`
(selectedEvent client-supplied declarative; gamePk client-supplied intent,
server-verified; both null-suppressed.)
stored run provenance: InputJson carries the block verbatim; run row identity
fields unchanged (server-authoritative after resolution).
refusal response: existing failed-run shape + typed reason from section 20.
replay request: byte-identical resubmission -> DuplicateRunGuard outcome.

## 22. implementation-slice decomposition (NOT authorized)

**2-iii-a -- contract + trust boundary (RED-first):** additive
SelectedEvent/gamePk on TS+C# input, null-suppression byte-compat pins,
selection refusal vocabulary types, validation fail-closed, gamePk/selectedEvent
conflict rule. No translation yet (selectedEvent alone -> typed
not-yet-supported refusal or pass-through-if-gamePk). Excludes: binding rules,
frontend wiring.
**2-iii-b -- server-owned translation + execution authority (RED-first):**
retrieval-time provider-event -> candidate matching (WI-0035 rule reuse),
delta/ambiguity refusals, staged-resolver handoff, DuplicateRunGuard
interaction pins, replay-after-reschedule fixtures.
**2-iii-c -- frontend propagation + provenance surfacing:** analyzer +
dev-artifact-review send SelectedEvent(+gamePk when server-supplied later);
DH end-to-end vitest pins (two pills -> two distinct pks -> two runs);
run-inspection provenance display; legacy-marker documentation.
Each slice: independent adversarial review, publication gate, residuals listed;
cross-runtime parity vectors where PS operator tooling displays identity.

## 23. test and fixture strategy

Pure unit: input validation, refusal typing, conflict rule. Contract: InputJson
byte-compat (with/without blocks), legacy deserialization. Integration
(offline fake-HTTP): DH pair -> two selections -> two pks -> two runs; forged
providerEventId -> selection_event_not_in_schedule; stale StartUtc ->
selection_start_mismatch; ambiguous candidates; duplicate submission -> guard;
replay after simulated reschedule (fixture pair); old-client DH -> WI-0006
ambiguity unchanged; no-model-call-before-refusal pins (handler request
counting). Cross-runtime: none required unless operator tooling grows a
selection surface. All fixtures offline; no live captures.

## 24. risks and open questions

(1) Binding-rule extraction: the WI-0035 matcher lives in the market-contrast
path; reuse must not couple analysis retrieval to market-contrast internals --
extraction seam to be designed in 2-iii-b review. (2) StartUtc tolerance: the
delta threshold for selection_start_mismatch must align with WI-0035 rules
(same constants, single source). (3) Discovery-supplied gamePk (the optional
optimization) would add statsapi load to discovery -- deferred decision,
implementation-review gated. (4) Tenant scoping: SelectedEvent carries no
tenant data; existing tenant boundary on runs is unchanged -- confirm in
2-iii-a review. (5) The generic-input-envelope note (`AgentRunContracts.cs:143`)
-- adding fields to CompetitionMatchupInput is consistent with current
single-run-type reality; revisit if a second run type lands first.

## 25. draft first implementation authorization -- NOT AUTHORIZED

Draft title: "DAI WI-0037 Slice 2-iii-a -- Selected-Event Contract and Trust
Boundary". Scope: exactly the 2-iii-a bullet above; RED-first; one dai commit +
one vault commit; no translation, no frontend wiring, no live calls; expected
status WI0037_SLICE2IIIA_IMPLEMENTED_LOCAL_REVIEW_REQUIRED. (Full prompt to be
issued by the operator only after the independent architecture review passes
and the architecture is published.)
