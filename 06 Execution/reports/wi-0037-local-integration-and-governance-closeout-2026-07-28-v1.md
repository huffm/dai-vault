---
title: "WI-0037 Local Integration and Governance Closeout 2026-07-28 v1"
type: "evidence-report"
date: "2026-07-28"
status: "complete"
project: "DAI"
slice: "WI-0037 local integration and governance closeout (Slice 2-ii-c + c1 fast-forward; WI-0037 close)"
repos:
  dai: "code (Slice 2-ii-c + c1 fast-forward to local main 4c6cd98; unpublished, undeployed)"
  dai-vault: "docs-only (this report, WI, MOC, current-slice, slice report)"
tags:
  - system-development
  - sports-v1
  - game-state
  - closeout
related:
  - "02 Platform/system-development/work-items/WI-0037-game-state-correctness-v1.md"
  - "06 Execution/reports/wi-0037-slice-2-ii-c-operator-harness-parity-2026-07-27-v1.md"
  - "06 Execution/patterns/game-status-recheck-discipline-v1.md"
---

# wi-0037 local integration and governance closeout

## purpose and authorization boundary

Complete one bounded, local-only operation: integrate the already independently reviewed
WI-0037 Slice 2-ii-c + c1 source package onto local source `main` by fast-forward, run
post-integration verification, complete WI-0037's governance obligations in the reviewed vault
topic worktree, create exactly one new vault closeout commit, and atomically advance local
vault `main` to it.

This operation was authorized for those exact local actions only. It was NOT authorized to and
did NOT fetch, push, publish, open a PR, deploy, activate selected-event execution, apply a
migration, start/stop services, query a database, call a live provider, capture/reconcile/settle
a game, or spend money. Remote publication and deployment belong to a future release program and
are not prerequisites for local slice completion in this hardening phase.

## opening path/ref/preservation evidence

Repository roots verified with `git rev-parse --show-toplevel`: `<DAI_REPO_ROOT>` and
`<DAI_VAULT_ROOT>`. Placeholders resolved from the tracked template and the uncommitted local
map; durable output uses placeholders.

- Source canonical worktree on `main` at `baf5e90` (pre-integration), clean; `origin/main` =
  `aee1ade` (observational; `git fetch` deliberately NOT run).
- Reviewed source topic `wi/0037-game-state-correctness-slice-2-ii-c` tip `4c6cd98`, linear
  chain `baf5e90 -> 80e8d1a -> 4c6cd98`, no merge commits, no remote-tracking branch, local
  `main` an ancestor of the tip.
- Reviewed vault records topic `wi/0037-game-state-correctness-slice-2-ii-c-records` tip
  `462d7a4`, linear chain `849eb9d -> d98d1a5 -> 462d7a4`, clean, no remote-tracking branch,
  local vault `main` = `849eb9d` an ancestor of the tip; `origin/main` = `e217a3f`.
- Protected WI-0035 canonical vault worktree on `wi/0035-provider-event-binding` at `de5791f`
  with exactly six pre-existing dirty paths and fingerprint (`git diff | git hash-object
  --stdin`) `86aa8b74a3ec5d0856d7e043cff24241c54b69f7`.
- Runtime/release state: `SelectedEventActivation.Enabled=false` with blank DeploymentId/
  EvidencePath; migration `20260727133845_AddAgentRunDomainExecutionProvenance` present and
  unapplied (established from preserved governance authority; no database queried).

## independent review authority and verdict

The independent adversarial review of the complete corrected 2-ii-c + c1 package returned
**WI0037_SLICE2IIC_C1_INDEPENDENT_REVIEW_PASSED_LOCAL_INTEGRATION_READY** with **zero findings**
(recorded in the external prompt ledger,
`.../dai/prompts/2026/07/2026-07-28-wi-0037-slice-2-ii-c-c1-independent-review.md`). That review
authorized a later local-only integration and governed closeout; it did not itself integrate or
close anything.

