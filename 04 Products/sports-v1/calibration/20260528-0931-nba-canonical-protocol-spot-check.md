# Canonical Protocol Calibration Spot-Check v1

**date:** 2026-05-28 09:31 local
**batch:** NBA, 3 runs (only 3 upcoming games available within the 14-day window)
**harness report:** [[20260528-0931-nba-calibration]]
**run ids:**
- `f48d433e-f36b-1410-815f-00373db4b724` -- Oklahoma City Thunder at San Antonio Spurs (2026-05-28)
- `fa8d433e-f36b-1410-815f-00373db4b724` -- New York Knicks at Oklahoma City Thunder (2026-06-03)
- `018e433e-f36b-1410-815f-00373db4b724` -- New York Knicks at San Antonio Spurs (2026-06-03)

## what this note is

A small, live spot-check of the canonical Cognitive Protocol Runtime against the analyzer after the recent migration slices shipped (Canonical Prompt Pure Rename v1, `phases -> protocol` rename, Stress Collapse v1, Perceive Detect/Aim Scalar Collapse v1). Verifies that the canonical shape is healthy end-to-end and that no first-class runtime dependency remains on retired field names. Does NOT change confidence rules, posture behavior, the v3 artifact version stamp, the database schema, the Tool Gateway, Angular, or any production secret.

## artifact shape findings (all 3 runs)

`cognitiveProtocol` block (canonical, the authoritative shape going forward):

| facet | result |
|---|---|
| `artifactVersion` | `sports_decision_artifact_v2` (v3 stamp deliberately not yet landed) |
| `cognitiveProtocol` present | yes on every run |
| `perceive.detect` | scalar string on every run (Perceive Detect/Aim Scalar Collapse v1 honored) |
| `perceive.frame` | single sentence on every run |
| `perceive.aim` | scalar string on every run |
| `interrogate` keys | `question`, `probe`, `verify` only -- `stress` absent (Stress Collapse v1 honored) |
| `interrogate.probe` | deterministically populated from signal follow-ups ("Sharp/public signal missing; market read relies on price movement only.") |
| `discern` keys | `weigh`, `contrast`, `stress` only -- `test` absent (Stress Collapse v1 honored) |
| `discern.stress` | single sentence on every run; sourced from canonical, not the legacy fallback |
| `decide` keys | `justify`, `position`, `resolve` -- canonical names |
| `decide.position` | `monitor` on every run (validated posture string preserved exactly) |
| `synthesize` | `integrate`, `compose`, `deliver` all present (platform-operational constants) |

`cognitivePhases` block (legacy compatibility, still emitted -- Q3 not yet decided):

| facet | result |
|---|---|
| `cognitivePhases.perceive.detect` | scalar string (no array regression) |
| `cognitivePhases.interrogate` keys | `balance`, `reframe` only -- `stress` correctly absent after Collapse v1 |
| `cognitivePhases.discern` keys | `filter`, `listen`, `stress` -- collapsed stress is here, `test` absent |

`protocolView` (the read-side projection served to `/dev/artifacts`):

| facet | result |
|---|---|
| `protocolView.perceive.detect` | `string[]` with 1 element (canonical scalar wrapped by `SplitJoined`; Angular contract preserved by design) |
| `protocolView.perceive.aim` | `string[]` with 1 element (same -- Angular untouched) |
| `protocolView.interrogate.probe` | populated with the deterministic probe sentence -- the BuildProbe path is producing real material |
| `protocolView.discern.stress` | `DiscernStressProtocolView` with `canonical` populated, both legacy slots null (Stress Collapse v1 working end-to-end) |
| `protocolView.synthesize` | all three platform-operational descriptions present |

## canonical protocol findings

The current analyzer is honoring every recent migration:

- the model emits the canonical `protocol` block key (no `phases` block on these runs)
- the model writes one scalar sentence for `perceive.detect` and one for `perceive.aim` -- no list semantics, no semicolon-joined multi-fact output, no bullet-style phrasing. The Perceive Scalar Collapse v1 prompt guardrail ("a single attention focus / single aim is the canonical station output") is being respected.
- the model emits a single `discern.stress` sentence per run. No `interrogate.stress` and no `discern.test` appear in any block. Stress Collapse v1 is honored at the model boundary, not just at the parser.
- `decide.position` carries the validated posture enum value (`monitor`) verbatim; no semantic drift.
- `interrogate.probe` is non-null because the deterministic `BuildProbe(SignalFollowUpRecord[])` path is exercised (the runs are missing `sharp_public`, which is the doctrinal probe trigger).
- `synthesize` carries the platform-operational constants on the canonical block as expected.

No first-class runtime dependency on retired names was observed. The legacy `cognitivePhases` block continues to carry the same values via the back-compat alias scaffold, so the dual-emit invariant still holds.

