---
title: "WI-0037 Slice 2-iii-c Frontend and Consumer Selected-Event Continuity 2026-07-27 v1"
type: "evidence-report"
date: "2026-07-27"
status: "in-progress"
project: "DAI"
slice: "WI-0037 Slice 2-iii-c: frontend and consumer selected-event continuity"
repos:
  dai: "code (IMPLEMENTED LOCAL, review-required: branch wi/0037-selected-event-frontend-continuity from aee1ade; NOT integrated, NOT pushed)"
  dai-vault: "docs-only (this report, WI update, current-slice)"
tags:
  - system-development
  - sports-v1
  - identity
  - frontend
related:
  - "02 Platform/system-development/work-items/WI-0037-game-state-correctness-v1.md"
  - "06 Execution/reports/wi-0037-slice-2-iii-architecture-review-2026-07-26-v1.md"
  - "06 Execution/reports/wi-0037-slice-2-iii-b2-backend-activation-2026-07-27-v1.md"
---

# wi-0037 slice 2-iii-c frontend and consumer selected-event continuity

## purpose

Close the client-side identity-loss line so the operator's explicitly selected provider
event crosses the sports-app request boundary and reaches the published server-owned
selected-event authority (Slice 2-iii-b2), without changing any backend contract. Both
sports-app run-creation paths already held the selected `MatchupEventDto.providerEventId`
and `MatchupEventDto.startUtc` but discarded both when building `CompetitionMatchupInput`.
This slice carries them through as the additive null-suppressed `selectedEvent` intent,
adds one shared typed selected-event refusal parser/presenter, and surfaces the
server-recorded game identity on the internal dev inspection surface only.

Terminal state: **WI0037_SLICE2IIIC_IMPLEMENTED_LOCAL_REVIEW_REQUIRED**. Frontend stays
intent-only; the backend remains the sole verification and execution authority. No C#
production or test change; activation stays DISABLED; migration 20260727133845 stays
UNAPPLIED; nothing pushed or integrated.

## opening state and ancestry

- dai local main = origin/main = direct remote main = `aee1ade8a27da45d845510baffaabca7068be974` (clean).
- implementation branch `wi/0037-selected-event-frontend-continuity` from `aee1ade` (worktree dai-2iiic).
- dai-vault main = origin/main = direct remote main = `e217a3f0c5d197c06fd942eab4fc3bff98fd988f`.
- records branch `wi/0037-selected-event-frontend-continuity-records` from `e217a3f` (worktree dv-2iiic).
- architecture branch local-only at `d78545c`; ops at `21f532d`; both unchanged.
- appsettings.json: `SelectedEventActivation.Enabled=false`, blank DeploymentId, blank EvidencePath.
- migration `20260727133845_AddAgentRunDomainExecutionProvenance` present in source, UNAPPLIED.
- neither frontend-continuity branch pre-existed locally or remotely.

## skills and path gates

- dai-skill-router (skills gate), dai-slice-runner (lifecycle), dai-test-discipline
  (verification ladder), dai-typescript-angular-quality (TS boundaries), dai-code-reviewer
  (pre-completion review), dai-docs-architect (this record), prompt-ledger (pre-execution
  capture). Path-resolution gate run: `<DAI_WORKSPACE_ROOT>`, `<DAI_REPO_ROOT>`,
  `<DAI_VAULT_ROOT>`, `<OBSIDIAN_PROMPT_LEDGER_ROOT>` resolved from the tracked template
  plus `.local/agent-paths.md`; both repo roots verified with `git rev-parse --show-toplevel`.
- Graphify: not used (optional navigation only).

## baseline characterization and genuine RED

Two focused specs were written to require the `selectedEvent` block and run against the
unmodified component behavior. The failure output is the characterization evidence: the
current analyzer and batch request bodies serialize with `selectedEvent` absent.

- `analyzer/selected-event-propagation.spec.ts`: analyzer POST asserted to carry
  `selectedEvent`; RED = 3 failures (block absent), while orientation and no-event-guard
  assertions passed unchanged.
- `dev-artifact-review/selected-event-batch-propagation.spec.ts`: batch POST asserted to
  carry one distinct `selectedEvent` per selected provider event and to retain a refusal
  code + message + agentRunId; RED = 4 failures (blocks `[undefined, undefined]`; refusal
  fields undefined; agentRunId not retained).
