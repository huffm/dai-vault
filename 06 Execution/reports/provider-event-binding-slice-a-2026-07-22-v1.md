---
title: "Provider-Event Binding Slice A -- Canonical Qualification Producer 2026-07-22 v1"
type: "evidence-report"
date: "2026-07-22"
status: "Slice A producer work locally complete through correction Pass 2 (original findings A, B, C, D, E, F all closed at the producer boundary); UNINTEGRATED and unavailable to live workflow; execution retrieval remains a separate open downstream gap; vertical still in progress"
project: "DAI"
slice: "WI-0035 provider-event binding vertical, Slice A + corrections Pass 1A/1B/2"
repos:
  dai: "local branch wi/0035-provider-event-binding, commit 4f4e726 (Pass 2); NOT pushed"
  dai-vault: "local branch wi/0035-provider-event-binding; NOT pushed"
tags:
  - evidence-operations
  - sports-v1
  - market-contrast
  - implementation
related:
  - "06 Execution/reports/provider-event-binding-seam-audit-2026-07-22-v1.md"
  - "06 Execution/reports/market-contrast-start-instant-normalization-analysis-2026-07-22-v1.md"
  - "02 Platform/system-development/work-items/WI-0035-market-contrast-candidate-screen.md"
---

# provider-event binding slice A -- canonical qualification producer

## what is now true

**Canonical qualification production is implemented locally.** One pure, versioned
authority (`AgentRuns/ProviderEventBinding.cs`) decides whether an Odds provider event may
bind to an authoritative StatsAPI candidate, and **both** producers -- the zero-quota events
gate and the paid market-screen adapter -- derive their binding facts from it.
`MarketJoinDiagnostics` is now the diagnostic projection only and retains **no independent
matching predicate**; every field it emits is derived from the single qualification result.

**A unique per-event `+/-60s` binding is not a global time-normalization rule.** The
2026-07-22 observations refuted any transform from scheduled instant to provider instant,
and that refutation stands. What Slice A adds is a per-candidate uniqueness admission:
exact teams and orientation, both instants inside the authorized eastern bracket, and
exactly one admissible event inside that candidate's own window. A fixture pins the
distinction explicitly -- one scheduled instant produces `0s` for one game and `+60s` for
another with no shared rule between them.

## binding truth table

| condition | result |
|---|---|
| unique same-orientation event, delta `0s` | qualified, `exact_start` |
| unique same-orientation event, delta `-60..+60s` nonzero | qualified, `bounded_start_tolerance` |
| `+61s`, `-61s`, `-240s` | `outside_start_tolerance`, no binding |
| reversed orientation | `orientation_mismatch`, never binds |
| either normalized team differs | `team_pair_not_found` |
| two admissible events in one window (even if one is exact) | `ambiguous_admissible_events`, fail closed |
| blank provider id, or a provider id repeated in the response | `invalid_provider_identity` |
| candidate or event outside the eastern bracket | `outside_target_date_bracket` |
| sub-second instant | `non_whole_second_start` (structural, never an exception) |
| caller context status (not attempted / source failed / preblocked) | passes through, fails closed |

Doubleheaders bind correctly: two same-team-pair events hours apart each bind to their own
candidate, because multiplicity across the response is not ambiguity -- only multiplicity
*inside one candidate's window* is.

The `60`-second limit lives in `ProviderEventBindingPolicy` only and is emitted with every
binding as `admitted_max_abs_start_delta_seconds`, alongside the deciding policy version, so
a later consumer can validate the exact producer decision rather than re-deriving it.

## contract versions

New: `provider-event-binding/1.0`, `provider-event-binding-policy/1.0`.

Bumped because matching semantics or serialized shape changed:

| contract | from | to |
|---|---|---|
| market-contrast screen bundle | 1.3 | **1.4** |
| market-contrast source adapter | 1.3 | **1.4** |
| events-gate artifact | 1.0 | **1.1** |
| events-gate operator | 1.1 | **1.2** |
| events-gate required preflight bundle | 1.3 | **1.4** |

