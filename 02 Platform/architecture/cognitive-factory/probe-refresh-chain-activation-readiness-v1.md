# Probe Refresh Chain Activation Readiness v1

**date:** 2026-06-05
**status:** readiness review complete; activation remains deferred
**scope:** the dormant probe-refresh chain from `interrogate.probe` through `ProbeRefreshChainDiagnostics`

## Current dormant chain

The shipped chain is:

`interrogate.probe` -> `ProbeRequest` -> `ProbeRefreshDecision` -> `ProbeRefreshAuthorization` -> `ProbeRefreshExecutor` -> `ProbeRefreshPerceiveIntake` -> `ProbeRefreshDiscernReweigh` -> `ProbeRefreshDecideRecommendation` -> `ProbeRefreshSynthesizePreview` -> `ProbeRefreshArtifactMergePlan` -> `ProbeRefreshMergeReview` -> `ProbeRefreshMergeDryRunExecutor` -> `ProbeRefreshMergeAuditRecord` -> `ProbeRefreshMergeAuditStore` -> `ProbeRefreshMergeAuditReadService` -> `ProbeRefreshChainAssembly` -> `ProbeRefreshChainDiagnostics`.

The chain assembles as a dormant value-object path. It has diagnostics, audit contracts, an idempotent audit store, and a tenant-safe read surface. It is not activated in production and has no endpoint or pipeline consumer.

## Safe-by-default guarantees

- `ProbeRefreshChainAssemblyOptions.Enabled` defaults to `false`.
- `AllowGatewayExecution` defaults to `false`.
- `PersistAuditRecord` defaults to `false`.
- `AllowArtifactMutation` defaults to `false`.
- `AllowConfidenceMutation` defaults to `false`.
- `AllowPostureMutation` defaults to `false`.
- `AllowLeanMutation` defaults to `false`.
- `ProbeRefreshExecutorOptions.Enabled` defaults to `false` in production DI.
- Gateway execution, when ever allowed in a future slice, is constrained to `platform.retrieve`.
- The assembly persists only audit ledger rows, and only when `PersistAuditRecord = true`.
- Diagnostics inspect supplied options/results only. They do not execute the chain.

## Activation blockers

- No production pipeline consumer exists.
- No HTTP endpoint exists.
- No feature flag is bound to config or tenant entitlement.
- No telemetry emission pipeline exists beyond value-object diagnostics and merge-review telemetry shape.
- No operator review surface exists.
- No merge writer exists.
- No rollback executor exists.
- No tenant/Stripe economic boundary is wired to refresh cost or enablement.
- No calibration proof exists for confidence or posture changes.
- Interrogate remains a requester only; it does not trigger Perceive or directly call tools.

## Required feature flags

Before any limited activation, the platform needs explicit flags for:

- chain assembly enablement: disabled by default and scoped to dev/local first.
- gateway execution enablement: disabled by default and requiring `platform.retrieve`.
- audit persistence enablement: disabled by default and audit-ledger only.
- tenant-scoped enablement: disabled unless the tenant is explicitly included.
- future artifact merge writer enablement: separate from chain assembly and still blocked.
- future operator/read endpoint enablement: separate, admin/dev gated, and not public.

No current slice binds these flags to config or changes behavior.

## Required telemetry/diagnostics

Minimum required before limited activation:

- Diagnostics must report safe defaults.
- Diagnostics must surface gateway execution, audit persistence, and mutation flags.
- Diagnostics must surface protected fields.
- Diagnostics must inspect an existing result without executing the chain.
- Assembly results must retain step trace, final status, reason, and error message.
- Merge review telemetry shape must retain run, tenant, correlation, plan status, review status, change counts, and manual-review state.
- Any future runtime telemetry emitter must record tenant, run, correlation, idempotency key, requested signal, authorized tool id, chain status, and store status.
- Diagnostics must be captured before and after any future limited activation run.

## Tenant/run/idempotency boundaries

- Tenant key is carried through chain assembly, merge plan, review, dry-run, audit, store, and read service.
- Run id is carried through chain assembly, merge plan, review, dry-run, audit, store, and read service.
- Source artifact version is required for merge planning and audit boundary checks.
- Audit idempotency key is deterministic and includes tenant/run/requested signal/tool/source artifact/proposed change hash inputs.
- Duplicate audit writes return the existing record and do not create a second row.
- Audit read by idempotency key is tenant-scoped at the service boundary. A cross-tenant hit returns `NotFound`, not `TenantMismatch`.
- Audit read by run is tenant-scoped and ordered by created time, then ledger key.
- Future endpoints must derive tenant from caller identity, not from a public query parameter.

## Protected fields

These fields remain protected from automatic mutation:

