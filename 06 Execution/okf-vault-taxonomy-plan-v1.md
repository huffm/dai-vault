---
title: "OKF Vault Taxonomy Plan v1"
type: "plan"
date: "2026-07-01"
status: "complete"
project: "DAI"
slice: "OKF Vault Taxonomy Plan v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - okf
  - vault
  - metadata
  - taxonomy
related:
  - "06 Execution/okf-yaml-front-matter-pattern-v1.md"
  - "02 Platform/decisions/0005-persist-assembly-error-detail.md"
---

# OKF Vault Taxonomy Plan v1

## purpose

Design a small, practical OKF-style taxonomy for the DAI vault -- folders/subfolders, front matter `type`, and a
compact tag vocabulary -- so slices are easier to retrieve, compare, and assemble into handoffs, and so a future
catalog/export tool can index the vault without prose parsing. This is a **plan only**: it defines the target
structure and adoption rules and recommends a small first migration batch. It performs **no file moves and no
mass retrofit**. It extends `okf-yaml-front-matter-pattern-v1.md` (which defined the 9-field block) with the
folder + tag layer.

## current-state inventory (read-only, nothing edited)

- `06 Execution/`: **38 top-level `.md` docs** + 7 existing subfolders
  (`backlog`, `handoffs`, `launch-checklists`, `prompting`, `roadmap`, `skills`, `weekly-plans`).
- Front matter present on **6 of 38** top-level docs (everything created since OKF Front Matter Pattern v1);
  the other ~32 predate it and carry none.
- `02 Platform/decisions/`: **5 ADRs** (`0001`..`0005`), numbered, no OKF front matter.
- None of the proposed type-folders (`reports`, `exports`, `reconciliations`, `diagnostics`, `plans`,
  `patterns`) exist yet; `handoffs` already exists and holds the single rolling `current-slice.md` append log.

Observed shapes (the doc kinds actually in the vault), with representative members:

| shape | count (approx) | examples |
|---|---|---|
| evidence report (slice run/writeup) | ~15 | `persist-assembly-error-detail-v1`, `asymmetric-*`, `default-allowlist-widening-v1` |
| reconciliation (settlement pass) | ~7 | `outcome-reconciliation-follow-up-v1..v5`, `live-batch-and-settlement-reconciliation-gate-v1` |
| export (data/metrics snapshot or export endpoint) | ~5 | `calibration-metrics-export-2026-06-30`, `platform-side-prompt-provenance-calibration-export-v1` |
| plan (design/recommendation) | ~3 | `evidence-regime-taxonomy-recipe-expansion-plan-v1`, this doc |
| pattern / doctrine (reusable convention) | ~3 | `okf-yaml-front-matter-pattern-v1`, `agent-slice-workflow-doctrine-v1` |
| diagnostic (root-cause / audit) | ~2 | `registry-assembly-error-diagnostic-v1`, `prompt-ledger-hook-state-audit-v1` |
| state / synopsis | 1 | `system-state-synopsis-v1` (untracked/excluded) |
| handoff (rolling append log) | 1 | `handoffs/current-slice.md` |
| ADR (platform decision) | 5 | `02 Platform/decisions/0001..0005` |

## proposed folder taxonomy (target for NEW docs; existing docs migrate slowly)

Under `06 Execution/`:

```
06 Execution/
  reports/           # evidence-report: a slice's run/evidence writeup (the bulk)
  exports/           # export: a data/metrics snapshot or an export-endpoint slice
  reconciliations/   # reconciliation: a settlement/outcome pass
  diagnostics/       # diagnostic: root-cause or state audit
  plans/             # plan: design/recommendation (like this doc)
  patterns/          # execution-pattern: a reusable convention/doctrine
  handoffs/          # the rolling current-slice.md append log (already exists)
  (unchanged pre-existing: backlog, launch-checklists, prompting, roadmap, skills, weekly-plans)
02 Platform/
  decisions/         # ADRs (keep 000N numbering; add OKF-compatible metadata where practical)
```

`type` (front matter) is the **authoritative, machine-readable classifier** and maps 1:1 to the folder name:

| front matter `type` | folder |
|---|---|
| `evidence-report` | `06 Execution/reports/` |
| `export` | `06 Execution/exports/` |
| `reconciliation` | `06 Execution/reconciliations/` |
| `diagnostic` | `06 Execution/diagnostics/` |
| `plan` | `06 Execution/plans/` |
| `execution-pattern` | `06 Execution/patterns/` |
| (ADR) | `02 Platform/decisions/` |

Because `type` is authoritative, retrieval/catalog tooling keys on the **front matter field, not the folder path**
-- so a doc is correctly classified even before it is physically moved. The folder is a physical convenience that
NEW docs adopt immediately and OLD docs migrate into opportunistically. This is the reconciliation between "give
us folders" and "do not move large numbers of files": the taxonomy is real today via `type`; the folders fill in
over time.

## folder-vs-tag rule

- **Folder = exactly one, = primary document TYPE / lifecycle.** Answers "what KIND of doc is this?"
  (report / export / reconciliation / diagnostic / plan / pattern / handoff; ADRs live in `decisions`). Mirrors
  the front matter `type`.
- **Tags = zero-or-more, = cross-cutting TOPIC / subject.** Answers "what is this ABOUT?" A reconciliation doc and
  an export doc can share the `calibration` tag; the folder still separates them by kind.
- A doc has **one** type/folder and **as many tags as apply** (2-5 typical). Never encode a topic as a folder or a
  type as a tag.

