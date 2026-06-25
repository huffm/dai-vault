# evidence readiness gates v1

**status:** active doctrine -- canonical platform governance for when classes of engineering work become permissible
**date:** 2026-06-25
**scope:** platform-wide (all niches). decision-system engineering governance. not runtime; nothing executes this doc.

## purpose

A decision-intelligence system learns from outcomes. If work is done out of order -- tuning a model before its
verdicts are trustworthy, claiming performance before a clean baseline exists -- the learning signal is corrupted
and every later decision inherits the corruption. This doc makes explicit the gate sequence that recent slices
followed implicitly, so any contributor can answer one question from one document: **"are we allowed to do this work
yet?"**

It distinguishes three readiness kinds that are easy to conflate:

- **implementation readiness** -- the code exists, compiles, and is tested. (Can we build it?)
- **operational readiness** -- the capability runs reliably in the live environment: services up, idempotent,
  duplicate-safe, authoritative inputs. (Can we run it?)
- **evidence readiness** -- the data the capability produces is trustworthy enough to justify the NEXT class of work
  (calibration, then tuning, then performance claims). (Can we *learn* from it, and act on what we learned?)

A capability can be implementation-ready and operationally-ready while being evidence-unready. Most governance
failures in a learning system are evidence-readiness failures wearing an implementation-readiness disguise.

## problem it solves

Across Decision-Encoding Integrity v1, the Settlement Integrity Guard, Mismatch Remediation v1, and Calibration
Assessment v3, the same rule kept being re-derived per slice: do not let later work consume earlier work that is
not yet trustworthy. That rule lived in handoffs and individual calibration docs, not in one canonical place. This
doc freezes it so future agents do not relitigate sequencing and do not skip a gate under time pressure.

## strategic fit

Platform = factory; a decision product is only as good as the evidence loop behind it. These gates are the factory's
quality-control stations for the *learning line*: each gate is a checkpoint that protects the integrity of the
feedback signal the whole platform optimizes against. They add no product feature; they make optimization safe.

## core principle

> **Integrity before settlement. Settlement before calibration. Calibration before tuning. Evidence before
> optimization.**

Why violating the sequence corrupts future learning:

- **Settling before integrity** records verdicts against decisions that contradict themselves -- a contradictory
  run graded "correct" is a false positive injected into the corpus (this is exactly what happened to 01AA433E and
  BDDE423E before the guard existed; see Mismatch Remediation v1).
- **Calibrating before settlement** reads outcomes that are unconfirmed, duplicated, or guessed -- the accuracy
  number is then measuring noise, not the system.
- **Tuning before calibration** optimizes against an unmeasured or contaminated baseline -- improvements are
  indistinguishable from slate noise, and the change is unfalsifiable.
- **Claiming performance before evidence** makes a buyer-facing assertion the corpus cannot support.

Each step consumes the previous step's output as ground truth. If the previous step's output is untrustworthy, the
error does not stay local -- it becomes the reference frame for everything after it.

## readiness gates

Each gate states requirements (what must be true), what it unlocks, and the verified anchor (where this is enforced
in source/tests/docs). A gate is passed for a record/capability only when every requirement holds.

### Gate 0 -- Decision generated

- **requirements:** successful pipeline execution; a canonical artifact produced (`sports_decision_artifact_v3`)
  with a persisted identity where applicable.
- **unlocks:** integrity evaluation.
- **anchor:** `SportsComposer`, artifact contract (`current-agent-run-contract.md`).

### Gate 1 -- Decision Integrity

- **requirements:** persisted representations agree (denormalized `AgentRun.LeanSide` matches the artifact's
  structured `LeanSide`); the artifact is internally consistent (structured lean does not contradict its own prose
  direction); integrity verification passes (not `PotentialMismatch`).
- **unlocks:** settlement (a run may be graded).
- **anchor:** `SettlementDirectionIntegrity` + `ArtifactDirectionConsistencyEvaluator` (the settlement write paths
  return 422 and write nothing on a clear contradiction); `decision-encoding-integrity-077a-root-cause-v1.md`.

