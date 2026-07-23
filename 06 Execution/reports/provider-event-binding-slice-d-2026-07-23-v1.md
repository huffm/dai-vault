---
title: "Provider-Event Binding Slice D -- Execution-Retrieval Enforcement 2026-07-23 v1"
type: "evidence-report"
date: "2026-07-23"
status: "Slice D implemented locally and green on both runtimes; full A-D vertical LOCAL and UNINTEGRATED; authorizes no flight, capture, or runtime use; ready for independent review and operator-authorized integration"
project: "DAI"
slice: "WI-0035 provider-event binding vertical, Slice D (execution-retrieval enforcement: MarketSpreadInput binding member, gateway-handler validation, retriever carry, bound client re-verification, controller trust boundary)"
repos:
  dai: "local branch wi/0035-provider-event-binding; NOT pushed"
  dai-vault: "local branch wi/0035-provider-event-binding; NOT pushed"
tags:
  - evidence-operations
  - sports-v1
  - market-contrast
  - wildcard-flight
  - implementation
related:
  - "06 Execution/reports/provider-event-binding-slice-c-2026-07-22-v1.md"
  - "06 Execution/reports/provider-event-binding-seam-audit-2026-07-22-v1.md"
---

# provider-event binding slice D -- execution-retrieval enforcement

## what is now true

The frozen provider-event binding now CONSTRAINS execution retrieval instead of merely
accompanying it. Once a run carries a validated binding, market retrieval selects EXACTLY
the bound Odds provider event, independently re-verifies current provider state against the
frozen binding through the one existing qualification authority, and fails closed -- as a
typed, distinguishable integrity failure, never a benign "no market" -- on any missing
event, identity drift, substitute event, duplicate id, malformed wire, or unsupported
competition. The four execution-gap tests that previously DOCUMENTED the unsafe behavior
now ENFORCE the corrected behavior.

## ownership

WI-0035 owns the provider-event identity-integrity invariant end to end. WI-0036 is the
downstream flight vertical that consumes the binding; the board-2.3 migration (Slice C) was
part of preserving the WI-0035 invariant through the WI-0036 consumer. Slice D completes
LOCAL BRANCH implementation through execution retrieval. Neither work item is integrated,
closed, or pushed; the separate review-and-integration gate has not run.

## the enforced execution chain

request (flightSelection.providerEventBinding, verbatim bytes)
-> controller entry trust boundary: strict `ProviderEventBindingWire.Parse` + identity
   consistency vs gamePk / flight-selection identity / game date / competition, 400 BEFORE
   the run row on any drift
-> `AgentRunService`: raw wire bytes (GetRawText, never reserialized) onto the artifact
-> `SportsRetriever`: carries the wire verbatim into the tool input; refuses a binding on
   any non-mlb competition; post-retrieval defense re-parses and compares grounded identity
-> `MarketBaseballSpreadHandler` (tool gateway): re-validates through the ONE strict parser;
   malformed wires fail closed before any provider read; football/basketball handlers
   REJECT a supplied binding (no silent ignore)
-> `OddsMarketClient.GetBaseballSpreadByBindingAsync`: re-runs `ProviderEventQualifier.Qualify`
   (no second predicate) against the candidate + bracket frozen in the binding; requires
   Qualified AND the same provider event id; every failure throws
   `ProviderEventBindingIntegrityException` with a closed status
-> controller final defense: a bound execution without recorded `verified` market-binding
   agreement is failed truthfully (run row `failed` + 422), never returned as a valid
   buyer-facing result.

Unbound retrieval (every ordinary run) is hardened the same way screening decides: exact
caller orientation only (reversed listing never binds) and refusal on same-team-pair
doubleheader ambiguity (no first-match selection). The near-close by-id capture path now
refuses duplicate provider ids instead of picking the first occurrence.

## test conversion (RED -> enforcement)

The four `ProviderEventBindingGapTests` characterization tests were converted to twelve
enforcement tests. RED evidence: the doubleheader-refusal and reversed-orientation
conversions were run against unmodified production code and failed for exactly the intended
reason (first-listed event still bound; reversed orientation still bound) before any
production change; the binding-member and bound-path conversions could not compile against
the old contract (no member, no bound method), which is the intended red for a
contract-shape change. After implementation: all twelve green, plus three new
`SportsRetrieverTests` (bound carry + verified status; bound-absent fails closed; non-mlb
binding refused) and three new `FlightSelectionProvenanceEndpointTests` (mutated binding
400 before any row; missing gamePk 400 before any row; unverified bound execution 422 with
failed row) through the real ASP.NET host.

Enforcement now proven executably: binding required at bound execution; exactly the bound
event retrievable; doubleheader/first-match selection impossible; reversed orientation
rejected; missing event fails closed; identity drift under a stable id fails closed;
substitute qualifying event fails closed; duplicate ids refused on both paths; malformed
wire fails at the gateway boundary; no construction site can silently omit the binding
member (no default value -- proven by reflection); no compatibility overload reintroduces
unbound execution; binding bytes reaching retrieval are verbatim the flight-provenance
bytes (GetRawText round-trip, same fingerprint).