- RED run total (pre-implementation): 7 failed / 2 passed. The passing pair
  (orientation preserved; no post without a selected event) confirms unchanged behavior
  was not perturbed. The additive DTO type members (`SelectedEventIntent`,
  `CompetitionMatchupInput.selectedEvent`, `BatchRunEntry.refusalCode/refusalMessage`,
  the six `AgentRunArtifactDto` identity fields) were added first as an inert contract so
  the specs compile; the RED came from the still-unmodified component logic.

## changed files (all under apps/sports-app; production limited to sports-app)

Production:
- `core/models/agent-run.model.ts` -- add `SelectedEventIntent`; add optional
  `selectedEvent` on `CompetitionMatchupInput` (null-suppressed legacy byte-identity);
  update the stale MatchupEventDto residual comment; mirror the six read-only identity
  fields on `AgentRunArtifactDto`; extend `BatchRunEntry` with `refusalCode`/`refusalMessage`.
- `core/selected-event-refusal.ts` (new) -- the shared parser/presenter: closed
  16-code vocabulary, `selection_` discriminator, phase derived from agentRunId presence,
  per-code safe operator message, unknown-code preservation, non-selection null, no server
  detail exposure.
- `analyzer/analyzer.component.ts` -- add the `selectedEvent` block in `analyze()`;
  map refusals via the shared parser in the error handler (`describeSelectedRefusal`);
  add `ANALYSIS_FAILED_MESSAGE` generic fallback; remove the now-unused `getErrorMessage`.
- `analyzer/analyzer.component.html` -- render `{{ error() }}` instead of a fixed sentence.
- `dev-artifact-review/dev-artifact-review.component.ts` -- add the `selectedEvent` block
  in `runSelectedGames()`; retain refusal code/message/agentRunId per entry in the catch;
  add the read-only `serverRecordedIdentity` computed.
- `dev-artifact-review/dev-artifact-review.component.html` -- per-entry refusal message on
  failure; the "Server-recorded game identity" panel (complete / partial / not-recorded).
- `dev-artifact-review/server-recorded-identity.ts` (new) -- the pure identity classifier.

Tests (new): `analyzer/selected-event-propagation.spec.ts`,
`dev-artifact-review/selected-event-batch-propagation.spec.ts`,
`core/selected-event-refusal.spec.ts`, `core/selected-event-serialization.spec.ts`,
`dev-artifact-review/server-recorded-identity.spec.ts`.

## request-contract examples

Legacy input (no selectedEvent) -- byte-identical to the pre-2-iii-c shape:

```json
{"competition":"mlb","homeTeam":"Rays","awayTeam":"Jays","gameDate":"2026-07-27"}
```

Selected input -- additive block, providerEventId and startUtc byte-exact from the
selected MatchupEventDto (case-sensitive, never normalized):

```json
{"competition":"mlb","homeTeam":"Rays","awayTeam":"Jays","gameDate":"2026-07-27","selectedEvent":{"providerEventId":"Odds-EVT-Dh-1","startUtc":"2026-07-27T17:05:00.0000000Z"}}
```

Proven by `core/selected-event-serialization.spec.ts` (exact-keys, legacy byte string,
never-null, case-sensitivity) and the SportsApiService passthrough test (body posted
unchanged).

## refusal-code-to-message matrix

The parser recognizes the published closed vocabulary (backend `SelectedEventRefusals`).
Phase is derived at runtime from the presence of `agentRunId` (post-create) vs absence
(pre-create); the message is the safe operator text (server `detail` is never surfaced).

| code | phase (controller) | operator message (summary) |
|---|---|---|
| selection_intent_malformed | pre-create (400) | could not be sent; refresh and reselect |
| selection_identity_not_active | pre-create (422) | verified selected execution inactive here; no legacy substituted |
| selection_competition_unsupported | pre-create (422) | verified selection unavailable for this competition |
| selection_provider_source_failure | pre-create (422) | verification unavailable (provider); game not rejected |
| selection_verification_source_failure | pre-create (422) | verification unavailable (source); game not rejected |
| selection_provider_identity_conflict | pre-create (422) | selection disagreed with server identity; refresh/reselect |
| selection_event_not_in_schedule | pre-create (422) | event no longer in schedule; refresh/reselect |
| selection_start_mismatch | pre-create (422) | scheduled start changed; refresh/reselect |
| selection_identity_mismatch | pre-create (422) | selection disagreed with server identity; refresh/reselect |
| selection_ambiguous_candidates | pre-create (422) | could not match to a single game; refresh/reselect |
| selection_gamepk_conflict | pre-create (422) | selection disagreed with server identity; refresh/reselect |
| selection_game_status_refused | pre-create (422) | authoritative game status did not permit analysis |
| selection_provenance_invalid | pre-create (422) | server-side identity evidence could not be established |
| selection_binding_conflict | pre-create (422) | selection disagreed with server identity; refresh/reselect |
| selection_execution_identity_mismatch | post-create (422 + agentRunId) | run stopped before model analysis; executed identity != selection |
| selection_execution_verification_failed | post-create (422 + agentRunId) | run stopped before model analysis; verification failed |