## compact tag vocabulary (controlled, advisory)

Keep tags to this small controlled set; add a new tag only when a real doc does not fit and record it here.

| tag | meaning |
|---|---|
| `calibration` | calibration reads, accuracy, confidence discrimination |
| `reconciliation` | settlement of outcomes against finals |
| `prompt-registry` | registry recipes, canary, allowlist, assembly |
| `provenance` | prompt-route provenance, route keys, attribution |
| `metrics` | metrics/rows endpoints, aggregation, exports of numbers |
| `outcome` | AgentRunOutcome / evaluation / directional result |
| `source-depth` | evidence richness / source-depth regime signal |
| `buyer-safety` | buyer surface / copy / route guarantees, fail-closed |
| `observability` | diagnostics, audits, read-only inspection surfaces |
| `provisioning` | environment/db/migration application, dev infra |
| `okf` | vault metadata / taxonomy / front matter itself |
| `diagnostic` | root-cause analysis (pairs with the `diagnostic` type) |

Notes: `prompt-registry` covers the canary/recipe/allowlist cluster; `provenance` covers route-key/attribution.
`asymmetric` remains acceptable as a free extra where a doc is specifically about the asymmetric regime split, but
prefer the controlled set first.

## adoption rules

1. **New** `06 Execution` docs are created **in the correct type-folder** with the full 9-field OKF front matter
   and tags from the controlled vocabulary.
2. **Existing** docs are updated/moved **opportunistically only when already being touched** for another reason --
   never as a standalone mass retrofit. The ~32 un-front-mattered docs stay where they are until next touched.
3. **Large file moves require a separate migration slice** (`OKF Backfill Batch v*`), done in small batches
   (2-5 files) so link updates stay reviewable. A migration batch MUST `grep -rl <filename>` first and update
   every inbound `related:` path and prose link before/with the move.
4. **Rolling logs are not front-mattered.** `handoffs/current-slice.md` is a single append log of every slice's
   handoff, not one document; it keeps its per-entry `# Slice` headers and is never given a top-of-file block or
   reorganized as a single doc.
5. **ADRs keep their `000N` numbering** in `02 Platform/decisions/`; they MAY additionally carry OKF-compatible
   metadata (a `type: adr` / `tags` block) where practical, but the numbering + ADR body remain the primary form.
6. **Tag discipline:** use the controlled vocabulary; a genuinely new cross-cutting tag is added to the table
   above (in this doc) before use. Keep the block to the nine OKF fields; do not let per-doc fields proliferate.

## recommended first migration batch (do NOT execute this slice unless approved)

A safe, high-signal first batch of **3 files that demonstrate three folders at once**, each an unambiguous single
type with low coupling:

| file | -> folder | front matter today | link action for the migration slice |
|---|---|---|---|
| `registry-assembly-error-diagnostic-v1.md` | `06 Execution/diagnostics/` | none (add on move) | update inbound `related:` in `persist-assembly-error-detail-v1.md`, `calibration-route-attribution-fix-v1.md`, and ADR `0005` |
| `calibration-metrics-export-2026-06-30.md` | `06 Execution/exports/` | `type: export` (ready) | update inbound `related:` in `okf-yaml-front-matter-pattern-v1.md`, `calibration-rows-export-endpoint-v1.md` |
| `evidence-regime-taxonomy-recipe-expansion-plan-v1.md` | `06 Execution/plans/` | none (add on move) | grep inbound refs; add front matter (`type: plan`) on move |

Rationale: 3 distinct types prove the structure with one member each; only one file (`calibration-metrics-export`)
is already OKF-ready, the other two get front matter added as they move (allowed by rule 2). Natural **batch 2**:
the coherent `outcome-reconciliation-follow-up-v1..v5` series -> `06 Execution/reconciliations/` (whole series
together to avoid a split, with their mutual `related:` paths + handoff path references updated).

## what this slice deliberately did NOT do

- No file was moved, created (other than this plan doc), renamed, or retrofitted.
- No existing doc's front matter or body was edited.
- `handoffs/current-slice.md` was not reorganized (only appended, per its normal rolling-log use).

## verification

- YAML: this doc's front matter is a single `---`-delimited block of scalar/sequence values -> valid YAML
  (validated with a YAML parser). The two example blocks in this doc are illustrative fences, not front matter.
- No file moves occurred (`git status` shows only this new doc + the handoff append; no renames/deletes).
- `dai`: unchanged (no code, prompt, allowlist, buyer, schema, or reconciliation touch). `dai-vault`: docs-only.
- No paid model calls.

## risks / deferred items

- Folders and tags are advisory (no linter). Acceptable at ~40 docs; a tiny read-only validator/catalog tool
  could enforce `type`<->folder and the tag vocabulary later (deferred; the data will exist once docs are
  front-mattered).
- Physical migration is deferred to `OKF Backfill Batch v*` slices; until then docs are correctly classified by
  `type` even while still sitting in the flat `06 Execution/` root.
- Moving files will break Obsidian wikilinks / `related:` paths if not updated -- hence the grep-first rule (3).

## next recommended slice

- If StatsAPI shows **824818 (07-01) Final**: **Outcome Reconciliation Follow-up v6** (per-run
  `/{28bd433e}/outcome`; do NOT identity-reconcile 824818).
- If still not Final: **OKF Backfill Batch v1** on the 3-file first migration batch above (move + front-matter +
  inbound-link updates, small and reviewable).
