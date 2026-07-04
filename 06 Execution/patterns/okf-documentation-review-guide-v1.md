---
title: "OKF Documentation Review & Study Guide v1"
type: "execution-pattern"
date: "2026-07-04"
status: "complete"
project: "DAI"
slice: "OKF Documentation Hygiene + Review Guide v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - okf
  - metadata
  - handoff
  - review
related:
  - "06 Execution/patterns/okf-yaml-front-matter-pattern-v1.md"
  - "06 Execution/reports/okf-migration-closeout-v1.md"
  - "06 Execution/reports/okf-documentation-hygiene-audit-v1.md"
  - "06 Execution/plans/okf-vault-taxonomy-plan-v1.md"
---

# OKF Documentation Review & Study Guide v1

## why this exists

DAI's knowledge lives in the vault, not in anyone's head. Slices hand off to the next agent (or the operator, days
later) almost entirely through docs. If a doc cannot be found, classified, or resumed from without guessing, the
handoff has failed regardless of how good the underlying work was.

OKF (the DAI vault's "organized knowledge format") is the convention that keeps docs **findable, comparable, and
resumable**. Two prior slices settled the mechanics: the [[okf-yaml-front-matter-pattern-v1]] fixed the 9-field
front matter schema, and the [[okf-migration-closeout-v1]] moved OKF from active backfill to opportunistic
maintenance and locked the type->folder taxonomy. Those docs are the *specification*. This guide is the
*review-and-study companion*: how a human or a future agent should read, evaluate, and learn from OKF docs, and how
to tell whether a new doc actually earns its place.

This guide does **not** redefine the schema. Where the general OKF idea and DAI's concrete convention differ, DAI's
convention wins (see "one canonical schema" below).

## mental model

Hold four sentences and you have OKF:

- **OKF makes each doc an addressable knowledge object** -- findable, searchable, comparable, reusable.
- **The YAML front matter is the label on the object** -- small, consistent, machine-readable.
- **The body is the evidence and reasoning** -- the source of truth; the label never replaces it.
- **The `related:` links are the project-memory graph** -- they let an agent walk from one object to the next.

Corollary the whole convention rests on: **small, consistent metadata beats large, inconsistent metadata.** Nine
fields everyone actually fills in are worth more than thirty fields that drift. This is why DAI deliberately caps
the schema rather than modeling every attribute a doc might have.

This is the platform-as-factory model applied to knowledge. The platform is the factory; docs are the durable
record of what each assembly line produced and decided. A well-labeled part can be pulled from the shelf and reused;
an unlabeled one gets remanufactured from scratch. OKF is the labeling standard on the shelf.

## one canonical schema (DAI's, not the generic one)

A doc that carries OKF front matter uses **exactly these nine fields**, defined authoritatively in
[[okf-yaml-front-matter-pattern-v1]]:

```yaml
---
title: "<human title, matches the H1>"
type: "<evidence-report | reconciliation | export | execution-pattern | diagnostic | plan>"
date: "<YYYY-MM-DD, absolute, never relative>"
status: "<complete | in-progress | blocked | no-op | superseded>"
project: "DAI"
slice: "<the slice name>"
repos:
  dai: "<unchanged | docs-only | code | code+docs | tests-only>"
  dai-vault: "<same value set>"
tags:
  - <lowercase, 2-5, prefer the controlled vocabulary>
related:
  - "<vault-relative path to a directly relevant doc>"
---
```

`type` is the authoritative classifier and maps **1:1 to the folder** the doc lives in (`evidence-report` ->
`reports/`, `reconciliation` -> `reconciliations/`, `export` -> `exports/`, `execution-pattern` -> `patterns/`,
`diagnostic` -> `diagnostics/`, `plan` -> `plans/`). Keep `type == folder` true; that invariant is what lets the
folder tree and the metadata agree.

A note on other OKF templates you may have seen (fields like `id`, `created`/`updated`, `area`, or a
`draft|active|accepted|archived` status set): those are a reasonable *generic* OKF shape, but DAI does not use them.
Adopting a second schema would fork the vocabulary and break the "one small consistent label" promise. **Do not
introduce competing fields.** If a real need for a tenth field appears, add it to
[[okf-yaml-front-matter-pattern-v1]] first with justification, then use it -- never per-doc.

## structure -- body shape for a slice/report doc

Front matter labels the object; the body carries the evidence. A slice or report doc reads fastest when it moves in
this order (lowercase `##` headings, matching the vault's house style):

- **purpose** -- why this doc exists, in one or two sentences.
- **context** -- what led to it (the prior slice, the gap, the trigger).
- **scope** -- what is included and, importantly, what is excluded.
- **key decisions / findings** -- the actual result. The part a reader came for.
- **evidence** -- commands, tests, run ids, commits, observations. The receipts.
- **safety / non-actions** -- what was explicitly *not* changed (paid calls, writes, migrations, prompt/routing/
  buyer/metrics). DAI slices live or die on this ledger; it is not optional.
- **next step** -- the one recommended next slice or decision, stated plainly.
- **related** -- links to connected docs (usually mirrors the `related:` front matter).

Not every doc needs every heading, but a doc that omits **evidence**, **safety/non-actions**, or **next step** is
usually the one that will not survive a handoff.

## structure -- body shape for a pattern/guide doc

A convention or guide doc (like this one) reads best as: **why this exists -> mental model -> structure -> usage
rules -> examples -> review checklist -> related.** The reader should be able to stop after "mental model" and still
apply the pattern correctly; everything after is depth.

## usage rules

- **Born in the right folder.** A new doc starts life in its `type` folder with full 9-field front matter. Nothing
  new lands flat in `06 Execution/`.
- **Keep the label small.** Nine fields. Absolute dates. 2-5 lowercase tags from the controlled vocabulary where one
  fits.
- **Body is the source of truth.** Never move a fact into front matter that belongs in the body. The label points
  at the object; it is not the object.
- **Link the few that matter.** `related:` is for genuine predecessors, the spec a doc implements, or the diagnostic
  it answers -- not an exhaustive backlink dump. Obsidian `[[wikilinks]]` in the body survive file moves (they
  resolve by basename); vault-relative `related:` paths do not, so they must be re-pointed if a target moves.
- **Preserve, don't rewrite.** When touching an existing doc, add metadata/structure; do not alter its factual
  conclusions, run ids, or test evidence.
- **Opportunistic, not mass.** Retrofit an old doc only when you are already editing it for another reason. No
  standalone backfill slices -- that thread is closed (see [[okf-migration-closeout-v1]]).
- **Rolling logs stay rolling.** `06 Execution/handoffs/current-slice.md` is one append-log of every handoff, not a
  single knowledge object. It is never front-mattered or reorganized. The same applies to backlogs, roadmaps, the
  `next-slice.md` proposal, and skills inventories -- these are intentionally flat.
- **Don't force ambiguous docs.** If a doc does not clearly fit one `type`, leave it flat and note why, rather than
  mis-filing it under a label that lies.

## examples

**A compliant reconciliation doc label** (from the current workflow):

```yaml
---
title: "Registry-Routed v8 Backed-Depth Cohort Resume v1"
type: "reconciliation"
date: "2026-07-04"
status: "complete (generation) -- PASS 7/7 registry provenance; settlement pending finality"
project: "DAI"
slice: "Registry-Routed v8 Backed-Depth Cohort Resume"
repos:
  dai: "unchanged (a923db4)"
  dai-vault: "docs-only"
tags:
  - prompting
  - registry
  - routing
  - calibration
  - cohort
related:
  - "06 Execution/reconciliations/paid-registry-routing-canary-v1.md"
  - "02 Platform/decisions/0009-registry-routing-canary-ready.md"
  - "06 Execution/reconciliations/source-readiness-preflight-gate-v1.md"
---
```

Why it reads well: `type == folder`, the `status` is honest about the split between generation (done) and
settlement (pending) rather than flattening to a bare `complete`, `repos.dai` pins the exact commit, and `related:`
walks straight to the canary it descends from and the ADR that authorized it.

**An anti-pattern:** a doc titled "misc notes" in `06 Execution/` root, no front matter, no `next step`, that says
"ran some analyses, looked fine." It is unaddressable (no type/status), unverifiable (no run ids), and unresumable
(no next step). A future agent has to redo the work to trust it.

## review checklist

Use this to decide whether a doc actually follows OKF. A "no" on any of the first three is a fix-now; a "no" lower
down is a quality note.

- [ ] **Purpose is clear** -- one glance tells you why the doc exists.
- [ ] **Type and status are obvious** -- front matter present, `type == folder`, `status` from the allowed set and
  honest (a partial result says so).
- [ ] **Front matter stays small** -- the nine fields, no invented extras, absolute dates, controlled-vocab tags
  where one fits.
- [ ] **Body is structured for fast review** -- a reader can find the finding, the evidence, and the safety ledger
  without hunting.
- [ ] **Evidence preserved** -- important run ids, commits, test counts, and explicit non-actions are all present
  and unaltered.
- [ ] **Related docs linked** -- the genuine predecessors / spec / answer are reachable; links are not an
  exhaustive dump.
- [ ] **Next action explicit** -- exactly one recommended next slice or decision, stated plainly.
- [ ] **Resumable by a stranger** -- could another agent pick this up and continue **without guessing** what
  happened or what to do next? If not, the doc is not done.

## common mistakes

- **Over-modeling the label.** Adding `id`, `owner`, `priority`, `updated`, `area`, etc. per doc. This is the exact
  failure OKF exists to prevent -- large inconsistent metadata. Nine fields, no more.
- **A second schema creeping in.** Copying a generic OKF template with different field names. One canonical schema;
  extend the pattern doc, don't fork it.
- **`type` disagreeing with the folder.** Breaks the one invariant the taxonomy relies on.
- **Front matter that lies.** `status: complete` on a slice whose settlement is still pending, or `repos.dai:
  unchanged` when code moved. The label must match reality; an honest split status is better than a clean false one.
- **Mass retrofit.** Re-touching dozens of old docs "to be consistent." That thread is closed; churn is the cost,
  and there is no retrieval problem to justify it.
- **Front-mattering a rolling log.** A single top-of-file block on `current-slice.md` (12k+ lines of many handoffs)
  is misleading -- it labels the container as if it were one object.
- **Dropping the safety ledger.** For DAI runtime-adjacent slices, "what was not changed" is load-bearing evidence,
  not boilerplate.

## related

- [[okf-yaml-front-matter-pattern-v1]] -- the authoritative 9-field schema and per-field rationale.
- [[okf-migration-closeout-v1]] -- the settled taxonomy, the type->folder map, and the opportunistic-maintenance
  rule.
- [[okf-documentation-hygiene-audit-v1]] -- the audit this guide was written alongside; current compliance state.
- [[okf-vault-taxonomy-plan-v1]] -- the folder/tag taxonomy plan behind the type set.