Deliberately unchanged: `input-evidence-envelope/1.1` and `ProjectToPlannerEnvelopeJson`;
all daily-evidence planner request/board/planner/CLI contracts; all WI-0036 wildcard flight
contracts; `flight-selection-provenance/1.0`; `CompetitionMatchupInput`, the controller trust
boundary, `MarketSpreadInput`, `SportsRetriever`, the Tool Gateway handler,
`OddsMarketClient`, and `MarketSnapshot`; prompts, decisions, confidence, lean, posture,
buyer projection, settlement, and reconciliation.

## where propagation stops -- and why that is enforced, not just documented

**Binding propagation stops before the unchanged input-evidence envelope.** No newly emitted
binding is eligible for a paid flight.

This is now mechanically enforced rather than merely stated: the Python planner CLI still
accepts only `market-contrast-screen-bundle/1.3` for its replay operation, so a bundle
emitted by the new 1.4 producer is **not replay-eligible** until Slice B wires the envelope
and planner deliberately. A new paid screen would therefore produce a bundle Planner Pass 2
refuses, which is the correct fail-closed posture for an unfinished vertical -- and the
reason the chains must stay unintegrated and no live operator may use the new contract.

## what remains unsafe

The four execution-gap characterization tests remain **green and explicitly unresolved**.
Execution retrieval still:

- rebuilds market identity from team names and date;
- selects `FirstOrDefault` across **either** orientation, so a doubleheader can bind the
  wrong game;
- performs no independent verification when an event id is supplied;
- has no contract member capable of carrying a binding (`MarketSpreadInput`).

Slice A does not touch any of that. **The complete provider-event-binding vertical is still
in progress.**

## verification

| gate | result |
|---|---|
| Slice-A acceptance tests | **23/23** |
| full `DevCore.Api.Tests` | **1567/1567** |
| four execution-gap tests | **4/4 green, unresolved** |
| full agent-service pytest (no-drift; Python unchanged) | **617/617** |
| canonical determinism across fresh processes | identical |
| strict planning snapshot | 25 work items / 6 timeline entries / 0 warnings |
| `git diff --check` both repos | clean |
| added-line scans | no machine paths, secrets, authority grants, live calls, or stale versions |

RED evidence: the acceptance suite failed to compile against the missing authority before
implementation. Nine pre-existing tests then failed because they pinned the superseded
exact-`0s`-only predicate; each was updated to the new policy with a dated note, and the
corpus equivalence test was retargeted to an independent reference implementation of the
current policy rather than deleted.

## posture

No StatsAPI, Odds `/events`, Odds `/odds`, model, Tool Gateway, or other network call. No
paid call, AgentRun, capture, flight freeze, settlement, reconciliation, scheduling, or
activation. No database read or write, migration operation, or service start. No
integration, push, or remote mutation. **$0.**

The persisted gamePk `823438` canary remains historical non-wildcard, non-settled,
pre-binding evidence -- not backfilled, rewritten, or reinterpreted.

## next

**Slice B is binding propagation through input-evidence -> Planner Pass 2 board with
producer replay** -- not execution activation. Slices C-D then carry the binding to flight
provenance and finally enforce it at execution retrieval. No live operator, capture,
wildcard flight, or paid use is authorized by Slice A.

---

## addendum 2026-07-22 -- correction pass 1B (findings C and D)

Pass 1B corrects the two accounting/vocabulary findings raised against Slice A. It is a
producer-side correction only: **no binding block is emitted on the wire, no planner
propagation exists, and execution is still unbound.** Findings A and E remain open for
Pass 2; Slice B is not started.

### finding C -- the counts were conflated

The projection assigned `ExactMatchCount` from `q.Binding is not null ? 1 : q.AdmittedEventCount`.
Because Slice A admits a unique same-orientation event anywhere inside the inclusive
+/-60s policy window, **a bounded admission was published as an exact match** on both the
free events-gate artifact and the paid screen bundle. A reader could not distinguish
"the provider agreed to the second" from "the provider was 45s off and we accepted it".

Three independent counts now exist on `ProviderEventQualification` and are carried
verbatim onto `MarketJoinDiagnostic`:

| count | meaning |
|---|---|
| `ExactStartMatchCount` | literal zero-delta agreement, counted inside the in-bracket loop |
| `AdmittedEventCount` | events inside the authorized bracket **and** the inclusive +/-60s window; may exceed 1, which is ambiguity |
| `QualifiedBindingCount` | `1` only when exactly one admissible event produced a binding, else `0` |

`MarketJoinDiagnostic.ExactMatches` is renamed `AdmittedEvents`: those events were admitted
by the policy window, which is not a claim of exactness.

Selection is now guarded on qualification rather than on collection shape. The paid adapter
requires **all three** of `QualifiedBindingCount == 1`, `AdmittedEvents.Count == 1`, and
`Binding is not null` before reading market depth; a collection that happens to hold one
element is no longer sufficient. The events gate classifies readiness on
`QualifiedBindingCount`, and `matched_odds_event_id` is gated on the same fact.

Both artifacts emit `admitted_event_count`, `exact_start_match_count`, and
`qualified_binding_count` separately, and both summaries keep
`evaluated_candidate_exact_start_match_count` and
`evaluated_candidate_qualified_binding_count` as distinct totals.

### finding D -- the canonical status was not carried

`MarketJoinDiagnostic.CanonicalStatus` is copied **directly** from the qualifier and emitted
as `canonical_qualification_status`. It is never reconstructed from counts and never routed
through the legacy vocabulary. `diagnostic_status` is retained only as a secondary legacy
view for historical readers and must never contradict it.

`duplicate_exact_match` is replaced by `ambiguous_admissible_events`. An ambiguous window may
hold zero or one exact event, so the old token asserted something that was frequently false.
The terminal vocabulary follows:

| old | new |
|---|---|
| `exact_match_ready_for_separate_operator_decision` | `qualified_binding_ready_for_separate_operator_decision` |
| `no_exact_matches` | `no_qualified_bindings` |
| `duplicate_exact_match_blocked` | `ambiguous_qualified_binding` |

Explanations are written from the canonical status and never call a bounded admission exact;
the qualified-but-bounded explanation states `bounded tolerance, not an exact start` with the
signed delta, and the paid join detail states the match method rather than always claiming
`teams+start exact`.

### also corrected

`ProviderEventQualificationContext.TargetDate` is **removed**. It duplicated
`AuthorizedBracket.TargetDate` as a second independently mutable representation of the same
authorized fact. It was removed rather than left required-but-ignored, so the two can no
longer disagree.

### verification

| gate | result |
|---|---|
| new Pass-1B accounting suite (`ProviderEventAccountingTests`) | **24/24** |
| new producer-boundary tests (events gate + paid adapter) | **4/4** |
| binding/join/contrast focused suites | **240/240** |
| full `DevCore.Api.Tests` | **1608/1608** |
| four execution-gap tests | **4/4 green, unresolved** |
| full agent-service pytest (Python unchanged) | **617/617** |
| strict planning snapshot | 25 work items / 6 timeline entries / **0 warnings** |
| `git diff --check` both repos | clean |
| stale-vocabulary scan | no `duplicate_exact_match` / `ExactMatches` / `ExactMatchReady` token remains in production or test names |
| added-line scans | no machine paths, secrets, authority-ledger changes, or live-call surfaces |
| protected hashes | `DevCore.Data.csproj` unchanged (`63EF2488...`) |

RED evidence: **eight pre-existing tests failed immediately on the production change**, and
they failed exactly at the conflation seam -- four qualifier rows asserting
`ExactMatchCount == 1` for +/-30s and +/-60s admissions, two producer fixtures asserting an
exact count on a boundary admission, the corpus-equivalence test comparing the admissible
predicate against the exact count, and the ambiguity fixture pinned to the old token. Each
was corrected to state the separated facts. The new accounting truth table is written so its
rows **disagree with each other by construction** -- one exact plus one bounded is ambiguous
with exactly 1 exact; two bounded is ambiguous with **0** exact -- so no row can be satisfied
by copying one count into another.

Known deviation: within the operator-specified batch order, the four dedicated Pass-1B test
files were authored after the production edits in the same batch. The eight failing
pre-existing tests are the honest RED signal for this pass; the new tests are additive
pinning, not the original failure evidence.

