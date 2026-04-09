# niche: {{name}}

## thesis
what painful decision does this niche help users make faster, better, or more safely?
what is the specific moment of failure this product addresses?
what is the narrowest v1 wedge?

## customers
who is the most likely first paying customer — be specific about their role, budget, and current workaround.
who are the secondary customers?
what do they pay for today, and why is it not good enough?

## signals
what signal categories does this niche score?
list each category and describe what constitutes a favorable, neutral, or unfavorable reading.
note which signals are v1 and which are deferred.

## workflows
what are the named workflows (e.g. pre-game brief, daily sweep, earnings brief)?
for each workflow: trigger, steps, output format, agent roles used (collector, evaluator, synthesizer, compliance, delivery).
what does each workflow explicitly not do?

## monetization
what are the subscription tiers (free, starter, pro) and what does each include?
what drives willingness to upgrade?
what future monetization options exist?
note: billing enforced by platform — see `02 Platform/architecture/billing-and-stripe.md`.

## data-sources
list primary data sources with endpoint, cost, and use.
list secondary sources under consideration.
include a data freshness table (signal → refresh frequency).
note any sources with ToS risk or unstable api contracts.

## prompts
list the prompt docs in this folder:
- collector-prompt.md
- evaluator-prompt.md
- synthesizer-prompt.md
- delivery-prompt.md

note the base patterns they extend:
- `02 Platform/prompts/shared-system-prompt-pattern.md`
- `02 Platform/prompts/shared-evaluation-pattern.md`

## experiments
what are the first things to test before committing to a full build?
list as hypothesis → test → success condition.

## decisions
link to any decision memos recorded in the `decisions/` subfolder for this niche.
format: `- [NNNN-title](decisions/NNNN-title.md) — one-line summary`
