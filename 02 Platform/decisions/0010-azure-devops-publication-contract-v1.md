# decision 0010: azure devops publication contract v1 (create-only projection)

**date:** 2026-07-19
**status:** accepted (documentation-only slice; policy and contract established; no publisher
implemented; zero azure devops writes)

## context

The vault is authoritative for work definition, rationale, acceptance expectations, relationships,
approval, and durable closeout. Azure DevOps becomes authoritative for operational tracking after
publication (assignee, sprint placement, board state, PR/build links, execution status). Azure
DevOps is a projection target, not a second authoring system. This contract governs developer
workflow tooling only; it is never wired into the DAI application runtime, tenant platform, tool
gateway, API, persistence layer, or production orchestration.

A read-only discovery on 2026-07-19 (registered `mcp__azure-devops__*` tools, domains core / work /
work-items, default interactive authentication, status `ADO_PUBLICATION_TARGET_DISCOVERY_COMPLETE`)
established the publication target. The discovery report is evidence, not authorization: nothing in
it, or in this decision, authorizes any Azure DevOps mutation.

## retrieved facts (discovery, 2026-07-19)

- organization `jera-technologies`; project `dai`; project id `319ef4de-0139-41c3-bcd8-2573e3e28eb6`;
  visibility private.
- default team `dai Team`; default area path `dai`; backlog iteration `dai`; current sprint
  `dai\Sprint 1` (no start/finish dates).
- supported work-item hierarchy: Epic -> Issue -> Task. unsupported types: Feature, User Story, Bug
  (type retrieval returned null for all three).
- workflow states on all supported types: To Do / Doing / Done; new items enter at To Do.
- no dedicated acceptance-criteria field on any type (`System.Description` is the only rich body).
- zero existing work items; zero collisions on `WI-`, `vault-wi-`, or `ai-drafted` in titles/tags.
- the registered MCP domains expose no delete tool of any kind.

## derived conclusions

- the project runs the **Basic** process (derived from backlog categories and the null type
  lookups; the api surface used does not return the process name directly).
- `Issue` is the requirement-level type; a "user story" expectation cannot be met without a process
  migration, which this contract does not perform.
- with no custom fields available, canonical identity and provenance must travel in tags and in a
  governed description structure.
- an empty project plus a clean tag namespace means deterministic identity can be established from
  the first published item with no backfill.

## binding decisions

1. **process disposition.** Accept the Basic process for v1. No migration to Agile and no process
   customization in this contract's scope. Process migration is a separately governed future
   decision, never an implicit prerequisite of publication.

2. **type mapping.** An approved execution-grade vault work item maps to exactly one Azure DevOps
   **Issue**. **Epic** is reserved for an explicitly approved initiative- or portfolio-level
   parent. **Task** is used only for an explicit child implementation unit defined in the vault.
   Azure DevOps Tasks are never manufactured automatically from headings, acceptance criteria,
   plans, or inferred implementation steps.

3. **identity and idempotency.** The canonical machine join key is the normalized tag
   `vault-wi-####`. The human-readable title may use `WI-####: <title>`; the tag, not the title,
   is authoritative for duplicate detection. Before any future create, the publisher performs a
   WIQL pre-check for the exact canonical tag: zero matches permits consideration of creation; one
   match resolves to the existing projection; more than one match fails closed as an identity
   collision. Another item is never silently created.

4. **publication eligibility.** Only a canonically approved vault work item may enter a
   publication manifest. Draft, proposed, blocked, superseded, rejected, or structurally invalid
   records are ineligible. Eligibility is decided deterministically from governed vault fields and
   validation results, never inferred from prose by an agent. Batch membership is explicit; the
   publisher never scans the vault and publishes every apparently eligible record automatically.

5. **initial routing.** `System.AreaPath` defaults to `dai`. `System.IterationPath` defaults to
   the backlog root `dai`; publication never places work directly into Sprint 1 or any sprint —
   sprint assignment remains an Azure DevOps planning action. `System.AssignedTo` is left unset.
   New items enter in To Do; the publisher never changes board state.

6. **publishable projection.** Publication renders a normalized, typed projection containing only
   approved publishable content: canonical WI id, title, target work-item type, problem or
   purpose, desired outcome, acceptance criteria or verification expectations, explicit non-goals,
   relevant dependencies, canonical vault links, approved tags, source git revision, and the
   normalized projection hash. The hash is calculated over the normalized projection, not the
   entire markdown file, so unrelated metadata, formatting, or non-published sections cannot
   create false drift.

7. **description structure.** Because Basic has no acceptance-criteria field, the Issue
   description uses a stable, governed rendering with exactly these sections, in order:
   Purpose or Problem; Desired Outcome; Acceptance and Verification; Non-goals; Dependencies;
   Canonical Vault Reference; Publication Provenance. The provenance tag is `vault-managed`.
   `ai-drafted` is not used as an authority or lifecycle marker.