## reviewed source and vault chains

- Source: `baf5e90 -> 80e8d1a` (feat: operator/harness + status-contract parity) `-> 4c6cd98`
  (fix: powershell blank-state validation + comment correction). Exactly 11 changed paths
  (platform/dotnet + scripts/dev/sports), 329 insertions / 62 deletions, `git diff --check`
  clean; no dependency/lockfile/manifest/appsettings/schema/migration change; fixture corpus
  byte-identical `80e8d1a..4c6cd98` (blob `539f137afc5b6ef3665654bfa732ba34fc81edfd`).
- Vault records: `849eb9d -> d98d1a5 -> 462d7a4`. Exactly four changed paths (WI, current-slice,
  recheck-discipline doctrine, slice report), 440 insertions / 4 deletions, all `.md`,
  `git diff --check` clean.

## exact source fast-forward proof

From the clean canonical source worktree on `main` at `baf5e90`:
`git merge --ff-only wi/0037-game-state-correctness-slice-2-ii-c` -> `Updating baf5e90..4c6cd98`,
`Fast-forward`. After: source `main` = `4c6cd985662caebec686d4b96a12a4104440e857`; HEAD has a
single parent (`80e8d1a`), so no merge commit was created; `origin/main` unchanged at `aee1ade`;
the reviewed topic branch still exists at `4c6cd98`; the source worktree is clean.

## verification matrix (expected vs observed, on source main 4c6cd98)

| check | expected | observed |
|---|---|---|
| `test-check-game-status.ps1` | 195 / 0, exit 0 | 195 / 0, exit 0 |
| `test-check-settlement-finals.ps1` | 40 / 0, exit 0 | 40 / 0, exit 0 |
| `test-check-settlement-finals.ps1 -Probe` | tally reached, 1 pass + 1 deliberate fail, exit 1 | 1 / 1, tally reached, exit 1 by design |
| `GameStatusResolutionCorpusTests` | 63 | 63 |
| combined focused (corpus + adapter + starter + selected-event + selected-event-creation) | 189 / 0 | 189 / 0 |
| full `DevCore.Api.Tests` | 2059 / 0 skip / 0 fail | 2059 / 0 / 0 |
| build `DevCore.Api.slnx` | 0 errors | 0 errors |
| comment audit (`baf5e90..4c6cd98`, .cs/.ps1) | 69 added, 0 uppercase, 0 non-ascii | 69 / 0 / 0 |
| c# schedule-state alias tables | exactly one (`ScheduleStateNormalizer.cs`) | exactly one |
| production `GameStatusResolver.Resolve` typed | every call | 4/4 (2 NotRequired, 2 Required) |
| PowerShell live URI | `gamePks`, no `date=`, one GET, 30s, local bracket | confirmed |
| corpus shape | 25 fixtures / 31 vectors / 6 refusals / 8 normalized values | confirmed |
| network calls during verification | none | none |
| `git diff --check` + `main` clean | clean | clean |

## warning classification

Build succeeded with 0 errors. Warnings are pre-existing and NOT introduced by this package:
NU1903 advisories for `Microsoft.OpenApi` 2.0.0 and `System.Security.Cryptography.Xml` 10.0.7
(package vulnerability advisories), plus pre-existing compiler/nullability/xUnit analyzer
warnings (CS0108, CS0649, CS8604, CS8625, CS8631, xUnit2031). Every non-NU1903 warning is
located in a file OUTSIDE the 11-path changed set (TestAuthentication.cs,
MarketContrastSourceAdapterTests.cs, ProtocolFailureSeamTests.cs, BuyerArtifactProjectionTests.cs,
OddsScheduleClientTests.cs, SelectedEventCreationTests.cs); a warning in a file the range never
touched cannot have been introduced by the range. This is not a warning-free build; nothing was
remediated.

## no-network and cost evidence

