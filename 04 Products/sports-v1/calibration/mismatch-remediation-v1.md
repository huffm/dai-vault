# Mismatch-Remediation v1 -- already-settled PotentialMismatch runs

**date:** 2026-06-25
**status:** REMEDIATED. The two already-settled lean-vs-prose contradictions found by Decision Encoding Integrity
v1 are excluded as `invalid` historical integrity failures; their settled rows are preserved for audit. No tuning;
no code change; no verdict overwritten or deleted. Corpus settled unchanged (61); valid-calibration now 59.
**type:** data + doctrine remediation in the DB; this doc + handoff in `dai-vault`. No `dai` change.
**skills:** dai-slice-runner + dai-skill-router gate + dai-grill-with-vault + superpowers:systematic-debugging +
superpowers:verification-before-completion + dai-test-discipline + dai-agent-handoff. dai-code-reviewer not needed
(no code change).
**input:** `decision-encoding-integrity-077a-root-cause-v1.md` (sec 2 blast radius).

**Anchor:** A run whose persisted lean contradicts its own artifact prose has no trustworthy direction (the 077A
precedent). The two such runs that were graded BEFORE the guard existed must not count as calibration evidence in
either direction. Exclude them (preserving audit rows); do not re-grade (re-grading would assert a side a
contradictory artifact cannot prove); do not delete.

## 1. The two contaminated runs (DB evidence)

| run | key | gamePk | home / away | denorm + artifact lean | prose leans | settled verdict | true (prose) read | regime |
|---|---|---|---|---|---|---|---|---|
| BDDE423E | 180017 | 822889 | texas-rangers / minnesota-twins | home | **Twins (away)** | `incorrect` (away_win, lean home) | should be **correct** (away won) | 180x |
| 01AA433E | 190017 | 824910 | atlanta-braves / milwaukee-brewers | home | **Brewers (away)** | **`correct`** (home_win, lean home) | should be **incorrect** (home won) | **190x** |

Both were `PotentialMismatch` (structured `lean_side=home`, prose names the away team) and were settled before the
settlement guard existed (ResolvedUtc 2026-06-19 and 2026-06-22). Their recorded verdicts are each wrong relative to
the artifact's actual prose read -- one false-negative (BDDE: recorded incorrect, true correct), one false-positive
(01AA: recorded **correct**, true incorrect).

**Note on 190x:** 01AA433E sits in the "enriched starter-depth 190x" cohort that Calibration Read v2 reported at
6/6 = 100%. That headline rests partly on this false-correct; with 01AA excluded the 190x cohort drops to 5
valid decided runs. (Read v2's conclusion already discounted 190x as slate-confounded; this further weakens it.
This doc does not re-run the read -- it removes the contaminated input.)

## 2. Remediation policy (chosen + justification)

**Exclude both as `invalid` historical integrity failures; preserve the settled outcome/evaluation rows for
auditability; calibration reads must filter `ExclusionReason IS NULL`.**

Rejected alternatives and why:

- **Re-grade to the prose direction** -- rejected. A contradictory artifact has no provable decision direction;
  asserting "the prose is authoritative" picks a side the run itself contradicts. This is inconsistent with the
  077A precedent (077A was excluded, not re-graded). The model emitted the contradiction; we do not launder it into
  a verdict.
- **Delete the settled outcome/evaluation rows** -- rejected. The boundary forbids silent deletion, and deleting
  destroys the audit trail of what was wrongly recorded.
- **Preserve verdict but mark calibration-excluded** -- this is effectively what `ExclusionReason=invalid` achieves:
  the verdict row stays (audit), and selection/reads skip it. `invalid` is the contract reason "malformed/erroneous
  run that must not be evaluated," which fits an internally-contradictory decision.

This matches "prefer preserving auditability over making the corpus look cleaner."

## 3. Applied (before/after DB evidence)

| run | ExclusionReason before | ExclusionReason after | outcome row after | verdict row after |
|---|---|---|---|---|
| BDDE423E | null | **invalid** | preserved (1) | preserved (`incorrect`) |
| 01AA433E | null | **invalid** | preserved (1) | preserved (`correct`) |

