---
title: "WI-0037 Slice 2-ii-c Operator/Harness and Status-Contract Parity 2026-07-27 v1"
type: "evidence-report"
date: "2026-07-27"
status: "complete"
project: "DAI"
slice: "WI-0037 Slice 2-ii-c: operator/harness and status-contract parity (local-only)"
repos:
  dai: "code"
  dai-vault: "docs-only"
tags:
  - system-development
  - sports-v1
  - game-state
  - operator-harness
related:
  - "02 Platform/system-development/work-items/WI-0037-game-state-correctness-v1.md"
  - "06 Execution/reports/wi-0037-slice-2-ii-architecture-review-2026-07-26-v1.md"
  - "06 Execution/patterns/game-status-recheck-discipline-v1.md"
---

# wi-0037 slice 2-ii-c operator/harness and status-contract parity

> **CURRENT STATE (2026-07-28): COMPLETE -- independently reviewed, locally integrated,
> unpublished, undeployed.** Everything from "## purpose" down to (and including) the
> "## slice 2-ii-c1 correction" section is the ORIGINAL 2026-07-27 record, preserved verbatim
> as historical evidence: at that time the package was IMPLEMENTED LOCAL and REVIEW-REQUIRED,
> NOT yet reviewed/integrated. Those "in-progress", "review-required", "NOT integrated", and
> "NOT pushed" statements describe the 2026-07-27 snapshot only. They were resolved on
> 2026-07-28: the package passed independent adversarial review
> (WI0037_SLICE2IIC_C1_INDEPENDENT_REVIEW_PASSED_LOCAL_INTEGRATION_READY, zero findings) and
> was integrated by pure local fast-forward to dai main `4c6cd98`; local vault main was
> advanced to the closeout commit `74a8188`. The authoritative current state is the
> "## closeout reconciliation (2026-07-28)" section at the end of this file. There is no live
> contradiction: `status: complete` in the frontmatter reflects this current state.

## purpose

Close the final WI-0037 hardening obligations bound by the Slice 2-ii architecture review:
F3 (null-safe finals-harness evidence), F4 (authority-plus-context live query), a type-level
status-detail requirement replacing a defaulted boolean, one c# schedule-state normalizer
consolidating the resolver and adapter alias tables, and cross-runtime normalization parity
vectors consumed by both runners. Contract stays `game-status-resolution/1.1`; the 25 scenario
fixtures and six refusal reasons are unchanged; no runtime result shape, refusal vocabulary,
or normalized vocabulary changed.

Terminal state: **WI0037_SLICE2IIC_IMPLEMENTED_LOCAL_REVIEW_REQUIRED**. Local-only: no push,
deploy, activation, migration apply, live provider/model/paid call, capture, reconciliation,
or settlement. No frontend change. Success does NOT close WI-0037.

## opening state and ancestry

