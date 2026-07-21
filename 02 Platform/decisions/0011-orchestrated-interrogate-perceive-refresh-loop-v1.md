# decision 0011: orchestrated interrogate-perceive refresh loop v1

**date:** 2026-07-21
**status:** accepted (WI-0036 Slice 1; docs only; target architecture bound, no runtime change, no activation, no authorization granted)

## decision

The canonical governed refresh loop of the Cognitive Protocol Runtime is:

`Question -> Probe -> authorized retrieval -> Perceive refresh -> Verify -> Discern`

with these binding authority rules:

1. **Interrogate is a requester, never an execution authority.** `Interrogate.Question`
   names the consequential gap; `Interrogate.Probe` proposes; `Interrogate.Verify`
   dispositions. None of the three fetches, executes, or mutates.
2. **An orchestrator mediates the loop.** The orchestrator -- platform code, never a
   cognitive station -- owns authorization, re-entry, idempotency, Tool Gateway routing,
   audit, cost, and termination. Retrieval happens only through platform-owned
   retrieval/Tool Gateway boundaries (for the probe-refresh seam, constrained to
   `platform.retrieve` per the standing readiness review).
3. **Direct Interrogate-to-Perceive self-invocation remains forbidden.** This decision
   re-affirms deferred-runtime-decisions ledger entry 1 and the tool-gateway doctrine's
   fail-closed permission model; nothing here relaxes either.
4. **Refreshed evidence re-enters Verify before Discern.** A Perceive refresh produced by
   an authorized retrieval must pass back through `Interrogate.Verify` before Discern
   re-weighs. The shipped dormant probe-refresh chain does NOT do this today -- verified in
   source 2026-07-21, `ProbeRefreshChainAssembly` proceeds from Perceive intake directly
   to Discern/Decide/Synthesize preview with no Verify re-entry step. That is recorded as
   a target-contract gap (ledger entry 27), owned by WI-0036 Slice 5; it is not claimed to
   exist.
5. **Preflight is conceptually part of Perceive.** In target doctrine, the daily evidence
   orchestrator's preflight is Perceive-side work: it discovers and binds what evidence and
   candidate conditions are available before paid execution. Current implementation truth
   is different and stays truthfully documented: preflight today is the offline
   WI-0034/WI-0035 planner/screen path in `services/agent-service` and the market-contrast
   operators, while the runtime Perceive surface is the sports retrieval path. The
   architectural statement binds the target, not a claim about current code.
6. **Interrogate proposes signal needs; it does not add sources.** Discovery output is the
   proposal-only `SignalNeedProposal` concept (name tentative; contract in
   `02 Platform/architecture/cognitive-factory/wildcard-evidence-discovery-loop-v1.md`),
   whose authority fields are all false by contract. The execution-safe `ProbeRequest`
   keeps its closed template set. Source/retrieval additions are later, separately
   reviewed work items.

## independent protocol callability (target architecture)

Protocols and stations should be architected so they may eventually be invoked through the
full orchestrated pipeline OR as independently callable governed services by other system
services, under one contract:

- the shared versioned decision artifact remains the unit that moves;
- protocol and station calls accept bounded typed inputs and return bounded typed outputs;
- tenant, run, artifact version, correlation, idempotency, station-card version,
  authority, cost, trace, and termination state are explicit;
- the same protocol contract serves both invocation paths (same station cards, same
  permissions, same trace shape);
- remote callability never bypasses the Tool Gateway or station permissions; and
- a refresh cannot overwrite protected artifact fields or silently mutate confidence,
  posture, lean, identity, or history.

This is a target-architecture statement only. It grants no authority to add an endpoint,
split the current single analyze model call into station calls, or activate the dormant
probe-refresh chain. Any endpoint is its own runtime-surface slice under TDD and the
activation-ladder rules in
`02 Platform/architecture/cognitive-factory/cognitive-factory-runtime-activation-readiness-v1.md`.

## what this decision does not do

- does not change any runtime code, test, prompt, schema, config, or permission;
- does not activate the probe-refresh chain, the Stage-2 endpoint's scope, or any station
  runtime;
- does not widen `ProbeRequest`, station permissions, or Tool Gateway permissions;
- does not authorize retrieval, capture, generation, settlement, spend, or the July 22
  events-gate action (which remains separately governed);
- does not implement `SignalNeedProposal`, the wildcard preflight planner, or any service
  seam; and
- does not change confidence, posture, lean, buyer copy, pricing, Stripe, entitlements, or
  tenant behavior.

## why this matters

The dormant probe-refresh chain proved the seams (typed request, decision, authorization,
executor, intake, audit) but was built before the loop's cognitive re-entry contract was
settled; without decision authority, a future activation slice could wire
retrieval-refresh-reweigh while skipping Verify, silently converting unverified refreshed
context into decision input. Binding the loop order and the orchestrator's ownership now
lets WI-0036's implementation slices land against a stable target, exactly as decision
0004 did for the protocol vocabulary.

## references

- `02 Platform/decisions/0004-cognitive-protocol-runtime.md`
- `02 Platform/architecture/cognitive-factory/wildcard-evidence-discovery-loop-v1.md`
- `02 Platform/architecture/cognitive-factory/probe-refresh-chain-activation-readiness-v1.md`
- `02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md` (entries 1, 13, 27)
- `02 Platform/architecture/tool-gateway-and-agent-permissions-doctrine-v1.md`
- `02 Platform/architecture/cognitive-factory/protocol-station-blueprint-v1.md`
- `02 Platform/system-development/work-items/WI-0036-wildcard-evidence-discovery-loop-v1.md`
- source (verified 2026-07-21): `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshChainAssembly.cs`,
  `DevCore.Api/AgentRuns/ProbeRequest.cs`, `DevCore.Api/AgentRuns/CognitiveProtocolBuilder.cs`