### left deliberately unchanged

The schema-1.2 fixture in `services/agent-service/tests/test_daily_evidence_planner_cli.py`
still uses `evaluated_candidate_exact_match_count`. That fixture is version-pinned to the
older bundle shape and asserts backward compatibility; rewriting it would falsify what 1.2
actually looked like. There is **no production Python or Angular reader** of any of these
keys -- verified by scanning `services/agent-service/src` and `apps/`.

### posture

No StatsAPI, Odds `/events`, Odds `/odds`, model, Tool Gateway, or other network call. No
paid call, AgentRun, capture, flight freeze, settlement, reconciliation, scheduling, or
activation. No database read or write, migration operation, or service start. No integration,
push, or remote mutation. **$0.**

The persisted gamePk `823438` canary remains historical non-wildcard, non-settled,
pre-binding evidence.

### next

**Pass 2 -- findings A and E**: emit the binding block on both artifacts and pin the
fingerprint, then Slice B carries it through input-evidence into the Planner Pass 2 board.
Execution correction stays gated behind those. Nothing here authorizes a paid call, a
wildcard flight, or any live operator action.

---

## addendum 2026-07-22 -- correction pass 2 (original findings A and E)

Pass 2 closes the two remaining **original producer findings**. It is a producer-side
correction only: **no planner propagation exists, no flight or provenance carries a
binding, and execution is still unbound.**

### terminology correction (binding for all active readers)

Earlier Pass-1B wording in the appended handoff called finding E "execution-path
correction". **That label was wrong and is corrected here.** The earlier text is preserved
unedited as historical evidence under the append-only rule; this addendum is the active
disposition.

| label | what it actually is | status |
|---|---|---|
| original **Finding A** | neither versioned producer candidate wire emitted the canonical `provider_event_binding` block | **closed locally by Pass 2** |
| original **Finding E** | the binding fingerprint was not on any wire, and the nested binding authority ledger was an incomplete four-key subset with no pinned closed contract | **closed locally by Pass 2** |
| execution retrieval | team/date fallback, doubleheader mis-binding, missing `MarketSpreadInput` binding member, no by-ID re-verification | **separate, open, downstream — NOT finding E** |

The four execution-gap characterization tests remain **green and explicitly unresolved**.
Renaming that gap as finding E would have implied Pass 2 addressed it. It did not.

### the three concepts are now separate in code

| concept | method | what it is |
|---|---|---|
| canonical **content** | `ProviderEventBinding.CanonicalContentJson` | the complete closed binding facts **without** the fingerprint |
| content **fingerprint** | `ProviderEventBinding.ComputeFingerprint` | lowercase 64-hex SHA-256 over the UTF-8 bytes of exactly that content |
| canonical **wire** | `ProviderEventBinding.CanonicalWireJson` | the same facts **plus** `binding_fingerprint_sha256` |

The fingerprint field never participates in its own hash, so the calculation is not
recursive. Both key orders are written by hand rather than delegated to a serializer, so
ordering is not serializer-dependent. `binding_fingerprint_sha256` sorts between
`authority_ledger` and `binding_schema_version`, making the wire order the content order
with exactly one insertion.

### the emitted wire, pinned

Exact-start case (sanitized fixture; fingerprint computed **outside** the test process and
asserted byte for byte against both producers):

```json
{"admitted_event_count":1,"admitted_max_abs_start_delta_seconds":60,"authority_ledger":{"capture_authorized":false,"concrete_tool_selection":false,"execution_authorized":false,"mutation_authorized":false,"scheduling_authorized":false,"screening_authorized":false,"settlement_authorized":false,"tuning_authorized":false},"binding_fingerprint_sha256":"b0b6f2753f02f7b83deaeb7f3586c806657d2662fc9b5a31fc3608d219111d77","binding_schema_version":"provider-event-binding/1.0","bracket_from_utc":"2026-07-22T04:00:00Z","bracket_to_utc":"2026-07-23T04:00:00Z","candidate_away_team_ref":"away-824001","candidate_home_team_ref":"home-824001","candidate_scheduled_start_utc":"2026-07-22T23:10:00Z","external_game_id":"824001","match_method":"exact_start","observed_at_utc":"2026-07-22T14:30:00Z","policy_version":"provider-event-binding-policy/1.0","provider_commence_time_utc":"2026-07-22T23:10:00Z","provider_event_id":"odds-evt-0","provider_name":"odds_api","same_orientation_team_pair_count":1,"source_provider":"mlb_statsapi","start_delta_seconds":0,"target_date":"2026-07-22"}
```