### Gate 2 -- Settlement

- **requirements:** an authoritative outcome (e.g. MLB StatsAPI final, never guessed); idempotent reconciliation
  keyed by `agentRunId` (a retry is a 409, never a duplicate); duplicate-risk protection (same `gamePk`, distinct
  runs settle per run; `/reconcile` returns `MultipleMatches` and writes nothing rather than collapsing).
- **unlocks:** calibration inclusion (the verdict may enter the corpus).
- **anchor:** `AgentRunsController` (`RecordOutcome`/`Reconcile`), `OutcomeReconciliationMatcher`/`Service`;
  `outcome-reconciliation-matcher-v1.md`, `pre-settlement-idempotency-duplicate-risk-manifest-v1.md`.

### Gate 3 -- Calibration Eligibility

- **requirements:** settled (an `AgentRunOutcome` + `AgentRunEvaluation` exist); `ExclusionReason IS NULL`;
  integrity-clean (contradictory/invalid runs already excluded, not counted).
- **unlocks:** calibration assessment (the run counts in the denominator).
- **anchor:** the matcher selects only `ExclusionReason == null` (`OutcomeReconciliationService.cs`); the binding
  denominator doctrine `valid = settled AND ExclusionReason IS NULL` (`mismatch-remediation-v1.md`). **Current
  verified denominator: 59.**

### Gate 4 -- Calibration Sufficiency

- **requirements:** adequate sample size for the question asked; balanced cohorts (not slate-confounded); market
  comparison available for the runs being compared; confidence behavior understood (is it predictive?). A clean
  denominator (Gate 3) is necessary but NOT sufficient -- this gate is about statistical power and discrimination,
  not hygiene.
- **unlocks:** tuning proposals.
- **status: NOT YET ACHIEVED.** Calibration Assessment v3 found: market baselines cover only 13 of 59 runs (22%);
  the decisive DAI-vs-market disagreement sample is n=1; confidence is non-predictive (clustered at 0.75, the 0.80+
  band inverted); no regime shows a slate-independent edge. Readiness verdict: **No**.
- **anchor:** `calibration-assessment-v3.md`; `confidence-calibration-rules-v1.md` (no-tuning-until-discrimination).

### Gate 5 -- Evidence-backed Optimization

- **requirements:** a measurable, integrity-qualified baseline (Calibration Assessment v3 is the current baseline);
  a before/after comparison on the same denominator and regimes; statistical justification (the change beats noise,
  not a single slate); reproducible improvement across independent slates.
- **unlocks:** production tuning; buyer performance claims; model replacement.
- **status:** locked behind Gate 4.
- **anchor:** baseline `calibration-assessment-v3.md`.

## decision matrix

| activity | required gate |
|---|---|
| Settlement (grade a run) | Gate 1 |
| Calibration inclusion (count a verdict) | Gate 2 |
| Calibration assessment / read | Gate 3 |
| Tuning proposals | Gate 4 |
| Production tuning | Gate 5 |
| Buyer performance claims | Gate 5 |
| Model replacement | Gate 5 |

(Settlement requires the run to have passed Gate 1; calibration assessment requires Gate 3; tuning requires Gate 4;
optimization/claims require Gate 5. Lower gates are preconditions of higher ones -- a record at Gate 3 has by
definition cleared Gates 0-2.)

## current platform status (2026-06-25)

- **Gate 1 Integrity -- COMPLETE.** Settlement integrity guard implemented, tested, and verified live; the four
  known contradictory runs are excluded.
- **Gate 2 Settlement -- MATURE.** Idempotent-by-`agentRunId`, duplicate-risk-safe, graded only on authoritative
  finals.
- **Gate 3 Calibration eligibility -- OPERATIONAL.** Integrity-qualified denominator established and verified (59
  valid = settled AND `ExclusionReason IS NULL`).
- **Gate 4 Calibration sufficiency -- NOT ACHIEVED.** Evidence accumulation in progress; balanced market-backed
  slates and a real disagreement sample are still needed.
