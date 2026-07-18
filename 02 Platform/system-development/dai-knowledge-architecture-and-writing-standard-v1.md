---
title: "DAI Knowledge Architecture and Writing Standard v1"
type: "execution-pattern"
date: "2026-07-18"
status: "in-progress"
project: "DAI"
slice: "WI-0032 DAI Knowledge Architecture and Writing Standard v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - knowledge-system
  - documentation
  - governance
  - okf
related:
  - "02 Platform/system-development/knowledge-system.md"
  - "02 Platform/system-development/work-item-traceability.md"
  - "06 Execution/patterns/okf-documentation-review-guide-v1.md"
  - "02 Platform/system-development/work-items/WI-0032-dai-knowledge-architecture-and-writing-standard-v1.md"
---

# dai knowledge architecture and writing standard v1

## purpose

Define, in one normative place, how durable knowledge is identified, organized, authored,
related, validated, evolved, superseded, and discovered in the DAI vault. This standard is
prospective: it governs new durable knowledge and substantive edits; it does not invalidate
legacy records or authorize broad migration. It complements (does not replace) the OKF registry
and authority contract ([[knowledge-system]]), the OKF review guide, work-item traceability, and
the operating model. It is a writing/organization standard, not a runtime or tooling spec; no
parser, validator, manifest, hook, CI job, or schema is implemented by it.

## problem it solves

The vault has grown coherent per-record profiles (architecture topic docs, work items, plans,
reports, ADRs, MOCs, the rolling handoff log) and an OKF registry, but the *architecture* of the
knowledge -- how records aggregate into cohesive modules, which of physical/logical/navigational
organization answers which question, when a local module map or a machine manifest is justified,
how authority is prevented from leaking through links or inheritance, and how a writing slice
declares its knowledge impact before touching files -- has been implicit and re-derived per
slice. WI-0031 (a six-record capability module) is the first evidence that the per-record rules
are sufficient but the module-level architecture deserves a normative statement.

## strategic fit

Platform = factory; the vault is the factory's durable memory. The factory scales safely only if
a worker (human or agent) can find the one authoritative record, trust it, and change it in a
small, reviewable, reversible way without restructuring the whole shelf. This standard protects
the same long-term stock as the rest of the platform -- operator and agent trust in the knowledge
-- by making knowledge modules discoverable, authority explicit, and change lightweight but
traceable. It favors information flows and rules (high-leverage) over folders and manifests
(low-leverage).

## mental model

```text
Vault -> Knowledge areas -> Knowledge modules -> Typed records
```

Three organizational dimensions answer three different questions and must not be collapsed into
one directory tree:

- **Physical** -- where a record lives, by record profile and existing taxonomy (folders).
- **Logical** -- which cohesive capability/system/decision-thread/body of knowledge it belongs
  to (the module).
- **Navigational** -- how a human or agent discovers and traverses it (MOCs, module entry
  points, explicit links, work-item traceability, search, validated relationships).

A record has exactly one physical home and one primary purpose; it may participate in one logical
module and be reachable through several navigational paths. Authority is always explicit; context
may be inherited only when deterministic and non-authoritative.

## design principles (systems-thinking framing)

- **Modularity.** A knowledge module has one primary purpose, a clear boundary, a bounded set of
  reasons to change, one discoverable entry point, and explicit dependencies, authority, and
  lifecycle.
- **Small, safe changes.** Documentation changes are bounded, independently reviewable, validated
  before integration, traceable to a work item, and reversible without broad restructuring.
- **Fast feedback.** Where practical, changes support automated validation of metadata, profile,
  location, naming, links, relationships, authority, supersession, machine paths, secrets, and
  duplicate identities. (This standard identifies the levels; it implements no tooling.)
- **Empiricism.** No manifest, folder, local map, version number, or relationship mechanism is
  introduced because it looks elegant; each must solve a demonstrated discovery, validation,
  ownership, packaging, dependency, or lifecycle problem.
