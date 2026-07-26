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

## part 3 -- dr-1 disposition (delta review + source-hygiene correction, 2026-07-26)

The independent delta review of `5a11a2c..c545117` / `fd2d20b..3f31c84` PASSED
every behavioral and architectural attack (selection identity, cardinality
parity, track keys, integrity ordering, malformed groups, source-order
independence, utc precision, frontend seven-digit runtime compatibility incl.
the documented sub-millisecond Date.parse bound with providerEventId tie-break,
chronological ordering, occurrence labels, missing commence, typed batch
failure, analysis identity boundary, slice 2-iii governance, 2-ii-c exclusion,
vault accuracy) and returned exactly one finding:

**DR-1 (Medium, audit-integrity; no runtime-behavior impact).**
`apps/sports-app/src/app/core/matchup-event-labels.ts` line 26 contained two RAW
U+0000 bytes as the group-key separators inside a template literal (byte offsets
55 and 70 of the line; the only NUL-bearing file in the whole slice delta). Git
therefore classified the committed source blob as BINARY: `file` reported
"data", ordinary grep degraded to "Binary file matches", and the complete-chain
numstat showed `- -`. Runtime behavior was correct throughout (NUL is a legal,
collision-safe separator; 155/155 + build were green), but the blob was opaque
to textual diff, blame, and grep-based audit -- the same sweep machinery this
program's reviews depend on.

**Correction (dai `b8d0c0b`, new commit; nothing amended).** The raw bytes are
now the six ascii characters backslash-u-0-0-0-0, which the javascript runtime
evaluates to the IDENTICAL U+0000 separator. Distinction recorded: a raw NUL is
a byte in the source file; the escape is ascii source that produces the same
character at runtime -- grouping semantics are untouched. One
collision-resistance vector was added to the EXISTING
`matchup-event-labels.spec.ts` because the pre-existing tests could not
distinguish the correct NUL runtime from a defective printable separator: the
new vector aliases under a printable "0000" or space separator and passes only
with the real NUL, proving runtime equivalence (separator character code 0).

**Text-classification proof.** Committed blob NUL count: 0 (was 2). `file`:
javascript source, utf-8 text. Ordinary grep inspects the file without binary
overrides. Complete chain `841ae26..b8d0c0b`: the helper appears as a textual
file, numstat `63 0` (numeric, not `- -`), and NO source file in the chain is
binary-classified. No `.gitattributes`, diff driver, base64, or generated
replacement was used -- the blob itself is clean text. Git nuance recorded: the
PARENT-pair diff `c545117..b8d0c0b` alone still renders as binary because the
parent blob contains the raw bytes; that is expected and not a failed
correction -- the authoritative checks are the new blob and the complete
published-candidate chain, both text.

**Verification.** Frontend **156/156** (155 baseline + 1 dr-1 pin) + production
build SUCCESS; full .NET **1853/1853, 0 skipped**; operator **187/187**; finals
guard **40/40**; `git diff --check` clean; secret/machine-path scans clean;
dependency and lockfile diffs empty. Scope: exactly `matchup-event-labels.ts` +
its existing spec (justified above). The optional sub-millisecond observation
test was NOT added (explicitly unauthorized this turn). Zero live/paid/model/
db/capture/reconciliation/settlement activity.

**State.** dai chain: `841ae26 -> 5a11a2c -> c5ab834 -> c545117 -> b8d0c0b`
(local only). Slice 2-ii-b: DR-1 corrected locally; INDEPENDENT DR-1 DELTA
REVIEW REQUIRED before any integration claim. 2-ii-c unauthorized; 2-iii
defined/unauthorized; WI-0037 in-progress.

## part 4 -- dr-2 disposition (grouping collision-safety correction, 2026-07-26)

The independent DR-1 delta review confirmed DR-1 fully closed (blob purity, exact
ascii escape spelling, runtime U+0000 equivalence proven against the committed
implementation, complete-chain text visibility, no concealment, behavior
identical) and returned one new finding:

**DR-2 (Medium).** The part-3 statement "NUL cannot appear inside date or team
fields, so concatenated grouping fields can never alias each other" is hereby
EXPLICITLY SUPERSEDED as overstated (the original wording is preserved above as
history). Reachability evidence: a provider payload whose team name embeds an
escaped NUL survives the REAL sampler path into MatchupEventDto (no schema,
parser, validation, or normalization excludes control characters from
provider-controlled team strings), and at the frontend helper the
delimiter-concatenated key aliased the distinct ordered tuples ("A", NUL+"B")
and ("A"+NUL, "B") -- probe-reproduced and re-reproduced RED this turn
(dr2-red.txt). Impact was label-grouping ONLY: spurious shared-group time/Game N
discriminators; provider ids, angular track keys, row identity, selection, and
execution-entry cardinality were never affected.

**Correction (dai `af59853`, new commit; nothing amended).** Ordered-string-tuple
contract verified first: date, homeTeam, and awayTeam are REQUIRED strings on
the canonical MatchupEventDto (backend record guarantees non-null strings;
stubs and call sites conform) -- GROUP_KEY_MEMBERS_STRING_GUARANTEED, so no
runtime guard or coercion is needed. The group key is now
`JSON.stringify([date, homeTeam, awayTeam])`. Guarantee claimed, precisely: JSON
array serialization is an unambiguous encoding for THIS ordered tuple of
strings -- embedded control characters, quotes, backslashes, delimiter
lookalikes, empty strings, and unicode cannot cause distinct tuples to alias,
and identical tuples always share one key. No claim is made for non-string
values (unreachable here by contract). The runtime NUL separator construction
and the superseded exclusion claim are removed from source.

**Collision vectors (table-driven, committed in the existing spec).** Embedded
NUL at the boundary (the exact DR-2 pair, now non-aliasing); printable "0000";
spaces; colons; pipes; quotes; backslashes; json-looking content (brackets,
commas, null-text); empty strings in either position (order significance);
unicode (accented + cjk) -- each proving distinct ordered tuples yield distinct
keys under both source orders; plus an identical-tuple vector (quotes +
backslash + NUL content) proving identical tuples still group. NUL is
constructed programmatically (String.fromCharCode(0)); zero raw control bytes
exist in either changed file.

**Source hygiene.** Both files: NUL 0, unexpected C0 0, valid utf-8; helper
classifies as javascript source text; the complete chain `841ae26..af59853`
contains no binary source entry and the helper numstat is numeric (67/0); the
parent-pair diff `b8d0c0b..af59853` is itself textual now that both blobs are
clean; no gitattributes or diff drivers.

**Verification.** Frontend **157/157** (156 baseline - 1 superseded dr-1 vector
+ 2 structural tests) + production build SUCCESS; full .NET **1853/1853, 0
skipped**; operator **187/187**; finals guard **40/40**; git diff --check
clean; secret/machine-path scans clean; dependency and lockfile diffs empty.
Scope: exactly the helper + its existing spec. Zero live/paid/model/db/capture/
reconciliation/settlement activity. Backend control-character sanitization
remains deliberately OUT of scope (recorded hardening appetite, 2-iii-adjacent).

**State.** dai chain: `841ae26 -> 5a11a2c -> c5ab834 -> c545117 -> b8d0c0b ->
af59853` (local only). Slice 2-ii-b: DR-2 corrected locally; INDEPENDENT DR-2
DELTA REVIEW REQUIRED before any integration claim. 2-ii-c unauthorized; 2-iii
defined/unauthorized; WI-0037 in-progress.