- **Gate 5 Optimization -- LOCKED.** Tuning intentionally deferred. No production tuning, buyer performance claim,
  or model replacement is permitted until Gate 4 is cleared.

## future guidance

Every future slice should, in its boundary statement, declare its relationship to these gates:

- **which readiness gate it advances** (e.g. "advances Gate 4 by adding a balanced market-backed slate"),
- **whether it crosses a gate** (changes the permissible-work frontier -- this is rare and must be explicit and
  evidence-backed),
- or **whether it merely strengthens an existing gate** (e.g. hardens Gate 1, grows the Gate 3 denominator) without
  crossing into new permissions.

This becomes part of normal slice planning alongside the Skills Gate. A slice that would cross a gate without
meeting that gate's requirements is out of bounds, regardless of how implementation-ready it is.

## truth hierarchy

These gates sit ON TOP of, and do not replace, the per-slice truth hierarchy (observed runtime/tests > source >
contracts/vault docs > handoffs > generated graphs; see `agent-slice-workflow-doctrine-v1.md`). The slice-workflow
gates govern HOW one slice runs; these evidence-readiness gates govern WHEN a class of work across slices is
permitted. A slice obeys both: it runs through its lifecycle gates and it respects the evidence gate its activity
requires.

## what it is / what it is not

- **is:** a program-level governance sequence; a permission model for engineering activity; a single place to check
  whether a class of work is allowed yet.
- **is not:** a runtime component; a per-slice lifecycle (that is the slice-workflow doctrine); a calibration method
  (that is the calibration reads); a buyer-facing artifact. It introduces no tuning and changes no behavior.

## risks and failure modes

- **Gate laundering:** dressing a Gate-4/5 activity (tuning, a performance claim) as a Gate-1/2 fix to skip the
  sequence. Mitigation: every slice declares the gate it touches (future guidance).
- **False sufficiency:** treating a clean denominator (Gate 3) as license to tune. Gate 3 is hygiene; Gate 4 is
  power. They are distinct on purpose.
- **Silent contamination:** a calibration read that forgets the `ExclusionReason IS NULL` filter re-imports excluded
  runs. Mitigation: the denominator doctrine is binding and the matcher enforces it at selection.

## deferred decisions

- Whether to encode these gates as a machine-checkable precondition (e.g. a calibration-read query template or a CI
  assertion) rather than doctrine. Deferred -- doctrine first; automation only if the gates are skipped in practice.
- Per-niche gate thresholds (what "adequate sample" / "balanced cohort" means numerically per sport). Deferred to
  the calibration-sufficiency work itself.

## related docs

- `06 Execution/agent-slice-workflow-doctrine-v1.md` -- per-slice lifecycle + truth hierarchy (these gates layer on top).
- `02 Platform/architecture/decision-intelligence-model.md` -- the evidence-backed learning doctrine these gates protect.
- `04 Products/sports-v1/outcome-reconciliation-matcher-v1.md` + `calibration/pre-settlement-idempotency-duplicate-risk-manifest-v1.md` -- Gate 2.
- `04 Products/sports-v1/calibration/decision-encoding-integrity-077a-root-cause-v1.md` -- Gate 1.
- `04 Products/sports-v1/calibration/mismatch-remediation-v1.md` -- Gate 3 denominator doctrine.
- `04 Products/sports-v1/calibration/calibration-assessment-v3.md` -- Gate 4 status + Gate 5 baseline.
- `04 Products/sports-v1/confidence-calibration-rules-v1.md` -- the no-tuning-until-discrimination rule (Gate 4/5).

## acceptance criteria

- A contributor can determine, from this doc alone, the gate required for any engineering activity and whether it is
  currently open.
- Every gate names a verified anchor (source/test/doc), not an assertion.
- Current status reflects the verified state: Gates 1-3 cleared, Gate 4 not achieved, Gate 5 locked.

## recommended next slice

No follow-up doc is required. The next *evidence* slices (Cohort Capture v3 + Market Baseline v2, then Calibration
Assessment v4) advance Gate 4; each should cite this doctrine in its boundary statement. If the gates are ever
skipped in practice, write the machine-checkable enforcement deferred above.
