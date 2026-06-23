# tool gateway and agent permissions doctrine v1

**status:** active doctrine -- canonical anchor over the tool/permission charter, the station blueprint, and the probe-refresh readiness review
**date:** 2026-06-23

## purpose

State, in one canonical place, how DAI governs tool access and agent permissions: what the Tool Gateway is, how capabilities are granted, what an agent/station may and may not do, and how DAI avoids implicit agent power. The charter (`security-and-permissions.md`) states the principles; this doc ties them to the implemented mechanism so the relationship between principle and enforcement is explicit.

## problem it solves

The permission rules are stated as a short charter, and the actual mechanism (`ToolGateway`, `IProtocolToolAccessPolicy`, `AllowedProtocolNodes`, the dormant probe-refresh chain) is spread across the station blueprint, the probe-refresh readiness review, and code. An agent reading one piece can miss the whole: e.g. assume a model can request a tool and get it, or that the probe-refresh chain can mutate an artifact. This doctrine fixes the relationship so capability boundaries are not re-derived (or quietly widened) per slice.

## strategic fit

The factory only scales safely if workers have exactly the capability their station needs and no more. The Tool Gateway is the capability boundary of the factory floor: it is what lets DAI add tools, sources, and (later) memory without any of them becoming implicit agent power or a cross-tenant leak. It protects the same long-term stock as the rest of the platform -- buyer and tenant trust -- by making capability explicit, scoped, and observable.

## mental model

Tools are shared platform infrastructure; agents/stations differ by role, not by exclusive tool ownership (`orchestration.md`). Every tool call passes through one gateway that fails closed. A station does not choose its tools; its permission is declared config, and the model filling a station never selects tools or widens access. Capability flows one way: the platform grants a scoped, observable capability to a station; the station cannot grant itself more.

Three layers:
- capability declaration -- each tool's `ToolDefinition` declares `AllowedProtocolNodes` (which caller nodes/stages may invoke it) plus metadata (kind, transport, cost class, idempotency, secrets scope).
- enforcement -- `ToolGateway.InvokeAsync` checks the registry, then `IProtocolToolAccessPolicy.IsAllowed(definition, callerNode)`; either failing throws and dispatches nothing.
- observation -- every invocation emits one structured `ToolGatewayInvocation` log (tool id, protocol node, run/tenant/correlation ids, cost class, outcome, duration).

## what it is

- The canonical statement of DAI's tool/permission model and its enforcement seam.
- A controlled capability boundary: tools reachable only through the gateway, fail-closed.
- The rule set for safe, default-off execution of dormant capabilities (probe-refresh chain).
- An anchor naming which doc/code owns each piece.

## what it is not

- Not a general plugin layer or an agent autonomy layer. The gateway is a governed boundary, not a marketplace.
- Not a claim that all listed governance is enforced today. v1 enforces `AllowedProtocolNodes` only; idempotency caching, cost-class rate limiting, correlation-header injection, and tenant-tier enforcement are declarative/deferred (`IToolGateway.cs`, `ToolDefinition.cs`).
- Not a runtime change. This is doctrine over implemented behavior.
- Not authorization to activate the dormant probe-refresh chain (still deferred).

## approved uses

- Adding a tool: register a `ToolDefinition` with an explicit `AllowedProtocolNodes` set; reach it only through the gateway; rely on the fail-closed default.
- Reviewing a capability change for least privilege: does the caller node actually need this tool, and is the grant scoped and observable?
- Keeping retrieve-only behavior the default; treating mutation-capable behavior as a separate, explicitly-approved, audited step.
- Keeping dormant capabilities default-off and tenant-scoped until explicitly activated with diagnostics and an operator review path.
- Verifying that tool/permission claims hold by reading the gateway, the policy, and their tests.

## disallowed uses

- Granting an agent/station implicit tool power, or letting a station widen its own permissions.
- Letting a model's request for a tool expand permissions -- permission is the declared policy, not the model's ask.
- Tool output silently overwriting source facts or buyer artifacts (Synthesize must not create new facts; merges go through an audit envelope).
- Any cross-tenant data access; deriving tenant from a public query parameter instead of caller identity.
- Activating the probe-refresh chain (gateway execution, audit persistence, or any confidence/posture/lean/artifact mutation) without the explicit, tenant-scoped, audited, operator-reviewed gate.
- Enabling gateway execution for probe-refresh outside `platform.retrieve`.