Unknown `selection_*` values still produce a truthful selected-verification failure with
the raw code preserved for inspection; they never become success, legacy retry, or silent
fallback. Non-selection errors (e.g. `invalid gamePk`, `duplicate_candidate_identity_invalid`,
network) return null from the parser and keep existing fallback handling. Proof: one test
per code plus extraction/unknown/fallback/non-disclosure in
`core/selected-event-refusal.spec.ts`.

## analyzer and batch propagation evidence

- Analyzer (`analyze()`): the existing no-event guard is unchanged; the exact selected
  `ev` supplies `selectedEvent.providerEventId`/`startUtc`; competition, server-resolved
  home/away, and gameDate are unchanged; no array position/labels/date+teams used as
  identity; no normalization or reserialization; a selected request is never retried as a
  legacy request; refusals render the mapped message.
- Batch (`runSelectedGames()`): providerEventId remains the selection/tracking/cardinality
  key; each entry emits `selectedEvent` from its own game object; sequential execution and
  the BATCH_MAX (10) provider-event cap preserved; one failed entry records its refusal
  without mutating another; a returned agentRunId is retained on a post-create refusal;
  no failed entry is retried without selectedEvent.

## doubleheader cross-layer traceability

1. sports-app: two doubleheader selections -> two distinct `selectedEvent` intents
   (distinct providerEventId + startUtc) -- proven by the analyzer and batch propagation
   specs (occurrence 1 vs 2; two same-date/same-team entries).
2. published `SelectedEventCreationTests` (56/56 in the focused run): two provider events
   -> two verified gamePks -> two runs (server-side, unchanged).
3. published persistence assertions: each selected run stores the complete row identity and
   provenance atomically (unchanged).
4. published `OutcomeReconciliationMatcherTests`: reconciliation matches on exact
   (SourceProvider, ExternalGameId) (unchanged).

No C# file changed to manufacture this chain; the frontend now supplies the distinct
identity that the published backend chain already preserves.

## read-only identity presentation and its honesty bound

`server-recorded-identity.ts` classifies the six stable identity fields the artifact
endpoint already returns into complete / partial / not-recorded. The dev panel is labelled
"Server-recorded game identity" -- never selected-event provenance, because this read model
does not distinguish a selected run from any other identity-bearing run. externalGameId is
labelled neutrally (not gamePk); the raw DomainExecutionProvenanceJson is never read or
parsed in TypeScript; complete shows all six facts; not-recorded shows an honest legacy /
unmatched state; partial shows an incomplete-record warning rather than inventing values;
buyer routes and buyer copy are unchanged.

## caller and stub inventory

- Exactly two production run-creation callers: `AnalyzerComponent.analyze` and
  `DevArtifactReviewComponent.runSelectedGames` (through `SportsApiService`); both now emit
  `selectedEvent`. Verified by repo scan for `createMatchupAnalysis` and POST `/api/agent-runs`.
- `SportsApiService.createMatchupAnalysis` passes the request body unchanged on the live
  HTTP path (passthrough test).
- `generateStubEvents` yields deterministic, distinct providerEventId/startUtc (day-3 stub
  doubleheader); the stub api response claims no server verification.
- `auth-config.ts` only registers the `/api/agent-runs` URL for MSAL protection (not a caller).
- No alternate identity-blind caller; operator scripts do not create runs through this UI contract.

## tests and build

- Focused new specs: `npx ng test --no-watch --include="src/app/**/selected-event-*.spec.ts"
  --include="src/app/**/server-recorded-identity.spec.ts"` -> 38 passed (5 files).
- Full frontend: `npx ng test --no-watch` -> 195 passed (22 files) (baseline 157/17 + 38/5).
  `npx ng build` -> success (0 errors).
- Focused backend regression (no backend diff): `SelectedEventCreationTests |
  OutcomeReconciliationMatcherTests` -> 56 passed, 0 failed.
- Full .NET: `dotnet test DevCore.Api.Tests.csproj --no-restore -v minimal` -> **2025
  passed, 0 skipped, 0 failed** (exact published baseline; zero backend impact).
