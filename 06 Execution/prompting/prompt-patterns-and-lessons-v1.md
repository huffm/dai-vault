# Prompt Patterns and Lessons v1

**date:** 2026-06-24
**status:** LIVING. Reusable prompting lessons for DAI slices. Appended by `dai-slice-prompt-architect` (MODE 3)
after prompt-effectiveness reviews. Read before drafting a next-slice prompt.
**classification:** agent-ops reference. Governed by Slice Prompt Architecture Doctrine v1.

## How to add a lesson

Each lesson is one entry: **lesson** (one falsifiable line); **evidence** (the slice/doc it came from);
**when to reuse**; **when NOT to reuse**; **affected skill/doctrine** (if any); **update now or later**. Add via
MODE 3; do not edit runtime code. A lesson that recurs across >=2 reviews without action becomes a MODE 4 skill
proposal (cite both instances; state overfitting risk). One slice is an anecdote, not a rule.

## Reusable patterns

- **State the non-goals explicitly.** A bounded "do not change X / Y / Z" list prevents scope drift more reliably
  than a tight goal alone.
- **Say what not to change.** Name the protected surfaces (confidence, lean, posture, buyer copy, Tool Gateway,
  reconciliation, billing, auth, tenant) so the executor cannot drift into them.
- **Compile from verified state, not memory.** Every recommendation cites a handoff/doc/repo fact; verify the
  claimed repo head against the live head before trusting it.
- **Carry a next-prompt package in the handoff** when the next slice is clear, so the loop continues cleanly.
- **Preserve commits; never rewrite history.** Prompts should instruct commit-on-scope, push only when explicitly
  told, and never amend/rebase shared history.
- **A next-slice prompt is a proposal until validated by repo state.** The runner checks live heads before acting.
- **Specialize the prompt by slice type.** Implementation, calibration-read, cohort-capture, settlement, docs, and
  skills slices need different required skills, verification styles, and non-goals (see the doctrine's type table).
- **Lead with a Strategic Snapshot + Opportunity Cost (MODE 1).** Compress WhyNow / Factory-vs-Product / RevenuePath /
  DecisionScarcity / Horizon / Invalidation / OpportunityCost into 5-7 lines + one comparison sentence (< 60 tokens).
  Prefer slices that resolve scarce information (DAI-vs-market disagreements, confidence outliers, high-confidence
  misses, sparse cohorts) over volume. Prioritization only -- no execution authority.

## Anti-patterns

- **Tuning before measurement.** Do not propose changing confidence/depth/lean before a settled, slate-balanced read
  shows a discrimination signal. Tuning on noise fits the slate, not the system.
- **Treating Graphify as authority.** It locates code; it never proves behavior. Verify every finding against
  source/tests.
- **Mixing unrelated domains in one slice.** Runtime + buyer UX + billing + calibration in one prompt produces a
  muddy handoff and unsafe scope.
- **Pooling regimes in a calibration read.** Different cohorts sit on different slates; pooling hides slate
  confounding and invents false signal.
- **Guessing outcomes.** Reconcile only against authoritative results; never fabricate a score to "complete" a pass.
- **Prompt bloat.** Redundant restatement lowers signal; keep prompts dense (pair with `dai-token-tight`).
- **Self-modifying tooling.** A skill that edits itself silently is a strategic-autonomy risk; propose, get
  approval, apply in a separate slice.

## Current known lessons (seeded from recent project history)

1. **Do not tune before measurement.** Evidence: Reconciled-Outcome Calibration Read v1 + v2 (confidence flat 0.75,
   non-discriminating). Reuse: any calibration slice. Not when: a settled balanced read already shows signal.
   Affected: Confidence Calibration Rules. Update: already doctrine; keep enforcing.
2. **Treat Graphify as navigation, not authority.** Evidence: Advertised-Strength slices + Tool Gateway doctrine.
   Reuse: any code-orientation slice. Not when: never an exception. Affected: Graphify Navigation Doctrine. Now.
3. **Separate server authority from frontend rendering.** Evidence: Advertised-Strength Server Authority v1 ->
   Frontend Adoption v1 (closed dual authority). Reuse: any contract/projection split. Not when: trivial one-side
   value. Affected: none new.
4. **Prefer read-only containment for buyer-trust contradictions before deeper rewrites.** Evidence: LeanSide
   Direction Consistency Fix v1 (contained the structured/prose mismatch at read time, rule-12-safe). Reuse:
   buyer-trust correctness bugs. Not when: the persist path is the actual defect and is in scope.
5. **Keep regimes separate in calibration reads.** Evidence: Read v1/v2 (190x 100% vs 200x 29%, same depth opposite
   slates). Reuse: every calibration read. Not when: never pool. Affected: the calibration reads.
6. **Exclude contaminated runs before reconciliation.** Evidence: Directional-Contrast Settlement Pass (excluded 7
   duplicate reruns + 722d433e before posting outcomes). Reuse: any settlement with dupes/known-bad runs. Not when:
   a clean cohort with unique identities.
7. **Capture a market baseline when evaluating sports signal.** Evidence: Cohort Capture v2 + Market Baseline v1
   (DAI matched the market favorite 12/13). Reuse: any sports calibration capture. Not when: non-market domains.
8. **Require explicit non-goals to prevent scope drift.** Evidence: every clean slice this session named protected
   surfaces. Reuse: all non-trivial slices. Not when: a one-line read.
9. **A prompt should say what NOT to change.** Evidence: same as 8. Reuse: all. Affected: this skill's rubric (items
   2, 8, 14).
10. **Handoffs should include the next-prompt package when appropriate.** Evidence: this slice (the architect reads
    the handoff to emit the next prompt). Reuse: when the next slice is clear. Not when: direction is genuinely open.
11. **Confidence may be non-discriminating even when artifacts look coherent.** Evidence: Read v2 (flat 0.75,
    60.0% ~ home base rate). Reuse: when tempted to trust the confidence number. Affected: Confidence Calibration.
12. **A good prompt preserves commits and avoids history rewrites.** Evidence: every commit this session was a new
    commit; pushes only on explicit instruction. Reuse: all. Affected: none.
13. **A next-slice prompt is a proposal until validated by repo state.** Evidence: this doctrine + MODE 5 semantics.
    Reuse: whenever `next-slice.md` is used. Affected: `dai-slice-runner` validation step.
