---
title: "OKF YAML Front Matter Pattern v1"
type: "execution-pattern"
date: "2026-06-30"
status: "complete"
project: "DAI"
slice: "OKF YAML Front Matter Pattern v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - vault
  - okf
  - metadata
  - handoff
related:
  - "06 Execution/exports/calibration-metrics-export-2026-06-30.md"
---

# OKF YAML Front Matter Pattern v1

## purpose

A small, OKF-style YAML front matter block for DAI execution (`06 Execution`) docs so slices are easier to
classify, search, compare, and hand off -- and so a future tool can index them without parsing prose. Deliberately
minimal: nine fields, a few allowed values, no large taxonomy. The block is the doc's first lines; the existing
`# Title` + body is preserved unchanged below it. Obsidian parses this natively as document properties, so it adds
no new tooling.

## schema (the nine fields)

```yaml
---
title: "<human title, matches the H1>"
type: "<one of the small type set below>"
date: "<YYYY-MM-DD>"
status: "<one of the small status set below>"
project: "DAI"
slice: "<the slice name>"
repos:
  dai: "<repo-effect value>"
  dai-vault: "<repo-effect value>"
tags:
  - <free-form, lowercase, 2-5 typical>
related:
  - "<vault-relative path to a related doc>"
---
```

## allowed values (small, advisory not enforced)

- **type** (what kind of doc): `evidence-report` (a slice's run/evidence writeup), `execution-pattern` (a
  reusable convention like this one), `diagnostic` (root-cause), `export` (a data/metrics snapshot), `plan`
  (design/recommendation), `reconciliation` (settlement pass). Add a new value only when an existing one clearly
  does not fit -- keep the set short.
- **status**: `complete`, `in-progress`, `blocked`, `no-op` (a valid gated hold that did nothing), `superseded`.
- **repos.dai** / **repos.dai-vault** (effect on each repo): `unchanged`, `docs-only`, `code`, `code+docs`,
  `tests-only`.
- **project**: `DAI` (constant for now; the field exists so a future multi-project vault can filter).
- **tags**: free-form lowercase keywords; no controlled list (avoid over-modeling). 2-5 is typical.
- **related**: vault-relative paths to directly relevant docs (predecessors, the spec it implements, the
  diagnostic it answers). Keep to the few that matter, not an exhaustive backlink list.

`title`, `date`, `slice` are free text. Dates are absolute `YYYY-MM-DD` (never relative).

## why each field exists (how it helps DAI)

| field | helps |
|---|---|
| title | handoff retrieval -- a clean human name independent of filename drift |
| type | slice comparison -- group all `diagnostic` or all `export` docs; filter the index |
| date | status tracking + ordering -- absolute date for a chronological view |
| status | status tracking -- find all `blocked`/`no-op` slices at a glance (we have several gated holds) |
| project | future multi-project filtering (constant DAI today) |
| slice | handoff retrieval -- ties the doc to its slice prompt + handoff |
| repos | slice comparison + audit -- see at a glance whether a slice was docs-only or touched code/tests |
| tags | search -- lightweight cross-cutting retrieval (e.g. `calibration`, `asymmetric`, `reconciliation`) |
| related | prompt/context assembly + navigation -- a small dependency graph a future tool (or an agent assembling
  context) can walk |

The combination directly serves: **handoff retrieval** (title+slice+status), **status tracking** (status+date),
**slice comparison** (type+repos+tags), **prompt/context assembly** (related+tags), and **future automated
exports** (the whole block is machine-readable YAML, so an index/report can be generated without prose parsing).

## sample applied (small, per the adoption rule)

Front matter was applied to exactly two docs in this slice -- this pattern doc (above) and the latest export:

- `06 Execution/okf-yaml-front-matter-pattern-v1.md` (this doc) -- `type: execution-pattern`.
- `06 Execution/calibration-metrics-export-2026-06-30.md` -- `type: export`.

`06 Execution/handoffs/current-slice.md` was intentionally **not** front-mattered: it is a single rolling
append-log of every slice handoff (11k+ lines), not one document, so a single top-of-file block would be
misleading. Per-entry metadata there is out of scope.

No older execution docs were retrofitted.

## adoption rule

- **New** `06 Execution` docs SHOULD start with this front matter block.
- **Existing** docs are updated **opportunistically only when they are already being edited** for another reason
  -- never as a standalone mass retrofit. The 33 existing execution docs stay as-is until next touched.
- Keep the block to these nine fields. If a real need for a tenth field appears, add it here first and justify it;
  do not let per-doc fields proliferate.

## verification

This file's front matter is valid YAML (single `---` delimited block at the top of the file, mapping with
scalar/sequence values only). The applied sample (calibration export) parses the same way. No code, prompt
registry, allowlist, buyer copy, DB schema, or reconciliation state is touched -- docs-only.

## risks / deferred items

- Front matter is advisory, not enforced -- without a linter, fields can drift. Acceptable at this scale; a tiny
  read-only validator could be added later if the vault grows.
- A future **vault index/export tool** could read these blocks to produce a slice catalog (status board, type
  histogram, dependency graph). Deferred -- the pattern exists first so the data is there to index.
