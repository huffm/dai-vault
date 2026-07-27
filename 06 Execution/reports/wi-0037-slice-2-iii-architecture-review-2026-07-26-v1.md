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

# part 2 -- architecture correction (af-1..af-5, l-1), 2026-07-26

The independent adversarial architecture review returned
WI0037_SLICE2III_ARCHITECTURE_REVIEW_CORRECTIONS_REQUIRED. Sections 1-25 above
are PRESERVED AS HISTORY; where this part conflicts with them, THIS PART IS THE
CURRENT AUTHORITY. The corrected canonical design is named
**E-PRIME-PRECREATE**.

## 2.1 af-1 (High) -- canonical pre-create authority order: SERVER_TRANSLATION_BEFORE_DUPLICATE_GUARD

Part 1 placed translation "during retrieval" -- AFTER DuplicateRunGuard and run
creation. That is wrong: the guard fails CLOSED on matchup identity whenever
either side lacks a known gamePk (`DuplicateRunGuard.cs:16-19,84-89`) and the
controller evaluates it with `req.Input.GamePk` straight from the request
before creating the run (`AgentRunsController.cs:157`). Under the part-1 order,
the second legitimate doubleheader selection (SelectedEvent-only, no pk) is
409-blocked -- the slice's flagship scenario fails. SUPERSEDED.

Canonical order (bound): request received -> validate SelectedEvent shape and
input bounds -> obtain the server-owned provider observation -> translate
providerEventId + StartUtc through the canonical rule owner -> candidate
gamePk -> existing staged gamePk verification (2-ii-a resolver) -> immutable
verified-resolution candidate -> ENTER the existing per-matchup creation gate
(`AgentRunsController.cs:129`) -> compare any client-supplied gamePk with the
verified gamePk (conflict -> refuse) -> DuplicateRunGuard WITH THE VERIFIED
AUTHORITATIVE GAMEPK -> freeze and persist intent + verified evidence +
gamePk atomically with run creation -> leave the gate -> model execution.
Invariants: no run row before translation + staged verification succeed; no
model call before evidence is persisted; no date/team-only duplicate decision
when a valid non-null SelectedEvent is present.

Gate/I-O boundary (bound): provider observation, translation, and staged
verification run OUTSIDE the creation gate (no network I/O under the lock);
the gamePk conflict check, duplicate evaluation, evidence persistence, and run
creation run INSIDE. The gate already serializes same-matchup creation, so the
outside-computed verified candidate cannot race a same-game duplicate past the
inside check. If implementation-time inspection proves different lock
semantics are required, the implementing slice must document the corrected
atomic design before merging.

Client/discovery-supplied gamePk is DEMOTED to optional intent and cross-check
only (never a co-equal canonical path): it never replaces server-owned
translation and staged verification; a conflict with the verified pk refuses
(`selection_gamepk_conflict`) before run creation.

Named af-1 RED scenarios (mandatory in the implementation slices): (1) first
DH selection creates run for gamePk A; (2) second selection, same teams/date,
different providerEventId, creates run for gamePk B; (3) the second selection
is never 409-blocked under matchup identity; (4) same selection twice follows
the existing duplicate policy for the same verified pk; (5) two provider ids
resolving to one gamePk -> the existing duplicate outcome, explicitly typed;
(6) conflicting client gamePk refuses before run creation; (7) no model call
before translation + verification; (8) no run row after an early selection
refusal.

## 2.2 canonical binding-rule owner (promoted from risk to requirement)

One sports-domain component -- architecture name
**ProviderEventGameBindingMatcher** (implementation name may follow repository
conventions) -- owns: provider-event normalization inputs, orientation
matching, date-bracket evaluation, commence-delta with sub-second handling,
candidate-set construction, unique-match selection, ambiguity/no-match
outcomes, and rule-version reporting. Bound rules: market-contrast and
selected-event translation consume the SAME matcher; neither duplicates the
rule family; selected-event translation never depends on market-contrast
orchestration, state, or workflow; the matcher is sports-domain, returns
evidence and typed outcomes only, never creates runs or invokes models; its
rule version is durably recorded in verified evidence.

## 2.3 af-2 (Medium) -- exact durable verified-evidence home: NEW_DURABLE_FIELD_AND_MIGRATION_REQUIRED

