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

### dai-slice-runner (new, 2026-06-23)
- **Purpose:** drive a DAI slice end to end through the canonical lifecycle (orient -> bound -> select skills -> execute -> verify -> review -> handoff -> commit/push discipline); the executable encoding of `06 Execution/agent-slice-workflow-doctrine-v1.md`. Coordinates the other skills; does not replace judgment or restate their internals.
- **Use when:** starting any non-trivial slice; coordinating repo+vault work; a task needing multiple skills or with code/docs/architecture/test/handoff implications.
- **Not when:** one-off explanation, pure Q&A, tiny wording edits, when the user opts out of the full process, or nested inside an already-running slice.
- **Pairs with:** `dai-skill-router` (its step 3 Skills Gate), `dai-agent-handoff` (final handoff / close output), `dai-code-reviewer` (code-changing slices), `dai-test-discipline`, `dai-docs-architect`, `dai-grill-with-vault`, `dai-token-tight`.
- **Runtime behavior:** none -- a process driver; it edits no files of its own and pushes only when explicitly instructed.
- **Graphify relationship:** uses Graphify for code orientation/blast-radius only (navigation evidence; verify against source/tests); never runs it on `dai-vault`, never enables semantic/cloud extraction, never commits `graphify-out/`.
- **Kind:** DAI-specific. **Provenance:** original, written this slice; encodes the slice-workflow doctrine. Routed from `dai-skill-router`.

### dai-slice-prompt-architect (new, 2026-06-24)
- **Purpose:** doctrine-backed prompt compiler -- convert verified project state (handoffs, current-slice, vault doctrine, skills inventory, calibration/backlog/repo state, Graphify orientation) into a reviewable, executable next-slice prompt. Proposes; does not execute or decide strategy.
- **Use when:** drafting the next slice prompt; turning a handoff into a prompt; reviewing previous prompt quality; saving/updating `next-slice.md`; proposing skill/doctrine improvements from repeated prompt failures.
- **Not when:** executing a slice (`dai-slice-runner`); non-prompt doctrine (`dai-docs-architect`); authoring a non-prompt skill (`dai-write-skill`); plain session handoff (`dai-agent-handoff`); deciding product/business strategy (user/architect).
- **Pairs with:** `dai-slice-runner` (proposes vs executes), `dai-agent-handoff` (handoff in, prompt out), `dai-docs-architect` (promote a stable lesson to doctrine), `dai-token-tight` (dense prompts), `dai-grill-with-vault` (challenge a draft), `dai-write-skill` (the only applier of its MODE 4 proposals).
- **Runtime impact:** none -- writes prompts and `06 Execution/prompting/*` vault notes only; edits no runtime code, product/model prompts, or other skills.
- **Graphify relationship:** navigation evidence only; never on `dai-vault`; never semantic/cloud; never commits `graphify-out/`.
- **Learning-loop boundary:** never self-modifies. MODE 4 PROPOSES skill/doctrine updates (>=2 instances or 1 severe failure, overfitting risk stated); a separate user-approved skill-update slice (under `dai-write-skill`) APPLIES them. The user/architect remains the strategic approver.
- **Kind:** DAI-specific. **Provenance:** original, written this slice. Routed from `dai-skill-router`. Encodes `06 Execution/prompting/slice-prompt-architecture-doctrine-v1.md`.

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

## Skill-layer update (2026-06-23, slice runner)

DAI Slice Runner Skill v1 added `dai-slice-runner` (skill count 12 -> 13): the executable encoding of `06 Execution/agent-slice-workflow-doctrine-v1.md` -- an 8-step lifecycle driver (orient, bound, select skills, execute, verify, review, handoff, commit/push discipline) with required `DAI SLICE START` and `DAI SLICE CLOSE` outputs. It coordinates the existing skills (it is not a replacement) and routes from `dai-skill-router` as its step-3 Skills Gate. No runtime code or behavior changed.

## Skill-layer update (2026-06-24, slice prompt architect)

