---
title: "OKF Registry v1"
type: "execution-pattern"
date: "2026-07-16"
status: "complete"
project: "DAI"
slice: "WI-0025 OKF Registry and Authority Contract v1"
repos:
  dai: "docs-only"
  dai-vault: "docs-only"
tags:
  - okf
  - registry
  - knowledge-system
  - authority
related:
  - "02 Platform/system-development/knowledge-system.md"
  - "02 Platform/system-development/work-items/WI-0025-okf-registry-and-authority-contract.md"
  - "06 Execution/patterns/okf-documentation-review-guide-v1.md"
  - "06 Execution/patterns/okf-yaml-front-matter-pattern-v1.md"
---

# OKF registry v1

## purpose

This document is the authoritative machine-readable contract for the DAI
Obsidian Knowledge Format (OKF): which front-matter fields exist, which types
and statuses are valid, where each type canonically lives, how authority,
freshness, and trust are classified, which documents are exempt, and which
generated artifacts are registered. The single fenced JSON block below is the
payload; tooling parses it with PowerShell `ConvertFrom-Json` and Python
`json.loads` and validates it against
`dai/scripts/knowledge/schemas/okf-registry.schema.json`. Prose here records
semantics and rationale only; it never duplicates the JSON enumerations.

Design source: DAI Knowledge System Architecture and Orchestration Plan v1.1
(workspace root, outside both repos), Phase 1, as corrected by the reviewed
WI-0025 authorization contract. The living contract that links this registry
to the rest of the system is
`02 Platform/system-development/knowledge-system.md`.

## change rule

Registry edits happen only through governed slices with a named work item.
A registry change is ADR-worthy, never a drive-by edit. History is versioned
by supersession: a future registry version is a new governed document or a
governed edit that records what changed and why, never a silent rewrite of
recorded semantics. Exemptions and legacy recognitions absorb history; they
are not licenses for new drift.

## classification semantics

Authority, freshness, trust, and projection status are derived at index time
and never written into source documents.

- The integer authority ladder 1 through 7 in `authority_precedence` applies
  only to source-eligible documents. Lower tiers never override higher tiers.
- Generated projections carry `source_eligible: false` and `authority: null`
  in every manifest record, plus a `sources` array of authoritative URIs. A
  projection may orient an agent; the agent cites and follows the sources.
- Freshness classes separate what is current from what is dated-but-valid,
  stale, or historical; the `stale_when` triggers make staleness mechanical,
  not editorial.
- Trust classes bound what retrieved text may do: only governing-class text
  may direct behavior. A completed work item is authoritative evidence about
  what its own slice authorized and delivered; it is operational evidence,
  not a live directive, and it never displaces the explicitly selected or
  active in-progress work item in the governing slot.

## work-item compatibility rule

Work items are identified by their canonical path, the work-items folder,
not by type alone. All pre-registry work items use `type: "plan"`; future
work items may use `type: "work-item"`. `work_item_folder_types` registers
both as valid in that folder. No mass migration is authorized; existing
documents keep their type until materially touched under a governed slice
that says otherwise.

## moc placement rule

A document with `type: "moc"` is valid if and only if its parent directory is
an exact member of `moc_locations`. Adding a location is a registry edit
through a governed slice. `type_folders` therefore records `null` for moc:
placement is governed by the membership list, not by a single canonical
folder.

## type-folder rule and grandfather gate

Each non-null `type_folders` value is a list of canonical parent directories
for that type. A document's exact parent directory must match one list member
(`path_rules.type_folder_match` is `exact-parent-directory`; matching is
never an unrestricted descendant-prefix match). Multiple canonical parents
are a fact of the live corpus, not a relaxation: execution-pattern doctrine
is canonical both under the platform system-development folder (including its
separately registered design-system child) and under the execution patterns
folder, and plan documents are canonical both as work items and as execution
plans.

Pre-registry placements that do not match are grandfathered by the
`legacy_type_folder_ok_before` date gate: a mismatch on a document dated
before the gate date is a warning, not an error, and never permission for a
new mismatch. The companion `legacy_status_ok_before` gate grandfathers prose
statuses on documents dated before the same date. Documents dated on or after
the gate dates must conform.

## exemption semantics and path rules

Exemptions come in exactly two shapes: `no_front_matter_files` lists exact
vault-relative files (rolling registries that are never front-mattered), and
`no_front_matter_path_prefixes` lists directory prefixes whose contents are
exempt. `path_rules` normalizes every comparison: paths are vault-relative
with forward slashes, compared case-sensitively against the canonical stored
form; a match that succeeds only case-insensitively on Windows is reported as
a portability warning, not accepted silently; directory prefixes must end
with a slash; wildcards are not interpreted unless a wildcard form is
separately registered in a future schema version.

## generated-artifact lifecycle rule

Every registered generated artifact carries a lifecycle `state` (`planned`,
`active`, or `retired`) and the delivery work item in
`available_from_work_item`. A `planned` artifact is a declaration: validation
must not report it as missing or stale before its delivery work item ships.
Delivery flips it to `active` through that governed slice, at which point the
determinism envelope and regenerate-compare checks apply. As of this version
all four artifacts are `planned`; none exists on disk yet, and Phase 1
deliberately creates none of them.

## workspace-bootstrap instruction sources