Part 1's "zero schema change" phrasing is WITHDRAWN. Inspection of the run
persistence surface (`AgentRun.cs`) shows no existing field satisfies the
requirements: `OutputJson` is composed at completion/failure (written AFTER
execution -- disqualified); `InputJson` is the serialized client request
(placing server evidence there would let client-supplied data masquerade as
verification -- disqualified); the scalar stable-identity columns cannot carry
the evidence bundle; `PromptRouteProvenanceJson` is owned by a different
provenance domain. Decision: a NEW nullable server-owned JSON column on the
run row -- sketch name `SelectedEventBindingJson` -- following the exact
`PromptRouteProvenanceJson` precedent (additive nullable string column, one
EF migration `AddAgentRunPromptRouteProvenance`-style, legacy rows null, no
backfill). Written INSIDE the creation gate atomically with the run row,
BEFORE model execution; survives failed and successful execution and every
retry; never touched by the composer; immutable for the run.

Conceptual separation (bound): `SelectedEventIntent { providerEventId,
startUtc }` -- client-supplied, nonauthoritative, lives in InputJson as part
of the request snapshot. `VerifiedSelectedEventBinding { providerNamespace,
observedProviderEventId, observedStartUtc, observedHomeTeam,
observedAwayTeam, operationalDate, candidateGamePks, authorizedGamePk,
bindingRuleVersion, bracketContext, translationOutcome,
providerObservationTimestampUtc, frozenAtUtc }` -- server-written only (sketch
names, not implementation authority); no client path can populate it.
Pre-create refusals (no run row): typed 409/422 response + structured log,
consistent with the existing duplicate-guard 409 precedent -- explicitly
accepted as the durable-record posture for pre-run refusals; post-creation
failures retain the persisted evidence block.

## 2.4 af-3 (Medium) -- provider namespace: server-derived and durable

providerNamespace is derived from the server's competition-scoped provider
registration (CompetitionCatalog/service registration -- the same mechanism
that selects the odds client today), NEVER client-supplied, frozen into
VerifiedSelectedEventBinding at observation time, immutable for the run and
its retries. Historical audit uses the persisted namespace, never current
configuration (rationale: legacy odds ids in `ExternalGameId`,
`DuplicateRunGuard.cs:97-99`, prove namespaces shift). If the provider
observed at execution differs from the namespace implied by the selection
context, fail closed (provider-source/selection typed outcome). Configuration
changes between discovery, submission, execution, retry, and audit are
resolved by the frozen per-run namespace.

## 2.5 af-5 (Medium) -- three replay classes (bound)