- Solution build: `dotnet build DevCore.Api.slnx --no-restore -v minimal` -> 0 errors, 12
  warnings (all pre-existing).
- Repo checks: `git diff --check` clean in both worktrees; source production changes
  confined to apps/sports-app; no platform/dotnet file changed; no dependency manifest or
  lockfile changed; no appsettings/migration/deployment/auth/billing/settlement/
  reconciliation file changed; no graphify-out/local/generated/secret/machine-path in the
  diff; both createMatchupAnalysis call sites emit selectedEvent; no client gamePk
  derivation or selected-to-legacy retry (only negation comments).
- Vault: strict snapshot against published vault main (e217a3f) = **26 work items, 0
  warnings** (the default run reported 25 only because it read the preserved WI-0035
  checkout at de5791f, which has 25 work-item files and predates WI-0037 registration on
  main; explained, not a change from this slice). Links resolve.

## review findings

| id | severity | file/location | behavior/risk | disposition | proof |
|---|---|---|---|---|---|
| FC-1 | high-visibility (by-design, not a defect) | analyzer + dev batch create paths | once integrated, both paths always send selectedEvent; in any environment with SelectedEventActivation disabled (the default, incl. production and dev) the backend refuses with selection_identity_not_active and no legacy analysis is substituted | authorized fail-closed design (prompt 6.2/6.4; residual requires client identity); surfaced truthfully via the mapped message; slice is local/review-required/NOT integrated; enabling for buyers is a separate activation + integration decision | selected-event-refusal.spec (identity_not_active message); propagation specs prove the block is always sent |
| FC-2 | low | analyzer.component.html error block | changed from a fixed generic sentence to `{{ error() }}`; load-failure strings now render their own (safe) text in that block | intended improvement; all error() values are safe operator literals or the mapped refusal; no raw HTTP/exception text is ever rendered | verified every `this.error.set(...)` call site |
| FC-3 | low (pre-existing) | analyzer.component.ts | `analyze()` and the new `describeSelectedRefusal` sit at column-0 indentation | pre-existing quirk (analyze was already column-0); out of scope to reflow; build + full suite green | ng build + 195/195 |

No blocking findings. Verdict: approve with notes (FC-1 recorded for the operator's
integration decision).

## docs-impact declaration

Frontend request contract gains an additive, null-suppressed `selectedEvent` intent and a
dev-only read-only identity panel; no backend contract, buyer contract, or public API
changes. Docs updated: this report; WI-0037 status/history; current-slice + Slice Synopsis;
external prompt-ledger record. No doctrine promotion; the WI-level glossary disposition pass
is deliberately NOT performed in this slice.

## no-live / no-paid / no-migration / no-activation evidence

No live Odds API / StatsAPI / model / Tool Gateway / paid call; no database mutation; no
migration executed (20260727133845 remains UNAPPLIED); no capture/reconciliation/settlement/
artifact generation; activation remains DISABLED (`SelectedEventActivation.Enabled=false`,
blank DeploymentId/EvidencePath). All HTTP in tests is answered by HttpTestingController.

## preservation manifest comparison

Preserved WI-0035 checkout (`dai-vault`, branch wi/0035-provider-event-binding) unchanged:
HEAD `de5791f1fdc6a5261ce3e710e0f7f0aff975abeb`; same six dirty paths; diff fingerprint
`86aa8b74a3ec5d0856d7e043cff24241c54b69f7` (git diff | git hash-object --stdin). Before-state
SHA-256 manifest of the six paths captured at opening and re-verified at close (see the
operator report / ledger). Architecture (d78545c), ops (21f532d), and the foundation/b2
worktrees were not touched.

## residual dispositions

- SELECTED_EVENT_IDENTITY_PROPAGATION_REQUIRED_BEFORE_WI0037_CLOSE:
  **RESOLUTION-PENDING-VERIFICATION** (client identity-loss line closed locally; resolution
  requires independent review, integration, publication, and verification -- not resolved).
- MULTI_INSTANCE_SELECTED_EVENT_ATOMICITY_REQUIRED_BEFORE_SCALE_OUT: unchanged, blocking.
- DURABLE_PREEXECUTION_SELECTION_DECISION_LEDGER_DEFERRED: unchanged, nonblocking.
- 2-ii-c remains unauthorized; WI-0037 stays `in-progress`.

## slice 2-iii-c1 correction and evidence hardening (2026-07-27)