DAI Slice Prompt Architect Skill v1 added `dai-slice-prompt-architect` (skill count 13 -> 14): a doctrine-backed prompt compiler that converts verified project state into a reviewable, executable next-slice prompt. It proposes (6 internal modes: next-slice-prompt, prompt-review, prompt-learning-note, skill-improvement-proposal, next-slice-file, slice-type-specialization), the runner executes, and the user/architect approves. It is explicitly NOT self-modifying: MODE 4 proposes skill/doctrine updates, a separate user-approved skill slice applies them. `dai-skill-router` now routes next-prompt drafting/review/saving/improvement-proposals to it and pairs with it. Companion vault area `06 Execution/prompting/` adds the doctrine (`slice-prompt-architecture-doctrine-v1.md`), the lessons log (`prompt-patterns-and-lessons-v1.md`), the live proposal (`next-slice.md`), and `archive/`. No runtime code or behavior changed.

## Skill-layer update (2026-06-24, prompt architect enhancement)

DAI Slice Prompt Architect Enhancement v1 added a Strategic Snapshot + Opportunity Cost step to `dai-slice-prompt-architect` (no new skill; count stays 14). MODE 1 now silently evaluates seven inputs (WhyNow, FactoryOrProduct, RevenuePath, DecisionScarcity, StrategicHorizon, Invalidation, OpportunityCost) and compresses them into a 5-7 line Strategic Snapshot plus a one-line Opportunity Cost (target < 60 added tokens), placed after the recommendation rationale -- improving prioritization/bottleneck/scarcity recognition without adding ceremony. Decision-scarcity guidance prefers slices that resolve scarce information over volume; invalidation folds into Primary Risk. The snapshot informs prioritization only -- no execution authority (architect proposes, runner validates, user approves). The live `next-slice.md` was retrofitted to the new format. No runtime code or behavior changed.

## Skill-layer update (2026-06-25, current-state alignment)

Decision Intelligence Model Current-State Alignment v1 added a Current-State Alignment review to `dai-slice-prompt-architect` (sec 17; no new skill; count stays 14). Architecture/docs prompts must now require, before claiming completion, a check for stale "current truth" across the vault and a three-layer milestone-completeness test (Layer 1 new architecture introduced; Layer 2 related architecture linked; Layer 3 older platform docs reconciled) -- a milestone is not complete until all three agree. The reconciliation pattern is `Originally / Today / See <doctrine>`: preserve historical context, reference rather than duplicate doctrine, tie every "Today" claim to verified source/tests/slice-history. The docs/doctrine slice-type row gained the matching risk/verification cells. This is documentation-integrity discipline -- it changes no runtime code or behavior and proposes no tuning. (Same slice reconciled the stale learning-loop + retrieval current-truth in `decision-intelligence-model.md` and the run-evaluation line in `current-agent-run-contract.md`.)

## Skill-layer update (2026-06-25, runtime activation ladder)

Cognitive Factory Observability Surface v1 added a Runtime-Activation-Ladder section to `dai-slice-prompt-architect` (sec 18; new slice-type row "runtime activation"; no new skill; count stays 14). Runtime-activation prompts must now declare activation stage, mode (observes/configures/executes/mutates/productionizes), why the previous rung is complete, rollback boundary, observability surface, affected runtime capability, and explicit non-goals; the ladder is Observability -> Configuration-bound control -> Read-only execution -> Controlled mutation (Gate 4/5) -> Production, one rung per slice. Adds the reminder that a new endpoint is itself a runtime surface change (TDD + code review + dev-gating + no-execution tests), not a docs slice. Pattern confirmed across Runtime Activation Readiness v1 (defined the ladder) and Observability Surface v1 (executed Stage 0). No runtime code or behavior changed by the skill edit.

## Skill-layer update (2026-06-25, stage-1 config-bound checklist)

Cognitive Factory Configuration-Bound Control v1 added a Stage-1 (configuration-bound) checklist to `dai-slice-prompt-architect` sec 18 (no new skill; count stays 14): config-bound rungs must bind options default-off, resolve missing/invalid config to all-off, default off in every environment, surface the bound posture, add NO execution consumer (a "config true does not execute" test must prove observable-but-blocked), and roll back by config only; bind narrow flags, never an EnableEverything. Skill doc only; no runtime change.

## Skill-layer update (2026-06-25, stage-2 read-only-execution checklist)

Cognitive Factory Read-Only Execution v1 added a Stage-2 (read-only execution) checklist to `dai-slice-prompt-architect` sec 18 (no new skill; count stays 14): Stage-2 prompts must not say "read-only" without specifying audit-only vs decision-read-only vs artifact-non-mutating vs DB-read-only; require an allowed-write ledger, config-gate refusal tests, forbidden-write assertions, proof a valid run does not call model/tools unless that is the stated goal, a structured result with truthful did-not-mutate/did-not-call flags, and a gate-naming refusal; prefer a proven-safe inert path; if no audit schema fits, return in-memory and document the deferral. Skill doc only; no runtime change.

