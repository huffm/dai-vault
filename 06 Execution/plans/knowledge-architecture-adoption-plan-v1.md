---
title: "Knowledge Architecture Adoption Plan v1"
type: "plan"
date: "2026-07-18"
status: "in-progress"
project: "DAI"
slice: "WI-0032 DAI Knowledge Architecture and Writing Standard v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - system-development
  - knowledge-system
  - documentation
  - planning
related:
  - "02 Platform/system-development/dai-knowledge-architecture-and-writing-standard-v1.md"
  - "02 Platform/system-development/work-items/WI-0032-dai-knowledge-architecture-and-writing-standard-v1.md"
---

# knowledge architecture adoption plan v1

## purpose

Sequence the prospective adoption of the DAI Knowledge Architecture and Writing Standard v1. The
standard defines the "what"; WI-0032 governs execution/status; this plan defines adoption order.
Adoption is prospective and evidence-driven -- no broad migration and no tooling are implemented
by this plan or authorized by it.

## context

Planning slice 2026-07-18 established the standard and evaluated WI-0031 as the pilot module.
Vault main `220f8f0` (WI-0031 integrated). This plan proposes five slices; only Slice 1 (the
standard) is done.

## scope

Included: slice objectives, boundaries, dependencies, and promotion criteria for prospective
adoption. Excluded: any tooling, validator, manifest, migration, or WI-0031 Slice 1 work.

## adoption principle

Apply the standard to new durable knowledge and when a module is substantively changed. Do not
migrate legacy records for cosmetic consistency; create focused migration work only on measurable
discovery/validation/maintenance value.

## slice sequence

### Slice 1 -- standard and pilot disposition (DONE this slice)
- Output: the normative standard + embedded WI-0031 pilot assessment + this plan.
- Boundary: docs-only; no tooling; no migration.

### Slice 2 -- documentation-impact declaration and authoring checklist
- Objective: codify the pre-write knowledge-impact declaration and a reviewer checklist using
  existing documentation mechanisms (a pattern doc and/or a checklist under
  `06 Execution/patterns/`), so future slices declare area/module/WI/records/profiles/relationships/
  supersession/validation/paths before writing.
- Boundary: no tooling, no hook, no CI; documentation mechanisms only.
- Dependency: Slice 1.

### Slice 3 -- record-profile validation gap assessment
- Objective: inventory what current validators cover (OKF validation, strict planning snapshot,
  front-matter checks) versus the five validation levels, and identify only the highest-value
  missing checks (candidate: vault-wide broken-link, duplicate-canonical, stale-MOC).
- Boundary: assessment only; implement nothing.
- Dependency: Slices 1-2.

### Slice 4 -- optional lightweight validator improvements
- Objective: implement only the proven validation gaps from Slice 3, under a separate
  authorization, as the smallest possible checks reusing existing script conventions.
- Boundary: separate authorization required; no broad tooling.
- Dependency: Slice 3 evidence.

### Slice 5 -- prospective adoption in a second knowledge module
- Objective: apply the standard to a second module (a real durable subject) to test whether it
  generalizes beyond WI-0031; capture findings; refine the standard only if evidence requires.
- Boundary: no forced migration of existing modules.
- Dependency: Slices 1-2 (3-4 optional).

## recommended follow-up (recorded, not actioned)

The system-development MOC registers through WI-0023 plus WI-0031/0032 but omits WI-0024 and
WI-0025 (integrated). Under the prospective migration posture this is a focused, high-value
discoverability correction (two MOC entries), suitable for a small hygiene slice -- not a broad
migration.

## safety / non-actions

No tooling, validator, manifest, migration, renaming, moving, or WI-0031 Slice 1 work is created or
authorized by this plan. Legacy records remain valid.

## recommended next slice

WI-0032 Slice 2 (documentation-impact declaration + authoring checklist), then Slice 3. WI-0031
Slice 1 remains sequenced after this standard is dispositioned.