This section records a narrow correction/evidence-hardening pass on top of the Slice
2-iii-c source commit d5d093c and vault records commit 9cee53b. The correction is source
commit `baf5e9088aff2a23453dbdc4f3a33d5ced2b9188` on top of d5d093c; this records update is
a new vault commit on top of 9cee53b. It preserves all accepted 2-iii-c behavior and settled
decisions; no runtime semantics changed.

### disclosed defect: comment-rule violation in the 2-iii-c delta

The original 2-iii-c commit d5d093c added production and test comments that carried
work-item/slice/finding labels (for example "wi-0037 2-iii-c:", "(RED-first)") and
uppercase symbol names in prose (for example matchup-event, request-input, and refusal
type names written in their code casing). This violates the standing code standard "keep
code comments lowercase and ascii only" and repeats the low-severity comment-hygiene class
the 2-iii-b2 final review already flagged. The miss is disclosed here rather than erased
from history.

Correction: every comment added by the branch delta (aee1ade through the 2-iii-c1
correction commit) was rewritten to lowercase ascii prose with no work-item, slice,
architecture, or finding labels, and with uppercase type/symbol names rewritten as ordinary
lowercase prose. pre-existing comments outside the branch delta were left untouched. an
end-of-slice audit confirms zero labels and zero uppercase or non-ascii characters in
comments added by the branch. no executable behavior was altered by the comment rewrite;
the focused frontend suite stayed green across the change.

### added consumer-boundary and template evidence

New focused tests pin the actual consumer behavior that the first pass proved only at the
request-body level:

- analyzer/selected-event-refusal-boundary.spec.ts (new): the analyzer's real
  request/error path renders the safe message for selection_identity_not_active, never
  surfaces the server detail string, sends exactly one selected request and never retries
  it, keeps the existing generic fallback for a non-selection error, presents the short run
  identifier for a post-create refusal, and (non-baseball) shows the unsupported-competition
  message with no retry.
- dev-artifact-review/server-recorded-identity-panel.spec.ts (new): the rendered dev
  inspection panel produces the complete, partial (incomplete-record warning, no invented
  values), and not-recorded states; the visible label stays "server-recorded game
  identity"; the surface never labels the data as selected-event provenance; and the
  identity panel never renders an out-of-contract raw provenance value placed on the row.
- non-baseball propagation was added to both propagation specs: an nba selection still emits
  the exact provider event id and start utc on both run-creation paths, and intent is not
  silently dropped by sport.

Note on the raw-artifact debug dump: the dev page also has a pre-existing "raw artifact"
json section that renders the whole artifact dto verbatim. the artifact contract carries no
raw execution provenance envelope field (only the six stable identity fields were mirrored
in this slice), so a real row never presents one there; the panel test scopes its
"not rendered" assertion to the identity panel element specifically.

### activation-off and non-baseball operational envelope

The staged backend execution envelope is baseball-only and settled for this slice; the
frontend adds no sport allowlist, no fallback, and no backend support. The deliberate
fail-closed posture is:

- with activation off (the default, including current production and dev), any selected
  request refuses with selection_identity_not_active and no legacy analysis is substituted;
- after activation, a competition outside the staged baseball backend still refuses as
  unsupported (selection_competition_unsupported);
- this posture is intentional and requires deployment/release review before any publication
  or activation. it is not authorization to add fallback or broaden backend support.

### corrected publication sequence (supersedes any same-commit fast-forward instruction)

Publication must NOT fast-forward the current vault records commit unchanged after source
publication. The required later sequence is:

1. one independent read-only review of the complete source and records package;
2. if review passes, publish and verify the SOURCE first;
3. create a NEW docs-only closeout commit atop the vault topic branch that reflects the
   verified published source (its integrated main sha);
4. publish and verify the VAULT second;
5. update the propagation residual only to an existing canonical resolved state supported
   by the work-item lifecycle -- do not invent a new state name.

Until then the package stays local, unintegrated, and unpushed; WI-0037 stays in progress;
the propagation residual stays RESOLUTION-PENDING-VERIFICATION; the scale-out and
durable-ledger residuals are unchanged.

### 2-iii-c1 verification

Frontend focused selected-event/identity specs 50/50; full frontend 207/207 (24 files) +
build clean; full .NET 2025/2025, 0 skipped (no backend diff); git diff --check clean;
branch-delta comment audit clean (zero labels, zero uppercase/non-ascii in added comments).
Source production changes remain confined to apps/sports-app; no platform/dotnet, lockfile,
manifest, appsettings, or migration change. activation remains disabled; migration remains
unapplied; nothing pushed or integrated.