## files changed (dai, 9 production + 4 test)

- `platform/dotnet/DevCore.Api/AgentRuns/ProviderEventBinding.cs` -- closed verification
  vocabulary + typed `ProviderEventBindingIntegrityException` (closed statuses:
  malformed_binding, unsupported_bound_competition, source_failure, bound_event_mismatch,
  plus qualifier statuses passed through verbatim).
- `platform/dotnet/DevCore.Api/Sports/OddsMarketClient.cs` -- bound retrieval
  `GetBaseballSpreadByBindingAsync` (internal; only reachable with a strictly-parsed
  binding; uncached, evidence-grade); extracted single day-event fetch authority; hardened
  matching (exact orientation only, ambiguity refused, duplicate ids refused).
- `platform/dotnet/DevCore.Api/Tools/Handlers/MarketSpreadHandlers.cs` --
  `MarketSpreadInput` + `ProviderEventBindingWire` member with NO default; baseball handler
  validates + dispatches bound; football/basketball handlers reject a binding.
- `platform/dotnet/DevCore.Api/AgentRuns/SportsRetriever.cs` -- non-mlb bound refusal,
  verbatim carry, defense re-check, verification recording.
- `platform/dotnet/DevCore.Api/AgentRuns/SportsRunArtifact.cs`,
  `IAgentRunService.cs`, `AgentRunService.cs` -- binding wire in (WI-0009 pattern),
  `MarketBindingVerification` out; run-row-adjacent only, never OutputJson.
- `platform/dotnet/DevCore.Api/Controllers/AgentRunsController.cs` -- entry validation +
  final 422 defense.
- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshExecutor.cs` -- explicit unbound.
- tests: `ProviderEventBindingGapTests.cs` (converted), `SportsRetrieverTests.cs`,
  `FlightSelectionProvenanceEndpointTests.cs`, `ToolGatewayMarketSpreadTests.cs`.

## validation proof

full `DevCore.Api.Tests` **1760/1760** (prior baseline 1746; +14 = converted/new
enforcement tests, 0 skipped); full agent-service pytest **691/691** (unchanged -- no
python edit, no cross-runtime vector change, no schema bump anywhere); focused runs:
gap-test class 12/12, retriever+gap+gateway 33/33, endpoint class 15/15 (including the
pre-existing bound reserve cross-runtime vector now passing the new strict controller
validation); `git diff --check` clean; `DevCore.Data.csproj` protected hash `63EF2488...`
unchanged; no machine-absolute path introduced; changed-file set exactly the declared
change map.

## db writes / external side effects / cost

None. **$0.** Zero StatsAPI, `/events`, `/odds`, model, or network calls (all tests fake
http). Zero database access, capture, settlement, scheduling, or activation.

## what did not change

Board/planner/request/replay/flight-plan/provenance schemas (no version bump -- the binding
already traveled in flight-selection-provenance/1.1; WI-0034/WI-0036 shared-planner coupling
untouched); cross-runtime vectors (byte-identical, untouched); the artifact OutputJson
contract and every buyer surface (verification status is run-row-adjacent metadata only);
python agent-service; WI-0036 flight semantics; migrations; prompts.

## residual risks (self-review; all tests green)

- `GetBaseballSpreadByEventIdAsync` (near-close capture) remains a by-id compatibility
  surface WITHOUT frozen-binding re-verification -- its caller has only a provider event id
  from market memory, no binding. hardened for duplicate ids; identity drift under a stable
  id on the CAPTURE path is a documented post-slice candidate, not an execution-path gap.
- unbound team/date hardening changes live behavior for any caller that previously matched
  with swapped home/away or on a doubleheader day: those now return ungrounded (degraded
  run) instead of guessing. this is the enforced screening semantics, but it is a real
  behavioral change on the buyer path.
- a refused unbound match (ambiguity) is cached null for 15 minutes, same as the
  pre-existing no-match caching.
- the controller 422 path persists the failed run row without the composed OutputJson
  (ErrorMessage carries the diagnosis).
- this is implementation SELF-review only; no independent review has run.

## commits

- dai `85af96d` on `wi/0035-provider-event-binding` (one coherent Slice D implementation
  commit; parent `88092eb`; 13 files, 838 insertions, 114 deletions).
- dai-vault: this report + current-slice append (one docs commit; parent `e9c0731`).
- Nothing merged, nothing pushed, no PR, no remote branch created; both mains unmoved
  (dai `48a2931`, vault `3a82af0`).

## review checklist and recommended integration order

1. independent review of the full A-D vertical (binding wire, replay gate, planner carry,
   provenance, execution enforcement) -- adversarial focus: escape paths, overloads,
   orientation guessing, byte mutation, exception translation, shared-planner coupling.
2. correction commits on the same branch if findings demand.
3. fast-forward integration: dai first (single branch, tests green at tip), then vault
   record; verify both suites green post-ff.
4. push only if separately and explicitly authorized.

## next authorization boundary

Independent review + operator-authorized integration of the A-D vertical. No flight, no
capture, no paid call, no push is authorized by this slice.