- confidence
- posture
- lean
- artifact version
- tenant
- run id
- raw retrieved signals overwrite
- historical audit deletion

The merge planner lists them as forbidden changes. Merge review blocks proposals that touch them. Dry-run projection filters them. Diagnostics surfaces the same boundary.

## Audit/store/read requirements

- Audit contract exists before any artifact writer.
- Audit store exists and persists audit ledger rows only.
- Audit read surface exists and is read-only.
- Store validation requires tenant key, run id, audit id, and idempotency key.
- Store duplicate writes are idempotent.
- Store and read service must not mutate artifacts or AgentRun rows.
- Read service must not leak cross-tenant existence.
- Any future writer must consume a valid audit record and must not bypass the audit envelope.
- Any future rollback execution must use audited before-payload data and must have its own review.

## Allowed future activation scope

The first limited activation may be considered only if it is:

- dev/local only.
- controlled by explicit feature flags.
- tenant-scoped.
- audit-persistence only.
- non-mutating for artifacts.
- non-mutating for confidence, posture, and lean.
- gateway execution only through `platform.retrieve`.
- preceded and followed by diagnostics capture.
- backed by a clear operator review path.

## Explicitly forbidden activation scope

The following are not approved:

- direct Interrogate -> Perceive self-invocation.
- direct tool power on `interrogate.probe`.
- production activation without tenant/auth boundary.
- mutation without an audit record.
- posture or confidence changes without calibration proof.
- public endpoint without an explicit gate.
- artifact overwrite without rollback.
- using the audit store as an artifact writer.
- enabling gateway execution outside `platform.retrieve`.
- changing FastAPI prompts, model call count, Angular, schema, MCP, pgvector, Azure Functions, Kubernetes, or secrets as part of activation.

## Readiness checklist

| Area | Item | Status |
|---|---|---|
| Safe defaults | Chain assembly default `Enabled=false` | ready |
| Safe defaults | `AllowGatewayExecution=false` by default | ready |
| Safe defaults | `PersistAuditRecord=false` by default | ready |
| Safe defaults | Artifact mutation flags false by default | ready |
| Safe defaults | Confidence/posture/lean mutation flags false by default | ready |
| Diagnostics | Diagnostics can report safe defaults | ready |
| Diagnostics | Diagnostics can surface gateway/audit/mutation flags | ready |
| Diagnostics | Diagnostics can surface protected fields | ready |
| Diagnostics | Diagnostics can inspect results without executing the chain | ready |
| Audit | Audit contract exists | ready |
| Audit | Audit store exists | ready |
| Audit | Audit read surface exists | ready |
| Audit | Idempotency key exists and is deterministic | ready |
| Audit | Duplicate audit writes are idempotent | ready |
| Audit | Tenant/run boundaries are enforced | ready |
| Audit | Audit read does not leak cross-tenant existence | ready |
| Protected fields | confidence is protected | ready |
| Protected fields | posture is protected | ready |
| Protected fields | lean is protected | ready |
| Protected fields | artifact version is protected | ready |
| Protected fields | tenant/run id are protected | ready |
| Protected fields | raw retrieved signals overwrite is protected | ready |
| Protected fields | historical audit deletion is protected | ready |
| Blocker | No production pipeline consumer yet | blocked |
| Blocker | No endpoint yet | blocked |
| Blocker | No feature flag bound to config yet | blocked |
| Blocker | No telemetry emission pipeline beyond value-object diagnostics yet | blocked |
| Blocker | No operator review surface yet | blocked |
| Blocker | No calibration proof for confidence/posture changes | blocked |
| Blocker | No merge writer yet | blocked |
| Blocker | No rollback execution yet | blocked |
| Blocker | No tenant/Stripe economic boundary integration yet | blocked |

## Recommended next slices

1. Probe Refresh Limited Dev Activation Plan v1: config and tenant-scoped flag design only; no production wiring.
2. Probe Refresh Operator Review Surface v1: service-level or dev/admin inspection surface for diagnostics and audit records, gated and tenant-safe.
3. Probe Refresh Merge Writer v1: dormant writer design that consumes `ReadyForPersistCandidate` audit records and proves rollback before any artifact mutation.
4. Probe Refresh Runtime Telemetry v1: emit structured telemetry from assembly/diagnostics/audit paths without changing activation state.

## References

- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/ProbeRefreshChainAssembly.cs`
- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/ProbeRefreshChainDiagnostics.cs`
- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/ProbeRefreshArtifactMergeContract.cs`
- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/ProbeRefreshMergeReview.cs`
- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/ProbeRefreshMergeDryRunExecutor.cs`
- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/ProbeRefreshMergeAuditStore.cs`
- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/ProbeRefreshMergeAuditReadService.cs`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md`
