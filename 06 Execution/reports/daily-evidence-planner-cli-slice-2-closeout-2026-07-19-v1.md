---
title: "Daily Evidence Planner CLI Slice 2 Closeout v1 (2026-07-19)"
type: "evidence-report"
date: "2026-07-19"
status: "complete"
project: "DAI"
slice: "WI-0034 Daily Evidence Planner Stage 0 (Slice 2: portable CLI + atomic publication)"
repos:
  dai: "code+tests (cli adapter; additive; branch wi/0034-daily-evidence-planner-cli)"
  dai-vault: "docs-only"
tags:
  - system-development
  - evidence-operations
  - sports-v1
  - implementation
related:
  - "02 Platform/system-development/work-items/WI-0034-daily-evidence-planner-stage-0.md"
  - "04 Products/sports-v1/daily-evidence-acquisition-orchestrator-v1.md"
  - "06 Execution/reports/daily-evidence-planner-stage-0-slice-1-closeout-2026-07-19-v1.md"
---

# daily evidence planner cli slice 2 closeout v1

## purpose

Record WI-0034 Slice 2: the portable, non-interactive offline CLI and atomic
artifact-publication boundary around the (independently reviewed and integrated) Slice-1
planner core. Committed locally on matching governed branches; NOT pushed / NOT merged.

## context

Executed under the two-phase Slice-1-review + Slice-2 authorization. Phase A completed
first: Slice 1 was independently reviewed (six corrections + the known closeout whitespace
defect), corrected in new commits (dai `e3ef9a5`, vault `47ff379`; originals `ff55398` /
`d5943a2` preserved), pushed, and fast-forward integrated (dai main `ac634b5..e3ef9a5`,
vault main `e5d90e9..6e5c99d`; both mains == origin verified post-push). Slice-2 branches
`wi/0034-daily-evidence-planner-cli` were created in both repos from those exact mains
before the first Phase B write.

## what shipped (dai, additive, 2 files)

- `services/agent-service/app/services/daily_evidence_planner_cli.py`: three explicit
  operations — `plan` (strict request file -> pure planner -> canonical board json on
  stdout, optional markdown projection, optional publication), `validate` (request or
  board structural validation, no planning side effects), `version` (machine-readable
  cli/request-schema/board-schema/planner/policy versions). Stable exit-code classes:
  0 success (any valid terminal board), 2 usage, 3 malformed/missing input, 4 stale
  evidence, 5 policy/schema/version mismatch, 6 unresolved identity, 7 inconsistent
  normalized state, 8 publication failure, 9 internal. Structured single-line json errors
  on stderr; no stack traces by default. No wall-clock reads, no ambient application data,
  no repository discovery, no inferred operator authorization, no network.
- `services/agent-service/tests/test_daily_evidence_planner_cli.py`: 29 offline tests
  (operations, strict boundary incl. duplicate keys / non-finite numbers / unknown and
  misspelled fields / null-vs-missing-vs-empty, every planner-error exit mapping, all six
  terminal outcomes exit 0, publication collision/replacement/partial-failure/cleanup,
  cross-process byte determinism, candidate reordering, out-of-repo invocation, module
  source scan for network/clock/ambient access, markdown-vs-published-board semantic
  equivalence).

## strict request boundary

Closed field sets at every level (request, verdict, candidate); duplicate json keys and
NaN/Infinity rejected at parse; unknown or misspelled fields rejected; missing key vs
explicit null vs empty list distinguished (missing candidates key = exit 3; null = planner
inconsistency exit 7 when addressable; empty list = evaluated zero-eligible board exit 0);
iso dates and explicit-utc timestamps validated; bool-as-int rejected; negative pool limits
rejected at the boundary (and again by the core). No input field can grant screening,
capture, scheduling, execution, settlement, tuning, mutation, or tool-selection authority —
the schema carries no such fields and `validate --board` additionally fails any board whose
safety-ledger entry is not `false`.

## publication transaction boundary (honest semantics)

Canonical json is the authoritative artifact and the commit marker. Per artifact: stage as
`<final>.staging` on the destination filesystem, fsync, independently re-parse and
byte-compare the staged json against the core renderer output, then one atomic
`os.replace`. Markdown commits first, json last, so a destination containing the json
artifact contains a fully committed board; markdown without its json sibling is
recognizably uncommitted. The multi-file pair is NOT claimed atomic — only each
`os.replace` is; the recovery rule (trust only a parsing `.json`) is documented in the
module header and tested. Without `--overwrite`, an existing destination fails closed
(exit 8) and the existing artifact stays byte-identical; failed runs remove staging files
and never leave a partial file at a final path. Artifact names are deterministic from
explicit inputs only (`daily-evidence-board-<target_date>-asof-<compact as_of>`).

