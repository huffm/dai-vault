---
title: "Platform Delivery Timeline v1"
type: "plan"
date: "2026-07-13"
status: "active"
project: "DAI"
slice: "WI-0008 Evidence-Grounded Next-Slice Planner v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - planning
  - timeline
  - roadmap
related:
  - "02 Platform/system-development/work-items/WI-0008-evidence-grounded-next-slice-planner.md"
  - "02 Platform/system-development/work-items/WI-0006-identity-safe-mlb-doubleheader-resolution.md"
---

# platform delivery timeline v1

**Operator-owned planning intent. Not an autonomous schedule.**

Rules of this document:

- `desired_by` and `due_by` are OPERATOR-owned. The system (any script, skill, or agent) must
  never write or overwrite them. A system suggestion goes in `proposed_by_system` only.
- `desired_by` = operator preference; `due_by` = real external commitment (only when one truly
  exists); `not_before` = sequencing/dependency constraint. Never collapsed into one field.
- No invented dates. Empty means "no operator date exists yet" -- that is a fact, not a gap to
  fill.
- Every entry cites the evidence that makes it real. Entries without canonical evidence do not
  belong here.
- Machine consumers parse the fenced yaml blocks below and the authorization block; prose is
  for humans.

## authorization posture (operator-owned, read by tooling; fail-closed)

```yaml
authorization:
  paid_model_calls: not-authorized
  sports_capture: not-authorized
  reconciliation_writes: not-authorized
  posture: no-spend
  source:
    - "06 Execution/handoffs/current-slice.md (2026-07-11 v2 cadence WRAP -- authorization ENDS)"
    - "02 Platform/system-development/work-items/WI-0006-identity-safe-mlb-doubleheader-resolution.md (approval boundary)"
  as_of: "2026-07-13"
```

## initiatives

### gamepk propagation through the generation request

```yaml
initiative_id: gamepk-propagation
title: propagate gamePk through CompetitionMatchupInput for doubleheader capture
status: candidate
priority: high
desired_by:
due_by:
not_before:
proposed_by_system:
date_source: none
date_confidence: low
depends_on:
blocks:
  - doubleheader-capture-capability
economic_reason: doubleheaders are currently uncapturable (fail-closed); each ambiguous slate day forfeits capturable paid-product volume until the request can carry an exact event identity
operator_intent: named as deferred item 1 at WI-0006 close; contract-expansion work requiring its own WI and review (touches persisted InputJson and the public analyze body)
replan_triggers:
  - capture authorization resumes
  - a doubleheader-heavy slate is scheduled during an active capture window
related_work_items:
  - WI-0006
```

### first-class identity-status refinement

```yaml
initiative_id: identity-status-refinement
title: first-class no_match / ambiguous / source_failure statuses on /source-readiness
status: candidate
priority: low
desired_by:
due_by:
not_before:
proposed_by_system:
date_source: none
date_confidence: low
depends_on:
blocks:
economic_reason: diagnostic clarity only; today the states are already distinguishable via IdentityReason, so buyer/product value is indirect
operator_intent: named as deferred item 2 at WI-0006 close; independent of gamepk-propagation
replan_triggers:
  - a screening consumer misreads unmatched vs ambiguous in practice
related_work_items:
  - WI-0006
```

### artifact chip alignment (presentation backlog)

```yaml
initiative_id: wi-0002-artifact-chip-alignment
title: WI-0002 artifact chip primitive alignment
status: backlog-not-authorized
priority: low
desired_by:
due_by:
not_before:
proposed_by_system:
date_source: none
date_confidence: low
depends_on:
blocks:
economic_reason: presentation consistency; no correctness or capture impact
operator_intent: registered BACKLOG with disposition open and an activation gate in its spec
replan_triggers:
  - operator authorizes presentation work
related_work_items:
  - WI-0002
```

### shared chip module promotion (presentation backlog)

```yaml
initiative_id: wi-0003-shared-chip-module
title: WI-0003 shared chip and long-token module promotion
status: backlog-not-authorized
priority: low
desired_by:
due_by:
not_before: a concrete second consumer of the chip primitive exists
proposed_by_system:
date_source: none
date_confidence: low
depends_on:
  - wi-0002-artifact-chip-alignment
blocks:
economic_reason: abstraction promotion without a second consumer is speculative; explicitly gated in its spec
operator_intent: registered BACKLOG, gated on a concrete second consumer
replan_triggers:
  - a second consumer materializes
related_work_items:
  - WI-0003
```

## deferred decisions

- calibration volume expansion: not currently justified (Gate 4 posture; v2 cadence closed
  2026-07-11 with authorization ended) -- revisit only under a new operator authorization.
- cross-sport event-identity abstraction: rejected until a second sport path needs it
  (WI-0006 decision record).

## change log

- 2026-07-13: created (WI-0008). Seeded from WI-0006 deferred items and WI-0002/0003 backlog
  state. No operator dates exist yet; all date fields deliberately empty.