## Skill-layer update (2026-06-25, prompt ledger hook)

Prompt Ledger Hook v1 added a cross-cutting prompt-capture hook (`06 Execution/prompt-ledger-hook-v1.md`) and wired it by composition into three skills (no new skill; count stays 14): `dai-slice-prompt-architect` sec 19 (capture decision + capture_mode + ledger-ready summary on a finalized prompt; never drafts/secrets; external-first storage), `dai-docs-architect` (capture when a docs/milestone prompt materially changes canonical architecture), and `dai-agent-handoff` (link the final outcome back to a ledger record at slice close). Captures finalized, reusable prompts as compact Obsidian-friendly records at `<OBSIDIAN_PROMPT_LEDGER_ROOT>/<PROJECT_KEY>/prompts/...`; external-first, never forced into dai/dai-vault, one record per slice, no secrets/drafts. Docs + skill guidance only; no runtime/Cognitive-Factory change. Stage 2 (read-only execution) was already committed+pushed and was NOT touched; its prompt is backfilled retroactively as the worked example in the hook doc.

## Skill-layer update (2026-07-05, continuation-grade handoff standard)

Continuation-Grade Handoff Brief v1 made the richer, self-contained handoff format canonical and wired it by composition into three skills (no new skill; count stays 14): `dai-agent-handoff` gained a "Continuation-grade handoff brief" section that defines the concept and holds the **canonical 13-section template** (objective, outcome, repo state before/after, services, work performed, files changed, DB writes/side effects, paid calls/cost, validation proof, what did not change, open issues, recommended next slice, suggested next prompt) plus the rule "do not bury critical state in prose -- use bullets, paths, IDs, counts, commit hashes, service names, validation evidence"; `dai-slice-prompt-architect` sec 9 now requires every produced execution prompt to include a `HANDOFF BRIEF REQUIREMENT` section by default (referencing the canonical template, not restating it), and MODE 1's output structure names it; `dai-docs-architect` step I now requires a continuation-grade brief on any closeout/report/execution slice. Applies whenever an execution prompt is architected or a closeout/report is written. Template lives in `dai-agent-handoff`. Standard applied to the DAI-workspace pack (`dai/.claude/skills/`) only -- the generic `jera-workspace-skills` copy of `dai-agent-handoff` (already diverged into the light general-purpose dev-pack form) is intentionally NOT given the DAI-operational fields. Report: `06 Execution/reports/continuation-grade-handoff-skill-augment-2026-07-05-v1.md`. Docs/skills only; no runtime code or behavior changed.

## Skill-layer update (2026-07-09, system-development router)

DAI System Development v1 added `dai-system-development` (skill count 14 -> 15): the workflow
router for the development operating model under `02 Platform/system-development/`. Every
meaningful change (behavior/contract/UI/schema/doctrine-adjacent, or multi-session) starts
from a work item (`WI-####`, local-spine mode until Azure DevOps is wired), gets an OKF spec
with tests defined before code, executes through the unchanged slice stack
(`dai-skill-router` gate -> `dai-slice-runner` -> TDD -> `dai-test-discipline` ->
`dai-code-reviewer`), and closes with 8 required traceability links plus recorded lessons.

- **Purpose:** intake/spec seam before the first slice, traceability closeout after the last;
  routes six lenses (frontend-implementation, backend-implementation, architecture-contracts,
  testing-quality, ui-system-design, okf-curation) to the existing domain skills.
- **Use when:** starting/resuming/closing a `WI-####`; any change above the meaningful-change
  threshold in `02 Platform/system-development/operating-model.md`.
- **Not when:** typos/comments/doc hygiene/dep bumps; vault strategy or calibration work;
  mid-slice execution mechanics (slice doctrine owns those; never nested inside a running slice).
- **Kind:** DAI-specific. **Provenance:** DAI-native (this slice). Single-file SKILL.md;
  canonical docs live in the vault, the skill is the router.