## workflow impact

- Tool calls: route through `ToolGateway.InvokeAsync`; never call a tool handler or external client directly to bypass the policy and telemetry.
- Permission grants: declared on the tool (`AllowedProtocolNodes`) and cross-checked against station cards (`allowed_tools`); both directions must agree (station blueprint section 6). Today enforcement is at the stage level (`platform.reference`, `platform.retrieve`, `platform.analyze`) because the cognitive stations are produced in one analyze call; per-station enforcement is a later configuration step, not a redesign.
- Default-off execution: dormant chains stay disabled by default (`Enabled`, `AllowGatewayExecution`, `PersistAuditRecord`, `AllowArtifactMutation`, `AllowConfidenceMutation`, `AllowPostureMutation`, `AllowLeanMutation` all default `false`); diagnostics inspect options/results without executing the chain.
- Mutation safety: any merge goes through an audit record (idempotent, tenant-scoped, read-only store); protected fields (confidence, posture, lean, artifact version, tenant, run id, raw signal overwrite, historical audit deletion) are forbidden changes; a future writer must consume a valid audit record and prove rollback first.
- Review: a code-changing slice touching tools/permissions runs `dai-code-reviewer` against this doctrine and verifies safe defaults with tests before completion.

## truth hierarchy

1. Observed runtime behavior and tests (`ToolGateway` tests, `ProtocolToolAccessPolicyTests`, probe-refresh safe-default tests).
2. Source code (`Tools/ToolGateway.cs`, `IProtocolToolAccessPolicy`/`ProtocolToolAccessPolicy`, `ToolDefinition.cs`, `Protocols/ProbeRefresh*`).
3. Explicit contracts and vault docs (this doctrine; the station blueprint; the probe-refresh readiness review; the charter).
4. Slice handoffs and the deferred-runtime-decisions ledger.
5. Generated graphs (Graphify) and prior assumptions -- navigation only.

Graphify can locate gateway/policy/probe/merge code; it cannot authorize a permission or prove runtime behavior. A permission claim is only true if the policy and its tests say so.

## source or vault references to verify

- `02 Platform/architecture/security-and-permissions.md` -- the charter (never implicit power; explicit/scoped/observable/rate-limited; no silent cross-tenant access).
- `02 Platform/architecture/orchestration.md` -- tools are shared platform infrastructure; agents differ by role, not tool ownership.
- `02 Platform/architecture/cognitive-factory/protocol-station-blueprint-v1.md` -- station cards derive gateway permissions; stations cannot widen permissions; memory/document tools default-off; stage-level enforcement today.
- `02 Platform/architecture/cognitive-factory/probe-refresh-chain-activation-readiness-v1.md` -- dormant chain; safe-by-default flags; merge audit; protected fields; forbidden activation scope.
- `02 Platform/architecture/cloud-tool-runtime-plan.md` -- gateway/registry design rationale; deferred enforcement.
- code (verify, do not assume): `platform/dotnet/DevCore.Api/Tools/ToolGateway.cs`, `IToolGateway.cs`, `ToolDefinition.cs`, `ToolTelemetry.cs`, the `IProtocolToolAccessPolicy`/`ProtocolToolAccessPolicy` pair and `ProtocolToolAccessPolicyTests.cs`; `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshExecutor.cs`, `ProbeRefreshMergeAuditStore.cs`, `ProbeRefreshMergeAuditReadService.cs`.

## example usage

A retrieve-stage call to an odds tool: the caller passes `ToolInvocationContext` with `ProtocolNode = platform.retrieve`; `ToolGateway.InvokeAsync` finds the tool in the registry, calls `policy.IsAllowed(definition, "platform.retrieve")`; allowed -> the keyed handler runs and one `Success` telemetry line is emitted with tenant/run/correlation. If the same tool is invoked from a node not in its `AllowedProtocolNodes`, the gateway emits a `Denied` warning and throws `ToolNotAllowedForProtocolNodeException` -- no handler resolves, no transport call is made. The dormant probe-refresh chain, asked to run with default options, does nothing: `Enabled=false` and every mutation flag is `false`; diagnostics report the safe defaults without executing it.

