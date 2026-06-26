# prompt ledger hook v1

**status:** active doctrine -- canonical cross-cutting process hook for capturing finalized slice prompts.
**date:** 2026-06-25
**type:** process + skill-architecture slice. no runtime/app/Cognitive-Factory code; docs + skill guidance only.

**Prompt Ledger Hook is a cross-cutting lifecycle hook.** It is invoked by composition (not inheritance) from
`dai-slice-prompt-architect`, `dai-docs-architect`, and `dai-agent-handoff`; future skills can invoke the same hook
without copying its rules.

## purpose

Preserve finalized, reusable slice prompts as compact, Obsidian-friendly provenance records -- so important
implementation, architecture, governance, and skill-evolution prompts become reusable project history without
bloating the repos or duplicating conversational drafts. It is a lightweight prompt provenance trail.

What it is NOT: not runtime code; not model memory; not a replacement for vault docs (those capture decisions/
doctrine -- this captures the *prompt* that drove a slice); not an archive of every draft.

## A. storage model (external-first)

Records live external-first, addressed by convention, never hard-coded into app code:

```
<OBSIDIAN_PROMPT_LEDGER_ROOT>/<PROJECT_KEY>/prompts/<YYYY>/<MM>/<YYYY-MM-DD>-<SLICE_NAME>.md
```

Placeholders: `<OBSIDIAN_PROMPT_LEDGER_ROOT>`, `<PROJECT_KEY>` (e.g. `dai`), `<SLICE_NAME>`, `<DATE>`.

**Configured (2026-06-26):** `<OBSIDIAN_PROMPT_LEDGER_ROOT>` is now set on the working machine to a `prompt-ledger`
folder that is a sibling of `dai/` and `dai-vault/` under the workspace root (outside both git repos). The literal
path lives only in the gitignored local path map (`<DAI_WORKSPACE_ROOT>/.local/agent-paths.md`, template at
`dai/docs/examples/agent-paths.example.md`); committed docs reference the placeholder only. Records resolve to
`<OBSIDIAN_PROMPT_LEDGER_ROOT>/dai/prompts/<YYYY>/<MM>/...`.

The ledger may live: outside the git repo; in an Obsidian-synced folder; in a workspace folder that is gitignored;
or -- only if the human explicitly wants versioning -- inside the vault (e.g. `06 Execution/prompting/ledger/`).
**Do not force the ledger into `dai` or `dai-vault`**, and **do not commit external Obsidian files** unless the
workspace clearly contains the agreed ledger folder. If the ledger root is unavailable, embed the record as an
example in this doc (as done below) rather than fabricating an external path.

## B. capture rules

**Capture a finalized prompt when it:** initiates a slice; changes platform architecture; changes runtime
activation posture; introduces or reinforces a reusable engineering pattern; creates/updates a skill; governs
settlement, reconciliation, calibration, evidence readiness, tenant/factory doctrine, or Cognitive Factory
activation; or is a reference prompt likely to be reused.

**Do NOT capture:** rough drafts; duplicate revisions; minor wording changes; casual discussion; prompts containing
secrets/credentials/private personal data; prompt fragments with no execution value; content already fully
represented by an existing record. **One canonical record per slice** -- not one per revision.

## capture timing

- **Primary:** after prompt finalization, *before* sending to Claude/Codex/agent (`capture_mode: pre-execution`).
- **Secondary:** after slice close, to link the outcome (`post-execution`), or to backfill a slice whose prompt
  predates this hook (`retroactive`).

## C. record format

Compact markdown. Required frontmatter, then the four required sections; optional sections only when they add value.

```markdown
---
date: <YYYY-MM-DD>
project: <PROJECT_KEY>
slice: <slice-name>
lane: <platform-runtime | architecture | governance | calibration | skill-evolution | ...>
activation_stage: <none | 0 | 1 | 2 | 3 | 4>
prompt_type: <implementation | architecture | docs | governance | skill | calibration | ...>
agent_target: <claude | codex | human>
capture_mode: <pre-execution | post-execution | retroactive>
status: <proposed | executing | executed>
source: <chat-handoff | current-slice | next-slice.md | local-notes>
repo_target: <dai | dai-vault | both | none>
related_docs: [<path>, ...]
related_commits: [<repo sha>, ...]
---

# Prompt Summary
<3-7 lines: what the slice does and why this prompt matters>

# Guardrails
<compact bullets: the hard boundaries / non-goals the prompt enforced>

# Final Prompt
<the finalized canonical prompt verbatim, OR a reference to where it lives>

# Outcome
<link to the slice handoff / commits; one-line result>

# Reusable Pattern        (optional)
# Skill Updates           (optional)
# Follow-Up Prompt        (optional)
```

Space discipline: summary 3-7 lines; guardrails compact; include the full prompt only for finalized canonical
prompts; outcome links to the handoff/commit rather than copying it; prefer references over duplication.

## privacy / security boundaries

