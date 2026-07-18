---
title: "WI-0032 DAI Knowledge Architecture and Writing Standard v1"
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
  - work-item
  - knowledge-system
  - documentation
related:
  - "02 Platform/system-development/dai-knowledge-architecture-and-writing-standard-v1.md"
  - "02 Platform/system-development/knowledge-system.md"
  - "06 Execution/plans/knowledge-architecture-adoption-plan-v1.md"
---

# WI-0032 DAI Knowledge Architecture and Writing Standard v1

Category: knowledge-system architecture + documentation governance + vault writing standard.
Governing parent for a normative standard defining how durable DAI knowledge is identified,
organized, authored, related, validated, evolved, superseded, and discovered. Planning and
documentation only; no runtime or tooling implemented. Status stays `in-progress`.

## problem

DAI has coherent per-record profiles and an OKF registry, but the module-level knowledge
architecture -- how records aggregate into cohesive modules; which of physical/logical/
navigational organization answers which question; when a local module map or a machine manifest
is justified; how authority is prevented from leaking through links or inheritance; how a writing
slice declares its knowledge impact before touching files -- has been implicit and re-derived per
slice. WI-0031 (a six-record capability module) is the first evidence that per-record rules are
sufficient but the module-level architecture needs a normative statement.

## desired behavior

A normative standard exists so future durable knowledge is placed, profiled, related, validated,
and discovered consistently, with explicit authority and a lightweight-but-traceable change flow;
WI-0031 is evaluated as the pilot; adoption is prospective (no broad migration, no tooling).

## affected surfaces

- New standard `02 Platform/system-development/dai-knowledge-architecture-and-writing-standard-v1.md`.
- New adoption plan `06 Execution/plans/knowledge-architecture-adoption-plan-v1.md`.
- This work item + the system-development MOC entry.
- Planning closeout + rolling `06 Execution/handoffs/current-slice.md`.
- No DAI runtime/source surface; no existing record relocated or rewritten (WI-0031 records
  untouched).

## non-goals

No parser, validator, CLI, skill, hook, CI, schema, manifest, dashboard, search/graph service, or
model call. No broad migration, mass renaming, moving records, global front-matter normalization,
new top-level folders, or new metadata fields. No `bundle.yaml`/`module.yaml`/`README.md`
requirement. No relocation or rewrite of WI-0031. No universal front-matter dialect. No WI-0031
Slice 1 implementation.

## proposed vocabulary

Vault; knowledge area; knowledge module (logical, not necessarily one directory); record; record
profile; map (MOC / area map / module entry point); manifest (optional, evidence-driven); work
item; evidence; decision. OKF "bundle" recorded as an interoperability concept, not adopted.

## normative content (summary; full text in the standard)

Physical/logical/navigational separation; the record-profile table (no forced nine-field OKF);
the authority hierarchy (links never grant authority; handoff never supersedes a standard; a plan
never grants runtime authority beyond its WI); naming/identity rules; relationship/dependency
conventions using existing mechanisms; the inheritance principle (context may be inherited only
when deterministic and non-authoritative; v1 avoids inheritance); the optional-manifest trigger
policy; OKF compatibility without bundle terminology; the mandatory documentation-slice impact
declaration; the change flow with a lightweight evidence path; supersession/deprecation; five
validation levels with tooling gaps recorded; and a prospective migration posture.

## wi-0031 pilot findings

Embedded in the standard. Worked: coherent module boundary; one authoritative standard/WI/plan;
MOC discoverability; explicit relationships; reuse over fork; completion + continuation state.
Difficult/implicit: no single module entry point (MOC is WI-keyed, not capability-keyed); logical
membership implicit; different-but-valid front-matter profiles across records. Dispositions: local
module map = marginal value, defer; manifest = no value yet; all rules prospective (WI-0031 already
complies, not retrofitted).

## authority model

Records answer distinct questions (rule / why / decided / authorized / sequenced / happened /
current / where). A MOC or module map is navigation only. A handoff never supersedes a normative
standard. A plan never silently grants runtime authority beyond its governing work item.

## documentation-slice declarations

Future vault-writing slices declare area, module, governing WI, records created/modified, profiles,
MOCs/maps affected, relationships, supersession/versioning impact, validation required, and exact
allowlisted paths before writing; stop for a documentation-architecture decision if a doc has no
area/module/profile/authority role.

