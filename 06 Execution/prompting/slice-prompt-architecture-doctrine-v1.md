# Slice Prompt Architecture Doctrine v1

**date:** 2026-06-24
**status:** DOCTRINE. Canonical definition of how DAI compiles next-slice prompts. Encoded by the
`dai-slice-prompt-architect` skill (`dai/.claude/skills/dai-slice-prompt-architect/SKILL.md`). Skills + docs only;
no runtime behavior.
**classification:** agent-ops doctrine. Pairs with Agent Slice Workflow Doctrine v1 (the runner side).

## Why this exists

DAI slices are driven by Claude prompts. Until now those prompts were authored ad hoc from memory of the last
handoff. As the project compounds (calibration reads, cohort captures, settlement passes, frontend adoption), the
quality of the next slice depends on how faithfully the next prompt is compiled from verified state. This doctrine
makes prompt creation a first-class, reviewable step of the operating system -- not an afterthought.

## The prompt architect role

`dai-slice-prompt-architect` is a doctrine-backed prompt compiler. It converts verified project state -- handoffs,
current-slice, vault doctrine, skills inventory, calibration/backlog state, repo state, Graphify orientation -- into
a reviewable, executable next-slice prompt. It **proposes**; it does not execute, decide strategy, invent direction,
or silently modify skills.

## Relationship to the runner

The runner executes; the architect proposes. `dai-slice-runner` runs the 8-step lifecycle and is the only executor.
`dai-slice-prompt-architect` produces the prompt the runner will run. A produced prompt -- including a saved
`next-slice.md` -- is a **proposal until the runner validates live repo state at execution time**. If the file's
expected repo heads differ from the live heads, the runner re-derives rather than trusting the stale proposal.

## Relationship to the handoff

The prior `dai-agent-handoff` output is the architect's primary input. A handoff records what happened; the
architect reads it (plus current-slice and doctrine) and emits the next prompt. The produced prompt's required
handoff format aligns with `dai-agent-handoff` so the loop stays consistent.

## Relationship to Graphify

Graphify is navigation evidence only. The architect may use it to locate a symbol or estimate blast radius when a
code-area question needs orienting, then verifies against source/tests. It never runs Graphify on `<DAI_VAULT_ROOT>`,
never enables semantic/cloud extraction, never commits `graphify-out/`, and never asserts behavior from the graph.

## Relationship to vault doctrine

The architect compiles prompts that **align with** existing doctrine (Source Depth & Evidence Sufficiency, Tool
Gateway & Agent Permissions, the calibration reads, Buyer Copy Safety, etc.). It cites the doctrine a recommendation
rests on. It never freezes speculation as doctrine -- that is `dai-docs-architect`'s job, and only after verification.

## Relationship to user/architect approval

The human/architect remains the strategic approver. The skill proposes the next slice and (in MODE 4) proposes
skill/doctrine improvements; it applies neither. Strategy, prioritization, and any skill edit are the user's call.
This is the guardrail against strategic autonomy and self-modification.

## Anti-goals

- Not a strategist: it does not decide what DAI should do next as a business or product.
- Not an executor: it never runs the slice it proposes.
- Not self-modifying: it never edits itself or another skill; MODE 4 proposes, a separate user-approved skill slice
  (under `dai-write-skill`) applies.
- Not an automation layer: no shell scripts, Git hooks, CI, or background jobs.
- Not a source of truth above the repo: a proposal is provisional until validated against live state.

## Learning loop

1. Draft prompt (MODE 1). 2. Execute (`dai-slice-runner`). 3. Receive handoff. 4. Review prompt effectiveness
(MODE 2). 5. Record a prompt-learning note (MODE 3) in `prompt-patterns-and-lessons-v1.md`. 6. Propose a
skill/doctrine update only if warranted (MODE 4: >=2 instances or 1 severe failure, with overfitting risk stated).
7. User/architect approves. 8. A separate skill-update slice applies the change.

The hard boundary: steps 6-8 are proposal -> human approval -> separate slice. The skill never closes that loop on
its own.

## next-slice file semantics

- Canonical home: `06 Execution/prompting/next-slice.md` (this doctrine consolidates the proposed `prompts/` and
  `prompting/` locations into one to prevent drift). Archive: `06 Execution/prompting/archive/`.
- The file is a **proposal**, not standing authority. Before overwriting, the prior `next-slice.md` is archived.
- It carries: generated date, generated-by, source handoff/doc, expected repo heads, supersession/expiration, the
  full prompt, and a runner validation checklist.
- `dai-slice-runner` validates live repo state against the expected heads before executing; on mismatch it
  re-derives.

## Truth hierarchy

Authority top to bottom; lower never overrides higher:

1. Observed runtime behavior and tests.
2. Source code.
3. Explicit contracts and vault doctrine.
4. Slice handoffs.
5. Generated graphs (Graphify) and prior assumptions / a saved `next-slice.md` -- navigation/proposal only.

## What did not change

Skills + docs only. No runtime code, no product/model prompts, no confidence/source-depth/advertised-strength/Tool
Gateway/reconciliation/billing/auth/tenant/buyer-copy behavior. No automation, hooks, or CI.
