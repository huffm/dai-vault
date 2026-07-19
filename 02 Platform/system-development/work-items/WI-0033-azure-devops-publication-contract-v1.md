---
title: "WI-0033 Azure DevOps Publication Contract v1"
type: "plan"
date: "2026-07-19"
status: "complete"
project: "DAI"
slice: "WI-0033 Azure DevOps Publication Contract v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - system-development
  - work-item
  - azure-devops
  - governance
related:
  - "02 Platform/decisions/0010-azure-devops-publication-contract-v1.md"
  - "02 Platform/system-development/work-item-traceability.md"
  - "02 Platform/system-development/dai-knowledge-architecture-and-writing-standard-v1.md"
  - "06 Execution/patterns/documentation-slice-impact-declaration-v1.md"
---

# WI-0033 azure devops publication contract v1

## problem  <!-- LITE -->

A read-only Azure DevOps discovery (2026-07-19, org `jera-technologies`, project `dai`) confirmed a
live, empty, private publication target, but no governed contract exists for projecting approved
vault work items into it. [[work-item-traceability]] records an "ado adoption path" as deferred and
still states (verified 2026-07-09) that no Azure DevOps org exists — that reality has changed.
Without a binding contract, any future publisher would improvise identity, routing, type mapping,
and update semantics, risking duplicate items, drift between vault and board, and Azure DevOps
silently becoming a second authoring system.

## desired behavior

A single decision record (ADR 0010) binds how vault work items may later be projected into Azure
DevOps: vault authoritative for definition and approval; Azure DevOps authoritative for operational
tracking after publication; create-only v1 semantics with deterministic tag-based identity; explicit
batches; dry-run distinct from canary; a closed list of prohibited behaviors. Policy and contract
only — no publisher, validator, wrapper, adapter, or Azure DevOps mutation exists after this slice.

## affected surfaces

- `02 Platform/decisions/0010-azure-devops-publication-contract-v1.md` (new, the contract)
- `02 Platform/system-development/work-items/WI-0033-azure-devops-publication-contract-v1.md` (this spec)
- `02 Platform/system-development/MOC - DAI System Development.md` (registry entry)
- `06 Execution/reports/azure-devops-publication-contract-v1-closeout-2026-07-19-v1.md` (closeout)
- `06 Execution/handoffs/current-slice.md` (append-only handoff)

## non-goals

- No publisher, validator, PowerShell wrapper, python code, MCP wrapper, or REST adapter.
- No Azure DevOps mutation of any kind; no process customization, area/iteration creation.
- No wiring into DAI application runtime, tenant platform, tool gateway, API, persistence, or
  production orchestration — this is developer workflow tooling governance only.
- No process migration to Agile (separately governed future decision).
- No adapter selection (MCP vs REST remains open).
- No `dai` repository change; no schema/frontmatter extension.

## acceptance criteria  <!-- LITE -->

- ADR 0010 exists, distinguishes retrieved facts / derived conclusions / binding decisions /
  deferred decisions / explicit non-goals, and covers all fourteen contract areas (process
  disposition, type mapping, identity and idempotency, eligibility, routing, projection,
  description structure, create-only semantics, reference capture, operational-state boundary,
  batch and recovery, dry-run vs canary, prohibitions, adapter boundary).
- ADR 0010 preserves the discovery facts (project id `319ef4de-0139-41c3-bcd8-2573e3e28eb6`,
  Basic process, Epic/Issue/Task only, To Do/Doing/Done, no acceptance-criteria field, zero
  existing items, zero namespace collisions, no delete tool, default interactive auth).
- Reference capture reuses the existing links-block convention from [[work-item-traceability]];
  no new frontmatter field is introduced.
- WI-0033 registered in the system-development MOC; strict planning snapshot exit 0 / 0 warnings
  after edits; `git diff --check` clean; all new links resolve; current-slice append-only.
- Zero Azure DevOps mutations; `dai` repo byte-identical (pre-existing csproj drift preserved).

