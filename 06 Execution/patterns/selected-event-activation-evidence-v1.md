---
title: "Selected-Event Activation Deployment Evidence v1"
type: "execution-pattern"
date: "2026-07-27"
status: "complete"
project: "DAI"
slice: "WI-0037 Slice 2-iii-b2: verified selected-event backend activation"
repos:
  dai: "code (wi/0037-selected-event-backend-activation; activation DEFAULT-OFF)"
  dai-vault: "docs-only"
tags:
  - system-development
  - sports-v1
  - operations
related:
  - "02 Platform/system-development/work-items/WI-0037-game-state-correctness-v1.md"
  - "06 Execution/reports/wi-0037-slice-2-iii-b2-backend-activation-2026-07-27-v1.md"
---

# selected-event activation deployment evidence v1

## purpose

The operator process for producing, validating, expiring, revoking, and auditing the
EXTERNAL deployment-evidence record that selected-event run creation requires before it
can activate. The shared creation gate is process-local, so activation is safe only when
exactly one run-creating API process exists -- a topology fact the application cannot
prove about itself. This evidence record is how the operator proves it. Production
default is DISABLED; nothing in this pattern activates anything.

## mental model

The application asserts nothing about deployment topology. The operator observes the
real deployment, writes one signed-by-citation evidence record, and points configuration
at it. The gate activates only while the record is complete, current, and names exactly
this deployment. Remove or expire the record and the feature turns itself off.

## producing the record

Write one json file OUTSIDE the repository (the path is deployment configuration):

```json
{
  "deploymentId": "<the running deployment/revision identity>",
  "observedAtUtc": "<when the topology was actually observed>",
  "expiresUtc": "<must be after observedAtUtc; window capped at 24 hours>",
  "deploymentConfigurationCitation": {
    "artifact": "<the config artifact, ex: compose.prod.yaml>",
    "reference": "<digest/revision/detail an audit re-checks>"
  },
  "processObservationCitation": {
    "artifact": "<the process/revision listing artifact>",
    "reference": "<the concrete observation detail>"
  },
  "topology": {
    "singleRunCreatingProcess": true,
    "oneWorkerPerProcess": true,
    "noWebGarden": true,
    "noOverlappingRecycle": true,
    "noRevisionOverlap": true,
    "noMixedVersionPool": true,
    "noSecondManualProcess": true,
    "noAlternateRunCreator": true,
    "oldProcessStoppedBeforeNewAccepts": true,
    "provenanceMigrationPresent": true
  }
}
```

Every topology member must be an explicit observation, not a hope: one active
run-creating API process; one worker per process; no web garden; no overlapping
app-pool recycle; no old/new revision overlap receiving run-creation traffic; no
blue/green or mixed-version backend pool; no second manually started API process; no
background worker or alternate service capable of run creation; the old process stopped
or unable to accept run creation before the new enforcement-capable process accepts
selected requests; the domain-provenance migration applied to the target database.
Citations are STRUCTURED, not prose: each carries a bounded `artifact` identifier and
a bounded `reference` (digest, revision, timestamp, or listing detail) that a later
audit re-checks. Unstructured or partial citations refuse.

## validating

Configuration binds `SelectedEventActivation` (`Enabled`, `DeploymentId`,
`EvidencePath`). The gate (`SelectedEventActivationEvidence.Evaluate`) refuses --
keeping the feature off -- when the flag is off, the deployment identity is unset or
mismatched, the record is missing/malformed/expired, a citation is absent or
unstructured, or ANY topology assertion is missing or false. Freshness is
semantically enforced (b2 correction F-B2-4): the observation instant may not sit in
the future beyond 5 minutes of clock skew, may not be older than 24 hours, the expiry
must be strictly after the observation, and the total validity window is capped at
24 hours -- evidence can never be made effectively permanent, so activation always
rests on a topology observation made within the last day. Every requirement is
individually pinned by `SelectedEventActivationTests`.

## expiring and revoking

- expiry: `expiresUtc` bounds validity within the 24-hour maximum window; renewal
  requires a FRESH observation, never an edit of the old timestamps.
- revoke immediately by deleting the record or setting `Enabled` to false -- both take
  effect on the next request; no restart or deploy is required for revocation.
- any deployment change (scale-out, revision overlap, new worker topology) INVALIDATES
  the observation: revoke first, re-observe after the change settles.

## auditing

Keep superseded evidence records with their observation timestamps; each activation
window should be reconstructable as (record, validity interval, deployment identity).
The record never contains secrets and is never client-supplied.

## safety / non-actions

This pattern creates no production evidence and enables nothing. The tracked
configuration ships `Enabled: false` with blank identity and path. Scale-out remains
blocked by MULTI_INSTANCE_SELECTED_EVENT_ATOMICITY_REQUIRED_BEFORE_SCALE_OUT
regardless of this record -- the evidence proves single-process topology, it does not
create multi-process safety.

## next step

None until an operator prepares a real deployment: then produce the record per this
pattern, configure the three settings, and verify the gate reports active before the
first selected request.

## related

- [[WI-0037-game-state-correctness-v1]] -- governing work item and residual states.
- [[wi-0037-slice-2-iii-b2-backend-activation-2026-07-27-v1]] -- the implementation this
  pattern operates.