- source local main = baf5e90 (2 commits ahead of origin/main aee1ade); this slice branch
  `wi/0037-game-state-correctness-slice-2-ii-c` from baf5e90 (worktree dai-2iic); source
  commit `80e8d1a2ab6ded8dd046c7e135a33698c52e93b5` ("feat(sports): operator/harness and
  status-contract parity (wi-0037 2-ii-c)") on top of baf5e90 (local, unpushed).
- vault local main = 849eb9d (contains 9cee53b, c497262, 849eb9d); origin/main = e217a3f;
  records branch `wi/0037-game-state-correctness-slice-2-ii-c-records` from 849eb9d (worktree dv-2iic).
- protected wi/0035 checkout de5791f, fingerprint 86aa8b74, six dirty paths (unchanged).
- selected-event activation disabled; migration 20260727133845 present, unapplied.

## baseline and red evidence

Baselines before edits: PowerShell `test-check-game-status.ps1` 187 passed / 0 failed;
`test-check-settlement-finals.ps1` 40 passed / 0 failed; full .NET 2025 passed / 0 skipped;
solution build 0 errors; git diff --check clean.

- F3 red: with malformed / non-json child output, the old inline detail interpolation
  `"(got '$($j.games[0].reason)')"` dereferences an absent games/reason and terminates under
  strict mode instead of reaching the tally. The new `-Probe` mode reproduces this: the probe
  asserts the old inline interpolation throws (pass), then records an ordinary failed
  assertion with a safe `<no json>` detail, reaches its tally, and exits nonzero.
- F4 red: the live schedule request contained `date=$BracketDate`; the desired request is a
  broad `sportId=1&gamePks=<pk>` query with no `date=`. The harness now statically asserts the
  corrected URI shape; no test issues a network request.
- Status requirement red: `Resolve` carried a defaulted `bool requireStatus = true`, letting
  callers omit the decision. The signature now requires a mandatory enum; a new test proves an
  undefined enum value is rejected.
- Parity red: the runners did not consume a complete alias matrix; the new corpus
  normalization-vector collection is now consumed generically by both runners with no-skip proof.

## f3 null-safe finals harness

`scripts/dev/sports/test-check-settlement-finals.ps1` only (no production guard change):

- added `Get-FirstGameReason` (null-safe extraction: guards missing `$Parsed`, missing/empty
  games array, and missing reason property; returns null when any level is absent) and
  `Get-ReasonDetail` (binds `(got '<no json>')` when no reason is readable).
- the two bottom corpus assertions (date-bucket reason; duplicate reason) now bind the safe
  reason and safe detail before the assertion instead of dereferencing inside interpolation.
- fixture ids, normal assertions, pass/fail tally semantics, and exit codes preserved.
- added a deterministic self-contained `-Probe` mode proving the harness reaches its tally on
  malformed output and exits nonzero; the normal harness still finishes green (40/40).

## f4 authority plus context

`scripts/dev/sports/check-game-status.ps1` + `test-check-game-status.ps1`:

- live request changed to `https://statsapi.mlb.com/api/v1/schedule?sportId=1&gamePks=$GamePk`;
  `date=` removed from the transport query. Exactly one GET, 30-second timeout preserved.
- `-BracketDate` remains mandatory and the sole local bracket-selection authority downstream in
  `Resolve-GameStatus`; offline `-ScheduleJsonPath` behavior unchanged; fail-closed resolution
  and all exit codes unchanged. `sourceRef` and the comment now truthfully describe a broad
  gamePk query with a locally selected bracket.
- no completeness field, new refusal, retry, fallback, positional bucket selection, or
  current-date inference added.
- the harness statically proves the URI shape (fetches by exact gamePk; no `date=` in the
  schedule query), the one-GET boundary, the 30-second timeout, and single GET method, without
  any live call.

## type-level status requirement

`GameStatusResolver.cs`, `MlbStarterClient.cs`, `SelectedEventResolution.cs`,
`GameStatusResolutionCorpusTests.cs`:

- new closed internal enum `GameStatusDetailRequirement { Required, NotRequired }`; it is a
  mandatory `Resolve` parameter with no default, placed before the optional
  `ExpectedGameIdentity`. An undefined enum value throws `ArgumentOutOfRangeException` (tested).
- every call site chooses explicitly: corpus and selected-event verification paths pass
  `Required`; starter identity-grounding paths pass `NotRequired`.
- `GameStatusResolution.Resolved` now accepts a nullable normalized status; the `null!`
  suppression is removed, so a `NotRequired` resolution with no detailed state resolves with a
  truthful null normalized status (tested), while `Required` still refuses `status_malformed`
  on a blank detailed state. No public HTTP contract, refusal code, stage ordering, or
  consumer requirement changed.

## normalizer consolidation

- new `platform/dotnet/DevCore.Api/Sports/ScheduleStateNormalizer.cs`: one sports-owned c#
  normalizer with the exact current mappings (scheduled/pregame/in_progress/final/postponed/
  cancelled/suspended, else unknown), preserving trimming and case-insensitivity.
- `GameStatusResolver` and `MarketContrastSourceAdapter` both route through it; both duplicated
  c# alias tables are removed (one c# alias table remains, in ScheduleStateNormalizer.cs).
- the PowerShell operator runner keeps its own `ConvertTo-NormalizedStatus` as a separate
  runtime implementation by design; parity is by corpus, not shared executable code.

## cross-runtime parity vectors

- one top-level `normalizationVectors` collection added to
  `scripts/dev/sports/fixtures/game-status-resolution-v1.json` (31 vectors: every documented
  alias, mixed-case, outer-whitespace, unknown-nonempty, and null/empty/whitespace-only with
  `blankDetailedState` truthfully flagging the required-status refusal semantics). The 25
  scenario fixtures are unchanged. Expectations live once in the corpus.
- both runners consume every vector generically with no-skip proof: the c# corpus runner adds
  `every_normalization_vector_is_selected_none_skipped` + a per-vector theory through
  `ScheduleStateNormalizer.Normalize`; the PowerShell harness dot-sources the operator runner's
  `ConvertTo-NormalizedStatus` and iterates every vector, asserting the blank-flag semantics.
- contract stays `game-status-resolution/1.1`; the vectors are additive conformance metadata
  with no runtime shape, refusal, or semantic change.

## contract and doctrine reconciliation

Contract version unchanged (`game-status-resolution/1.1`). No new refusal reason, status
semantic, or public contract. The recheck-discipline doctrine
(`06 Execution/patterns/game-status-recheck-discipline-v1.md`) is updated to record the
authority-plus-context live query (broad gamePk fetch + local bracket selection), the single
c# normalizer plus the separate PowerShell normalizer, and the shared parity vectors.

## tests and build

- PowerShell: `test-check-game-status.ps1` 195 passed / 0 failed (baseline 187 + 8 F4/vector
  assertions); `test-check-settlement-finals.ps1` normal mode 40 passed / 0 failed (exit 0),
  `-Probe` mode 1 passed / 1 failed by design (exit 1, reaches tally on malformed output).
- Focused .NET: GameStatusResolutionCorpusTests 63 passed; adapter + starter + selected-event
  + selected-event-creation 126 passed.
- Full .NET: 2059 passed, 0 skipped, 0 failed (baseline 2025 + 34 new corpus/vector/enum tests).
- Solution build: 0 errors.
- Warnings (accurate, pre-existing, not introduced by this slice): NU1903 high-severity
  advisories for Microsoft.OpenApi 2.0.0 and System.Security.Cryptography.Xml 10.0.7, plus
  pre-existing compiler/nullability/xUnit analyzer warnings. Not remediated. This is not a
  warning-free build.

## static audits and no-network proof

- one c# alias table remains (ScheduleStateNormalizer.cs); no defaulted boolean status
  requirement remains; every real `GameStatusResolver.Resolve` call names Required/NotRequired
  (the one `(GameStatusDetailRequirement)999` call is the deliberate undefined-value rejection
  test); the adapter uses `ScheduleStateNormalizer.Normalize` at all four sites.
- both runtime runners consume all 31 vectors (no-skip proven). The live URL contains
  `gamePks` and no `date=`. No test issues a network call (harness Invoke-RestMethod references
  are static source inspections; the offline path uses `-ScheduleJsonPath` or dot-source).
- the 25 scenario fixtures and six refusal reasons are unchanged. git diff --check clean;
  added comments lowercase ascii; changed paths confined to the authorized scope.

## release state and residuals

Release state: LOCAL-ONLY; UNPUBLISHED; UNDEPLOYED; selected-event activation DISABLED;
migration 20260727133845 UNAPPLIED. No product/deployment/activation/live-execution claim.

- SELECTED_EVENT_IDENTITY_PROPAGATION_REQUIRED_BEFORE_WI0037_CLOSE: remains RESOLVED (Slice
  2-iii-c local integration); unaffected by this slice.
- MULTI_INSTANCE_SELECTED_EVENT_ATOMICITY_REQUIRED_BEFORE_SCALE_OUT: unchanged (blocking scale-out).
- DURABLE_PREEXECUTION_SELECTION_DECISION_LEDGER_DEFERRED: unchanged (deferred/nonblocking).

Slice 2-ii-c is IMPLEMENTED LOCAL; INDEPENDENT REVIEW REQUIRED. WI-0037 remains in-progress.
Next governed action = one independent adversarial review of the complete source+vault
2-ii-c package; local integration and WI-0037 closeout occur only under a later authorization
after that review passes.

## slice 2-ii-c1 correction (2026-07-27, review findings)

An independent review of the 2-ii-c package (source 80e8d1a / vault d98d1a5) found three
issues, corrected in place on top of the originals (originals preserved as history; correction
commits sit on top: source `4c6cd985662caebec686d4b96a12a4104440e857` on 80e8d1a, vault records
commit on d98d1a5). Slice 2-ii-c and WI-0037 remain in-progress and review-required.

### F-1 (blocking): cross-runtime blank-state divergence

For a bracketed game whose authoritative `detailedState` is whitespace-only (e.g. three
spaces), the PowerShell `Resolve-GameStatus` stage-5 check used truthiness
(`-not "$($g.status.detailedState)"`), so a non-empty whitespace string was treated as present
and the game resolved as `disposition=resolved`, `normalizedStatus=unknown`. The C# `Required`
resolution correctly refuses `status_malformed` (it uses `IsNullOrWhiteSpace`). The original
normalization-vector tests only exercised the direct normalizer, so both runtimes returned
`unknown` at that level and the end-to-end divergence was invisible.

Why the original green tests missed it: the vector loops asserted `Normalize(input) == expected`
only; they never drove each vector through the staged resolver, where the blank-state refusal
lives. Both runtimes normalize whitespace to `unknown`, so the direct-level assertion passed on
both sides while the staged dispositions differed.

Correction (test side, strengthened first as red): both runners now drive every vector through
staged `Required` resolution over a minimal bracketed schedule -- blank inputs must refuse
`status_malformed` with a null normalized status; nonblank inputs must resolve to the expected
normalized value. Against 80e8d1a the strengthened PowerShell test failed for
`nv-31-whitespace-only` only (resolved/unknown instead of refused/status_malformed); the C#
staged assertion already passed. Correction (production side): `check-game-status.ps1` stage-5
now uses `[string]::IsNullOrWhiteSpace([string]$g.status.detailedState)`, matching the platform
fail-closed behavior. No C# resolver or normalizer change (C# was the correct side). All other
behavior preserved: absent/null/empty/whitespace refuse `status_malformed`, nonempty
unrecognized detail resolves as `unknown`, aliases normalize unchanged, bracket-first authority,
refusal vocabulary, output shape, exit codes, and contract version unchanged.

### F-2 (hygiene): inaccurate added-comment audit

The original report and current-slice claimed the added-comment audit was lowercase-ascii clean.
That was inaccurate. The exact original audit over `baf5e90..80e8d1a` added C#/PowerShell
comments was: **55 added comment lines, 19 uppercase-bearing, 0 non-ascii**. Correction: every
comment added across the branch was rewritten to generic lowercase prose (referring to concepts
generically instead of reproducing mixed-case symbol names; no code symbol, parameter, function,
test, or runtime string was renamed to lowercase). Corrected audit over the full corrected range:
**69 added comment lines, 0 uppercase-bearing, 0 non-ascii**.

### F-3 (low): stale documentation

- `scripts/dev/sports/game-status-resolution-contract-v1.md` called the C# conformance runner
  future work; corrected to state it currently consumes the corpus (25 scenario fixtures + 31
  normalization vectors).
- `game-status-recheck-discipline-v1.md` said the corpus has 24 vectors; corrected to
  distinguish 25 scenario fixtures, 31 normalization vectors, and six refusal reasons.

### corrected verification

PowerShell: `test-check-game-status.ps1` 195/0 (all vectors end-to-end, including staged blank
refusal); `test-check-settlement-finals.ps1` normal 40/0, `-probe` reaches its tally and exits
nonzero. Full .NET 2059/2059, 0 skipped; solution build 0 errors. 25 scenario fixtures, 31
normalization vectors, six refusal reasons byte-for-byte unchanged; one c# alias table; live URL
still `gamePks` with no `date=`; no network call. Warnings unchanged and pre-existing (NU1903
Microsoft.OpenApi 2.0.0 + System.Security.Cryptography.Xml 10.0.7 + compiler/nullability/xUnit;
not remediated). Original commits 80e8d1a / d98d1a5 preserved; correction commits on top;
local-only, unpushed, undeployed; activation disabled; migration unapplied.

## closeout reconciliation (2026-07-28)

The "in-progress / review-required / next = one independent adversarial review" surfaces above
are historical. They are now resolved and superseded by this dated section; the original
review-finding disclosures and the c1 correction record above are preserved unchanged for
history.

- **Independent review.** The complete corrected 2-ii-c + c1 package (source
  `baf5e90..4c6cd98`, vault records `849eb9d..462d7a4`) was independently adversarially
  reviewed with verdict **WI0037_SLICE2IIC_C1_INDEPENDENT_REVIEW_PASSED_LOCAL_INTEGRATION_
  READY** and **zero findings**.
- **Source fast-forward.** dai local `main` was advanced by pure fast-forward
  `baf5e90 -> 4c6cd98` (no merge commit, reviewed topic branch retained). dai `origin/main`
  remains `aee1ade` (observational, deliberately not refreshed); nothing was pushed.
- **Post-integration verification on source main `4c6cd98`.** PowerShell
  `test-check-game-status.ps1` 195/0; `test-check-settlement-finals.ps1` 40/0 and `-Probe`
  reaches its tally with one pass + one deliberate failure (exit 1 by design); focused
  `GameStatusResolutionCorpusTests` 63; combined focused 189; full `DevCore.Api.Tests`
  **2059/2059, 0 skipped**; solution build 0 errors; comment audit 69/0/0; one c# alias table;
  live URI `gamePks` with no `date=`; no network call. Warnings pre-existing (NU1903 +
  compiler/nullability/xUnit), not introduced, not remediated.
- **Vault closeout transaction.** A single documentation-only vault closeout commit -- the
  commit containing this closeout section, the new
  [[wi-0037-local-integration-and-governance-closeout-2026-07-28-v1]] report, and the WI/MOC/
  current-slice updates -- was created directly on top of `462d7a4`; local vault `main` was
  then advanced to that closeout commit by an atomic ref update (the final transaction step),
  without checking out or modifying the protected WI-0035 worktree. Vault `origin/main`
  remains `e217a3f`; nothing was pushed.
- **Retained release boundaries.** LOCAL-ONLY; UNPUBLISHED; UNDEPLOYED; selected-event
  activation DISABLED; migration `20260727133845` UNAPPLIED. Residuals: propagation RESOLVED;
  scale-out atomicity blocking before scale-out; durable ledger deferred/nonblocking. WI-0037
  is now **complete** (engineering + governance completion only; not scale-out, deployment,
  product, or production-activation readiness).
