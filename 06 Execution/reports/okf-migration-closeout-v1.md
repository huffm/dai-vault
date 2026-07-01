---
title: "OKF Migration Closeout v1"
type: "evidence-report"
date: "2026-07-01"
status: "complete"
project: "DAI"
slice: "OKF Backfill Batch v7 + OKF Migration Closeout v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - okf
  - metadata
  - observability
related:
  - "06 Execution/plans/okf-vault-taxonomy-plan-v1.md"
  - "06 Execution/patterns/okf-yaml-front-matter-pattern-v1.md"
---

# OKF Migration Closeout v1 -- Evidence Report

## purpose

Close out the OKF vault migration. Backfill Batches v1-v7 moved the clearly-classifiable `06 Execution` docs into
the approved type-folders with 9-field OKF front matter. This note records the final taxonomy, totals, the one
intentionally-flat doc, and the go-forward rule: **OKF moves from active cleanup to opportunistic maintenance.**

## final folder taxonomy (06 Execution)

`type` (front matter) is the authoritative classifier and maps 1:1 to the folder:

| folder | type | count |
|---|---|---|
| `reports/` | evidence-report | 20 |
| `reconciliations/` | reconciliation | 8 |
| `exports/` | export | 4 |
| `patterns/` | execution-pattern | 3 |
| `diagnostics/` | diagnostic | 2 |
| `plans/` | plan | 2 |

Pre-existing non-OKF subfolders left untouched: `backlog`, `handoffs`, `launch-checklists`, `prompting`,
`roadmap`, `skills`, `weekly-plans`. ADRs remain in `02 Platform/decisions/` (000N numbering).

## docs moved across batches

| batch | moved | into |
|---|---|---|
| v1 | 3 | diagnostics / exports / plans |
| v2 | 5 | reconciliations (follow-up v1-v5) |
| v3 | 3 | exports (calibration export machinery) |
| v4 | 4 | reports (provenance/routing cluster) |
| v5 | 4 | reports (asymmetric cluster) |
| v6 | 5 | reports (capture/allowlist/canary) |
| v7 | 14 | patterns(3) / plans(1) / diagnostics(1) / reports(7) / reconciliations(2) |
| **total** | **38** | -- |

(Plus the v6 reconciliation doc, which was *born* directly in `reconciliations/` under the new rule.) Every move
used `git mv` (history preserved); every moved doc carries 9-field front matter with `type == folder`, validated
with pyyaml.

## intentionally left flat

- `06 Execution/system-state-synopsis-v1.md` -- **untracked/excluded by durable rule** (a point-in-time state
  snapshot, deliberately not committed). Not moved, not staged, not front-mattered. This is the only remaining
  flat doc and is intentional.

## link discipline used

- Inbound **`related:` YAML paths** (vault-relative) break on move -> grep-first and updated to the new path in
  every batch (v7 updated 9 such links).
- **Obsidian wikilinks `[[basename]]`** resolve by filename -> survive moves untouched (never edited).
- **Preserved, never rewritten:** `handoffs/current-slice.md` rolling history, bare-name inventory references in
  planning docs, and historical prose mentions.

## adoption rule going forward (settled)

1. **New docs are born in the correct type-folder** with the full 9-field OKF front matter and controlled-vocab
   tags. No new doc lands flat in `06 Execution/`.
2. **Old docs are updated/moved opportunistically only when already being touched** for another reason -- never
   as a standalone mass retrofit.
3. **Stop active OKF backfill after this slice.** Do not open further `OKF Backfill Batch` slices unless a
   concrete retrieval problem appears (e.g., a doc that is genuinely mis-filed and blocks a real lookup).
4. **Rolling logs stay rolling** -- `current-slice.md` is never front-mattered or reorganized.
5. **Ambiguous docs are not forced** -- if a doc does not clearly fit one `type`, leave it flat and note why
   rather than mis-filing it.

## controlled tag vocabulary (reference)

`calibration, reconciliation, prompt-registry, provenance, metrics, outcome, source-depth, buyer-safety,
observability, provisioning, okf, diagnostic`. Justified free tags in use: `asymmetric`, `capture`, `workflow`
(each materially improves retrieval for a specific doc class). Add a new controlled tag only via the taxonomy
plan.

## verification

- All 14 v7 moves validated with pyyaml (9 fields, `type == folder`). No stale vault-relative `related:` paths
  remain for any moved doc (grep-clean). `git status` shows only approved renames + the necessary `related:`
  edits + this closeout + the handoff append. `dai` unchanged; `dai-vault` docs-only.

## paid-call / activity status

**None.** No paid model calls, no game runs, no reconciliation writes, no code/prompt/allowlist/buyer/schema
change. Docs-only.

## next recommended slice

OKF is in **opportunistic maintenance** now. The next substantive thread is a calibration one:
**Live Calibration Cohort Capture vNext** -- run a small controlled set of fresh MLB analyses to grow calibration
sample size (especially `enriched_market_backed_depth` and `enriched_market_missing`). Before ANY paid analysis:
identify candidate games, show expected regimes, show which calls are paid, and **pause for explicit approval**.
Reconciliation of the 9 pending 07-02 backlog runs remains event-gated (Follow-up v7 when they are Final).
