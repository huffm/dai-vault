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
status: complete
priority: high
desired_by:
due_by:
not_before:
proposed_by_system:
date_source: none
date_confidence: high
delivered_by: WI-0009
completed: 2026-07-13
status_source: operator-confirmed integrated WI (WI-0009 fully closed)
integration_commit: d493f84
aliases:
  - gamePk
depends_on:
blocks:
economic_reason: was -- doubleheaders uncapturable (fail-closed); DELIVERED -- the initiating request now carries exact event identity; capture OPERATION remains separately gated
operator_intent: delivered per WI-0009; corrected under WI-0010 explicit authorization
replan_triggers:
related_work_items:
  - WI-0006
  - WI-0009
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
aliases:
  - first-class
  - identity outcome statuses
depends_on:
blocks:
economic_reason: diagnostic clarity only; today the states are already distinguishable via IdentityReason, so buyer/product value is indirect
operator_intent: named as deferred item 2 at WI-0006 close; re-deferred at WI-0009 close; independent of gamepk-propagation
replan_triggers:
  - a screening consumer misreads unmatched vs ambiguous in practice
related_work_items:
  - WI-0006
  - WI-0009
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

### doubleheader capture operation (operational gate, not an engineering slice)

```yaml
initiative_id: doubleheader-capture-operation
title: doubleheader capture operation under the delivered gamePk capability
status: operational-gated
priority: medium
desired_by:
due_by:
not_before:
proposed_by_system:
date_source: none
date_confidence: low
aliases:
  - capture OPERATION
depends_on:
  - gamepk-propagation
blocks:
economic_reason: the code capability is delivered (WI-0009); realizing paid-product volume from doubleheaders requires an operator capture authorization, which is an operational decision outside the WI system
operator_intent: named in the WI-0009 close NEXT list; requires explicit capture authorization; never a ranked engineering candidate
replan_triggers:
  - operator authorizes capture
related_work_items:
  - WI-0009
```

### wildcard evidence discovery loop (wi-0036)

```yaml
initiative_id: wi-0036-wildcard-evidence-discovery-loop
title: WI-0036 wildcard evidence discovery loop (documentation Slice 1 complete; implementation separately gated)
status: in-progress
priority: medium
desired_by:
due_by:
not_before: PAID wildcard use requires a future explicit operator flight authorization; the separately governed 2026-07-22T12:00:00Z events-gate observation remains its own ungated action (2026-07-21 operator sequencing override -- offline implementation is no longer gated on it)
proposed_by_system: after the 2026-07-22 producer-replay correction (findings M-S) and its final review, coordinated fast-forward integration of the wi/0036-wildcard-capture-flight-plan branches (dai + dai-vault); thereafter the Slice-3 remainder remains the next proposed step
date_source: none
date_confidence: high
aliases:
  - wildcard
  - wildcard evidence discovery
  - signal-need proposal
depends_on:
blocks:
economic_reason: widens safe evidence acquisition toward underrepresented recognized recipes/regimes/signal combinations and turns artifact interrogation into proposal-only retrieval inputs; no spend authority of its own -- paid model calls and sports capture remain fail-closed per the authorization block above
operator_intent: operator decisions 2026-07-21 -- Slice 1 docs authorized/completed/integrated; then a sequencing override authorized the offline/default-off implementation NOW (Slice 2 + minimum Slice-3 seam, delivered on local branches). The July 22 observation is a separate still-unexecuted action; wildcard use in any future PAID flight remains explicitly operator-approved
replan_triggers:
  - the 2026-07-22 events-gate observation completes
  - an operator implementation authorization for WI-0036 Slice 2 is issued
related_work_items:
  - WI-0034
  - WI-0035
  - WI-0036
```

## deferred decisions

- calibration volume expansion: not currently justified (Gate 4 posture; v2 cadence closed
  2026-07-11 with authorization ended) -- revisit only under a new operator authorization.
- cross-sport event-identity abstraction: rejected until a second sport path needs it
  (WI-0006 decision record).

## change log

- 2026-07-22 (WI-0036 Slice 2 producer-replay correction, findings M-S): the wi-0036
  initiative's `proposed_by_system` now names coordinated fast-forward integration after
  the correction and its final review. Full plan validity became exact producer
  re-production; the WI-0036 request/plan/planner/realization/cli contracts bumped to
  `1.1`. No `desired_by`/`due_by` invented; `not_before` is UNCHANGED (paid wildcard use
  still requires a future explicit operator flight authorization and the 2026-07-22
  events-gate observation remains its own separately governed action); the authorization
  posture block is UNCHANGED (paid model calls and sports capture remain fail-closed /
  not-authorized).
- 2026-07-21 (WI-0036 Slice 2, operator-authorized sequencing override): the offline
  wildcard flight-plan implementation was un-gated from the July 22 observation and
  delivered on local branches; the wi-0036 initiative's not_before/proposed_by_system/
  operator_intent updated accordingly. The authorization posture block is UNCHANGED
  (paid model calls and sports capture remain fail-closed / not-authorized); no
  desired_by/due_by invented.
- 2026-07-21 (WI-0036 Slice 1, operator-authorized): wi-0036-wildcard-evidence-discovery-loop
  added. Records the operator sequencing decision that WI-0036 implementation is not before
  the separately governed 2026-07-22 events-gate observation plus a new implementation
  authorization. No desired_by/due_by invented; system suggestion confined to
  proposed_by_system; the authorization posture block is unchanged (paid model calls and
  sports capture remain fail-closed / not-authorized).
- 2026-07-13 (WI-0010): doubleheader-capture-operation added as an operational-gated
  initiative (identity metadata for the WI-0009 deferred bullet; never a ranked candidate).
- 2026-07-13 (WI-0010, operator-authorized manual correction): gamepk-propagation marked
  complete, delivered_by WI-0009 (integration commit d493f84); aliases + WI-0009 relation
  added to both initiatives for deterministic candidate mapping. No desired/due/proposed date
  invented. Tooling never writes this file.
- 2026-07-13: created (WI-0008). Seeded from WI-0006 deferred items and WI-0002/0003 backlog
  state. No operator dates exist yet; all date fields deliberately empty.