Bounded `+60s` case differs in exactly three fields and therefore in the fingerprint:
`"match_method":"bounded_start_tolerance"`,
`"provider_commence_time_utc":"2026-07-22T23:11:00Z"`, `"start_delta_seconds":60`,
fingerprint `8911c44be22fa20a339990a80d808360e7f6af11df04e9c5f669dc257b6b118c`. A bounded
binding still reports the **literal** `exact_start_match_count` of `0`.

Null case (every non-qualifying candidate):

```json
"provider_event_binding": null
```

### finding E -- the closed eight-key ledger

The four-key subset (`capture` / `execution` / `mutation` / `screening`) is replaced by the
closed WI-0036 vocabulary, all eight keys always present, every value the JSON boolean
`false`:

```text
capture_authorized      concrete_tool_selection  execution_authorized  mutation_authorized
scheduling_authorized   screening_authorized     settlement_authorized tuning_authorized
```

This is the same canonical key set the WI-0034/WI-0036 safety ledgers use
(`FlightSelectionProvenance.LedgerKeys`), so a reader never has to learn a second authority
dialect. The incomplete subset was the real defect: it let a reader infer that an unlisted
authority was merely *unstated* rather than *explicitly denied*.

The fingerprint covers the complete nested eight-key ledger.

**The enclosing operation ledgers are untouched and remain separately versioned.** The
screen bundle keeps its own six-key ledger; the events-gate artifact keeps its own eight-key
(different) ledger. The nested binding ledger does not silently replace either, and both are
asserted explicitly.

### the independent wire validator

`ProviderEventBindingWire.Validate` reads an emitted wire **as data**. It never calls the
emitter and never compares against a re-rendered expected string, so it detects mutations a
producer-side round trip would hide. Ledger mutation matrix, each **rehashed** so the
attacker's fingerprint is internally consistent:

| mutation | verdict |
|---|---|
| a key missing | rejected |
| an extra key outside the closed set | rejected |
| a duplicate key | rejected (strict parse; no last-value collapse) |
| a renamed key | rejected |
| a string-valued entry (`"false"`) | rejected |
| a null entry | rejected |
| `execution_authorized: true` | rejected |
| absent / uppercase / short fingerprint | rejected |
| an extra or missing top-level wire key | rejected |

**A recomputed fingerprint is integrity evidence, never authority and never producer
authenticity.** Authority is decided by the closed vocabulary and the all-false rule, never
by hash agreement.

### finding A -- one shared wire from both producers

Both candidate writers emit exactly one key, `provider_event_binding`, whose value comes
**verbatim** from the shared `ProviderEventBinding.EmittedWireJson`. Neither writer
hand-composes the block, adds a field of its own, or parses and reserializes it through a
second serializer.

The complete object is emitted **only** when every internal invariant agrees:
`QualifiedBindingCount == 1`, a non-null `Binding`, and exactly one admitted event stated
consistently by *both* the count and the admitted-event list. Everything else emits explicit
JSON `null`: unqualified, ambiguous, outside-window, reversed, invalid-identity,
bracket-rejected, non-whole-second, preblocked, not-attempted, source-failed — and, notably,
an internally inconsistent state, which **fails closed rather than emitting either a
plausible block or a caller-invented repair**.

Under identical frozen binding inputs the paid screen and the free events gate emit
**byte-identical** binding objects, proven on exact, `+60s`, and `-60s` cases.

### versions -- finalized, not re-bumped

