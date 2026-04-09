# decision 0001: separate vault from monorepo

**date:** 2026-04-09
**status:** accepted

## decision
keep the strategic knowledge base (dai-vault) in a separate folder and separate git repository from the implementation monorepo (dai).

## context
the workspace has two sibling folders under one root:
- `dai` — source code, agents, backend, niche implementations
- `dai-vault` — strategy, architecture docs, niche briefs, decisions, research

the question was whether to keep them together (one repo) or separate (two repos).

## decision drivers
- git history for code and for docs serve different purposes. a commit to a niche thesis doc should not appear in the code repo's log.
- the vault may be shared with non-technical collaborators (researchers, advisors, strategists) who should not have code access.
- obsidian works better when the vault root is clean — no `node_modules`, no build artifacts, no `.env` files.
- strategic documents and implementation history are audited separately. conflating them adds noise to both.

## consequences
- changes to platform docs or niche strategy do not trigger CI or code review workflows.
- keeping the two aligned is a manual discipline — when a niche doc changes its workflow design, the corresponding code must be updated separately.
- claude code and similar tools must be aware of both paths to work across the boundary.

## rejected alternative
single monorepo with a `docs/` or `vault/` subfolder. rejected because it violates the clean vault root obsidian requires and conflates doc and code history.