- **Systems thinking.** The vault is a living system with stocks (trusted knowledge, stale
  knowledge, unresolved decisions, evidence, operator trust), flows (authoring, review,
  integration, supersession, deprecation, migration), feedback loops (discoverability,
  consistency, reuse, maintenance), and delays (system change -> knowledge update). Rules and
  information flows are the major leverage points; ceremony that discourages timely documentation
  is a negative leverage point and is avoided.

## vocabulary

- **Vault** -- the complete governed DAI knowledge repository.
- **Knowledge area** -- a broad, durable domain of responsibility (e.g. platform architecture;
  agent orchestration; system development; evidence operations; tenant and permission
  management). Areas map loosely to top-level vault folders; the mapping is not required to be
  exact.
- **Knowledge module** -- a cohesive *logical* body of knowledge about one system, capability,
  standard, or decision thread. A module is a logical boundary and does not inherently require
  one physical directory. WI-0031 is the pilot module.
- **Record** -- an atomic durable document with one declared record profile and one primary
  purpose.
- **Record profile** -- the valid metadata, location, authority, lifecycle, and validation
  contract for a record type.
- **Map** -- a navigational record: a MOC, an area map, or a module entry point.
- **Manifest** -- an optional machine-readable module-level declaration, introduced only when
  machine-level module behavior is justified (see the manifest policy). Not required; "bundle" is
  not the default DAI term.
- **Work item** -- the governed authorization and execution lifecycle for meaningful change.
- **Evidence** -- a record demonstrating what happened, what was observed, or why a claim is
  supportable.
- **Decision** -- an explicit choice with rationale, alternatives, and consequences (an ADR).

OKF "bundle" is recorded as an interoperability concept, not adopted as DAI vocabulary.

## record-profile standard

DAI uses several valid record profiles; the nine-field OKF contract is **not** forced onto record
types that validly use another established profile. For each profile: purpose; canonical location;
metadata contract; authority level; mutability; supersession behavior.

| Profile | Canonical location | Metadata contract | Authority | Mutability |
|---|---|---|---|---|
| Architecture topic document | `02 Platform/architecture/**` | topic-doc `**status:**/**date:**` header | doctrine (why the system works this way) | revised in place; version suffix when a hard break |
| Normative standard | `02 Platform/**` (area-appropriate; system-development docs use OKF `execution-pattern`) | matches its area's neighbor profile | normative (the rule) | revised in place; version suffix on a breaking change |
| Work item | `02 Platform/system-development/work-items/WI-####-<slug>.md` | 9-field OKF `type: plan` (template) | authorization + state for one change | status lifecycle in place; never renumbered |
| Implementation plan | `06 Execution/plans/**` | 9-field OKF `type: plan` | sequencing (how change is ordered); no runtime authority beyond its WI | revised in place; version suffix per lifecycle |
| Decision / ADR | `02 Platform/decisions/####-<slug>.md` | ADR header | decision record (the choice) | immutable once accepted; superseded by a new ADR |
| Evidence report | `06 Execution/reports/**` | 9-field OKF `type: evidence-report` | evidence (what happened) | immutable after close; corrections append |
| Reconciliation / diagnostic / export | `06 Execution/{reconciliations,diagnostics,exports}/**` | 9-field OKF matching folder | evidence | immutable after close |
| Runbook | `06 Execution/plans/**` or area-appropriate | 9-field OKF or topic-doc per neighbor | operational procedure | revised in place |
| MOC / map | area root or `02 Platform/system-development/MOC - *.md` | navigation conventions (no OKF front matter) | navigation only -- never behavioral authority | append/curate in place |
| Current-state synopsis | `06 Execution/system-state-synopsis-v1.md` (untracked) | intentionally flat | current-state snapshot | overwritten; never front-mattered |
| Rolling handoff log | `06 Execution/handoffs/current-slice.md` | intentionally flat, append-only | continuation state | append-only; never reorganized |

Rules: `type == folder` for OKF records; do not force nine-field OKF onto architecture topic docs,
ADRs, MOCs, or the rolling logs; do not create a competing universal front-matter dialect; a
different valid profile is not, by itself, a defect.

