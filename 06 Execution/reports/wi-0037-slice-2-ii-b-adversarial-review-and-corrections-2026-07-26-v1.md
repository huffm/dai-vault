---
title: "WI-0037 Slice 2-ii-b Adversarial Review and Corrections 2026-07-26 v1"
type: "evidence-report"
date: "2026-07-26"
status: "corrections applied locally -- delta review required; NOT integrated, NOT pushed"
project: "DAI"
slice: "WI-0037 Slice 2-ii-b: correction pass for review findings F-A..F-G"
repos:
  dai: "corrections on local branch wi/0037-game-state-correctness-slice-2-ii-b: 5a11a2c -> c5ab834 -> c545117 (base 841ae26); NOT integrated"
  dai-vault: "docs on the matching local branch (base 664cd4a -> fd2d20b -> this record); NOT pushed"
tags:
  - system-development
  - sports-v1
  - correctness
  - review
  - corrections
related:
  - "02 Platform/system-development/work-items/WI-0037-game-state-correctness-v1.md"
  - "06 Execution/reports/wi-0037-slice-2-ii-b-discovery-identity-2026-07-26-v1.md"
---

# wi-0037 slice 2-ii-b adversarial review and corrections 2026-07-26 v1

## part 1 -- durable record of the independent adversarial review

The independent adversarial review of the 2-ii-b chain (dai `5a11a2c`, vault
`fd2d20b`) executed 2026-07-26 and returned
**WI0037_SLICE2IIB_REVIEW_CORRECTIONS_REQUIRED**. Method: live opening-gate
verification, full base-to-tip diff inspection, independent production-path
reconstruction, a 16-area mandatory attack matrix, 17 temporary C# probes through
the real HTTP seam plus 4 TestBed runtime-DOM probes (all removed; both trees
proven byte-identical to their tips afterward), and post-probe full-suite reruns
(.NET 1831/1831, operator 187/187, guard 40/40, frontend 136/136 + build). The
operative pre-correction status of the chain is the review verdict; the earlier
formal status IMPLEMENTED_LOCAL_REVIEW_REQUIRED is superseded.

Findings (severity, location, essence, machine evidence):

- **F-A (High)** `dev-artifact-review.component.ts:414` (+ template :175/:228):
  `gameKey = date:home:away` aliased legitimate doubleheader pairs the corrected
  sampler path now delivers -- one checkbox toggled both games, `selectedCount`
  showed 1 while the paid batch filter matched 2 (duplicate `@for` track keys in
  two lists; BATCH_MAX capped keys, not runs). Probe: TestBed component instance,
  `gameKey(dh1)===gameKey(dh2)`, one toggle selected both, 1-count filter matched 2.
- **F-B (Medium)** `OddsScheduleClient.cs` pair path: the team filter ran BEFORE
  `NormalizeEvents`, so a same-id conflict on a team field whose other variant
  failed the filter escaped the whole-id exclusion (sampler excluded the same
  input). Probe: pair path emitted 1 row, sampler 0 rows, identical input.
- **F-C (Medium)** StartUtc: fractional-second commence values were accepted by
  the deserialization boundary and silently truncated by the fixed
  `yyyy-MM-ddTHH:mm:ssZ` format, contradicting the recorded "exact provider
  commence instant" claim; no enforceable whole-second guarantee existed. Probe:
  `.500Z` and `.1234567Z` inputs both emitted `...:00Z`.
- **F-D (Medium)** analyzer labels: two distinct provider ids with identical
  date/teams/StartUtc rendered byte-identical pills (selection stayed id-correct).
  Probe: runtime DOM, two pills with equal text.
- **F-E (Low)** missing `commence_time` bound default(DateTimeOffset) and emitted
  an accepted event with sentinel StartUtc `0001-01-01T00:00:00Z`. Probe: event
  present with year-one StartUtc.
- **F-F (Low)** "stay distinguishable end to end" overstated: the analysis-request
  payload carries competition/homeTeam/awayTeam/gameDate only, so selected
  provider identity terminates before analysis-request serialization
  (`analyzer.component.ts` analyze payload; `dev-artifact-review.component.ts`
  batch payload).
- **F-G (Low)** the duplicate-batch test asserted `SourceFailure` non-null, not
  the typed reason.

Review cleanliness: probes removed, both review trees byte-identical to
`5a11a2c`/`fd2d20b`, mains unchanged, ops branch untouched, wi/0035 hash
`86aa8b74` re-verified.

## part 2 -- correction pass (this turn)