No `git fetch/pull/push`, no PR, no deployment, no activation, no migration apply, no service
start/stop, no database query, no provider/model call, no capture/reconciliation/settlement, no
dependency install/restore. PowerShell suites ran offline via fixtures / `-ScheduleJsonPath`; C#
corpus tests use in-memory JSON; the single live `Invoke-RestMethod` branch was never triggered.
**DAI application/provider spend = $0.00.** Coding-agent host/session billing is outside
repository observability and is deliberately not represented as a value.

## glossary disposition table

Rules applied: promote only a durable concept with demonstrated meaning beyond WI-0037; merge
same-concept labels (name the survivor); otherwise retain WI-local; never erase from WI history;
do not add a glossary entry for a merged or retained-WI-local term.

| term | disposition | reason |
|---|---|---|
| provider-scoped candidate identity | MERGED into `provider-qualified duplicate identity` | Slice 2-iii-b2 F-B2-8 unified provider-scoped candidate discovery and provider-qualified duplicate identity as one WI-local concept over the governing `(SourceProvider, ExternalGameId)` pair; survivor = `provider-qualified duplicate identity`. |
| provider-qualified duplicate identity | RETAINED WI-local | load-bearing selected-event duplicate-detection evidence in WI-0037 but an implementation-level identity rule with no demonstrated use beyond WI-0037; promoting it would distort the shared glossary. |
| activation deployment evidence | RETAINED WI-local | structured `{artifact, reference}` freshness-bounded evidence gating default-off `SelectedEventActivation`; specific to the activation runbook; no demonstrated platform-wide meaning. |
| validated writer | RETAINED WI-local | the validated-writer contract (provenance builder proven readable via `TryRead` before commit) is a useful WI-0037 write-path pattern with no demonstrated reuse beyond WI-0037. |

`domain execution provenance` is already PROMOTED in `01 Operating System/glossary.md` (confirmed
present, not duplicated). Because no term was promoted, `glossary.md` is unchanged.

## eight-link traceability disposition

The full eight-link block lives in the WI-0037 spec's `## links` section. Summary: (1) work item
WI-0037, local-spine mode, no ADO item; (2) branches per slice, all retained; (3) no PR --
published slices merged direct by fast-forward + push (dai origin `aee1ade`, vault origin
`e217a3f`), local-only slices merged direct by fast-forward UNPUSHED including this closeout
(dai `baf5e90..4c6cd98`); (4) commit hashes per slice ending at dai main `4c6cd98` and the vault
commit containing this report; (5) test paths (corpus/adapter/starter/selected-event C# suites +
the two PowerShell harnesses); (6) verification notes = the independent PASS verdict + this
matrix; (7) docs updated = WI, MOC, current-slice, the 2-ii-c slice report, and this report;
(8) lessons = staged-resolution vectors catch cross-runtime divergence direct-only tests hide,
and a warning outside the changed-path set cannot be introduced by the change.

## documents updated

- `02 Platform/system-development/work-items/WI-0037-game-state-correctness-v1.md` (status
  complete; closeout block; glossary dispositions; eight-link block; acceptance/handoff).
- `02 Platform/system-development/MOC - DAI System Development.md` (WI-0037 entry -> complete
  local state).
- `06 Execution/reports/wi-0037-slice-2-ii-c-operator-harness-parity-2026-07-27-v1.md` (dated
  closeout reconciliation; original findings/c1 disclosure preserved).
- `06 Execution/handoffs/current-slice.md` (appended closeout entry + Slice Synopsis).
- `06 Execution/reports/wi-0037-local-integration-and-governance-closeout-2026-07-28-v1.md`
  (this report, new).
- `01 Operating System/glossary.md` unchanged (no term promoted).
- `06 Execution/backlog/sports-v1-backlog.md` unchanged (WI-0037 is not registered there and
  doctrine does not require adding completed items -- not required).

## vault closeout transaction