## authority hierarchy

Records answer distinct questions; authority does not flow through links.

| Question | Answered by |
|---|---|
| What is the rule? | normative standard |
| Why does the system work this way? | architecture doctrine |
| What was decided (and the alternatives)? | ADR / decision |
| What change is authorized? | work item |
| How is the change sequenced? | implementation plan |
| What actually happened? | evidence report / handoff |
| What is the current state? | current-state synopsis (+ live verification) |
| Where do I find the related knowledge? | MOC / module map |

Invariants: a MOC or module map never becomes behavioral authority merely because it links to a
record; a handoff never supersedes a normative standard; a plan never silently grants runtime
authority beyond its governing work item; source and tests remain the behavioral truth once code
exists.

## physical, logical, and navigational organization

- **Physical placement** follows the record profile and existing taxonomy. Do not place a file
  because a folder is convenient.
- **Logical membership** is represented initially through existing mechanisms -- titles, explicit
  links, `related` metadata where the profile uses it, work-item relationships, and MOC placement
  -- **not** through a new metadata field. A new field requires a formal, documented extension
  decision (see manifest/extension policy).
- **Navigation** uses vault-wide/area MOCs for discovery; a module entry point or focused
  architecture map only when a subject is complex enough to justify it; explicit related links;
  work-item traceability; and stable search terms. Not every module needs a local index file.

A local module map is justified only when the module contains multiple durable record types, area
MOC discovery is insufficient, the subject has multiple dependencies or lifecycle threads, or
agent context assembly benefits from one bounded entry point.

## physical filesystem structure (binding)

The conceptual dimensions above are not enough; the physical vault has concrete rules.

### canonical hierarchy

- **Numbered top-level areas** (`01 Operating System`, `02 Platform`, `03 Niches`, `04 Products`,
  `05 Research`, `06 Execution`) are the durable knowledge areas; they are stable and are not
  renamed or added without an explicit decision.
- **Record-category folders** inside an area group records by profile/type: e.g.
  `02 Platform/architecture/`, `02 Platform/decisions/`, `02 Platform/system-development/`,
  `02 Platform/system-development/work-items/`, `06 Execution/{plans,reports,reconciliations,
  diagnostics,exports,patterns,handoffs}/`. `type == folder` for OKF records.
- **Durable topic subfolders** group several related durable records about one stable subject
  inside a category (e.g. `02 Platform/architecture/cognitive-factory/`,
  `04 Products/sports-v1/calibration/`).
- **Flat category folder** is the default: a record stays directly in its category folder until a
  durable topic subfolder is genuinely justified. Most records never need a subfolder.

### topic-folder creation threshold (reasoned, not a file-count rule)

Consider a durable topic subfolder when several of these hold: multiple durable records already
exist or are authorized; the topic has a stable architectural identity; the records change for
related reasons; the topic is expected to grow beyond one work item; the containing folder is hard
to scan; local navigation would materially improve discovery; the subject has multiple dependencies
or sub-concepts. This is a judgment threshold, never an inflexible file count. When in doubt, stay
flat and revisit when the records actually accumulate.

### folder anti-patterns (discouraged)

