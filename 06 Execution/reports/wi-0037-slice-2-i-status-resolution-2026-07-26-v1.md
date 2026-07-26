---
title: "WI-0037 Slice 2-i Status Resolution 2026-07-26 v1"
type: "evidence-report"
date: "2026-07-26"
status: "implemented local -- RED-first, both harnesses green; NOT reviewed, NOT integrated, NOT pushed"
project: "DAI"
slice: "WI-0037 Slice 2-i: status-resolution contract, fixture corpus, finals-guard bracket correction, canonical operator status script"
repos:
  dai: "scripts+fixtures on local branch wi/0037-game-state-correctness-slice-2-i (base 0a9129b); zero C# changes; NOT integrated"
  dai-vault: "docs on local branch wi/0037-game-state-correctness-slice-2-i (base 5577f55); NOT pushed"
tags:
  - system-development
  - sports-v1
  - correctness
  - implementation
related:
  - "02 Platform/system-development/work-items/WI-0037-game-state-correctness-v1.md"
  - "06 Execution/reports/wi-0037-slice-2-status-resolution-design-2026-07-24-v1.md"
  - "06 Execution/patterns/game-status-recheck-discipline-v1.md"
---

# wi-0037 slice 2-i status resolution 2026-07-26 v1

## opening state and planning publication

Gate passed 2026-07-26 12:43Z: dai main == origin == direct remote == `0a9129b` (clean,
suite baseline 1780/1780); vault main == origin == direct remote == `14c3926`; planning
tip `5577f55` (parent `14c3926`, docs-only, local). Planning commit `5577f55`
adversarially re-reviewed (docs-only, scans clean, cited source lines re-verified live,
strict snapshot 26/0 on its tree) and PUBLISHED UNCHANGED by plain fast-forward: vault
main -> `5577f559da26503a52ac95d850756b2be7b431a1` (remote verified).
`ops/2026-07-24-daily-evidence` (clean, 0 unique) fast-forwarded `af69725` -> `5577f55`
(not pushed). Slice branches: `wi/0037-game-state-correctness-slice-2-i` in dai (base
`0a9129b`) and dai-vault (base `5577f55`). Preserved wi/0035 worktree untouched
(six dirty paths, hash `86aa8b74`).

## pre-edit contract map (verified against source before any edit)

Finals guard: params Competition/GamePks/ManifestPath/OutputPath/JsonOnly/FailOnPartial/
CheckLocalRows/RequireUnreconciled/ApiBaseUrl/BearerToken/ScheduleJsonPath; live fetch
`?sportId=1&gamePks=...` (no date filter); flattened ALL `dates[]` buckets before
per-pk matching (`:255` pre-edit); duplicate = any two flat matches; finality
abstract=Final AND coded=F; exit contract 0/1/2/3; sole harness
`test-check-settlement-finals.ps1` (offline via `-ScheduleJsonPath`; single-bucket
`Write-Schedule` shape); no script-to-script callers (operator + tests only). No
contradiction with the reviewed design.

## deliverables

1. **Contract** `scripts/dev/sports/game-status-resolution-contract-v1.md` --
   `game-status-resolution/1.0`: five-stage resolution (payload/bracket validation ->
   bracket selection -> exact-pk within bracket -> in-bracket uniqueness -> status
   validation), closed refusal vocabulary (`bracket_missing`, `game_not_in_bracket`,
   `duplicate_in_bracket`, `bucket_malformed`, `status_malformed`,
   `identity_mismatch`), normalized-status vocabulary identical to the platform's
   `NormalizeScheduleState`, reschedule context as data, no execution authority.
2. **Corpus** `scripts/dev/sports/fixtures/game-status-resolution-v1.json` -- exactly
   24 fixtures (gsr-01..gsr-24) incl. legit doubleheaders, the two-bucket
   postponed+makeup class shaped from the stored 823042 evidence, historical-bracket
   resolution, three-bucket context, true in-bracket duplicate, missing
   bracket/pk, malformed bucket/status, ET/UTC boundary, rescheduled start,
   suspended/cancelled/live/final, warmup alias, identity mismatch, and the guard
   READY/DEFECT pair. Consumed by name from the corpus by both harnesses (no copied
   constants); Slice 2-ii's xunit runner consumes the same file.
