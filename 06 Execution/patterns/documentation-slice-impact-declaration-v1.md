---
title: "Documentation-Slice Impact Declaration and Authoring Checklist v1"
type: "execution-pattern"
date: "2026-07-18"
status: "active"
project: "DAI"
slice: "WI-0032 DAI Knowledge Architecture and Writing Standard v1 (Slice 2)"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - knowledge-system
  - documentation
  - governance
  - checklist
related:
  - "02 Platform/system-development/dai-knowledge-architecture-and-writing-standard-v1.md"
  - "06 Execution/patterns/okf-documentation-review-guide-v1.md"
  - "02 Platform/system-development/work-items/WI-0032-dai-knowledge-architecture-and-writing-standard-v1.md"
---

# documentation-slice impact declaration and authoring checklist v1

## purpose

The operational companion to the [[dai-knowledge-architecture-and-writing-standard-v1]]: a
copy-usable pre-write **impact declaration** plus **authoring** and **reviewer** checklists that any
vault-writing slice fills in before touching files. It uses existing documentation mechanisms only
-- no tooling, hook, CI, validator, or manifest. The standard states the rules; this pattern is how
a slice applies them in practice.

## when to use

- Before the first vault write in any slice that creates or materially edits durable knowledge.
- During review, to check a proposed documentation change against the standard.

## when not to use (lightweight path)

Urgent evidence records and operational handoffs keep traceability but skip this full declaration:
an append to `06 Execution/handoffs/current-slice.md`, or a single evidence report under an existing
plan, needs only a governing/referenced WI, the correct profile, and append-only where required.
Routine governed operational cadence under an existing OKF plan does not mint a work item (WI-0007).

## pre-write impact declaration (fill before writing)

Copy this block into the slice's work item, prompt, or handoff and complete every field
(`none` is a valid value):

```text
Knowledge impact declaration
- affected knowledge area:            <01..06 area>
- affected knowledge module:          <logical module / capability / decision thread, or none>
- governing work item:                <WI-#### + path, or "does not qualify: <reason>">
- records created:                    <profile + path each>
- records modified:                   <profile + path each>
- record profiles:                    <profile per record; confirm type == folder for OKF>
- target directory (per record):      <exact folder each record lives in>
- new subfolder proposed?:            <yes/no; if yes: which, and why vs the topic-folder threshold>
- folder-depth impact:                <area->category->topic->record depth; justify unusual depth>
- MOCs / maps affected:               <which, and how, or none>
- relationships added / removed:      <links / related / WI-traceability / ADR / supersession>
- paths moved (if any):               <old -> new; requires git mv + link/MOC updates + supersession>
- supersession impact:                <predecessor/successor + status change, or none>
- versioning impact:                  <revise-in-place vs new -vN, or none>
- validation required:                <record / relationship / module / discoverable / slice-complete>
- exact allowlisted paths:            <the complete write allowlist>
```

Stop rule: if a proposed document has no obvious area, module, record profile, authority role, or
target directory, do not place it in a convenient folder -- stop for a documentation-architecture
decision (`DOCUMENTATION_ARCHITECTURE_BLOCKED:<code>`).

## authoring checklist (before commit)

- [ ] each record has one declared profile and one primary purpose; `type == folder` for OKF records
- [ ] placement follows record profile + existing taxonomy (not folder convenience); topic subfolder
      only if the reasoned threshold is met; no folder anti-patterns (one-file/temp-WI/symmetry/
      `misc`/vendor/duplicate-nesting/excessive-depth)
- [ ] no new metadata field, manifest, or new top-level folder without an explicit extension decision
- [ ] authority is explicit; no authority inherited via links or defaults; MOC/map is navigation only;
      a plan grants no authority beyond its WI; a handoff does not supersede a standard
- [ ] relationships explicit (`related` where the profile uses it; body wikilinks; WI-traceability;
      ADR/supersession links); reciprocal links added where the profile requires
- [ ] naming: descriptive kebab-case; stable name for durable doctrine; date only for time-bound
      records; version suffix only on a breaking revision; no `notes/misc/new/temporary`
- [ ] no machine-specific absolute paths; no secrets/tokens/private URLs in durable output
- [ ] `current-slice.md` append is append-only (prior byte prefix preserved); rolling logs not
      front-mattered or reorganized
- [ ] any moved/regrouped record used `git mv`, updated links/MOC/maps, and recorded supersession/
      compatibility -- and was not a cosmetic-only move
- [ ] the write matches the declared allowlist exactly

## reviewer checklist (before integration)

- [ ] delta contains only the declared allowlisted paths; no unrelated paths
- [ ] each record valid against its actual profile (a different valid profile is not a defect)
- [ ] related/`related` links resolve; no duplicate canonical record; no broken/one-way required link
- [ ] authority hierarchy intact (rule/why/decided/authorized/sequenced/happened/current/where each
      answered by the right profile); no link- or inheritance-granted authority
- [ ] physical-structure rules honored (hierarchy, threshold, anti-patterns, depth, grouping,
      map-trigger, structural-change discipline)
- [ ] manifest/OKF: no forced bundle/module/README; manifests evidence-driven; OKF as interoperability
- [ ] supersession/migration posture respected; historical evidence immutable; no cosmetic mass move
- [ ] strict planning snapshot exit 0 / 0 warnings; OKF validation where applicable; `git diff --check`
      clean; machine-path/secret scan clean; protected/classified paths byte-identical
- [ ] closeout + handoff agree with the work item and the changed records (slice-complete)

## validation levels (which checklist item maps where)

Record valid -> authoring + OKF validation. Relationship valid -> related-link checks. Module
coherent -> reviewer authority/relationship items. Vault discoverable -> MOC/map review. Slice
complete -> closeout/handoff agreement + strict snapshot. Tooling for the currently manual levels
(broken-link, module-coherence, duplicate-canonical, stale-MOC) is a recorded gap for a later
assessment slice; none is implemented by this pattern.

## example (this slice, dogfooded)

- area: 06 Execution (+ 02 Platform system-development for the WI); module: DAI knowledge
  architecture (WI-0032); governing WI: WI-0032.
- records created: this pattern (`execution-pattern`, `06 Execution/patterns/`); a closeout
  (`evidence-report`, `06 Execution/reports/`). records modified: WI-0032 spec (links/disposition);
  `current-slice.md` (append-only). target directories as above; new subfolder proposed: no;
  folder-depth impact: none (records stay in existing category folders).
- MOCs affected: none (WI-0032 already registered; patterns are not WI-MOC entries). relationships:
  this pattern links to the standard, the OKF review guide, and WI-0032. paths moved: none.
  supersession: none. versioning: new v1. validation: record + relationship + slice-complete.

## related

- [[dai-knowledge-architecture-and-writing-standard-v1]] -- the normative standard this operationalizes.
- [[okf-documentation-review-guide-v1]] -- per-record OKF placement and review checklist.
- `02 Platform/system-development/work-item-traceability.md` -- ids, branch/commit/link conventions.

## recommended next slice

WI-0032 Slice 3 (record-profile validation gap assessment): inventory what current validators cover
versus the five validation levels and name only the highest-value missing checks; implement nothing.