- one-file folders with no durable growth reason;
- folders named after a temporary work item (a WI is authorization, not a subject);
- folders created only for visual symmetry;
- `misc`, `other`, `new`, `temporary`, or similarly ambiguous folders;
- implementation-vendor folders for technology-independent doctrine;
- unnecessary duplicate nesting (a folder that only repeats its parent's meaning);
- excessive directory depth.

### depth guidance

Normal structural shape:

```text
vault area -> record category -> durable topic -> record
```

Deeper nesting is allowed only when a real, stable domain hierarchy justifies it. There is no
arbitrary universal maximum, but unusual depth requires explicit justification in the governing
work item.

### topic versus record-type grouping

- Architecture and doctrine **may** be grouped by durable topic (topic subfolders under
  `02 Platform/architecture/`).
- Work items remain grouped under the canonical work-item location
  (`02 Platform/system-development/work-items/`), never scattered by topic.
- Plans, reports, and handoffs **may** remain grouped by record type under `06 Execution/`.
- One logical knowledge module may therefore span several physical locations (its standard in
  architecture/system-development, its WI under work-items, its plan/report under 06 Execution) --
  this is expected, and navigation (MOC/map/links), not physical colocation, ties the module
  together.

### local map or MOC trigger

A durable topic folder receives a local map only when it contains multiple durable record types,
discovery through the parent MOC is insufficient, the subject has multiple dependencies or
subtopics, or bounded agent context assembly materially benefits. Not every folder gets a map.

### structural change discipline

Moving or regrouping existing records is itself a governed change and requires: a governed work
item; an exact path-impact declaration; link updates (body links, `related`, MOC/map entries);
MOC and map updates; validation; preserved Git history where practical (`git mv`, not
delete+create); explicit supersession or compatibility treatment when a stable path changes; and
**no cosmetic-only mass movement**. Regrouping for visual tidiness alone is disallowed.

## wi-0031 pilot assessment (embedded)

WI-0031 (Model-Assisted Capability Recommendation and Tool Selection) is the first evidence-backed
knowledge module. Integrated records reviewed: the parent work item; the capability-recommendation
normative standard; the implementation plan; the system-development MOC entry; the planning
closeout; and the integration handoff in `current-slice.md`.

What worked:
- Coherent module boundary (one subject: model-assisted capability recommendation + deterministic
  tool selection).
- One authoritative standard, one governing work item, one sequencing plan -- authority questions
  each answered by exactly one record profile.
- Discoverable via the system-development MOC entry, which links both the WI and the standard.
- Explicit relationships through `related` front matter and body wikilinks; reused the Tool
  Gateway doctrine and ADR-0007 provenance dialect rather than forking.
- Evidence of completion (closeout) and continuation state (handoff) both present.
- Limited duplication: the standard is normative, the WI governs state, the plan sequences.

What was difficult or implicit:
- The module has **no single entry point**: discovery relies on the system-development MOC, whose
  primary axis is work items, not capabilities -- a reader looking for "tool selection" finds the
  WI first, then the standard. A future area/module map keyed by capability would reduce this
  friction (candidate, not required yet).
- Logical membership is implicit (inferred from links/titles), not declared -- acceptable at one
  module, but the reason a future `module` label or manifest could earn its place.
- The architecture standard (topic-doc profile) and the WI/plan/closeout (OKF profile) validly use
  **different** front-matter profiles; this is correct but must be documented so a validator does
  not flag it as drift.

Dispositions:
- A local module map would add **marginal** value at one module; defer until a second module or an
  agent-context-assembly need appears.
- A manifest would add **no** value yet; no machine module behavior is required.
- Every rule here is **prospective**; WI-0031's records are not retrofitted (they already comply).

## naming and identity

Favor descriptive nouns/noun-phrases; lowercase kebab-case filenames consistent with existing
conventions; stable names for durable doctrine; dates only for inherently time-bound records
(evidence, daily artifacts); version suffixes only where an existing lifecycle rule justifies them
(a breaking revision of a standard/plan). Work-item titles restate the WI subject; headings match
the H1. Avoid `notes.md`, `misc.md`, `article-one.md`, `new-document.md`, repeated words,
ambiguous abbreviations, and names based only on the current implementation technology. Deprecated
records keep their names and are marked superseded (see below). Do not mass-rename historical
records.

## relationship and dependency conventions

Use existing mechanisms before inventing new ones. Roles: ordinary Markdown/wikilinks for body
references; canonical `related` metadata where the profile uses it; work-item traceability links
for authorization threads; MOC entries for discovery; ADR predecessor/successor links for decision
threads; explicit supersession links; explicit dependency statements in the body. Relationships
must be explicit -- no invisible semantic inheritance. Required validation (levels below;
tooling not implemented here): broken links; duplicate canonical records; unresolved supersession;
stale MOC entries; missing reciprocal relationships when a profile requires them; circular
dependencies where relevant. Do not implement a graph database or typed-edge runtime.

## metadata inheritance

Principle: **context may be inherited only when deterministic and non-authoritative; authority is
always explicit.** Never implicitly inherit approval, execution authority, policy status, decision
status, evidence sufficiency, effective version, supersession, tenant permissions, or calibration
outcomes. Non-authoritative defaults that *could* be inherited later (owning area, broad navigation
context, default tags, module label) are documented as *possible*, not implemented. **v1
recommendation: avoid inheritance entirely** -- declare context explicitly; the cost of a repeated
field is lower than the risk of a leaked authority.

## optional manifest policy

`bundle.yaml`/`module.yaml`/`README.md` are **not required**. A future manifest becomes justified
only when at least one concrete need exists: stable machine-readable module identity; module-level
dependency resolution; export/packaging; compatibility checking; ownership discovery; module-level
versioning; automated bounded context assembly; explicit entry-point discovery; or machine
validation not reliably expressible through existing records. A manifest must not be introduced to
mirror an external format. If later authorized: prefer the intuitive name from actual use; define a
versioned schema; keep authority-bearing state in the canonical records (not the manifest); avoid
duplicating canonical data; preserve ordinary Markdown readability; and document OKF export
compatibility separately.

## okf compatibility

Open Knowledge Format is an interoperability influence, not DAI's governing vocabulary. DAI records
are already OKF-compatible in spirit: typed Markdown with explicit metadata and links supports
agent consumption. Where DAI is stricter/different: DAI uses per-profile metadata (not one schema),
folder-as-type, and an explicit authority hierarchy rather than bundle directories. Future export
to an OKF-compatible representation could map a knowledge module to an OKF bundle and each record
to an OKF document, generating a manifest at export time -- **without** reorganizing the vault.
DAI does not require logical modules to become physical OKF bundles, and `bundle.yaml`/`README.md`
are **not** treated as core OKF requirements.

## documentation-slice impact declaration (mandatory, prospective)

Before writing, a future vault-writing slice declares: affected knowledge **area**; affected
knowledge **module**; governing **work item**; records **created**; records **modified**; record
**profiles**; **target directory** for each record; **whether a new subfolder is proposed** and, if
so, **why it is justified** against the topic-folder threshold; **folder-depth impact**; MOCs/maps
affected; relationships added/removed; **paths moved (if any)**; supersession impact; versioning
impact; validation required; exact **allowlisted paths**. If a proposed document has no obvious
area, module, record profile, authority role, or target directory, stop for a
documentation-architecture decision rather than placing it in a convenient folder.

## change flow

```text
governed work item
-> knowledge impact declaration
-> branch-before-write gate
-> typed record changes
-> relationship and navigation updates
-> validation
-> review
-> local commit
-> separate integration
-> truthful closeout and handoff
```

Lightweight path: urgent evidence records and operational handoffs keep traceability (a governing
or referenced WI, correct profile, append-only where required) but skip feature-class ceremony --
an append to the rolling handoff log or a single evidence report under an existing plan does not
require a new standard or module map. Routine governed operational cadence under an existing OKF
plan does not mint a work item (per WI-0007).

## supersession and deprecation

Revise in place for non-breaking corrections/additions. Require a new versioned record when a
change breaks the contract or the conclusion (a new `-v2`), linking predecessor/successor
explicitly and marking the old record `superseded`. MOCs stop surfacing obsolete records by
pointing to the successor. Historical evidence remains immutable -- newer doctrine never edits or
deletes what an evidence/reconciliation report recorded. Deprecated terminology is noted with its
replacement, not erased. Current-state records (mutable snapshots) are distinct from historical
reports (immutable); do not conflate them.

## validation model (levels; tooling not implemented here)

- **Record valid** -- satisfies its profile (metadata, location, naming). Today: OKF validation +
  the OKF review guide + front-matter checks.
- **Relationship valid** -- required links/dependencies resolve. Today: manual link checks; no
  automated broken-link validator across the vault (gap).
- **Module coherent** -- the logical module has a governing entry point, explicit authority, and
  noncontradictory records. Today: manual; no module-level validator (gap).
- **Vault discoverable** -- area MOCs expose the current canonical entry points. Today: manual MOC
  curation.
- **Slice complete** -- work item, changed records, verification, closeout, and handoff agree.
  Today: strict planning snapshot + the slice-runner/handoff discipline.

Missing-tooling gaps (broken-link, module-coherence, duplicate-canonical, stale-MOC) are recorded
for a future validation-gap-assessment slice; none is implemented here.

## migration posture

Prospective. Use the standard for new durable knowledge and when a module is substantively
changed. Avoid broad migration, mass front-matter rewrites, global renaming, and cosmetic moves.
Create focused migration work only when there is measurable discovery, validation, or maintenance
value. Legacy records are not invalid merely for predating the standard. (Recorded follow-up: the
system-development MOC currently registers through WI-0023 plus WI-0031/0032 but omits WI-0024 and
WI-0025 -- a concrete, small discoverability gap that would justify a focused correction under this
posture, not a broad migration.)

## approved uses

- Deciding where a new durable record lives, which profile it uses, and which module it joins.
- Declaring a writing slice's knowledge impact before touching files.
- Judging whether a module needs a local map or (rarely) a manifest.
- Reviewing a proposed doc against the authority hierarchy.

## disallowed uses

- Forcing nine-field OKF onto architecture topic docs, ADRs, MOCs, or rolling logs.
- Introducing a manifest/folder/field/version without a demonstrated need.
- Letting a MOC/map or a handoff act as behavioral authority.
- Mass migration, renaming, or moving records for cosmetic consistency.

## acceptance criteria

- Areas, modules, records, maps, manifests, work items, evidence, and decisions are defined.
- Physical, logical, and navigational organization are distinct and each answers its question.
- Valid record profiles are documented without a universal front-matter dialect.
- The authority hierarchy is explicit and links do not grant authority.
- WI-0031 is evaluated as a real pilot with concrete findings and dispositions.
- Naming/identity rules are prospective; inheritance never grants authority; manifests are optional
  and evidence-driven; OKF compatibility is documented without forcing bundle terminology.
- Future documentation slices must declare knowledge impact; supersession/deprecation, validation
  levels, and a prospective migration posture are defined.
- No broad migration or tooling was implemented.

## risks and failure modes

- Over-ceremony discouraging timely documentation -> mitigated by the lightweight evidence/handoff
  path.
- A universal front-matter dialect creeping in -> blocked by per-profile validation and the
  "different valid profile is not a defect" rule.
- Authority leaking through links or inheritance -> blocked by the explicit-authority invariant.
- Premature manifest/module-map machinery -> gated behind objective triggers and empiricism.
- Standard treated as retroactive -> blocked by the prospective migration posture.

## deferred decisions

- Whether to add a local module map for WI-0031 (defer to a second module or agent-context need).
- Whether any non-authoritative inheritance default is worth implementing (v1: none).
- Which validation-tooling gaps to close, and in what order (Slice 3/4).
- The second pilot module to test generalization (Slice 5).

## related docs

- [[knowledge-system]] -- the OKF registry and authority contract this standard builds on.
- `02 Platform/system-development/work-item-traceability.md` -- ids, branch/commit/link conventions.
- `02 Platform/system-development/operating-model.md` -- the development spine and thresholds.
- `06 Execution/patterns/okf-documentation-review-guide-v1.md` -- per-record OKF placement/review.
- `02 Platform/system-development/work-items/WI-0031-model-assisted-capability-recommendation-and-tool-selection-v1.md` -- the pilot module.
- `06 Execution/plans/knowledge-architecture-adoption-plan-v1.md` -- the prospective adoption plan.

## recommended next slice

WI-0032 Slice 2 (documentation-impact declaration + authoring checklist): codify the pre-write
declaration and reviewer checklist using existing documentation mechanisms only, no tooling. Then
Slice 3 (record-profile validation gap assessment). WI-0031 Slice 1 (offline domain contracts +
selection trace) remains sequenced after this standard is dispositioned.