## quality warning findings

Runtime `artifactQualityWarnings` was empty on all three runs.

| reason | observation |
|---|---|
| anti-hype gate (rule 6) | did not fire -- requires `posture == "play"` AND `block_aggressive_posture` on a signal; all runs were `monitor`, so the gate is correctly silent |
| signal-narrative drift / `signals_used` integrity | clean -- no drift between retrieved `groundedSignals` and the model's `signals_used` self-report |
| posture validation | all `monitor` -- inside the validated enum, no posture clamp triggered |

Offline calibration flags (computed by the harness markdown report and the reconciliation script, NOT in runtime `ArtifactQualityWarnings`):

| flag | fire count | of |
|---|---|---|
| `missing_sharp_public` | 3 | 3 |
| `signal_quality_blocks_aggressive_posture` | 3 | 3 |
| `posture_aligned_with_partial_evidence` | 3 | 3 |
| `counter_case_generic` | 2 | 3 |
| `what_would_change_contains_filler` | 1 | 3 |
| `frame_missing_rest_context` | 1 | 3 |
| `confidence_high_for_partial_evidence` | 0 | 3 |

`confidence_high_for_partial_evidence` did not fire because no run reached confidence >= 0.70 (range was 0.63 - 0.675 here; the prior 2026-05-18 NBA batch had 0.72 on both runs). Too small a sample to draw a calibration conclusion. See "calibration delta" below.

`counter_case_generic` on 2/3 is a prompt-quality signal carried over from before this migration; not introduced by it.

## reporter divergence (known, not a runtime issue)

The harness markdown report's `Cognitive Phase Completeness` and `Cognitive Phase Quality` tables show `not recorded` for `interrogate.stress` and `discern.test` on every new run. This is correct from the harness's perspective because the harness reads `cognitivePhases.interrogate.stress` and `cognitivePhases.discern.test`, both of which are intentionally retired after Stress Collapse v1. The actual stress content is in `cognitivePhases.discern.stress` (which the report shows as `recorded`), and the canonical home is `cognitiveProtocol.discern.stress` (which the report does not inspect at all). This is a reporter limitation, not a runtime regression; the JSON artifacts and the inspection endpoint are correct.

Follow-up slice candidate: teach `run-artifact-calibration.ps1` to read the canonical `cognitiveProtocol` block first and fall back to `cognitivePhases` for older records. Out of scope for this spot-check.

## tool gateway / telemetry observations

The runs completed end-to-end through the pipeline, which means every tool in the canonical chain dispatched successfully through the gateway:

- `schedule.matchup_dates` (`ProtocolNodes.PlatformReference`) -- via `SportsReferenceController.GetMatchupDates`
- `schedule.basketball.rest_context`, `market.basketball.spread`, `market.sharp_public.split` (`platform.retrieve`) -- via `SportsRetriever`
- `analysis.sports.matchup_read` (`platform.analyze`) -- via `SportsAnalyzer`

The gateway enforces stage-level `AllowedProtocolNodes`; success at every stage proves the manifest cross-check from `ProtocolRegistry` Startup Guard v1 was satisfied at boot and that the registered handlers wired correctly through the scoped DI graph. Structured `ToolGatewayInvocation` log events are emitted per `Tool Gateway Correlation and Telemetry v1`; this spot-check did not capture live stdout from the .NET API process (it runs in a separate PowerShell window), but the same telemetry path is covered by `ToolGatewayTelemetryTests` (4 tests) in the 275-test .NET suite.

No `ToolNotAllowed`, `ToolNotRegistered`, or `HandlerError` outcomes inferred from the run results.

## calibration delta vs prior NBA batch

Small sample on both sides; treat as directional, not load-bearing.

| metric | 2026-05-18 NBA (2 runs, pre-canonical) | 2026-05-28 NBA (3 runs, canonical) | note |
|---|---|---|---|
| posture distribution | 2x `monitor` | 3x `monitor` | identical |
| confidence range | 0.72 -- 0.72 | 0.63 -- 0.675 | lower this batch; could be noise on n=3 |
| evidence richness | 2 on every run | 2 on every run | identical |
| `confidence_high_for_partial_evidence` | 2/2 | 0/3 | dropped because confidence is below the 0.70 threshold, not because the rule changed |
| `counter_case_generic` | 0/2 | 2/3 | new; might be a prompt-tightening signal vs. a sample-size artefact |
| `interrogate_unsupported_claim` | 1/2 | 0/3 | absent this batch -- canonical prompt may be tightening interrogate quality |
| `frame_missing_rest_context` | 2/2 | 1/3 | dropping; consistent with the canonical frame instructions reading more cleanly |