Applied via `POST /api/agent-runs/{id}/exclude {"reason":"invalid"}` (200 each) -- the audited eligibility
mechanism, same as 077A. No row deleted; no verdict overwritten; no LeanSide/artifact/confidence/posture change.

Counts:

| metric | before | after |
|---|---|---|
| corpus settled (AgentRunOutcomes) | 61 | **61** (unchanged -- rows preserved) |
| valid-calibration (settled AND `ExclusionReason IS NULL`) | 61 | **59** |
| runs with `ExclusionReason='invalid'` total | 2 (077A, Athletics@Giants) | **4** (+ BDDE, 01AA) |

## 4. Why no code change was required

- **Selection already excludes them.** `OutcomeReconciliationService.MatchAsync` selects only
  `r.ExclusionReason == null` (`OutcomeReconciliationService.cs:44`), so the matcher can never re-select an excluded
  run. This is covered by existing tests `reconcile_excluded_only_run_returns_no_match`,
  `reconcile_ignores_legacy_runs_with_null_identity`, `legacy_runs_with_null_identity_are_never_matched` (3/3 green
  this slice). These runs are also already settled (a re-settle attempt would 409), and the settlement guard
  (Decision Encoding Integrity v1) would 422 any new PotentialMismatch.
- **No code-level calibration read exists.** Calibration reads (Read v1/v2) are ad-hoc `sqlcmd` over
  `AgentRunEvaluations`. The non-test reads of `AgentRunEvaluations` in code are single-run existence/lookup checks
  (`AgentRunsController` idempotency + GET evaluation, `CloseMarketConfirmationService`), not calibration
  aggregation. There is nothing to modify, so nothing to add a test to.

Therefore the remediation is data + doctrine, not code.

## 5. Calibration read doctrine (binding for the next read)

Any calibration read/aggregation MUST filter `AgentRuns.ExclusionReason IS NULL`. The canonical denominator is the
**valid-calibration set = settled runs (AgentRunOutcomes present) with `ExclusionReason IS NULL`** -- currently
**59**. The matcher already enforces this at selection; reads must mirror it. A read that joins `AgentRunEvaluations`
without this filter will silently re-include the four excluded runs (Athletics@Giants invalid capture, 077A, BDDE,
01AA) and is non-conformant.

## 6. Disposition summary

- **BDDE423E:** excluded `invalid` (ungradable historical integrity failure); settled rows preserved for audit.
- **01AA433E:** excluded `invalid` (ungradable historical integrity failure); settled rows preserved for audit.
- **722D433E** (the remaining PotentialMismatch run): UNSETTLED and not part of this remediation -- it never
  produced a verdict, and the settlement guard blocks it from ever being graded while the contradiction stands. Left
  as-is; flagged for the analyzer-hardening / next-capture slice.

## 7. Effect on sequencing

- **Calibration Read is now safe** to run, provided it filters `ExclusionReason IS NULL` (sec 5). The valid set is
  59 settled runs. (Still subject to the standing conclusion: insufficient balanced data; do not tune.)
- **Cohort v3 capture remains safe** -- the settlement guard prevents new contradictory runs from being graded.

## 8. Verification

- DB before/after captured (sec 3): both `ExclusionReason` null -> invalid; outcome+evaluation rows preserved;
  corpus 61 unchanged; valid-calibration 61 -> 59; invalid-total 2 -> 4.
- Tests: no code change, so no new tests. Existing selection-exclusion tests rerun green (3/3) as evidence that an
  excluded run is never selected.
- `dai` unchanged (no code); `dai-vault` adds this doc only. No tuning; no prompt/model/confidence/posture/lean/
  buyer/frontend change; no verdict overwritten or deleted.

## 9. Recommended next slices

1. **Analyzer Lean-Agreement Hardening v1** (generation side): make `_parse_response` degrade `lean_side` to null
   when it disagrees with the prose, so contradictions stop being produced at the source. Separate lane.
2. **Reconciled-Outcome Calibration Read v3:** now unblocked; must use the `ExclusionReason IS NULL` denominator
   (59) and keep regimes separate. Pool only after >=3 settled market-baseline slates.
3. **Directional-Contrast Cohort Capture v3 + Market Baseline v2:** proceed; guard protects new settlements.