## agent prompt guidance

- Never assume a tool is callable; state the caller node and the tool's `AllowedProtocolNodes`, and that the gateway fails closed.
- Treat a model's "I need tool X" as a request to evaluate against policy, never as a grant.
- Keep retrieve-only the default; call out any mutation path as separate, audited, and explicitly approved.
- Never propose activating the probe-refresh chain or relaxing a safe-default flag without the full tenant-scoped, audited, operator-reviewed gate.
- Distinguish current enforcement (stage-level `AllowedProtocolNodes`) from deferred enforcement (cost-class, tenant-tier); do not assert deferred controls as live.

## acceptance criteria

- Every tool call goes through the gateway; no direct handler/client bypass.
- Each tool has an explicit `AllowedProtocolNodes`; the gateway fails closed on registry miss or policy deny.
- Stations/agents cannot widen their own permissions; permission is declared config.
- Retrieve-only is the default; mutation requires an audit record and explicit approval.
- Dormant capabilities stay default-off; diagnostics never execute the chain.
- No cross-tenant access; tenant derives from caller identity; reads do not leak cross-tenant existence.
- Every claim about current behavior is backed by the gateway/policy/probe source and tests.

## risks and failure modes

- Implicit power creep: a new tool with an over-broad `AllowedProtocolNodes`, or a model-driven grant -> mitigated by least-privilege review and the policy being declared, not model-chosen.
- Silent fact overwrite: tool/merge output overwriting source facts or artifacts -> blocked by Synthesize-creates-no-facts and the audit envelope with protected fields.
- Cross-tenant leak: tenant from a public parameter, or a read leaking existence -> blocked by identity-derived tenant and tenant-scoped reads returning `NotFound`.
- Premature activation: flipping a probe-refresh safe-default without the gate -> blocked by the readiness review's forbidden-activation list.
- Over-claiming enforcement: citing deferred controls (cost-class, tenant-tier) as live -> mitigated by the what-it-is-not and current-vs-deferred framing.
- Doctrine drift as the gateway gains real enforcement -> mitigated by anchoring to the gateway source and tests and re-checking on changes.

## deferred decisions

- Per-station (vs stage-level) gateway enforcement, gated on the FastAPI canonical-field migration.
- Cost-class rate limiting, idempotency caching, outbound correlation-header injection, and tenant-tier enforcement in the gateway.
- `ProtocolRegistry v0` (machine-readable station cards) and the startup validator cross-checking `allowed_tools` against `AllowedProtocolNodes`.
- Governed memory tools (`memory.*`) and document tools (`document.*`), `AllowedProtocolNodes` empty at launch.
- Any probe-refresh activation (limited dev activation, operator review surface, merge writer, runtime telemetry), each its own gated slice.
- Tenant/Stripe economic boundary wired to tool cost or capability enablement.

## related docs

- `02 Platform/architecture/security-and-permissions.md`
- `02 Platform/architecture/orchestration.md`
- `02 Platform/architecture/cloud-tool-runtime-plan.md`
- `02 Platform/architecture/cognitive-factory/protocol-station-blueprint-v1.md`
- `02 Platform/architecture/cognitive-factory/probe-refresh-chain-activation-readiness-v1.md`
- `02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md`
- `06 Execution/agent-slice-workflow-doctrine-v1.md` -- the lifecycle this slice followed.

## recommended next slice

DAI Slice Runner Skill v1 -- the documentation-doctrine backlog's three P1 items (slice workflow, source depth, tool gateway) are now complete, so the runner skill can encode a workflow proven across these doctrine slices rather than a single-use process. Cognitive Protocol Doctrine Anchor v1 (P2) is the alternative if more doctrine consolidation is preferred first; Tenant as Economic Boundary Doctrine v1 (P2) pairs naturally with this doc's deferred tenant/cost boundary.