Opening gate 2026-07-26 20:16Z: dai main == origin == direct remote == `841ae26`;
vault main == `664cd4a`; source tip exactly `5a11a2c` (1 unique, clean, local
only, no upstream, zero 0037 remote refs); vault tip exactly `fd2d20b` (same
checks); ops branch `664cd4a` clean 0-unique; wi/0035 six dirty paths, hash
`86aa8b74` (re-verified at close). No concurrent-session evidence.

RED preserved BEFORE any production edit (scratchpad `2iib-corr-red-backend.txt`,
`2iib-corr-red-frontend.txt`): backend 18/20 new integrity tests failing
(normalize-before-filter, fixed-width precision, fraction preservation,
typed-instant ordering, missing/malformed commence, mixed-group survival);
frontend 9 failing (all 8 dev-artifact-review selection-identity tests + the
identical-instant label test).

### f-a -- dev-artifact-review identity (corrected in c545117)

`gameKey` now returns `providerEventId`. Because selection membership, toggling,
`selectedCount`, both `@for track` expressions, the `runSelectedGames` filter, and
BATCH_MAX all flow through `gameKey`/`selectedGameKeys`, the invariant holds
structurally: one selected providerEventId = one selected game candidate = one
intended execution entry. Date, teams, date:home:away, and array index are no
longer identities anywhere in the surface. Pinned (8 tests in
`matchup-selection-identity.spec.ts`): distinct keys + unique track keys,
independent selection either way, count/filter parity at 1 and 2, independent
deselection, BATCH_MAX enforced against provider-event count (DH pair counts as
two runs), cardinality parity, reversed source order. The analysis-request payload
was NOT changed.

### f-b -- integrity before business filtering (corrected in c5ab834)

Both production paths now share the fixed order: raw records -> id validation ->
identity-field validation -> group by id -> whole-id integrity decision ->
canonical projection -> deterministic ordering -> (pair path only) requested-team
filtering -> output. `FetchEventsAsync` calls `NormalizeEvents` on the FULL
response and filters the normalized DTOs afterward (order-stable Where). Pinned:
team-field conflict excluded on BOTH paths in BOTH source orders; a clean id
adjacent to a conflicted id survives on both paths.

### group-level malformed-record policy (c5ab834)

A nonblank provider id defines an integrity group over identity-bearing fields
(id, home, away, commence instant). All-equivalent -> coalesce to one; any
conflicting value -> whole group excluded; any member with missing/malformed
commence -> whole group excluded (a valid member of a malformed group never
survives -- pinned in both source orders with an adjacent clean id surviving);
missing/blank id -> the unidentifiable record alone is omitted, never synthesized,
never date/team-deduped. Structured warn logs carry id/teams/commence context
only -- no payload dumps, no urls, no credentials.

### f-c -- temporal precision (corrected in c5ab834)

Binding decision: **STARTUTC_FIXED_WIDTH_UTC_100NS**. `commence_time` binds as the
RAW string; a strict parser requires an explicit utc designator or numeric offset
(an offset-less value would assume the machine timezone -- nondeterministic across
hosts -- so it is malformed, never an invented timezone) and at most seven
fractional digits (.net tick, 100ns); anything unparseable is malformed (group
excluded). StartUtc serializes invariant-culture fixed-width
`yyyy-MM-ddTHH:mm:ss.fffffffZ`, zero-padded, preserving the instant to tick
precision; excess precision fails the group closed rather than truncating.
Ordering and equivalence compare the TYPED utc instant, never the serialized
string; providerEventId (ordinal) is the final tie-breaker. Pinned: no-fraction,
1/3/7-digit fractions, 8-digit fail-closed, offset normalization, offset-equivalent
coalescing, sub-second chronological ordering under both source orders. Eastern
Date semantics unchanged. C#/TS comments, stubs, and fixtures updated to the
fixed-width form. Side effect (documented): a malformed commence VALUE is now a
deterministic per-group exclusion instead of a whole-response transport failure
(the raw-string binding moved that decision out of the deserializer); structurally
invalid JSON documents remain transport failures.

### f-d -- deterministic occurrence labels (corrected in c545117)

One shared pure helper `core/matchup-event-labels.ts`
(`matchupEventDiscriminators(events, baseLabel)`) consumed by the analyzer pills
and BOTH dev-artifact-review lists (game selection and run progress): single event
in its date/team grouping -> plain label; shared date -> localized start-time
suffix; still label-identical (identical instants, or base-label collisions such
as swapped orientation under a date-only base) -> deterministic ` · Game N`
ordinals assigned AFTER canonical ordering by date, parsed start instant, then
providerEventId (ordinal) -- never source index, never the raw provider id as
display text. Pinned by a single shared test-vector source
(`matchup-event-labels.spec.ts`, 7 vectors incl. identical-instant, reversed
order, three-way cluster, cross-matchup isolation) plus surface tests
(`analyzer-occurrence-labels.spec.ts`, 4).

