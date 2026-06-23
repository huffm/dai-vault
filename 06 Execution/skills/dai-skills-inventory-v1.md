# DAI Skills Inventory v1

**date:** 2026-06-11
**status:** governance / source of truth for the DAI skill layer. updated whenever a skill is added, adapted, or retired (enforced by `dai-write-skill`'s closing checklist).
**skills root:** `dai/.claude/skills/` (project skills, loaded dynamically by Claude Code from the filesystem). there is no workspace-level or user-level skills folder in use, and no `.claude-plugin/plugin.json` -- that shape belongs to the Jera pack, not to this project-skills layout.

## Why this doc exists

Before DAI Skills Normalization and Expansion v1 (2026-06-11), only `dai-signal-follow-up-diagnostics` was tracked in the `dai` repo; six other skill folders had been copied from the Jera workspace but sat untracked, and earlier sessions could not see them at all -- so prompts referenced skills that were not actually loadable. This inventory plus `dai-skill-router`'s Skills Gate prevent that class of silent failure.

## Current DAI skills

### dai-signal-follow-up-diagnostics
- **Purpose:** diagnose signal coverage gaps (missing/weak/stale/conflicting/proxy-only) on a completed run; recommend next slice and owning cognitive phase.
- **Use when:** reviewing a specific run, artifact JSON, or calibration report; before proposing fixes to signal retrieval/quality/posture/confidence when the complaint is "the read is missing context."
- **Not when:** general feature work with no specific run under review.
- **Kind:** DAI-specific. **Provenance:** DAI-native (the only skill tracked in `dai` before this slice). Multi-file (SKILL.md + purpose/inputs/outputs/rules/anti-patterns/examples).
- **Note:** description starts with "Diagnose..." not "Use when..." -- minor convention deviation, left as-is to avoid churn.

### dai-agent-handoff
- **Purpose:** compressed, structured handoff for a fresh session/agent/compaction.
- **Use when:** finishing a milestone, wrapping a session, context getting large.
- **Not when:** mid-task with no session boundary coming.
- **Kind:** general (workspace-flavored). **Provenance:** copied verbatim from `jera-workspace-skills/skills/dai/` (byte-identical at copy time). Upstream attribution: rewritten ideas inspired by mattpocock/skills (MIT) per the pack's NOTICE.
- **Note:** body still references Jera repos in places; candidate for later localization, not blocking.

### dai-grill-me
- **Purpose:** decision-by-decision interrogation of a fuzzy plan before implementation.
- **Use when:** scope/ordering/success criteria are ambiguous and code would be wasted.
- **Not when:** the slice spec is already explicit (most DAI slice prompts are).
- **Kind:** general. **Provenance:** copied verbatim from the Jera pack; mattpocock/skills-inspired rewrite per NOTICE.

### dai-grill-with-vault
- **Purpose:** challenge a plan against existing repo code and vault doctrine before deciding.
- **Use when:** a proposal may conflict with prior ADRs, the deferred-decisions ledger, or current behavior; repo/vault consistency checks.
- **Not when:** pure mechanical execution of an already-grilled spec.
- **Kind:** DAI/Jera-specific. **Provenance:** copied verbatim from the Jera pack. Body references jera-vault/jera-site; in DAI sessions read "vault" as `dai-vault`. Localization candidate.

### dai-token-tight
- **Purpose:** dense senior-engineer prose; kill preamble/recap bloat.
- **Use when:** sessions get verbose; the user says "tighten up" / "token-tight"; bounded-scope reporting.
- **Not when:** the reader needs explanatory depth (new architecture, onboarding).
- **Kind:** general. **Provenance:** copied verbatim from the Jera pack; mattpocock/skills-inspired rewrite per NOTICE.

### dai-write-skill
- **Purpose:** discipline for adding/updating skills: tight scope, triggers, anti-triggers, repo boundary, attribution, no runtime protocol code.
- **Use when:** creating or reviewing any skill in this workspace.
- **Not when:** one-off workflows (don't make a skill); raw skill craft alone (pair with `superpowers:writing-skills`).
- **Kind:** general (workspace-flavored). **Provenance:** copied from the Jera pack, then **adapted in this slice**: repo boundary and closing checklist re-pointed from the Jera pack layout (`jera-workspace-skills/skills/`, `.claude-plugin/plugin.json`, `skills/dai/README.md`) to the DAI layout (`dai/.claude/skills/<name>/SKILL.md`, this inventory doc). Attribution: mattpocock/skills (MIT), back-pointer at `dai/.claude/skills/references/mattpocock-skills-attribution.md` (copied from the Jera pack in this slice so the chain resolves locally).

### product-ui-design-architect
- **Purpose:** frontend UI/UX refinement -- layout rhythm, spacing, copy rhythm, responsive QA, accessibility, launch readiness.
- **Use when:** buyer-facing UI polish on the sports app or marketing surfaces.
- **Not when:** type/structure work in TS (use `dai-typescript-angular-quality`); backend work.
- **Kind:** general. **Provenance:** copied verbatim from `jera-workspace/.claude/skills/` (Jera workspace project skill, not the pack). Has examples/references/scripts subfolders.

### dai-skill-router (new, this slice)
- **Purpose:** mandatory Skills Gate at slice start: list available skills, select, name missing, state fallback, proceed/blocked verdict.
- **Use when:** starting any DAI slice; resuming after compaction; a prompt names skills.
- **Not when:** mid-slice after the gate has run; trivial Q&A.
- **Kind:** DAI-specific. **Provenance:** original, written this slice.

### dai-test-discipline (new, this slice)
- **Purpose:** bounded verification ladder for red/green work; 90-second rule; full breadth only at explicitly-declared final verification.
- **Use when:** any TDD cycle in `dai`; choosing what to run after an edit; a test command runs long.
- **Not when:** final verification itself (run the full breadth there, once).
- **Kind:** DAI-specific (commands name this repo's projects; includes the sports-app `npm test -- --watch=false` vs bare-vitest gotcha observed in a real session).
- **Provenance:** original, written this slice.

### dai-typescript-angular-quality (new, this slice)
- **Purpose:** practical TS/Angular correctness rules for the buyer app: boundary types, no `any`/broad `as`, `unknown`+narrowing, thin components, pure projection helpers, scope discipline.
- **Use when:** touching `apps/sports-app` TypeScript or DTO mappers.
- **Not when:** copy/visual slices (pair with `product-ui-design-architect`); backend C#.
- **Kind:** DAI-specific. **Provenance:** original; editor-feedback discipline inspired by Matt Pocock / Total TypeScript (attribution section in the skill; no text copied).

### dai-docs-architect (new, 2026-06-23)
- **Purpose:** turn DAI concepts into durable vault doctrine: enforce one-doctrine-per-doc, a standard topic template, a documentation-slice workflow, and a verify-before-assert / doctrine-vs-deferred discipline.
- **Use when:** documenting or canonicalizing a major architectural/strategic/agent-workflow topic; a concept keeps being re-explained across slices.
- **Not when:** one-off notes or run logs; pure session handoff (`dai-agent-handoff`); authoring a skill (`dai-write-skill`); the claim/decision is not yet verified (`dai-grill-with-vault` first).
- **Kind:** DAI-specific. **Provenance:** original, written this slice. Routed from `dai-skill-router`. Partially absorbs the doctrine-layer intent of the recommended `dai-vault-doc-conventions` (which remains a candidate for low-level format mechanics).

### dai-code-reviewer (new, 2026-06-23)
- **Purpose:** pre-completion code review for implementation slices through a productized, production-minded, minimal-design lens; catch correctness/syntax/idiom/simplicity/naming/comment/architecture/test issues and refuse to rubber-stamp unverified completion.
- **Use when:** a code-changing slice (application code, DTOs/contracts, db/read models, frontend projections, agent-service models, or tests) is about to complete or hand off.
- **Not when:** docs-only slices (unless code/technical claims need review); pure planning; one-off notes.
- **Pairs with:** `dai-test-discipline` (test strategy), `dai-typescript-angular-quality` (TS/Angular depth), `dai-grill-with-vault` (architecture alignment), `dai-agent-handoff` (final handoff), `superpowers:verification-before-completion`.
- **Runtime behavior:** none -- read-only review skill; edits no files, recommends fixes that the implementing slice applies.
- **Graphify relationship:** may use Graphify for orientation/blast-radius (`query`/`explain`/`path`/`affected`) as navigation evidence only; verifies all findings against source/tests/vault; never enables semantic/cloud extraction, never runs on `dai-vault`, never commits graph artifacts.
- **Kind:** DAI-specific. **Provenance:** original, written this slice. Routed from `dai-skill-router`.

## Jera skills intentionally NOT brought into DAI

- None currently identified beyond what was already copied; the Jera pack's five dai-* skills and the workspace's product-ui skill are all present. Future Jera-side skills should pass `dai-write-skill`'s duplication check before being copied here.

## Known missing / previously phantom references

- Prior slice prompts referenced "skills repo / <JERA_SKILLS_ROOT>" as "not present" in handoffs -- resolved: the active root is `dai/.claude/skills/` and is now tracked in git.
- "planning" as a named skill: not a DAI skill; route to `superpowers:writing-plans` / `superpowers:executing-plans`.
- "buyer safety guidance" as a skill: does not exist; doctrine lives in vault docs (Buyer Copy Safety v1, Confidence Calibration Rules v1). Recommended future skill (below).

## Recommended future skills

- **dai-buyer-copy-safety** -- distill Buyer Copy Safety v1 + band-gate doctrine into a skill for any slice touching buyer-visible language (currently doc-only knowledge).
- **dai-slice-preflight** -- the recurring git-status/expected-state/stop-if-different preflight as a skill instead of per-prompt boilerplate.
- **dai-vault-doc-conventions** -- report/handoff/ledger formats (date/status/scope headers, addendum shape, ascii rules) as a skill so docs stay uniform without copying prior docs.
- Localization pass over the verbatim Jera copies (grill-with-vault, agent-handoff) to name DAI repos -- low priority, content is usable as-is.

## The DAI Skills Gate (reusable prompt snippet)

The Skills Gate is step 1 of the canonical slice lifecycle; the full lifecycle (gates, truth hierarchy, handoff contract) lives in `06 Execution/agent-slice-workflow-doctrine-v1.md`.

Include this in future slice prompts:

> Before starting this slice, run the DAI Skills Gate:
> - invoke dai-skill-router
> - list available DAI skills
> - select skills for this slice
> - report missing expected skills
> - proceed only after the skill plan is explicit

## Validation notes (this slice)

- All 10 skills have `SKILL.md` with `name:` matching the folder and a third-person trigger description.
- New skills follow the pack shape: Overview + core posture, when to use, when not to use, body, repo boundary, pair-with, attribution where adapted.
- Skills were validated structurally and by live session loading (the three new skills appeared in the active session's skill list after creation). Subagent pressure-testing per `superpowers:writing-skills` was NOT performed in this slice -- recorded honestly as a gap; candidates for a later hardening pass if the skills underperform in real sessions.

## Operating-rule update (2026-06-15)

Claude Native Skill Registration Audit v1 (`claude-native-skill-registration-audit-v1.md`) confirmed all 10 skills are correctly registered for native Claude Code discovery (correct path, names match dirs, no `disable-model-invocation`, strong descriptions, all loadable this session). No frontmatter was changed -- the existing descriptions already meet/exceed Claude's auto-selection criteria. Decision: **native discovery is the primary skill-selection mechanism; `dai-skill-router`'s gate is retained as an audit/high-risk layer** (mandatory when a prompt/handoff names skills or for high-risk slices; lightweight on trivial/read-only slices). The gate's durable value is the expected-but-missing check + accountability + seeding the handoff skills-used line, not re-listing obviously relevant skills. Cross-repo note: `jera-workspace/.claude/skills/` holds 6 duplicate copies -- a drift risk, no canonical-source decision taken.

## Skill-layer update (2026-06-23)

DAI Documentation Skill and Topic Slice System v1 added `dai-docs-architect` (skill count 10 -> 11). It governs durable topic doctrine: one-doctrine-per-doc decomposition, a standard topic template, and a documentation-slice workflow. `dai-skill-router` now routes doctrine-documentation work to it and pairs with it. A companion documentation backlog lives at `06 Execution/backlog/documentation-doctrine-backlog-v1.md`. No runtime code or behavior changed.

## Skill-layer update (2026-06-23, code review)

DAI Code Review Skill v1 added `dai-code-reviewer` (skill count 11 -> 12): a read-only pre-completion review for code-changing slices, covering correctness/syntax/idiom/simplicity/naming/comments/architecture/product/testing/security with a fixed `DAI CODE REVIEW` output and an approve / approve-with-notes / request-changes verdict. `dai-skill-router` now routes any code-changing slice to it before final handoff and lists its pairings. No runtime code or behavior changed.