Never capture secrets, credentials, tokens, connection strings, or private personal data. Redact before capture. A
prompt that cannot be captured without a secret is not captured (or is captured with the secret removed and a note).

## relationship to the skills (composition)

- **`dai-slice-prompt-architect`** invokes the hook when it produces a *finalized* slice prompt: decide whether the
  prompt warrants capture; if so, set `capture_mode` (pre/post/retroactive) and emit a compact ledger-ready summary;
  never capture drafts or secrets; classify by lane, stage, and reusable pattern.
- **`dai-docs-architect`** invokes the hook when a prompt *materially changes canonical architecture* -- the doc
  slice may trigger a ledger record so the architecture change has a prompt provenance link.
- **`dai-agent-handoff`** invokes the hook at slice close: when a ledger record exists, link the final outcome
  (handoff/commits) back to it.

The hook owns the rules once; the skills compose it. A new skill can invoke it without restating the format.

## retroactive capture

For a slice whose prompt predates the hook, recover the finalized prompt from local notes, `current-slice`, vault
docs, or chat handoff text; create one record marked `capture_mode: retroactive`; link the already-committed
outcome. Do not reconstruct from memory if the prompt text is unavailable -- note the gap instead.

## avoiding ledger bloat

One canonical record per slice; capture only finalized prompts that meet the criteria in B; prefer references over
copying large handoffs; archive or supersede rather than accumulate near-duplicates; if a prompt is already fully
represented by an existing record, do not add another.

## example (retroactive backfill -- Cognitive Factory Read-Only Execution v1 / Stage 2)

The Stage-2 prompt was finalized and executed (committed + pushed) before this hook existed, so it is captured
retroactively. In a real setup this file would live at
`<OBSIDIAN_PROMPT_LEDGER_ROOT>/dai/prompts/2026/06/2026-06-25-cognitive-factory-read-only-execution-v1.md`; here it
is embedded as the worked example.

```markdown
---
date: 2026-06-25
project: dai
slice: cognitive-factory-read-only-execution-v1
lane: platform-runtime-architecture
activation_stage: 2
prompt_type: implementation
agent_target: claude
capture_mode: retroactive
status: executed
source: chat-handoff + current-slice
repo_target: both
related_docs: ["02 Platform/architecture/cognitive-factory/cognitive-factory-read-only-execution-v1.md"]
related_commits: ["dai a6c1bc5", "dai 9de1cd3", "dai-vault f83e0d2"]
---

# Prompt Summary
Stage 2 of the Cognitive Factory activation ladder: the first controlled runtime execution.
A dev-only, config-gated, audit-only, non-mutating probe endpoint that proves the factory can
enter and exit a runtime path without changing any product behavior. Execution produces
observability, not product behavior.

# Guardrails
- dev-only (404 otherwise); execute only when Enabled && AllowExecution && AuditOnly && !AllowArtifactMutation.
- no model call, no tool/gateway/external call, no artifact/decision mutation, no db write.
- audit-only; durable persistence deferred if the existing schema does not fit (in-memory result).
- no settlement/reconciliation/calibration/buyer/frontend change; no tuning.

# Final Prompt
The finalized slice prompt is the verbatim "DAI SLICE START -- Cognitive Factory Read-Only Execution v1"
prompt (Parts A-K). Stored by reference here to respect space discipline; the canonical copy is the
chat handoff for that slice. (For a pre-execution capture, the full finalized prompt would be inlined.)

# Outcome
Implemented POST /api/dev/cognitive-factory/probe/audit-only running one deterministic
ProtocolNodeRunner.ExecuteAsync(interrogate.probe); diagnostics ReadOnlyExecution block + activationStage 2.
TDD; dai-code-reviewer clean; full DevCore.Api.Tests 928/0. See related_commits + related_docs.

# Reusable Pattern
Activation-ladder Stage 2 = audit-only execution behind config gates over a proven-safe inert path;
forbidden-write assertions + a throwing-tool-gateway test prove non-execution of heavier paths.

# Skill Updates
dai-slice-prompt-architect sec 18 gained a Stage-2 read-only-execution checklist.
```

## related docs

- `06 Execution/prompting/slice-prompt-architecture-doctrine-v1.md` -- prompt compilation doctrine (this hook is the
  capture step at finalization).
- `06 Execution/prompting/prompt-patterns-and-lessons-v1.md` -- lessons log (distinct: patterns, not prompt records).
- `06 Execution/agent-slice-workflow-doctrine-v1.md` -- slice lifecycle (the hook fires at finalization + close).

## recommended next slice

The ledger root is now confirmed and configured (see the **Configured (2026-06-26)** note in section A). Adopt the
hook in practice: the next finalized canonical prompt produces a `pre-execution` record at
`<OBSIDIAN_PROMPT_LEDGER_ROOT>/dai/prompts/<YYYY>/<MM>/...` -- no skill change is needed; the wiring already exists in
`dai-slice-prompt-architect` (sec 19), `dai-docs-architect`, and `dai-agent-handoff`.