### f-e -- missing commence rejected (corrected in c5ab834)

Covered by the group policy above: omitted field, json null, empty string,
offset-less, and unparseable values are malformed members excluding their whole
nonblank-id group; the year-one sentinel event no longer exists. Pinned including
the mixed valid+malformed same-id group in both source orders.

### f-f -- identity boundary (corrected in c545117 + this record)

All "end to end" wording corrected in `OddsScheduleClient.cs` and
`agent-run.model.ts`: distinguishability runs discovery -> dto -> rendering ->
providerEventId-keyed operator selection, and terminates BEFORE analysis-request
serialization (payload = date + teams only). The payload was deliberately NOT
changed. Durable residual:

**SELECTED_EVENT_IDENTITY_PROPAGATION_REQUIRED_BEFORE_WI0037_CLOSE** -- two
distinct provider selections may still serialize to equivalent analysis requests;
count parity does not prove semantic run identity; changing the analysis payload
may affect WI-0035/WI-0036 provider binding, stored runs, API compatibility,
reconciliation provenance, and other callers; it requires a separate
architecture-binding review. Slice 2-iii "Selected-Event Identity Continuity" is
DEFINED (unauthorized) in the WI-0037 spec and must be reviewed and completed
before WI-0037 closes; it is NOT folded into 2-ii-c, whose obligations (F3, F4,
requireStatus hardening, normalization consolidation, cross-runtime parity) are
unchanged.

### f-g -- typed batch assertion (corrected in c5ab834)

The duplicate-batch theory now asserts `MlbStarterClient.DuplicateBatchInput`
exactly, empty result map, `ScheduleAttempts == 0`, and zero HTTP requests, across
adjacent, nonadjacent, and reversed-order duplicate inputs; the distinct-DH-pks
valid case is unchanged.

## verification (exact, post-correction)

- focused backend: integrity + client suites **36/36** green (previously 18/20 RED)
- full .NET: **1853 passed / 0 failed / 0 skipped** (baseline 1831 + 20 integrity
  + 2 added F-G theory rows)
- operator harness: **187/187**; finals guard: **40/40** (no PowerShell change)
- frontend: **155 passed / 0 failed (17 files)** (baseline 136 + 8 selection
  identity + 4 occurrence labels + 7 label vectors); production build SUCCESS;
  lockfile untouched (no install needed)
- `git diff --check` clean; machine-path/secret scans clean; dependency diff
  empty; zero live/paid/model/db/capture/reconciliation/settlement activity

## scope

Correction paths (14): `OddsScheduleClient.cs`, `OddsScheduleClientTests.cs`,
`OddsScheduleClientIntegrityTests.cs` (new), `analyzer.component.ts`,
`analyzer-occurrence-labels.spec.ts` (new), `matchup-event-distinguishability.spec.ts`,
`agent-run.model.ts`, `matchup-event-labels.ts` (new),
`matchup-event-labels.spec.ts` (new), `dev-artifact-review.component.ts`,
`dev-artifact-review.component.html`, `matchup-selection-identity.spec.ts` (new),
`review-feedback.spec.ts`, `sports-api.service.ts`. UNTOUCHED (verified):
GameStatusResolver, GameStatusPayloadReader, MlbEventResolver, contract 1.1,
fixture corpus, every PowerShell script, F3, F4, requireStatus, adapter
normalizer, the analysis-request DTO and endpoint contract, provider-event
binding, reconciliation/settlement/db/schemas/migrations, dependencies.

## commits and branch state

- dai: `841ae26` -> `5a11a2c` (original, never amended) ->
  **`c5ab834`** "fix(sports): harden provider event integrity" (3 files,
  +297/-44) -> **`c545117`** "fix(sports): preserve doubleheader selection
  identity" (11 files, +293/-24); tree clean; local only, NOT pushed.
- dai-vault: `664cd4a` -> `fd2d20b` (original, never amended) -> this correction
  record + spec state + handoff in one new docs commit; local only.

## next governed action

Delta review limited to `5a11a2c..c545117` and `fd2d20b..<vault tip>`, then a
separate integration/publication authorization. 2-ii-c and 2-iii remain
unauthorized.