**Same-run retry** (another attempt for an existing run): reuse persisted
VerifiedSelectedEventBinding and authorized gamePk; never re-contact the
provider to retranslate; namespace and candidate set immutable; can never
bind the other DH game. **New resubmission** (a new request from the same
intent): a new decision -- re-observe, re-translate, re-verify, persist NEW
evidence, may refuse if the event moved/disappeared/became ambiguous;
duplicate policy applies on the newly verified pk. Cross-run rebinding policy
(bound): if the same namespace + providerEventId later resolves to a
DIFFERENT gamePk, the resubmission is PERMITTED as a new versioned decision
with visible provenance (each run's evidence stands alone); it is NOT refused
at runtime because gamePk authority + staged verification already prevent
wrong-game execution, and runtime refusal would require queryable cross-run
evidence (see risk 7). The divergence is surfaced in audit by evidence
comparison. **Historical audit replay**: nonmutating; reads persisted intent +
evidence + outcome; zero provider or StatsAPI calls; no new translation; no
mutation.

## 2.6 af-4 (Medium) -- activation safety and all-or-none semantics

Invariant (bound): any non-null SelectedEvent presented to a server without
active translation + enforcement receives the typed refusal
`selection_identity_not_active`; it NEVER falls through to legacy date/team
resolution. All-or-none: both fields absent -> legacy request; both present ->
selected-event path; exactly one present, blank id, or malformed StartUtc ->
`selection_intent_malformed`, fail closed, never legacy fallback.

Corrected decomposition (internal-first; SUPERSEDES part 1 section 22):
**2-iii-a -- canonical internal translator:** extract/establish
ProviderEventGameBindingMatcher + internal selected-intent domain types +
typed translation outcomes; NO public request field, NO frontend change, NO
persistence change. RED: DH selection resolves to intended pk; forged id,
stale StartUtc, ambiguous, and no-candidate refusals; rule parity with
WI-0035/36; no market-contrast dependency; no run/model creation.
**2-iii-b -- atomic contract activation + execution authority:** additive
null-suppressed SelectedEventIntent + all-or-none validation + pre-guard
translation/verification per 2.1 + the SelectedEventBindingJson column and
migration + namespace freeze + guard integration + retry/resubmission
semantics + legacy preservation. Contract and enforcement publish TOGETHER;
a non-null block can never be accepted without translation. RED: legacy
byte-identity serialization (see 2.8), historical deserialization, second-DH-
selection-creates-second-run, duplicate policy, client-pk conflict, evidence-
before-execution, no-run/no-model on early refusal, same-run retry uses
frozen evidence.
**2-iii-c -- frontend and consumer continuity:** analyzer + dev-artifact-
review propagation, stubs/alternate callers, provenance surfacing, offline
end-to-end DH tests, reconciliation/settlement continuity verification. RED:
two pills -> two intents -> two verified pks -> two evidence blocks -> two
runs -> two reconciliation identities; no semantic request collapse.
No slice is authorized by this document.

## 2.7 refusal-domain corrections

Retained: selection_event_not_in_schedule, selection_ambiguous_candidates,
selection_identity_mismatch, selection_start_mismatch,
selection_gamepk_conflict. Added: `selection_intent_malformed` (shape/bounds),
`selection_identity_not_active` (pre-activation). Provider transport/
availability failures map to the EXISTING provider-source-failure domain
(retryable; no authority decision; no model call) -- never forced into the
selection vocabulary. Binding-history divergence across runs is a permitted
versioned decision (2.5), not a runtime refusal. For every selection refusal:
detected pre-create in the controller translation step; client-correctable
except transient source failures; no run row (pre-create) or standard failed
run (post-create); durable record = typed response + structured log
(pre-create) or run row (post-create); provider observation happens before
refusal by necessity; model calls forbidden. Game-status-resolution codes are
never overloaded.

## 2.8 input bounds and legacy serialization pin (l-1)

Bounds (to be bound + tested in 2-iii-b from the provider contract; observed
odds ids are 32-char hex -- cap selected from contract/fixture evidence, not
invented): providerEventId nonblank, bounded length, control characters
rejected (reuse the 2-ii-b strict posture); StartUtc exact fixed-width
STARTUTC_FIXED_WIDTH_UTC_100NS form only; bounded candidate counts and
diagnostic lengths; structured logs only; no payload/url logging. The client
can never choose namespace, rule version, candidates, authorized pk, or
verified evidence. Mandatory 2-iii-b RED serialization pins: byte-equality of
legacy request serialization before/after the additive field; historical
InputJson deserialization; populated-block additivity; every serializer
option-set used on the path. If byte identity fails, STOP and reassess -- no
silent downgrade.

## 2.9 traceability evidence table (durable homes bound)

submitted providerEventId / startUtc -> InputJson (client intent, immutable);
providerNamespace, observed event, candidate pks, rule version, bracket
context, outcome, frozen timestamp -> SelectedEventBindingJson (server,
immutable); authorized gamePk -> run row ExternalGameId + evidence block;
refusal outcome -> typed response + log (pre-create) / run row (post-create);
run/correlation id -> run row; reconciliation + settlement -> existing
records. All nine audit questions in the review's section 22 are answerable
from these homes.

## 2.10 corrected scoring and recommendation

**E-PRIME-PRECREATE** is the recommended canonical architecture: additive
client intent; server-owned namespace; one canonical matcher; translation
BEFORE DuplicateRunGuard; staged verification before run creation; durable
server-owned evidence (new nullable column + migration); gamePk sole
execution authority; reconciliation/settlement authority unchanged;
internal-first activation; precise retry/resubmission/audit semantics.
Rescored vs part 1: idempotency STRONG (guard now receives the verified pk);
ordering STRONG; doubleheader safety STRONG; persistence STRONG (explicit
home); migration risk ACCEPTABLE (one additive nullable column, established
precedent) -- honest downgrade from part 1's overstated "zero change";
operational complexity STRONG. Mandatory client/discovery gamePk as the
primary design is REJECTED (shifts correctness pressure to client-supplied
identity and still requires server verification); retained as cross-check /
later optimization only.

## 2.11 risks and open decisions (dispositions)

(1) evidence home: DECIDED (2.3). (2) gate boundary: DECIDED (2.1), with an
implementation-time verification obligation. (3) cross-run rebinding:
DECIDED permit-with-provenance (2.5). (4) rule owner: DECIDED (2.2).
(5) namespace changes: DECIDED frozen-per-run (2.4). (6) provider
unavailability: source-failure domain (2.7). (7) queryability: JSON evidence
is NOT efficiently queryable cross-run -- accepted for v1 because no runtime
decision reads it cross-run; revisit only if a future slice needs
binding-history enforcement. (8) pre-run refusal durability: DECIDED typed
response + structured log (2.3), consistent with the dup-guard 409 precedent.

## 2.12 state

Slice 2-iii: architecture CORRECTED locally; independent delta architecture
review of this correction required; implementation unauthorized; publication
not claimed. 2-ii-c unauthorized; WI-0037 in-progress.

# part 3 -- bound deployment, persistence, and lifecycle constraints (df-1..df-7), 2026-07-26

The delta review returned WI0037_SLICE2III_ADR_DELTA_REVIEW_CORRECTIONS_REQUIRED.
Parts 1-2 are preserved as history; where part 3 conflicts, PART 3 IS CURRENT
AUTHORITY. E-PRIME-PRECREATE remains canonical only under these bindings.

## 3.1 df-1 -- single-host constraint: CREATION_GATE_PROCESS_LOCAL_SINGLE_INSTANCE_CONSTRAINT

Source evidence: `CreationGates` is a static in-process
`ConcurrentDictionary<string, SemaphoreSlim>` (`DuplicateRunGuard.cs:52`); its
own contract states "the api runs as a single operator-host instance in v1...
a db-level uniqueness constraint would need a schema migration...; that is the
documented multi-instance follow-up" (`DuplicateRunGuard.cs:31-35`). Bound:
(1) the gate serializes only within one API process; (2) it is not distributed
authority; (3) E-PRIME-PRECREATE assumes exactly one active API process
creating agent runs; (4) no multi-replica/scaled deployment may claim the
selected-event atomicity guarantee; (5) no mixed-version rolling activation is
permitted; (6) database-backed or distributed authority is required before
scale-out. Named residual (retained here, in the spec, and the handoff):
**MULTI_INSTANCE_SELECTED_EVENT_ATOMICITY_REQUIRED_BEFORE_SCALE_OUT**. No
durable work item currently owns the db-level uniqueness follow-up beyond the
source comment; ownership must be assigned before any scale-out authorization
(no new work item minted in this turn). This residual does not block the
single-host WI-0037 path (the single-host constraint is standing program
posture).

## 3.2 deployment and activation posture: SINGLE_CUTOVER_DEPLOYMENT_ASSUMED

Phases: (1) inert nullable provenance migration; (2) canonical matcher +
provenance-capable backend, public SelectedEventIntent inactive; (3) atomic
contract + enforcement activation with exactly one enforcement-capable API
process; (4) frontend/consumer propagation. Rollback bindings: migration-only
rollback -> code ignores the nullable field, data intact; backend rollback
after evidence exists -> old code tolerates the nullable field, historical
evidence never deleted; frontend rollback -> legacy requests remain supported;
no rollback state may accept-and-ignore a selected block. Rolling
multi-replica activation is deferred with the 3.1 residual.

## 3.3 df-2 -- freshness: VERIFIED_CANDIDATE_STALENESS_ACCEPTED_WITH_TWO_GATE_VERIFICATION

Gate 1 (pre-create): observe -> canonical matcher -> candidate gamePk ->
staged verification -> immutable verified-resolution candidate with
providerObservationTimestampUtc. Gate 2 (execution): retrieval independently
re-verifies the authoritative gamePk through the staged 2-ii-a resolver
(bracket, in-bracket uniqueness, identity_mismatch) before any model call.
Correctness: gamePk is stable statsapi identity; no bounded candidate age is
required; provider/schedule changes between gates may FAIL the run but can
never silently execute another game; after a Gate-2 refusal no model call
occurs and the original frozen evidence remains immutable on the failed run.
Flow bound: pre-create success -> run+evidence persisted -> Gate-2 detects
changed authority -> run fails before execution -> evidence preserved. No
provider or StatsAPI call occurs while the creation gate is held.

## 3.4 df-3 -- divergence: CONCURRENT_SELECTION_DIVERGENCE_VERSIONED_WITHIN_SINGLE_HOST_BOUND

The gate key (tenant|competition|matchup) serializes both DH games' creation
in-process; concurrent requests may pre-translate before acquiring the gate;
inside, each is evaluated with its own verified gamePk -- same pk follows the
existing duplicate policy, different pks become separate versioned decisions
with separate immutable evidence. This prevents wrong-game execution but IS
NOT selected-event-level idempotency, and no claim is made across multiple
API processes. Execution idempotency = authoritative gamePk under existing
doctrine; selection-level history = one namespace+providerEventId may
accumulate multiple verified decisions (permit-with-provenance); runtime
cross-run lookup remains non-load-bearing; future audit/conflict features may
project a fingerprint or index separately. The policy is NEVER described as
"one provider event always creates one run."

## 3.5 df-4 -- persistence form: GENERIC_OPAQUE_DOMAIN_PROVENANCE_EXTENSION_SELECTED

Part 2's precedent-based sports column on generic AgentRun is SUPERSEDED
(precedent is evidence, not authority). Selected form: one generic nullable
platform field, conceptually `DomainExecutionProvenanceJson`, holding the
envelope `{ domain: "sports", type: "selected_event_binding", schemaVersion,
payload: { providerNamespace, observedProviderEventId, observedStartUtc,
observedHomeTeam, observedAwayTeam, operationalDate, candidateGamePks,
authorizedGamePk, bindingRuleVersion, bracketContext, translationOutcome,
providerObservationTimestampUtc, frozenAtUtc } }` (sketch, not implementation
authority). Charter rationale: the generic platform owns storage, retrieval,
atomic persistence, immutability, tenant/run association, and generic
type/version metadata ONLY -- it never interprets providerEventId, gamePk,
matching rules, or sports refusals; the sports domain owns and interprets the
payload; future niches reuse the same field without one-column-per-decision
proliferation. Atomicity: same-row nullable column -> single insert with the
run row. Query behavior: opaque, non-load-bearing (3.4). Migration: one
additive nullable column -- migration REQUIRED (unchanged conclusion, new
name). Rejected: sports-specific generic-row column (perpetuates schema-level
niche coupling; precedent-only rationale disallowed); sports-owned child
record (second record breaks single-insert atomicity or adds transaction and
lifecycle overhead with no gain at current scale).

## 3.6 df-5 -- evidence schema and evolution

Every evidence record carries: domain discriminator, evidence type,
schemaVersion, bindingRuleVersion, providerNamespace, provider
adapter/normalization version where needed for reproduction, frozenAtUtc.
Bound: sports domain owns payload-schema evolution; platform stores opaquely;
readers support known historical versions; unknown versions FAIL CLOSED for
authoritative reuse (displayable as opaque preserved material only); evidence
is immutable after insert; corrected decisions create NEW decisions and new
evidence -- history is never rewritten; any future same-run retry consumes
the frozen version in place, never upgrades it. Evidence type/schemaVersion
are versioned independently of bindingRuleVersion.

## 3.7 atomic persistence (implementation obligation)

Submitted intent snapshot + authoritative gamePk + verified provenance + run
id + tenant + initial status persist atomically before execution: one
transaction or one atomic ORM save; no run without provenance when selected
intent is active; no provenance without its run; no model call before
persistence succeeds; failed persistence creates no executable run; execution
code cannot overwrite frozen provenance. Slice 2-iii-b2 must prove the save
boundary.

## 3.8 df-6 -- lifecycle: RESUBMISSION_ONLY_CURRENTLY

No same-run retry/resume/rerun endpoint exists at published af59853; the
platform pattern is retire/exclude then submit a NEW run
(`AgentRunsController.cs:1349-1350`; guard doctrine "failed -> never blocks").
Each resubmission independently observes, translates, verifies, and persists
new evidence. Implementation slices must not claim same-run retry support.
Future invariant (explicitly future-proofing, not current behavior): any
future same-run retry must reuse frozen evidence + authoritative gamePk and
never retranslate; it requires separate implementation and review authority.
Historical audit replay: read-only, persisted evidence only, zero provider or
StatsAPI calls, no mutation.

## 3.9 df-7 -- refusal durability: PRECREATE_REFUSAL_DURABILITY_LIMITED_TO_RESPONSE_AND_OBSERVABILITY

Pre-create refusals produce a typed HTTP response + correlation-linked
structured log, create no AgentRun, and are NOT durable learning-loop data
(no audit-event or failure-corpus store exists; that facility remains
separately gated). WI-0037 traceability guarantees apply fully to accepted
decisions and persisted runs; pre-run refusal learning completeness is
deferred. Named nonblocking residual:
**DURABLE_PREEXECUTION_SELECTION_DECISION_LEDGER_DEFERRED** (a future generic
decision-attempt surface or sports refusal ledger; never masquerading as an
executed run; not authorized here). All traceability/learning claims in parts
1-2 are scoped accordingly.

Refusal durability matrix: PRE-CREATE (malformed intent, not-active, event
not in schedule, ambiguity, start mismatch, pk conflict) -> typed response +
log, no run row, no durable learning record. POST-CREATE (Gate-2 staged
re-verification failure) -> run row exists with persisted intent + frozen
evidence + failure status/reason; no model call after refusal. The two
classes are never equivalent.

## 3.10 final implementation decomposition (bound)

**2-iii-a canonical matcher**: extract ProviderEventGameBindingMatcher (typed
inputs/evidence/outcomes; single-source rules and tolerances; consumed by the
existing binding workflow AND future translation); no public field, no
persistence change, no frontend change; invariant: zero behavior drift in
WI-0035/36 matching, no runs, no model calls.
**2-iii-b1 inert provenance persistence**: DomainExecutionProvenanceJson
column + envelope schema/type/version model + ownership boundary +
historical-row compatibility; NO public SelectedEventIntent, no activation;
invariant: the migration introduces no new accepted input or execution
behavior; publication gate: old code tolerates the column, rollback leaves
data intact, platform/domain boundary independently reviewed.
**2-iii-b2 atomic contract activation and enforcement**: null-suppressed
SelectedEventIntent, all-or-none validation, pre-create translation + staged
verification, guard fed the verified pk, atomic persistence, namespace
freeze, refusal handling, legacy byte-identity serialization,
resubmission-only semantics; invariants: non-null block never
accepted-and-ignored, no run/model before authority, second DH selection
creates its distinct run; gate: one active API process, migration present,
contract + enforcement together, no mixed pool.
**2-iii-c frontend and consumer continuity**: analyzer + dev-artifact-review
propagation, stubs/consumers, approved provenance presentation, offline
end-to-end DH verification, reconciliation/settlement continuity; invariant:
two selected events -> two verified pks -> two persisted decisions -> two
runs -> two settlement identities. No slice authorized.

## 3.11 rescoring and residuals

E-PRIME-PRECREATE rescored WITH the bound costs (single-host constraint,
migration, generic-provenance boundary, two-gate freshness, versioned
divergence, resubmission-only lifecycle, non-durable pre-run refusals, four
slices, scale-out follow-up): still strongest on authority correctness,
doubleheader safety, compatibility, provenance, settlement continuity, and
implementation containment. NOT scored as solved: distributed concurrency,
same-run retry, durable refusal learning. Candidate C is not recreated: no
server-minted binding reference, no registry lookup lifecycle, no request-
carried durable binding id -- evidence is historical execution provenance.
Residuals: BLOCKING before implementation = final independent delta review +
persistence-form and evidence-schema approval (this part's decisions);
BLOCKING before scale-out =
MULTI_INSTANCE_SELECTED_EVENT_ATOMICITY_REQUIRED_BEFORE_SCALE_OUT; DEFERRED =
DURABLE_PREEXECUTION_SELECTION_DECISION_LEDGER_DEFERRED and future same-run
retry; standing deployment assumption = single operator-host, controlled
cutover, no mixed pool; queryability non-load-bearing.

# part 4 -- shared creation gate, enforceable activation, and immutable provenance cardinality (fr-1..fr-3), 2026-07-26

The closing review of part 3 returned CORRECTIONS_REQUIRED (FR-1 High gate-key
authority; FR-2 Medium activation enforceability + residual scope; FR-3 Medium
provenance cardinality). Parts 1-3 are preserved; PART 4 IS CURRENT AUTHORITY
where it corrects part 3 sections 3.1 (process/deployment enforcement), 3.4
(creation-gate and divergence), 3.5-3.6 (provenance cardinality/evolution),
3.10 (2-iii-b2 obligations), and 3.11 (residual scope and scoring). All other
part-3 decisions stand.

## 4.1 fr-1 -- SHARED_TENANT_COMPETITION_CREATION_GATE_V1

The interim split-key idea (legacy matchup key + selected gamePk key) is
REJECTED: separate semaphores would not serialize a legacy request racing a
selected request for the same physical game. Bound v1 strategy: EVERY
run-creation path -- legacy and selected -- uses the SAME process-local gate
namespace keyed by server-authenticated tenant key + server-canonical
competition code (resolved and validated against the server competition
catalog BEFORE acquisition). The key contains no raw client teams, game date,
providerEventId, StartUtc, or client gamePk. The deliberately coarser
per-competition serialization is accepted for the single-process
single-operator product because only the database read/check/insert boundary
is held -- no provider, StatsAPI, model, or other network I/O occurs under
the gate. Gate 1 for selected requests still completes before acquisition
(observe -> canonical matcher -> candidate gamePk -> staged verification ->
immutable verified candidate). Inside the shared gate the selected path
evaluates duplicates with the authoritative VERIFIED gamePk,
server-observed/verified team references for any fallback comparison, and the
verified operational date where the candidate query needs one; raw client
team strings and client gamePk are validation/cross-check inputs only and
never authorize the selected-path gate or duplicate result. The legacy path
keeps its existing DuplicateRunGuard identity policy and externally visible
behavior, executing its read/check/insert boundary inside the same shared
gate. The gate is held through candidate read, conflict evaluation, atomic
run/provenance insert, and successful save. Distinct doubleheader gamePks
remain separate decisions (their short creation sections serialize); the same
authoritative gamePk follows existing duplicate policy. Selection-level
idempotency and multi-process safety remain explicitly unclaimed.

Mandatory 2-iii-b2 RED scenarios: (1) same selected event, two client team
spellings, concurrent -> exactly one run; (2) two provider events -> same
verified gamePk -> existing duplicate outcome; (3) two DH selections ->
distinct verified gamePks -> two runs; (4) selected racing legacy for the
same game, BOTH arrival orders -> exactly one active run; (5) selected-path
duplicate evaluation receives verified gamePk/server identity, never raw
client matchup identity; (6) tenants never block, disclose, or confuse one
another; (7) no provider/StatsAPI/model/network call under the shared gate.
Illustrative .NET names are PascalCase, JSON camelCase; WI/slice identifiers
remain traceability labels, never production type names.

## 4.2 fr-2 -- enforceable activation/publication gate

Verified deployment truth (cited): `platform/dotnet/Dockerfile` launches one
API process per container (single entrypoint) but does not constrain the
number of containers, revisions, or overlapping replacements;
`compose.smoke.yaml` is explicitly non-production and proves no production
topology; no production pipeline or checked-in configuration currently proves
one active process, one worker, max-replicas-one, or non-overlapping cutover.
Bound: ARCHITECTURE publication may proceed after the closing review (it
publishes an OBLIGATION, not a claim that activation exists); 2-iii-b2
remains DEFAULT-OFF and may not activate SelectedEventIntent until a
deployment evidence record proves: exactly one active run-creating API
process; one worker per process; no web garden; no overlapping app-pool
recycle; no old/new revision overlap able to receive run-creation requests;
no blue/green or mixed-version pool; no second manually started API process;
no background worker or alternate service capable of creating runs. The old
process must be stopped or unable to accept run creation before the new
enforcement-capable process accepts selected-event requests. Absent or stale
topology proof -> activation stays disabled and a complete selected block
returns `selection_identity_not_active`; never accepted-and-ignored. The
evidence record must cite the actual configuration and process/revision
observation at activation time; prose alone is insufficient. No claim is
made that application code can self-prove external topology unless a later
implementation supplies such a mechanism. Residual
MULTI_INSTANCE_SELECTED_EVENT_ATOMICITY_REQUIRED_BEFORE_SCALE_OUT is
EXTENDED to cover: multiple processes on one host; overlapping
deployment/recycle processes; multiple containers or revisions; future
background workers/services capable of run creation; horizontal scale-out.
Ownerless disclosure retained; ownership assignment required before any such
topology is authorized.

## 4.3 fr-3 -- ONE_IMMUTABLE_DOMAIN_DOCUMENT_PER_RUN

DomainExecutionProvenanceJson cardinality is zero-or-one document per run; a
selected-event run has exactly one. The owning domain constructs the COMPLETE
document before run creation; the platform persists it opaquely in the same
atomic insert as the run. The field is SINGLE-ASSIGNMENT: no later merge,
append, replacement, or last-writer-wins update to the persisted JSON.
Selected-event binding evidence can never be removed or replaced. For a
sports run only the sports domain constructs the payload; generic platform
code stores, retrieves, associates, and enforces single-assignment, never
authoring or interpreting sports fields. Another niche constructs its own
document for its own runs and may not write or replace the sports document on
a sports run. v1 document type = selected_event_binding; schemaVersion is the
sports-owned domain-document schema version, independent of
bindingRuleVersion. No separate platform envelopeVersion in v1 (generic
metadata shape fixed; changing it later is a separately reviewed contract
change). Future evidence types never mutate existing rows: a future schema
may define a versioned aggregate for NEWLY CREATED runs, fully assembled
before insert; existing documents remain byte-preserved and readable;
evidence discovered after run creation requires a separately designed
append-only child/event surface and cannot be forced into this field.
Part 3's "versioned append-only envelope extension" wording is SUPERSEDED
insofar as it suggested in-place extension of an existing document. Unknown
versions stay displayable as opaque history and fail closed for
authoritative reuse.

## 4.4 decomposition and scoring updates

2-iii-b2 owns ALL load-bearing activation behavior together:
SelectedEventIntent contract; all-or-none validation; Gate-1 translation and
verification; the shared creation gate; verified-gamePk duplicate evaluation;
atomic run/provenance persistence; Gate-2 execution verification; refusal
behavior; and the deployment activation proof. No intermediate state accepts
and ignores selected identity. Scoring updated honestly: deliberately coarser
tenant+competition serialization in single-process v1; no distributed or
multi-process safety; activation dependent on external deployment evidence;
single-assignment provenance limits; later execution-time evidence requires a
separate future surface. E-PRIME-PRECREATE remains selected: even with these
costs it is the only candidate that adds no new authority, no registry, no
token lifecycle, and no client-trusted identity while completing the
selection -> gamePk -> settlement chain.

## 4.5 documentation and semantic dispositions

Documentation: architecture report versioned additively via part 4; WI state
and link updated; rolling handoff appended; no new standalone document
required. Terminology: added SHARED_TENANT_COMPETITION_CREATION_GATE_V1,
ONE_IMMUTABLE_DOMAIN_DOCUMENT_PER_RUN, and the activation-evidence record;
deprecated the split-key wording (rejected) and part 3's in-place append
phrasing; "architecture publication" (this vault decision record on main) is
distinct from "2-iii-b2 implementation activation/publication" (default-off,
evidence-gated); "creation gate" (serialization mechanism) is distinct from
"duplicate identity authority" (verified gamePk policy);
"single-assignment immutable document" is distinct from an append-only
evidence store (a possible FUTURE separate surface). The central DAI glossary
is outside this slice's allowlist; no cross-cutting glossary edit is required
now -- these terms are WI-scoped; revisit at WI-0037 close.

## 4.6 state

Slice 2-iii: part 4 bound locally; CLOSING delta review of 191402c..tip
required; architecture publication and implementation remain unauthorized;
2-ii-c unauthorized; WI-0037 in-progress.
