---
title: "Agent Slice Workflow Doctrine v1"
type: "execution-pattern"
date: "2026-06-23"
status: "complete"
project: "DAI"
slice: "Agent Slice Workflow Doctrine v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - workflow
  - okf
related:
  - "06 Execution/patterns/okf-yaml-front-matter-pattern-v1.md"
---

# agent slice workflow doctrine v1

**status:** active doctrine -- canonical lifecycle for a DAI slice
**date:** 2026-06-23

## purpose

Define, in one place, how a DAI slice is run from start to finish: its boundary, the gates it passes through, the order of authority when sources disagree, and the handoff it ends with. This is the canonical workflow doctrine that a future `dai-slice-runner` skill must encode rather than reinvent.

## problem it solves

Slice workflow behavior is currently spread across `dai-skill-router`, `dai-agent-handoff`, `dai-docs-architect`, `dai-code-reviewer`, `dai-test-discipline`, `<DAI_REPO_ROOT>/CLAUDE.md`, and the running handoff log. No single doc states the lifecycle, so each session re-derives it and the discipline drifts. This doc makes the lifecycle explicit and verifiable so agents stay aligned without re-discovery.

## strategic fit

Platform = factory; agents = workers; a slice is one controlled change on the line. The factory produces repeatable decision products only if each change is bounded, verified, and recorded the same way every time. This doctrine is the standard work instruction for a worker on the line: it does not add product features, it makes change throughput safe and repeatable.

## mental model

A slice is the unit of controlled platform change. It has five parts: boundary (what is in and out), purpose (the one outcome), constraints (hard rules), verification path (how "done" is proven), and handoff (what the next agent inherits). A slice moves through three phases -- orient, execute, verify-and-hand-off -- and passes named gates on the way. If any of the five parts is missing, the slice is underspecified; stop and pin it down before changing anything.

## what it is

- The canonical end-to-end lifecycle of a DAI slice.
- A statement of the required gates and the order of authority (truth hierarchy).
- The contract every slice handoff is expected to satisfy.
- The source of truth a future `dai-slice-runner` skill will encode.

## what it is not

- Not a runtime component; nothing in the platform executes this doc.
- Not the `dai-slice-runner` skill (explicitly deferred -- see deferred decisions).
- Not a replacement for the individual skills; it sequences them, it does not restate their internals.
- Not buyer-facing and not niche-specific.

## approved uses

- Orienting at the start of any slice: read the boundary, run the Skills Gate, decide which gates apply.
- Deciding which skills a slice must invoke (review, tests, grill, docs, handoff).
- Resolving disagreements between a generated graph, an assumption, and the source by applying the truth hierarchy.
- Holding a slice to honest completion: no inflated "done" without a verification path.
- Seeding the future `dai-slice-runner` skill.

## disallowed uses

- Using it to justify skipping verification because "the workflow was followed".
- Treating it as authority over actual source, tests, or contracts.
- Mixing unrelated domains into one slice because the doctrine "allows a slice".
- Expanding agent permissions or scope by citing the lifecycle.

## workflow impact

The slice lifecycle, in order:

1. Orient
   - Run the Skills Gate (`dai-skill-router`): list available skills, select deliberately, name what is missing, state fallbacks. Never claim a skill was used unless it was.
   - Establish the slice boundary, purpose, constraints, and intended verification path.
   - Read vault docs when strategic or architectural context matters; use `dai-grill-with-vault` for architecture/strategy-heavy slices.
   - Use Graphify for orientation or blast-radius when useful (`query`/`explain`/`path`/`affected`). Navigation evidence only, never authority; verify findings against source. Never run it on `<DAI_VAULT_ROOT>`; never enable semantic/cloud extraction.
2. Execute
   - Make the smallest change that satisfies the purpose. Keep diffs minimal and on-scope.
   - Do not mix unrelated domains (runtime, buyer UX, billing, agent doctrine) unless the slice explicitly scopes them.
   - Docs/doctrine work uses `dai-docs-architect`; TS/Angular work uses `dai-typescript-angular-quality`.
3. Verify and hand off
   - Test-relevant slices follow `dai-test-discipline` (narrowest meaningful command first; full breadth only at a declared final verification).
   - Code-changing slices run `dai-code-reviewer` before the final handoff; resolve blocking findings first.
   - Confirm diffs are on-scope; do not commit generated artifacts, local settings, graph output, or machine-specific files unless explicitly intended. Commit only when appropriate; push only when explicitly instructed.
   - Return a structured handoff via `dai-agent-handoff` (every field filled; `none` is valid).

## truth hierarchy

Authority runs top to bottom; lower tiers never override higher ones.

1. Observed runtime behavior and tests.
2. Source code.
3. Explicit contracts and vault docs.
4. Slice handoffs.
5. Generated graphs (Graphify) and prior assumptions -- navigation evidence only.