The corpus is inventoried by `git ls-files` plus explicitly registered
workspace-bootstrap instruction documents that live outside both
repositories: `AGENTS.md` and `CLAUDE.md` at the workspace root. Their
manifest records carry `repo: "workspace"`, `git_state: "workspace-bootstrap"`,
`source_commit: null`, a content hash, `source_eligible: true`,
`trust: "governing"`, and workspace scope. Tracked repository-level
instruction files (`dai/CLAUDE.md`, `dai-vault/CLAUDE.md`, and any future
tracked repository-level AGENTS.md) are included the same way with repository
scope; instruction scope follows directory ancestry.

## the registry block

```json
{
  "schema_version": "1.0",
  "fields": ["title", "type", "date", "status", "project", "slice", "repos", "tags", "related"],
  "types_active": ["evidence-report", "reconciliation", "export", "execution-pattern", "diagnostic", "plan", "handoff", "work-item", "moc", "state-snapshot"],
  "types_legacy_recognized": ["report", "pattern", "template", "product"],
  "statuses": ["complete", "in-progress", "blocked", "no-op", "superseded"],
  "type_folders": {
    "evidence-report": ["06 Execution/reports/"],
    "reconciliation": ["06 Execution/reconciliations/"],
    "export": ["06 Execution/exports/"],
    "execution-pattern": ["02 Platform/system-development/", "02 Platform/system-development/design-system/", "06 Execution/patterns/"],
    "diagnostic": ["06 Execution/diagnostics/"],
    "plan": ["02 Platform/system-development/work-items/", "06 Execution/plans/"],
    "handoff": ["06 Execution/handoffs/"],
    "work-item": ["02 Platform/system-development/work-items/"],
    "state-snapshot": ["06 Execution/state/"],
    "moc": null
  },
  "work_item_folder_types": ["plan", "work-item"],
  "moc_locations": ["02 Platform/system-development/", "06 Execution/", "06 Execution/patterns/"],
  "authority_precedence": {
    "1": "repository-and-runtime",
    "2": "work-item",
    "3": "final-integrated-handoff",
    "4": "canonical-plan-or-doctrine",
    "5": "rolling-registry",
    "6": "earlier-handoff",
    "7": "historical-report"
  },
  "freshness_classes": ["current", "dated-valid", "stale", "historical"],
  "stale_when": ["expiry-passed", "expected-heads-mismatch", "superseded-marker-present"],
  "trust_classes": ["governing", "operational-evidence", "informational", "historical", "generated-projection", "untrusted-external"],
  "edges": ["related_to", "links_to", "member_of", "contained_in", "governs", "evidence_of", "supersedes", "encodes"],
  "exemptions": {
    "no_front_matter_files": [
      "06 Execution/handoffs/current-slice.md",
      "06 Execution/handoffs/current-sports-matchup-analyzer.md"
    ],
    "no_front_matter_path_prefixes": [
      "06 Execution/prompting/",
      "06 Execution/backlog/",
      "06 Execution/roadmap/",
      "06 Execution/skills/",
      "06 Execution/weekly-plans/",
      "06 Execution/launch-checklists/"
    ],
    "legacy_status_ok_before": "2026-07-16",
    "legacy_type_folder_ok_before": "2026-07-16"
  },
  "path_rules": {
    "relative_to": "vault-root",
    "separator": "forward-slash",
    "comparison": "case-sensitive",
    "windows_case_mismatch": "portability-warning",
    "prefix_must_end_with_slash": true,
    "type_folder_match": "exact-parent-directory",
    "wildcards": "not-interpreted-unless-separately-registered"
  },
  "aliases": {},
  "generated_artifacts": [
    {
      "name": "current-state-view",
      "path": "dai-vault://06 Execution/state/current-state-view.md",
      "generator": "Invoke-DaiKnowledge.ps1 export -UpdateCurrentState",
      "committed": true,
      "state": "planned",
      "available_from_work_item": "WI-0027"
    },
    {
      "name": "execution-index",
      "path": "dai-vault://06 Execution/state/execution-index.md",
      "generator": "Invoke-DaiKnowledge.ps1 export -UpdateExecutionIndex",
      "committed": true,
      "state": "planned",
      "available_from_work_item": "WI-0027"
    },
    {
      "name": "operating-model-canvas",
      "path": "dai-vault://02 Platform/system-development/views/dai-operating-model.canvas",
      "generator": "Invoke-DaiKnowledge.ps1 export -UpdateCanvas",
      "committed": true,
      "state": "planned",
      "available_from_work_item": "WI-0027"
    },
    {
      "name": "corpus",
      "path": "dai://scripts/knowledge/out/",
      "generator": "Invoke-DaiKnowledge.ps1 build",
      "committed": false,
      "state": "planned",
      "available_from_work_item": "WI-0026"
    }
  ]
}
```

## verification

The registry parses with PowerShell `ConvertFrom-Json` and Python
`json.loads` (seventeen top-level keys) and validates against
`dai/scripts/knowledge/schemas/okf-registry.schema.json` via `Test-Json`.
Exactly one fenced json block exists in this document. The Phase 2 validator
(`Invoke-DaiKnowledge.ps1 validate`, WI-0026) will consume this same block;
until then, verification is the command battery recorded in the WI-0025
handoff.

## next recommended action

Phase 2 (WI-0026): extract the shared front-matter parser, implement
`Invoke-DaiKnowledge.ps1` with status, audit, validate, and build against
this registry — pending its own operator authorization.
