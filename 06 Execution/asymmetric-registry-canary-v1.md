# Asymmetric Registry Canary v1 -- Evidence Report

**status:** complete (non-paid canary verified; promotion blocker identified; tests-only, not pushed)
**date:** 2026-06-30

## purpose

Define and run a non-paid registry canary / shadow-validation path for `starter_asymmetric_market_backed_depth`,
proving the new asymmetric recipe assembles safely, routes predictably under default and explicit-override
settings, remains shadow_only, and is not default-authoritative. Identify exactly what is required to promote it.
No allowlist widening, no paid calls.

## start state

- `dai`: clean, synced, `a65fa19` (0/0). `dai-vault`: clean, synced, `cf56dd2` (0/0).
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- Manifest 9 templates / 10 recipes (check OK). Recipe
  `mlb.pregame.analysis.starter_asymmetric_market_backed_depth.v1`, template
  `mlb.overlay.starter.asymmetric.v1`, regime `starter_asymmetric_market_backed_depth`, and state `asymmetric`
  all present. `DEFAULT_ALLOWLIST` unchanged (4) and excludes asymmetric. No paid calls.

## canary / override mechanism

The canary allowlist is overridable without touching `DEFAULT_ALLOWLIST`:
- code: `RegistryPromptCanaryConfig(enabled=True, regimes=("starter_asymmetric_market_backed_depth", ...))`.
- env: `DAI_MLB_REGISTRY_PROMPT_CANARY_REGIMES` (comma-separated) overrides the default allowlist;
  `DAI_MLB_REGISTRY_PROMPT_CANARY=1` enables.
So a non-default regime CAN be canaried deliberately, exactly as this slice does in fixtures.

## default route behavior (verified)

Canary enabled, DEFAULT allowlist, asymmetric one-sided-quality backed_depth fixture:
- promptSource = **live**, selectedDataRegime = **starter_asymmetric_market_backed_depth**,
  regimeAllowlisted = **false**, legacyFallbackUsed = true, fallbackReason = **regime_not_allowlisted**,
  recipe id = null. The model receives the live bytes.

This is the win over the old behavior: the asymmetric game now fails closed BY POLICY (not allowlisted), not by
recipe failure (assembly_error) -- because the recipe assembles. (test
`test_canary_default_asymmetric_fails_closed_regime_not_allowlisted`.)

## override / canary behavior (verified)

Canary enabled with an EXPLICIT override allowlisting `starter_asymmetric_market_backed_depth`:
- the recipe assembles AND is compared to the live prompt, but the shadow render is **NOT byte-identical** to the
  live asymmetric rendering -> the canary returns **fallbackReason = mismatch**.
- promptSource = **live** (cardinal invariant: model only ever gets live bytes), regimeAllowlisted = **true**
  (override took effect), selectedPromptRecipeId/Version = **populated**
  (`...starter_asymmetric_market_backed_depth.v1` @ v1), assembledHash = **null** (set only on a registry-
  authoritative byte-match). It is **NOT registry-authoritative.** (test
  `test_canary_override_asymmetric_is_mismatch_not_registry_authoritative`.)

Root cause of the mismatch: the live prompt renders one-sided quality inline ("home starter season form: X"
between the two starter lines, with the "use the provided season stats" instruction), while the registry
asymmetric overlay renders "season form available for the home starter only: X" after both starter lines with a
different instruction. Same evidence, different bytes -> mismatch by design.

## recipe assembly result (verified)

`builder.assemble_recipe_for_migration` for an asymmetric backed_depth fixture returns the recipe
`mlb.pregame.analysis.starter_asymmetric_market_backed_depth.v1` with regime
`starter_asymmetric_market_backed_depth` and renders text containing the named starters + the one-sided form
line -- **no PromptAssemblyError.** Symmetric complete enriched still assembles to the enriched recipe and is
registry-authoritative under default (unchanged).

## provenance behavior

- default: live / regime_not_allowlisted / regime set / recipe id null / hash null.
- override (mismatch): live / mismatch / regime set / recipe id + version set / hash null.
- a registry-authoritative asymmetric decision (promptSource=registry, hash populated) is **not reachable today**
  because byte-equivalence to live fails.

## canary questions answered