## source or vault references to verify

- `<DAI_REPO_ROOT>/CLAUDE.md` -- operating directives (skills gate, graphify orientation, verify against source/tests/vault, code review before completion, structured handoffs).
- `<DAI_REPO_ROOT>/.claude/skills/dai-skill-router/SKILL.md` -- the Skills Gate and routing rules.
- `<DAI_REPO_ROOT>/.claude/skills/dai-code-reviewer/SKILL.md` -- pre-completion review gate.
- `<DAI_REPO_ROOT>/.claude/skills/dai-test-discipline/SKILL.md` -- verification ladder.
- `<DAI_REPO_ROOT>/.claude/skills/dai-grill-with-vault/SKILL.md` -- architecture/strategy alignment.
- `<DAI_REPO_ROOT>/.claude/skills/dai-docs-architect/SKILL.md` -- doctrine-doc discipline (this doc follows its template).
- `<DAI_REPO_ROOT>/.claude/skills/dai-agent-handoff/SKILL.md` -- mandatory handoff shape.
- `<DAI_VAULT_ROOT>/06 Execution/skills/dai-skills-inventory-v1.md` -- the skill layer this doctrine sequences.
- `02 Platform/architecture/devtooling/graphify-development-navigation-doctrine-v1.md` -- navigation-layer positioning and truth hierarchy.

## example usage

A code-changing sports slice: run the Skills Gate and select `dai-test-discipline` + `dai-code-reviewer`; read the relevant vault contract; use `graphify affected "<symbol>"` to scope blast radius, then verify each edge against the real source; implement the minimal change; run the narrowest test, widening to a declared final verification; run `dai-code-reviewer` and fix blocking findings; confirm the diff is on-scope and excludes `graphify-out/` and `.claude/settings.json`; return a `dai-agent-handoff` handoff. A docs slice (like this one): Skills Gate -> `dai-docs-architect` template -> link from the backlog/inventory -> update `current-slice.md` -> commit docs only -> handoff.

## agent prompt guidance

- Start every slice prompt with the Skills Gate; name expected-but-missing skills explicitly and the fallback in use.
- State the slice boundary, purpose, constraints, and verification path up front.
- Require the truth hierarchy when graph output or assumptions are in play.
- Require `dai-code-reviewer` before completion on any code-changing slice, and a `dai-agent-handoff` handoff at the end.
- Prefer honest partial completion over an inflated completion claim; say what was not verified.

## acceptance criteria

- The slice has a stated boundary, purpose, constraints, and verification path.
- The Skills Gate ran before the first change; selections and missing skills are recorded.
- Required gates for the slice type ran (review for code, tests for test-relevant work).
- Claims are verified against the highest available tier of the truth hierarchy.
- Diffs are on-scope; no unintended generated/local/machine-specific files committed; not pushed unless instructed.
- A structured handoff was returned with every field filled.

## risks and failure modes

- Phantom skills: claiming a skill was used when it was not loadable -> mitigated by the Skills Gate's expected-but-missing check.
- Inflated completion: declaring done without a verification path -> blocked by `dai-code-reviewer` and the acceptance criteria.
- Graph-as-truth: trusting a Graphify edge over source -> blocked by the truth hierarchy.
- Scope creep: mixing unrelated domains -> blocked by the boundary and the no-mixing rule.
- Doctrine drift: this doc going stale as skills evolve -> mitigated by linking it from the inventory and backlog and revising on skill-layer changes.

## deferred decisions

- `dai-slice-runner` skill -- a skill that enforces this lifecycle. Deferred until this doctrine is established (now). Build it as an encoding of this doc, not a new invention.
- Automated gate enforcement (hooks/CI that block completion without a review or handoff) -- deferred.
- A machine-checkable slice manifest (boundary/purpose/verification as structured fields) -- deferred.

## related docs

- `06 Execution/skills/dai-skills-inventory-v1.md`
- `06 Execution/backlog/documentation-doctrine-backlog-v1.md`
- `02 Platform/architecture/devtooling/graphify-development-navigation-doctrine-v1.md`
- `02 Platform/architecture/cognitive-worker-doctrine.md`
- `02 Platform/architecture/governance/evidence-readiness-gates-v1.md` -- program-level evidence gates; a slice should declare which gate it advances/crosses/strengthens as part of planning, alongside the Skills Gate. The per-slice lifecycle gates here govern HOW a slice runs; the evidence gates govern WHEN a class of work is permitted.
- `<DAI_REPO_ROOT>/CLAUDE.md`

## recommended next slice

Source Depth and Evidence Sufficiency Doctrine v1 (next P1 doctrine in the backlog). Defer `DAI Slice Runner Skill v1` until after at least one more doctrine slice has exercised this lifecycle in practice, so the runner encodes a workflow proven across more than one slice type.
