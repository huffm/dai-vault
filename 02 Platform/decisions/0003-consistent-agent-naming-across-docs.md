# decision 0003: consistent agent naming across platform and niche docs

**date:** 2026-04-09
**status:** accepted

## decision
all platform and niche documentation must use the canonical agent role names defined in `02 Platform/agents/canonical-agent-roles.md`. informal shorthand is acceptable in step descriptions but the canonical name must appear in every agent roles section.

canonical names: `collector`, `evaluator`, `synthesizer`, `compliance`, `delivery`

## context
during initial niche doc creation, all four niche workflow files independently used different names — `fetcher agent`, `scorer agent`, `summarizer agent`, `dispatcher agent` — without referencing the canonical set. this created a vocabulary split where a developer reading platform docs and niche docs would not recognize them as describing the same pipeline.

the split was caught in an audit and corrected (see decision 0002), but the pattern is easy to re-introduce when a new niche doc is written from scratch or a workflow section is edited in isolation.

## why consistency matters here

**developer handoff.** when implementation begins, the agent names in docs become the class names, function names, and log labels in code. if the docs are inconsistent, the code will be too.

**prompt construction.** the shared prompt patterns in `02 Platform/prompts/` inject the canonical role name into every system prompt. if a niche doc uses a different name, the prompt and the doc describe different things to different readers.

**compliance visibility.** the compliance agent is the role most likely to be omitted when someone copies a workflow section from another niche. naming consistency is what makes its absence obvious during review.

## rule
when writing or editing a niche workflow doc:
1. the "agent roles used" section must list all five canonical roles
2. each role must use the canonical name, not an informal variant
3. if a role genuinely does not apply to a workflow, it must still be listed with a note explaining why it is a no-op — silent omission is not acceptable

when adding a new canonical role in the future:
1. update `canonical-agent-roles.md` first
2. update all existing niche workflow docs before closing the change
3. record the addition as a new decision

## rejected alternative
allow niche docs to use their own local names as long as a mapping table exists somewhere. rejected because a mapping table is a second place to maintain and will drift.