## test plan  <!-- LITE, written BEFORE implementation -->

Documentation slice — no code tests. Verification is the strict planning snapshot (baseline exit 0 /
0 warnings recorded before editing), link resolution, append-only check on `current-slice.md`,
`git diff --check`, and a full branch-delta review against the base commit. Per
[[testing-strategy]], docs-only changes carry no spec files.

## implementation notes

Contract decisions and their evidence basis live in ADR 0010, not restated here (single-writer
rule). Discovery evidence came from the registered `mcp__azure-devops__*` tools (domains: core,
work, work-items; default interactive authentication); the discovery report is treated as evidence,
not authorization. Knowledge impact declaration for this slice:

```text
Knowledge impact declaration
- affected knowledge area:            02 Platform (decisions, system-development) + 06 Execution (reports, handoffs)
- affected knowledge module:          azure devops publication projection (developer-workflow governance)
- governing work item:                WI-0033 (this spec; next id after WI-0032 per filename registry)
- records created:                    work item (this spec); decision record 0010; evidence-report closeout
- records modified:                   MOC - DAI System Development (navigation); current-slice.md (append)
- record profiles:                    work item / ADR header / evidence-report; MOC navigation-only
- target directory (per record):      work-items/, decisions/, reports/, handoffs/ as listed above
- new subfolder proposed?:            no
- folder-depth impact:                none (existing category folders)
- MOCs / maps affected:               MOC - DAI System Development (add WI-0033 entry)
- relationships added / removed:      ADR 0010 <-> WI-0033; WI-0033 -> traceability/standard/pattern
- paths moved (if any):               none
- supersession impact:                none (ADR 0010 advances the deferred ado adoption path; does not supersede)
- versioning impact:                  new v1 records
- validation required:                record + relationship + slice-complete; strict snapshot before/after
- exact allowlisted paths:            the five paths listed under affected surfaces
```

## docs to update

All updates are within this slice's allowlist (MOC registration, closeout, current-slice append).
[[work-item-traceability]] "current reality" refresh (org now exists) is deliberately deferred to
the future publisher work item so this slice stays minimal; ADR 0010 records the new reality.

## verification commands  <!-- LITE -->

```text
pwsh <DAI_REPO_ROOT>/scripts/dev/planning/build-next-slice-snapshot.ps1 -OutputPath <tmp>/snapshot.json -Strict
git -C <DAI_VAULT_ROOT> diff --check
git -C <DAI_VAULT_ROOT> diff <base>..HEAD --stat   # delta = allowlisted paths only
```

## risks

- Contract decisions could later conflict with the chosen adapter's capabilities; mitigated by
  keeping the contract transport-independent and deferring adapter selection.
- The Basic process could be migrated later, changing type mapping; mitigated by recording
  migration as a separately governed decision with `WI-####` ids never renumbered.
- Vault/board drift if future work bypasses the contract; mitigated by the prohibited-behavior
  list and create-only-with-drift-detection semantics in ADR 0010.

## links  <!-- LITE; all 8 required at close, per work-item-traceability -->

- work item: WI-0033 (ADO: AB#— not yet published; this WI governs the contract for doing so)
- branch: `wi/0033-azure-devops-publication-contract` (dai-vault only)
- pr: merged direct planned; local-only at close of this slice (not pushed)
- commits: see closeout report (local commit on the branch above)
- tests: none (docs-only slice)
- verification notes: `06 Execution/reports/azure-devops-publication-contract-v1-closeout-2026-07-19-v1.md`
- docs updated: ADR 0010; MOC - DAI System Development; current-slice.md (append)
- lessons: contract-before-tooling holds for external projections exactly as it does for runtime
  surfaces; discovery output is evidence, never authorization

## final handoff requirements

Slice handoff appended to `06 Execution/handoffs/current-slice.md`; lessons recorded above;
definition of done in [[implementation-lifecycle]] checked for a docs-only slice.