## validation strategy

Five levels (record valid / relationship valid / module coherent / vault discoverable / slice
complete) mapped to existing checks (OKF validation, strict planning snapshot, slice-runner/handoff
discipline) with tooling gaps (broken-link, module-coherence, duplicate-canonical, stale-MOC)
recorded for a later gap-assessment slice. No tooling implemented here.

## supersession and deprecation

Revise in place for non-breaking changes; new versioned record on a breaking change with explicit
predecessor/successor links and `superseded` status; MOCs point to successors; historical evidence
immutable; deprecated terminology noted with replacement; current-state vs historical distinguished.

## optional manifest threshold

`bundle.yaml`/`module.yaml`/`README.md` not required; a manifest is justified only on a concrete
machine-level need (identity, dependency resolution, export, compatibility, ownership, module
versioning, context assembly, entry-point discovery, or machine validation not reliably expressible
in records). Never introduced to mirror an external format.

## okf compatibility

OKF is an interoperability influence, not the governing vocabulary; DAI records are OKF-compatible
in spirit; future export could map a module to an OKF bundle without reorganizing the vault;
`bundle.yaml`/`README.md` are not core OKF requirements.

## migration posture

Prospective: apply to new durable knowledge and substantive edits; avoid broad migration/renaming/
moves; focused migration only on measurable discovery/validation/maintenance value; legacy records
not invalid for predating the standard.

## decomposition

Slice 1: this standard + pilot disposition. Slice 2: documentation-impact declaration + authoring
checklist (existing mechanisms, no tooling) -- DELIVERED 2026-07-18 as
`06 Execution/patterns/documentation-slice-impact-declaration-v1.md`. Slice 3: record-profile
validation gap assessment.
Slice 4: optional lightweight validator improvements (separate authorization; proven gaps only).
Slice 5: prospective adoption in a second knowledge module (generalization test). Promote a slice
to a child WI only on an independent authority/tooling/rollback boundary.

## acceptance criteria

- The normative standard exists with areas/modules/records/maps/manifests/work-items/evidence/
  decisions defined; physical/logical/navigational organization distinct; record profiles
  documented without a universal dialect; authority hierarchy explicit; WI-0031 evaluated as a
  pilot; naming/relationship/inheritance/manifest/OKF rules present; documentation-slice
  declaration mandatory; supersession/validation/migration defined; no broad migration or tooling;
  local commits + continuation handoff complete.

## test plan

No source tests (docs-only planning). Verification is metadata/profile/link/OKF/snapshot/scan-based
(see verification commands).

## implementation notes

Standard placed in `02 Platform/system-development/` beside `knowledge-system.md`, matching the
`type: execution-pattern` OKF profile of its neighbors. Pilot embedded in the standard to avoid a
separate record. Adoption plan under `06 Execution/plans/`.

## docs to update

`02 Platform/system-development/MOC - DAI System Development.md` (register WI-0032). No historical
record rewritten.

## verification commands

- strict planning snapshot `-Strict` (exit 0, 0 warnings)
- OKF/front-matter + related-link validation on the standard, WI, plan; profile check per record
- `git diff --check`; staged-allowlist print; protected-file hash open/close; machine-path/secret
  scans

## risks

Over-ceremony; universal-dialect creep; authority leakage via links/inheritance; premature manifest
machinery; retroactive misreading -- each mitigated by an explicit rule in the standard.

## links

- work item: WI-0032 (ADO: AB#— when wired)
- branch: `wi/0032-dai-knowledge-architecture-and-writing-standard` (dai-vault; vault-only docs WI)
- pr: — (not pushed/merged this slice)
- commits: — (recorded in closeout at slice close)
- tests: none (docs-only planning)
- verification notes: see the planning closeout and current-slice handoff
- docs updated: standard, adoption plan, MOC, closeout, current-slice; Slice 2 added
  `06 Execution/patterns/documentation-slice-impact-declaration-v1.md` + its closeout
- lessons: separate physical/logical/navigational organization; keep authority explicit and out of
  links/inheritance; make manifests/maps evidence-driven, not elegant-by-default

## final handoff requirements

Slice handoff appended to `current-slice.md`; disposition: knowledge architecture standard and
pilot disposition complete; prospective adoption planned; no broad migration; no runtime or
validation tooling; local branch and commits only; not integrated. Status remains `in-progress`.
