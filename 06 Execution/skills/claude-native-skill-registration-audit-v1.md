# Claude Native Skill Registration Audit v1

**date:** 2026-06-15
**status:** audit only. no skill body or frontmatter changed, no code, no product-runtime change. confirms native discovery is healthy and reframes the skills gate as an audit/high-risk layer rather than a mandatory every-slice recital.
**classification:** tooling / workflow-quality audit.

## What was inspected

- `dai/.claude/skills/` -- the project skills root (10 skills + a shared `references/` folder).
- `jera-workspace/.claude/skills/` -- a separate repo carrying 6 copies of DAI skills.
- `dai-skill-router` (the skills gate) body and behavior.
- `dai-vault/06 Execution/skills/dai-skills-inventory-v1.md` (existing governance/source-of-truth).
- No workspace-level or user-level `.claude/skills/` exists; no `.claude-plugin/plugin.json` (that shape is the Jera pack, not this project-skills layout) -- consistent with the inventory doc.

## Skill inventory table (dai/.claude/skills/)

All have a valid `SKILL.md`, a frontmatter `name` matching the directory, no `disable-model-invocation`, and a description following the when/trigger/scope/boundary pattern. Native auto-selection readiness is HIGH for all; confirmed loadable (every one appears in this session's invocable-skills list).

| skill | kind | description quality | native-ready | overlaps gate? | action |
|---|---|---|---|---|---|
| dai-skill-router | meta/routing | strong | yes | (is the gate) | keep |
| dai-grill-me | workflow | strong | yes | no | keep |
| dai-grill-with-vault | workflow | strong | yes | no | keep |
| dai-token-tight | workflow | strong | yes | no | keep |
| dai-agent-handoff | workflow | strong | yes | no | keep |
| dai-test-discipline | workflow | strong | yes | no | keep |
| dai-write-skill | meta | strong | yes | no | keep |
| dai-signal-follow-up-diagnostics | DAI-specific | strong (opens "Diagnose..." not "Use when...") | yes | no | keep (minor convention deviation, already noted in inventory; native selection still works) |
| dai-typescript-angular-quality | DAI-specific | strong | yes | no | keep |
| product-ui-design-architect | domain | strong | yes | no | keep |

`references/` is a shared attribution/reference folder, not a skill -- correctly has no `SKILL.md`.

## Native Claude skill registration readiness

**Ready.** Skills are in the correct Claude Code filesystem location (`dai/.claude/skills/<name>/SKILL.md`), frontmatter names match directories, none are disabled for model invocation, and all 10 are discoverable this session. Descriptions already satisfy Claude's auto-selection criteria: each states when to use, the trigger phrases, the task scope, and the boundary/difference from neighbors. Native discovery is the working primary mechanism today.

## Description quality findings

No description is weak. Each carries explicit triggers (quoted user phrases) and a "not when"/boundary sense. The slice's proposed candidate descriptions (compact one-liners) are actually a **downgrade** from what exists -- the current descriptions are richer and more selective, which helps native auto-selection, not hurts it. Therefore **no frontmatter edits were made** (changing good descriptions for change's sake would add drift risk and churn for no gain). The single convention deviation (`dai-signal-follow-up-diagnostics` opens with "Diagnose..." rather than "Use when...") is cosmetic, already documented in the inventory, and does not impair discovery; left as-is.

## Cross-repo duplication finding

`jera-workspace/.claude/skills/` holds 6 copies of DAI skills (dai-agent-handoff, dai-grill-me, dai-grill-with-vault, dai-token-tight, dai-write-skill, product-ui-design-architect). This is the historical copy path (the inventory notes the DAI skills originated from the Jera workspace). It is not broken -- each repo discovers its own copy when that repo is the working tree -- but it is a **drift risk**: edits to a DAI skill will not propagate to the Jera copy and vice versa. Reconciling them (a canonical source + sync, or intentional divergence) is a cross-repo decision, out of scope here; flagged as an open risk, not acted on.

## Skills gate value assessment

What `dai-skill-router` does today: before the first file change it emits a 7-section gate report -- available DAI skills, available general skills, selected (with reasons), intentionally-not-selected (with reasons), expected-but-missing, fallback doctrine, proceed/blocked verdict.

Evaluated against native discovery:

- **Repeats native discovery? Partly.** Native auto-selection already surfaces relevant skills, so "which skills apply" overlaps.
- **Adds value native does not:** (1) detects **expected-but-missing** skills a prompt or handoff names -- the original failure mode (prompts referencing skills that were not loadable); native invocation silently omits a missing skill with no signal. (2) **Accountability** -- forces naming what was deliberately NOT selected, so omissions are decisions. (3) **Feeds the handoff** -- the "skills used" line in every final handoff comes from the gate. (4) A deliberate **pause before edits**.
- **Costs:** modest token overhead (the 7-section block) and friction on trivial read-only/docs slices where native selection is plainly sufficient.

Net: the gate is not redundant, but its full ceremony is overkill for low-risk slices. Its durable value is the missing-skill audit + accountability, not the re-listing of obviously-relevant skills.

## Recommendation

Native Claude skills are the **primary** selection mechanism (ready and working). Retain the gate, but **reduce it from mandatory-every-slice to an audit / high-risk layer**:

- **Run the full gate when:** a slice prompt or incoming handoff **names skills** (verify they are loadable -- the original failure mode); the slice is **high-risk** (product runtime, schema/migration, money/model spend, matcher/confidence/lean); or native selection looks unreliable.
- **Lightweight / optional when:** trivial, read-only, or docs-only slices -- let native auto-selection drive, and state skills used in one line.
- **Always:** keep the inventory (`dai-skills-inventory-v1.md`) as the source of truth, and keep the gate's missing-skill check available as the cheap insurance it was built to be.

## Proposed future operating rule

"Native skill discovery selects skills by default. The skills gate is an accountability/audit step, mandatory only for skill-naming or high-risk slices, where it verifies named skills are loadable, records deliberate omissions, and seeds the handoff's skills-used line. Do not recite the full gate on trivial slices." Most current DAI slice prompts name skills explicitly, so in practice the gate keeps running -- by the rule, not by reflex.

## Open risks

- Jera/DAI skill duplication can drift; no canonical-source decision taken.
- "Description strong enough for native auto-selection" is judged from frontmatter quality, not from an observed auto-invocation trace; a follow-up manual session should confirm Claude auto-selects (e.g. dai-grill-with-vault, dai-token-tight) without the gate naming them.
- Reducing the gate to optional risks reintroducing the silent-missing-skill failure if a future prompt names a skill on a slice classified "trivial"; the rule keeps the missing-skill check mandatory whenever skills are named, which mitigates this.

## How to test native auto-invocation (follow-up, manual)

In a fresh session in `dai-workspace`, give a slice prompt that does NOT name skills but clearly matches a skill's triggers (e.g. "tighten up the responses" -> dai-token-tight; "challenge this plan against the vault" -> dai-grill-with-vault) and observe whether Claude auto-invokes without the gate. If it does, the optional-gate posture is safe; if not, keep the gate mandatory for that skill's domain.

## Recommended next slice

No tooling change required. If the operating rule is adopted, a one-line note can be added to `dai-skill-router`'s "When NOT to use" (trivial/read-only slices) in a future micro-slice -- not done here to avoid editing the gate during its own audit. Product priority remains: reconcile the 9 active usable MLB runs after settlement.