8. **create-only v1 semantics.** V1 may create a missing projection. It never automatically
   overwrites or patches an existing Issue and never appends comments as an implicit
   synchronization mechanism. If the stored projection hash differs from the current normalized
   projection, the run returns a drift result; updating requires a separate operator-authorized
   update workflow. An ordinary create-only rerun does not touch title, description, tags, state,
   assignee, iteration, parent, or links of existing items.

9. **azure devops reference capture.** After a successful future publication the publisher
   captures the Azure DevOps work-item id, URL, publication timestamp, source git revision, and
   normalized projection hash. These are recorded in the work-item spec's existing links block per
   [[work-item-traceability]] (`AB#<id>` beside the local id, plus publication-provenance lines).
   No new frontmatter field is introduced; if a machine-readable per-item publication ledger is
   later wanted, that is a schema-extension prerequisite requiring its own decision.

10. **operational-state boundary.** Azure DevOps remains authoritative for assignee, sprint,
    board state, and execution links. The vault does not continuously mirror these volatile
    values; live status is retrieved from Azure DevOps when needed. Durable closeout may record
    the final Azure DevOps id, final state, and verification evidence without turning the vault
    into a second board.

11. **batch and recovery semantics.** A batch is explicit, bounded, and ordered; prefer small
    coherent batches. Parents are resolved before children. Azure DevOps batch creation is not
    assumed transactional: each item progresses independently through validated -> identity
    checked -> created or already resolved -> linked when authorized -> read-back verified ->
    reference recorded. A failed batch is safely resumable; retry re-runs identity resolution
    before any create.

12. **dry-run and canary distinction.** Dry-run performs validation, identity queries, routing
    resolution, and publication-plan rendering with zero Azure DevOps writes. A live canary is a
    separate, explicitly authorized operation that creates exactly one Issue. A mutating canary is
    never described as a dry-run.

13. **prohibited v1 behavior.** The publisher and any surrounding workflow must not perform:
    unapproved publication; repository scanning followed by autonomous backlog creation; automatic
    sprint assignment; automatic assignee selection; state transitions; deletes or removals;
    process customization; area-path or iteration creation; automatic republishing; bulk updates
    to existing items; inferring Tasks from prose; treating Azure DevOps as canonical
    work-definition storage; wiring this workflow into DAI runtime services.

14. **adapter boundary.** The contract is transport-independent. MCP and the Azure DevOps REST
    API are candidate adapters. The future deterministic publisher owns validation, identity
    resolution, field allowlisting, projection hashing, dry-run behavior, read-back verification,
    and recovery; the adapter only performs authorized retrieval or mutation. Adapter selection is
    not made here — no existing architecture makes it binding.

## deferred decisions

- process migration to Agile (or acceptance of Basic long-term).
- adapter selection (MCP vs REST) and publisher implementation design.
- the operator-authorized update/republish workflow for drifted projections.
- schema extension for a machine-readable publication ledger, if links-block capture proves
  insufficient.
- dependency-link publication (`System.LinkTypes.Dependency`) — phase 2.
- iteration cadence creation (Sprint 1 has no dates; no Sprint 2+ exists).
- authorization of the first live canary.

## explicit non-goals

No publisher, validator, PowerShell wrapper, python code, MCP wrapper, or REST adapter is
implemented by this decision. No Azure DevOps mutation is performed or authorized. Nothing here
touches DAI runtime services, tenant surfaces, prompts, registries, routing, confidence logic,
buyer surfaces, or pipelines.

## consequences

- A future publisher can be specified and reviewed against a fixed contract instead of improvised
  behavior; every mutating capability it needs is enumerable in advance, and everything else is
  prohibited by default.
- The deferred "ado adoption path" in [[work-item-traceability]] advances from hypothetical to
  contract-governed: the org and project now exist, `AB#<id>` becomes the id-capture channel, and
  local `WI-####` ids remain the canonical spine and are never renumbered.
- Vault/board drift is detectable (projection hash) rather than silently accumulating, and
  correction is an explicit operator action rather than an implicit sync.
- The smallest safe next steps are fully determined: a dry-run publication plan for one approved
  work item, then a separately authorized single-Issue canary.

## references

- Spec: `02 Platform/system-development/work-items/WI-0033-azure-devops-publication-contract-v1.md`.
- Conventions: `02 Platform/system-development/work-item-traceability.md` (ids, links block,
  `AB#` capture); `02 Platform/system-development/dai-knowledge-architecture-and-writing-standard-v1.md`;
  `06 Execution/patterns/documentation-slice-impact-declaration-v1.md`.
- Evidence: read-only azure devops discovery, 2026-07-19, org `jera-technologies`, project `dai`
  (`ADO_PUBLICATION_TARGET_DISCOVERY_COMPLETE`); closeout
  `06 Execution/reports/azure-devops-publication-contract-v1-closeout-2026-07-19-v1.md`.