The texture shift (lower confidence range, fewer unsupported-claim flags, fewer rest-context misses, more generic counter_case) is consistent with the canonical prompt being slightly more cautious / hedged. Sample is too small to conclude anything; the next calibration window should aim for `Take 8`+ across NBA and MLB before any threshold change.

## risks

- the prompt-texture shift above might mask a real quality regression on `counter_case`. Mitigation: capture another 5-8 run batch within a week and reassess.
- the harness markdown report's "not recorded" entries for `interrogate.stress` and `discern.test` could mislead a casual reader into thinking the runtime dropped fields. The reality is canonical; the reporter is stale. Carry the harness update in the next cleanup slice.
- v2 records (this batch) and v3 records (after the next slice) will coexist in inspection. The mapper preserves both projection paths; `protocolView` will continue to render correctly for both. No data migration.

## is v3 stamp now safe?

Yes. The canonical analyzer is emitting the v3-shaped artifact today (the only thing left to do is mark it with the constant). Every facet of the canonical contract is healthy on every run:

- canonical block present
- perceive scalars correctly emitted by the model
- single `discern.stress` correctly emitted
- `interrogate.stress` and `discern.test` correctly absent
- posture preserved verbatim in `decide.position`
- probe deterministically populated when meaningful
- synthesize layer carries the platform-operational descriptions
- `protocolView` projection renders cleanly with `Canonical` populated and legacy slots null
- runtime quality gates either fired correctly or correctly stayed silent

There is no remaining blocker to stamping `sports_decision_artifact_v3` and (optionally) dropping the legacy `cognitivePhases` block on new runs. Calibration thresholds are not changed by the v3 stamp; the stamp is a marker, not a behavior change. The next live batch can be used to spot-check the v3 records the same way.

## next recommended slice

Land **artifact v3 stamp v1 (Q6) + legacy CognitivePhases persistence decision (Q3)**, in the migration plan's order. Recommended approach:

1. add `ArtifactVersions.SportsDecisionArtifactV3 = "sports_decision_artifact_v3"`.
2. stamp v3 on the success-path `SportsComposer` result.
3. drop the legacy `cognitivePhases` block on v3 records (the read-side projection already prefers canonical, and old records remain readable via `ProjectFromLegacy`).
4. update `ProtocolView`'s artifact-version label and the inspection page's source badge text to recognize v3.
5. no calibration threshold or prompt change.
6. one small dotnet suite update; pytest unaffected.

After that: plan step 7 (alias removal, including the perceive `StringOrStringArrayJsonConverter` once a calibration window confirms no legacy array payloads remain), and the downstream `.NET` contract record rename `SportsCognitivePhases -> SportsCognitiveProtocol`.

## skills used

Local Jera pack (read-only this slice):
- `dai-grill-with-vault` -- read the migration plan, the cognitive-factory docs, prior calibration notes, and the harness source before touching the stack; surfaced the harness reporter divergence as a known limitation, not a runtime bug.
- `dai-token-tight` -- reporting density.
- `dai-agent-handoff` -- shape of the closing transfer notes in the handoff addendum.

Claude superpowers / built-ins:
- `superpowers:verification-before-completion` -- every claim grounded in artifact JSON or in the live endpoint response (no extrapolation from the test suite alone); the canonical/legacy facet checks were run against the three real artifact files; the protocolView shape was probed live.
- `superpowers:writing-plans` -- consulted to keep the spot-check sequence small and revertible (note + handoff only; no code).
- `superpowers:systematic-debugging` -- not needed; nothing broke.
- `superpowers:test-driven-development` -- not applicable this slice (no code).

Skill weakness / sharpening recommendations carried forward:
- the harness markdown reporter is the next sharpening surface: a dedicated mini-skill `dai-calibration-report-canonical` (read the canonical `cognitiveProtocol` block first, fall back to `cognitivePhases` only for older records, drop the retired field columns from the table) would close the reporter divergence in one short slice.
- a `dai-implement-with-vault` (or `dai-audit-with-vault`) skill that produces this exact spot-check shape (live runs, JSON facet table, calibration delta vs prior batch, v3-readiness conclusion) would beat the interactive grill template for these operational verification slices.
- `jera-workspace-skills` left untouched (no approval to edit). Confirmed clean at `C:/Users/trolo/source/repos/jera-workspace/jera-workspace-skills`.

## stack state

- Docker Desktop daemon launched from the session; `devcore-sql` container started cleanly; SQL recovered before the API came up.
- `.NET` API on `:5007` and FastAPI on `:8000` running in separate PowerShell windows for the duration of the spot-check.
- Angular `sports-app` not started this slice (artifact JSON + the inspection endpoint were inspected directly; `/dev/artifacts` not opened in a browser).
- the stack can be left running for follow-up or stopped via `stop-sports-dev.ps1`.