1. Classifier selects it from a fixture? **Yes** -- one-sided quality -> starter_asymmetric_market_backed_depth.
2. Assembles without PromptAssemblyError? **Yes.** 3. Default fails closed by policy not assembly? **Yes** --
regime_not_allowlisted. 4. Can override route it registry-authoritative? **No** -- override allowlists it, but it
is not byte-identical to live -> mismatch (still live bytes). 5. Provenance recipe/version/hash under override?
**recipe+version yes, hash no** (mismatch, not a registry match). 6. Byte- or schema-equivalence for promotion?
The current canary's cardinal invariant **requires byte-equivalence** (registry_text == live_prompt) before
registry bytes feed the model; the asymmetric overlay is not byte-equal, so byte-equivalence is the live blocker.
Two promotion paths: (a) align the asymmetric overlay to the live asymmetric bytes (a prompt-text change), or
(b) relax the criterion to schema/artifact equivalence for this regime (a canary-policy change that weakens the
cardinal invariant). **Recommend (a)** -- keep the strong byte invariant; do not relax it casually. 7. Evidence
threshold -> see below. 8. First paid proof fixture or live? **Live-scheduled** -- a real asymmetric game; the
fixture already proves assembly, the paid proof must confirm buyer-safe reasoning on real one-sided evidence.
9. Candidate signal? **Both probable pitchers announced, but exactly one has 2026 season stats** (the other a
debut/no-stats) -- detectable via StatsAPI probablePitcher present on both + `people/{id}/stats` present for one
side only. Distinct from the TBD signal that finds starter_missing.

## promotion threshold (proposed -- NOT implemented)

Add `starter_asymmetric_market_backed_depth` to DEFAULT_ALLOWLIST only when ALL hold:
1. manifest valid (9/10, hashes OK) -- **met now.**
2. non-paid fixture canary passes (classify + assemble + default regime_not_allowlisted + known override
   behavior) -- **met now.**
3. **byte-equivalence resolved**: the asymmetric overlay rendered bytes == the live asymmetric rendering, so an
   override routes registry-authoritative instead of mismatch -- **NOT met (the gating blocker).**
4. explicit-override registry-authoritative run works (promptSource=registry, recipe/version/hash populated) --
   blocked on (3).
5. >= 1 paid controlled LIVE canary on a real asymmetric game completes: artifact body schema unchanged, buyer
   copy safe, route provenance correct, no assembly_error, promptSource registry (or safe fallback).
6. metrics route `starter_asymmetric_market_backed_depth` appears separately (registry-source).
7. DEFAULT_ALLOWLIST unchanged until (1)-(6) all met. Optional: a second live proof if asymmetric candidates are
   easy to find.

Current position: steps 1-2 done; **step 3 (byte-equivalence) is the blocker**; 4-6 follow.

## is a future paid asymmetric canary ready?

**Not yet.** With byte-equivalence unresolved, an override-enabled paid run would route `mismatch` (safe, but it
proves nothing about registry-authoritative behavior; the model would use the live prompt anyway). A paid canary
becomes meaningful for PROMOTION only after the overlay is byte-aligned to live. So: align first (non-paid), then
paid-canary.

## tests run

TDD; tests-only (no production code change -- the asymmetric path already exists from the prior slice).
- 3 new canary tests added to `tests/test_asymmetric_regime_split.py`: default -> regime_not_allowlisted;
  override -> mismatch (not registry-authoritative, recipe/version set, hash null); symmetric enriched still
  registry-authoritative.
- `pytest` full agent-service suite -> **400 passed**.
- `check_prompt_manifest.py` -> OK (9 templates, 10 recipes).

## paid-call status

**None.** Fixture/dry-run canary + manifest check only.

## buyer-facing impact

**None.** Verification + tests only; no template/recipe/classifier/allowlist/route-key change; no .NET change; no
chain-of-thought or prompt text exposed. The asymmetric prompt is shadow_only and never reaches the model.

## default allowlist status

**Unchanged -- exactly four regimes; `starter_asymmetric_*` excluded** (asserted by tests).

## risks / deferred items

- Byte-equivalence is the promotion blocker; resolving it is a prompt-text change (align the overlay to the live
  asymmetric rendering) -- intentionally out of scope here and deferred to a dedicated slice.
- Relaxing the canary to schema-equivalence would weaken the cardinal byte invariant; not recommended.
- Only the backed_depth asymmetric recipe exists; asymmetric + missing/backed remain recipe-less (classify but
  fall back regime_not_allowlisted / fail closed on direct assembly).
- Justification still rests on n=1 observed assembly_error; deterministic from the recipe.

## next recommended slice

The asymmetric thread's true next step is **Asymmetric Overlay Byte-Alignment v1** (non-paid prompt-text slice):
make the asymmetric overlay rendered bytes identical to the live asymmetric rendering so an override routes
registry-authoritative -- this unblocks promotion threshold step 3/4. **Paid Asymmetric Canary v1 is premature**
until that lands (it would only route mismatch today).

If preferring non-blocked standing work: **Outcome Reconciliation Follow-up v1** once the 20-run backlog games
reach Final -- still the prerequisite for converting live route coverage into performance evidence.
