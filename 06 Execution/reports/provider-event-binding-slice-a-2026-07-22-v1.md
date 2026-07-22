---
title: "Provider-Event Binding Slice A -- Canonical Qualification Producer 2026-07-22 v1"
type: "evidence-report"
date: "2026-07-22"
status: "Slice A implemented locally; vertical still in progress; execution remains unsafe for bound flights"
project: "DAI"
slice: "WI-0035 provider-event binding vertical, Slice A"
repos:
  dai: "local branch wi/0035-provider-event-binding, commit 2e24782; NOT pushed"
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