These vault edits are committed as exactly one documentation-only commit -- **the commit
containing this report** -- created directly on top of `462d7a4` on
`wi/0037-game-state-correctness-slice-2-ii-c-records`, carrying the trailer `WI: WI-0037`.
Neither reviewed vault commit (`d98d1a5`, `462d7a4`) is amended. The immediately following and
final transaction step advances local vault `main` from `849eb9d` to that closeout commit by an
atomic compare-and-swap `git update-ref refs/heads/main <closeout> 849eb9d`, performed from a
safe worktree without checking out or modifying the protected WI-0035 worktree. The resolved
closeout SHA and post-move evidence are recorded in the prompt-ledger outcome and the final
response, not self-referentially inside this committed file.

## before/after protected WI-0035 inventory and fingerprint

Before and after every vault ref mutation, the protected canonical vault worktree remains on
`wi/0035-provider-event-binding` at `de5791f` with exactly the same six dirty paths (modified
`.obsidian/graph.json`, modified `06 Execution/handoffs/wi-0025-okf-registry-handoff-2026-07-16-v1.md`,
modified `CLAUDE.md`, deleted `Welcome.md`, untracked
`06 Execution/reports/preflight-settlement-manifest-2026-07-06-v1.json`, untracked
`06 Execution/system-state-synopsis-v1.md`) and fingerprint
`86aa8b74a3ec5d0856d7e043cff24241c54b69f7`. It was never read as WI-0037 output, edited, staged,
restored, deleted, moved, stashed, committed, or normalized.

## final repository / ref / worktree state

- Source local `main` = `4c6cd985662caebec686d4b96a12a4104440e857`; `origin/main` = `aee1ade`
  (unchanged); reviewed topic branch retained at `4c6cd98`; source worktree clean.
- Vault local `main` = the closeout commit (SHA in the prompt ledger + final response);
  `origin/main` = `e217a3f` (unchanged); records topic branch retained at the closeout commit;
  topic worktree clean.
- Protected WI-0035 worktree byte-for-byte preserved (fingerprint above).

## release-state truth and non-actions

LOCAL-ONLY; UNPUBLISHED; UNDEPLOYED; selected-event activation DISABLED; migration
`20260727133845` UNAPPLIED. Not performed: fetch, push, PR, deploy, activation, migration apply,
service start/stop, database query, provider/model call, capture, reconciliation, settlement,
dependency download, history rewrite (no reset/restore/checkout-paths/clean/stash/force/amend/
rebase/squash/cherry-pick), branch/worktree deletion.

## residual-risk ledger

- SELECTED_EVENT_IDENTITY_PROPAGATION_REQUIRED_BEFORE_WI0037_CLOSE: RESOLVED (Slice 2-iii-c local
  integration). Not reopened.
- MULTI_INSTANCE_SELECTED_EVENT_ATOMICITY_REQUIRED_BEFORE_SCALE_OUT: blocking before scale-out;
  NOT a blocker to this local WI closeout.
- DURABLE_PREEXECUTION_SELECTION_DECISION_LEDGER_DEFERRED: deferred/nonblocking.

## rollback guidance (preserves reviewed history)

Source: `git -C <DAI_REPO_ROOT> update-ref refs/heads/main baf5e90 4c6cd98` returns local `main`
to the pre-integration base; the reviewed topic branch and both reviewed commits are untouched.
Vault: `git -C <safe vault worktree> update-ref refs/heads/main 849eb9d <closeout>` returns local
vault `main` to its pre-advance base; the records topic branch and the closeout commit remain.
Do not use force/amend/rebase/squash/cherry-pick; never touch the protected WI-0035 worktree.

## continuation guidance

WI-0037 is complete (engineering + governance). Next governed action = an operator-selected next
slice against the integrated local baseline (dai main `4c6cd98`, vault main = the closeout
commit): a candidate evidence/capture slice, or -- only if a release program is separately
authorized -- remote publication + deployment + activation review (none authorized here). The
scale-out atomicity residual must be honored before any scale-out.