- **Companion vault area:** `02 Platform/system-development/` (MOC, operating-model,
  work-item-traceability, implementation-lifecycle, frontend/backend/architecture/testing
  docs, design-system/{component-rules, interaction-states, visual-qa-checklist},
  work-items/{_template, WI-0001}). The 9-field OKF front matter is deliberately extended to
  this subtree (placement extension recorded in the MOC; one new type value `moc`).
  No runtime code or behavior changed.

## Skill-layer update (2026-07-13, WI-0007 mandatory work items + slice synopsis)

WI-0007 (Mandatory Structured Work Items and Slice Synopsis Workflow v1) amended five existing
skills; no skill added or retired (count stays 15). Two invariants entered the canonical layer,
each with exactly one owner:

1. **Work-item qualification gate** -- owned by `dai-slice-runner` (blocking, four-line output
   before any production/script/workflow/skill/doctrine change; SLICE START/CLOSE templates
   extended). Qualifying categories, non-qualifying exceptions, status discipline
   (implementation complete vs merge ready vs integrated vs closed as body-level dispositions
   over the canonical front-matter statuses), the integration-continuation rule, and the
   meta-work rule live in `dai-system-development`. OKF placement/front matter/MOC
   registration for work items is owned by `dai-docs-architect` (born-correct rule).
   `dai-skill-router` surfaces the requirement at routing time.
2. **Slice Synopsis** -- canonical Change/Reason/Proof/State/Next format (~100 words, always
   the final section of every execution/implementation/integration/closeout handoff) defined
   once in `dai-agent-handoff`, together with mandatory work-item identity fields (governing
   WI, status, OKF path, MOC disposition). Other skills reference; none restate.

Routine governed operational cadence under an existing OKF plan still does not mint a WI
(v2-settlement precedent preserved); operational remediation, investigation, migration,
measurement, and workflow/doctrine changes now do. Files: the five SKILL.md bodies under
`dai/.claude/skills/`; governing WI:
`02 Platform/system-development/work-items/WI-0007-mandatory-work-items-and-slice-synopsis-workflow.md`.

## Skill-layer update (2026-07-13, WI-0008 next-slice planner)

WI-0008 added `dai-next-slice-planner` (skill count 15 -> 16): the evaluation half of the
evidence-grounded planning system.

- **Purpose:** consume the deterministic planning snapshot
  (`dai/scripts/dev/planning/build-next-slice-snapshot.ps1`) and recommend exactly one bounded
  next slice -- hard eligibility gates, <=3 ranked candidates across documented dimensions,
  alternatives explained, one explicit operator decision request, mandatory Slice Synopsis.
- **Use when:** choosing the next slice after a close/integration; ranking deferred candidates;
  auditing a proposed slice against the evidence.
- **Not when:** the operator already chose and authorized the next slice; mid-slice scope
  questions; anything requiring the planner to act on its own recommendation.
- **Kind:** DAI-specific. **Provenance:** DAI-native (WI-0008). Single-file SKILL.md.
- **Hard boundary:** advisory only -- never creates/authorizes/schedules/executes work, never
  writes dates (system suggestions are clearly-labelled proposals only), fail-closed on
  authorization (`unknown` is not permission).
- **Companion tooling:** `scripts/dev/planning/` (snapshot script + fixture test suite, 44
  asserts) and the operator-owned `06 Execution/plans/platform-delivery-timeline-v1.md`
  (timeline intent; authorization posture block).

## Skill-layer update (2026-07-13, WI-0010 planner evidence fidelity)

WI-0010 amended `dai-next-slice-planner` (compatibility only; count stays 16, ranking model
unchanged): candidates now carry `status` (active/delivered/gated), canonical `initiativeId`,
merged `origins`, and `deliveredBy` -- only `active` entries are rankable; `delivered` is
history; `gated` is reported, never ranked. `reconciliation.facts` (structured,
provenance-backed readout parsed from the canonical sidecar
`06 Execution/reports/reconciliation-planning-readout-v1.md`) is consumed directly; null facts
fall back to the pointer with an explicit statement. Snapshot schema 1.0 -> 1.1 (additive).
Governing WI: `02 Platform/system-development/work-items/WI-0010-planner-evidence-fidelity.md`.

> **Errata (WI-0025):** update-log entries above cite pre-migration paths (`06 Execution/agent-slice-workflow-doctrine-v1.md` and `06 Execution/prompt-ledger-hook-v1.md`); these documents now live under `06 Execution/patterns/`. Historical entries are preserved verbatim.