## real-game-shape manual smoke (2026-07-19)

Shape-validation exercise only — NOT an operational evidence run.

- competition eligibility: MLB is the repository's canonical, buyer-ready, in-season
  integration (V1 release plan is the MLB concierge; dedicated MlbAvailability/MlbStarter/
  MlbFatigue/BaseballMarket contexts; `mlb_statsapi` is the free schedule-source
  convention). WNBA (not canonical), NBA Summer League (not the canonical NBA contract),
  and NFL/NCAAF/NCAAMB (smoke-level integrations) were not substituted.
- target date re-derived at execution: 2026-07-22 (upcoming; the prior July 22 observation
  treated as a hint only). One authorized read-only statsapi schedule request (no retry
  needed): 15 games observed, all `Scheduled` with stable gamePk identity, start times,
  teams, no doubleheaders -> 15 accepted, 0 excluded.
- normalized request built only in the uncommitted session scratchpad with a controlled
  offline policy-verdict fixture (`insufficient_market_disagreement`, provenance labeled
  `controlled-offline-fixture (smoke; not an operational readout)`) and
  `available_input_capabilities = ["input.schedule_slate"]` (honest: no market-divergence
  input exists offline).
- result: exit 0; outcome `EVIDENCE_NEEDED_INPUT_TYPES_NOT_ADDRESSABLE`; missing
  capability `input.market_divergence_screen`; slate NOT evaluated (eligible count null);
  safety ledger all false; artifacts published to scratch only; canonical json sha-256
  `2cf5cc88167d1096377a620b9bd91d1f9106a0cea7f47f4f85ce4a24e0de04cb`. Inputs were not
  manipulated to force a cohort. No game was screened, selected for betting, captured,
  scheduled, or executed. Scratch request and artifacts NOT committed.

## wi-0031 consumer constraints (future seam; no runtime dependency added)

For a later, separately authorized WI-0031 continuation consuming planner output:

- stable missing-capability record fields: `capability_id`, `blocked_objective`,
  `required_input_type`, `reasons` (code/params/priority), `provenance`,
  `blocked_transition` — implementation-independent, never a tool id.
- boundaries: request schema `daily-evidence-planner-request/1.0`, board schema
  `daily-evidence-board/1.0`, planner `daily-evidence-planner/1.0`, cli
  `daily-evidence-planner-cli/1.0`; supported criterion `discrimination_hybrid_v1`.
- authorization-pending semantics: every board ships an all-false safety ledger;
  `NotEvaluated != Allowed`; a capability recommendation is not a recipe, tool choice, or
  execution authorization; a valid CLI output authorizes no screening or capture; any Tool
  Gateway integration, model-assisted recommendation, or recipe generation requires a
  separate authorization. The planner and CLI have no runtime dependency on WI-0031 types.

## verification

Targeted cli: 29/29. Full agent-service suite: **529 passed / 0 failed** (500 post-review
+ 29). Cross-process determinism proven by subprocess tests (repeated + reordered runs
byte-identical) and, for Slice 1 during Phase A, by two fresh python processes with equal
canonical-json sha-256
`2a012163eaef888647f1af8c7882c22b8277d946028ab18a9a08b1bbcf042d38`. `git diff --check`
clean in both repos (branch ranges). Strict snapshot 23 WIs / 0 warnings. Machine-path/
secret/network/authority scans clean. Protected/classified drift byte-identical (dai csproj
`63ef2488`; vault `b3d68588` / `9127e464` / `68948ebd` / `25835e6c`; Welcome.md deleted).

## safety / non-actions

0 model/paid/odds/market/screening/capture/settlement calls; network = Phase A attributable
git fetch/push + exactly one read-only statsapi schedule request; 0 database reads/writes;
0 services/endpoints/schedulers/persistence/schema/UI; 0 Tool Gateway or WI-0031 runtime
integration; 0 Azure DevOps reads/writes; Slice-2 branches NOT pushed / NOT merged.

## next step

Independent review + integration of the local `wi/0034-daily-evidence-planner-cli`
branches (both repos). Slice 3 (schedule adapter) stays deferred until Slice 2 is
integrated; Slice 4 stays deferred until the stable CLI has one reviewed manual use.
A recommendation is not an authorization.