Live source inspection found no contract reason the corrected wire cannot honestly be the
shape these already-announced, never-integrated versions always intended to carry, so they
are **pinned as the final Slice-A versions** rather than bumped again:

| contract | final Slice-A version |
|---|---|
| provider-event binding schema | `provider-event-binding/1.0` |
| provider-event binding policy | `provider-event-binding-policy/1.0` |
| market-contrast screen bundle | `market-contrast-screen-bundle/1.4` |
| market-contrast source adapter | `market-contrast-source/1.4` |
| events-gate artifact | `market-contrast-events-gate/1.1` |
| events-gate operator | `market-contrast-events-gate-operator/1.2` |
| events-gate required preflight bundle | `market-contrast-screen-bundle/1.4` |

One stale comment was corrected: the events-gate profile still claimed its "artifact schema
stays 1.0" while the constant already read `1.1`. Historical integrated 1.3/1.0 artifacts
remain history and were not rewritten.

### verification

| gate | result |
|---|---|
| new binding-wire suite (`ProviderEventBindingWireTests`) | **22/22** |
| new producer-wire suite (`ProviderEventBindingProducerWireTests`) | **34/34** |
| focused binding/join/contrast suites | **294/294** |
| full `DevCore.Api.Tests` | **1664/1664** (1608 baseline + 56 new) |
| four execution-gap tests | **4/4 green, explicitly unresolved** |
| full agent-service pytest (Python untouched) | **617/617** |
| fingerprint stability across a fresh process | pinned bytes computed outside the test process match both producers |
| strict planning snapshot | 25 work items / 6 timeline entries / **0 warnings** |
| `git diff --check` both repos | clean |
| protected hash `DevCore.Data.csproj` | `63EF2488...` unchanged |
| binding-token containment | `provider_event_binding` / `binding_fingerprint_sha256` appear in **no** buyer, envelope, planner, flight, controller, Python, or Angular file |
| authority-grant scan | no `*_authorized: true` on any added production line; the only `true` values are inside two negative tests that assert rejection |
| added-line scans | no machine paths, secrets, or live-call surfaces |

**RED evidence.** The acceptance suite failed to **compile** against `54873b3`:
`CanonicalContentJson`, `ComputeFingerprint`, `CanonicalWireJson`, `EmittedWireJson`, and
`ProviderEventBindingWire` did not exist. A second, unplanned red then caught a **real
defect**: the byte-parity and exact-start producer tests passed alone but failed when run
alongside the adapter suite, because `MarketContrastSourceGate` holds the paid single-use
run lease in process-wide static state and xUnit parallelizes across classes. One class was
consuming another's lease and turning a real paid path into a spurious `lease_unavailable`.
Both classes are now joined into one non-parallel collection.

### what remains unsafe and unfinished

Execution retrieval is **untouched** and remains a separate open downstream gap:

- market identity is still rebuilt from team names and date;
- `FirstOrDefault` still selects across **either** orientation, so a doubleheader can bind
  the wrong game;
- an id-supplied retrieval still performs no independent re-verification;
- `MarketSpreadInput` still has no member capable of carrying a binding.

The `1.4` bundle also remains **mechanically unusable** at Planner Pass 2: the Python replay
validator still accepts only `market-contrast-screen-bundle/1.1`, `/1.2`, and `/1.3`. A
bundle from the new producer is refused, which is the correct fail-closed posture for an
unfinished vertical.

### posture

No StatsAPI, Odds `/events`, Odds `/odds`, model, Tool Gateway, or other network call. No
paid call, AgentRun, capture, flight freeze, settlement, reconciliation, scheduling, or
activation. No database read or write, migration operation, or service start. No
integration, push, PR, remote branch, or history rewrite. **$0.**

Slice A producer work is **locally complete**, but remains **unintegrated and unavailable to
live workflow**. The persisted gamePk `823438` canary remains historical non-wildcard,
non-settled, pre-binding evidence.

### next

**Slice B** carries the binding through `input-evidence-envelope` into the Planner Pass 2
board with producer replay. Slices C-D then reach flight provenance and finally enforce the
binding at execution retrieval. No paid flight, wildcard capture, settlement, or runtime use
is authorized by Pass 2.
