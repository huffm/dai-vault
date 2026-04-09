# decision 0002: canonical agent role names

**date:** 2026-04-09
**status:** accepted

## decision
adopt the following canonical agent role names across all platform and niche documentation:

| canonical name | also seen as | role |
|---|---|---|
| collector | fetcher agent | gathers raw data from external sources |
| evaluator | scorer agent | scores, ranks, or filters collected data |
| synthesizer | summarizer agent | turns scored data into decision-ready narrative |
| compliance | (not yet used in niche docs) | checks outputs against safety and business constraints |
| delivery | dispatcher agent | formats and dispatches the brief to the tenant |

## context
the platform's `canonical-agent-roles.md` defined five role names. all four niche workflow documents independently used different names (fetcher, scorer, summarizer, dispatcher) without acknowledging the canonical set. this created a vocabulary split where platform docs and niche docs described the same pipeline with different words.

## decision drivers
- a consistent vocabulary reduces ambiguity when discussing or implementing the pipeline.
- the canonical names (collector, evaluator, synthesizer, compliance, delivery) are more abstract and transfer across niches without carrying domain assumptions.
- niche docs are allowed to use informal shorthand (e.g. "the scorer step") but should reference the canonical name on first use.

## consequences
- niche workflow docs should be updated to reference canonical names in the agent roles section. informal shorthand is acceptable in step descriptions.
- new niches must use the canonical names in their workflow docs from the start.
- the compliance agent must be explicitly referenced in every niche workflow, even if its role is to confirm no constraint applies.

## rejected alternative
adopt the niche-doc names (fetcher, scorer, summarizer, dispatcher) as the canonical set. rejected because those names are domain-flavored (fetcher is a data concept, dispatcher is a network concept) and do not generalize as cleanly as the abstract role names.