3. **Finals-guard correction** (`check-settlement-finals.ps1`): buckets preserved
   (never pre-flattened); new optional `-BracketDate` (validated `YYYY-MM-DD`; never
   inferred); per-game staged authority; out-of-bracket same-pk = reschedule context
   (`bucketDate` + `rescheduleContextCount` added to per-game output, additive);
   in-bracket duplicates remain DEFECT; without a bracket, single-bucket behavior is
   byte-compatible and a multi-bucket pk is a DEFECT naming the cause and instructing
   `-BracketDate`. Exit-code contract UNCHANGED (0/1/2/3).
4. **Operator script** `scripts/dev/sports/check-game-status.ps1` -- the canonical
   date-bracketed status query: required Competition/BracketDate/GamePk, optional
   expected identity, offline `-ScheduleJsonPath` seam, machine JSON (contract
   version, disposition, typed reason, normalized + raw status, identity, reschedule
   context, source mode/ref) with human output kept off the JSON stream; live mode
   implemented as `?sportId=1&date={bracket}&gamePks={pk}` but NOT executed this
   slice. Exit contract: 0 resolved / 1 refused / 2 source failure / 3 usage.
5. **Conformance harness** `scripts/dev/sports/test-check-game-status.ps1` -- corpus
   self-validation (version, 24 unique ids, closed-vocabulary refusals) + every
   operator-tagged fixture executed offline + context ordering/non-authority proofs +
   usage/source/transport exit distinctions + read-only and no-positional-bucket
   source checks.
6. **Doctrine** [[game-status-recheck-discipline-v1]] (ACTIVE pattern) -- the
   executable form of the July 23 corrective rule; historical reports unaltered.

## red proof (before any production edit)

`pwsh scripts/dev/sports/test-check-settlement-finals.ps1` -> exit 1 with EXACTLY the
new bracket assertions failing: gsr-23 with `-BracketDate` (capability absent -> exit 1
binding failure; READY/final/context assertions failed) and the no-bracket gsr-23 run
returning the verbatim false-duplicate reason `gamePk duplicated in source schedule
(2 entries)` where the corrected guard must name the multi-bucket cause. gsr-24's
bracketed duplicate check also RED pre-fix. Every pre-existing test stayed green
during RED. Output preserved in the session scratch (`s2i-red.txt`).

## green proof

- guard harness: **40 passed / 0 failed** (all pre-existing cases + gsr-23 READY with
  context 1 + no-bracket multi-bucket DEFECT naming date buckets + gsr-24 DEFECT
  retained with bracket).
- operator harness: **181 passed / 0 failed** (22 operator fixtures x full expectation
  set + corpus validation + behavioral/usage/source checks).
- no live network call anywhere (offline seams only); no C# suite run required (zero
  C# paths changed -- verified by scope audit).

## scope audit

Changed dai paths (exactly six): `check-settlement-finals.ps1`,
`test-check-settlement-finals.ps1`, `check-game-status.ps1` (new),
`test-check-game-status.ps1` (new), `fixtures/game-status-resolution-v1.json` (new),
`game-status-resolution-contract-v1.md` (new). Zero C# source or test paths; no
schema/migration/planner/binding/resolver(C#)/reconciliation/settlement-execution/
dependency change; `git diff --check` clean; machine-path and secret scans clean.
Slice 2-ii untouched (MlbEventResolver, OddsScheduleClient, nullable ScheduleState,
xunit runner all unimplemented).

## exit-code decisions

Finals guard: unchanged 0/1/2/3 (READY/BLOCKED/PARTIAL/DEFECT). Operator script: new
stable contract 0 resolved / 1 refused / 2 source-or-transport failure / 3 usage
error -- a typed refusal is never reported as success, and transport failure is never
conflated with refusal. Documented in the script header and the doctrine.

## commits and branch state

- dai: `dd760f9` "fix(sports): resolve game status within eastern bracket" on
  `wi/0037-game-state-correctness-slice-2-i` (base `0a9129b`), local only, NOT pushed,
  NOT integrated; tree clean.
- dai-vault: one docs commit on `wi/0037-game-state-correctness-slice-2-i` (base
  `5577f55`) -- this record, the doctrine pattern, the WI-0037 spec slice state, and
  the handoff.

## residual risks

The guard's live fetch still queries `gamePks=` (bracket filtering is applied to the
returned buckets); moving the live URL to `date=`+`gamePks=` was deliberately left out
to keep the change surface minimal -- the staged resolution makes the fetch shape
non-load-bearing. Recorded for reviewer attention, not a defect: offline semantics and
live semantics share the same resolution path.

## next governed action

Independent adversarial review of the Slice 2-i chain (both branches), then a separate
integration authorization. Slice 2-ii (C# conformance) remains unauthorized.
