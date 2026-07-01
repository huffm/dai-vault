---
title: "Asymmetric Overlay Byte-Alignment v1"
type: "evidence-report"
date: "2026-06-30"
status: "complete"
project: "DAI"
slice: "Asymmetric Overlay Byte-Alignment v1"
repos:
  dai: "code+docs"
  dai-vault: "docs-only"
tags:
  - prompt-registry
  - provenance
  - asymmetric
related:
  - "06 Execution/reports/asymmetric-enriched-recipe-regime-split-v1.md"
  - "06 Execution/reports/asymmetric-registry-canary-v1.md"
---

# Asymmetric Overlay Byte-Alignment v1 -- Evidence Report

**status:** complete (byte-equivalence achieved; override now registry-authoritative; full suite green; not pushed)
**date:** 2026-06-30

## purpose

Align the registry asymmetric overlay rendering with the live asymmetric rendering so
`starter_asymmetric_market_backed_depth` passes the explicit-override canary as **registry-authoritative**
instead of falling back with `mismatch`. This unblocks promotion threshold step 3 (byte-equivalence) from the
Asymmetric Registry Canary v1. No allowlist widening, no promotion, no paid calls.

## start state

- `dai`: clean, synced, `2614f08` (0/0). `dai-vault`: clean, synced, `8b5b50d` (0/0).
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- Manifest 9 templates / 10 recipes. Recipe `...starter_asymmetric_market_backed_depth.v1` present.
  `DEFAULT_ALLOWLIST` unchanged (4) and excludes asymmetric. No paid calls.

## mismatch root cause (byte diff, before)

For the asymmetric one-sided-quality fixture (home quality only), `render_recipe` (1122 chars) vs
`build_mlb_user_message` (1058 chars) differed in the starter block:

```
 home starter: Hunter Brown (RHP)
-home starter season form: ERA 3.45, WHIP 1.21, 95 K, 22 BB, 84.1 IP, 14 GS     # LIVE: inline, after home
 away starter: Jack Flaherty (LHP)
+season form available for the home starter only: ERA 3.45, ...                 # REGISTRY: after both, diff wording
 use these starters in the starting pitching factor. ...
-use the provided season stats (era, whip, k, bb, innings, games started) ...   # LIVE instruction
+use the provided season stats only for the side where they are given; ...      # REGISTRY instruction
```

Two divergences: (1) the form line position + wording, (2) the trailing instruction line.

## byte-alignment change

The asymmetric overlay was restructured to mirror the live rendering exactly, using per-side form **blocks**
slotted inline:

```
[starter data]
home starter: {{home_starter_name}} ({{home_starter_hand}}HP){{home_form_block}}
away starter: {{away_starter_name}} ({{away_starter_hand}}HP){{away_form_block}}
use these starters in the starting pitching factor. note handedness advantage vs lineup if relevant.
use the provided season stats (era, whip, k, bb, innings, games started) as pitcher quality/form. do not fabricate any stats beyond those provided.
```

Slot derivation (`_starter_slots`, asymmetric branch) now emits, using the SAME `format_pitcher_quality`
formatter the live path uses:
- `home_form_block` = `"\nhome starter season form: <home_q>"` when home has quality, else `""`.
- `away_form_block` = `"\naway starter season form: <away_q>"` when away has quality, else `""`.

So the grounded side renders its inline live "season form" line and the other renders nothing -- byte-identical
to live. The trailing instruction is the live enriched instruction (always emitted because exactly one side has
quality). No semantic change, no broadened content; buyer-safe language preserved (it IS the live wording).

Required slots changed from `enriched_side`/`enriched_starter_form` to `home_form_block`/`away_form_block`;
template `sha256` recomputed and updated in the manifest. Template count unchanged (9/10).

## before / after behavior

| asymmetric backed_depth | before (Canary v1) | after (this slice) |
|---|---|---|
| registry render == live | **no** (1122 vs 1058) | **yes** (byte-identical, both orientations) |
| default canary | regime_not_allowlisted | regime_not_allowlisted (unchanged) |
| explicit override | mismatch (promptSource=live, hash null) | **registry-authoritative** (promptSource=registry, hash set, fallbackReason null) |
| shadow validation | mismatch, not captured | **matched + captured** |

## default route behavior (verified, unchanged)

Canary enabled, DEFAULT allowlist, asymmetric backed_depth -> promptSource=live,
regime=starter_asymmetric_market_backed_depth, regimeAllowlisted=false,
fallbackReason=**regime_not_allowlisted**, recipe null. Still fails closed by policy (not promoted).

## explicit override behavior (verified, now registry-authoritative)

Override allowlisting the asymmetric regime -> msg == live (registry bytes ARE the live bytes),
promptSource=**registry**, regimeAllowlisted=true, legacyFallbackUsed=**false**, fallbackReason=**null**,
selectedPromptRecipeId/Version populated, **assembledHash populated (64 hex)**. No longer a mismatch.

## tests

TDD. `tests/test_asymmetric_regime_split.py`: updated slot assertions to `home_form_block`/`away_form_block`;
replaced the override "mismatch" test with `test_canary_override_asymmetric_is_registry_authoritative`; added a
parametrized `test_asymmetric_render_is_byte_identical_to_live` (home-only AND away-only); updated the
assembly-text assertion to the live wording. `tests/test_shadow_validation.py`: the one-sided-quality case now
**matches + captures** (was mismatch) -- updated to `test_enabled_asymmetric_one_sided_quality_matches_and_
captures`. Default-fails-closed, symmetric-enriched-unchanged, allowlist-unchanged, and shadow_only tests retained.

## verification

- Dry-run byte check: `render_recipe == build_mlb_user_message` -> **True** for both home-only and away-only
  asymmetric fixtures.
- `pytest` full agent-service suite -> **402 passed**.
- `check_prompt_manifest.py` -> OK (9 templates, 10 recipes); new overlay sha256 validated.
- Asymmetric recipe still `shadow_only`. DEFAULT_ALLOWLIST unchanged (asserted). Symmetric enriched unchanged
  (asserted). No .NET / buyer contract change. No paid calls.

## allowlist status

**Unchanged -- exactly four regimes; `starter_asymmetric_*` still excluded.** This slice does NOT promote; it
only makes the route promotable.

## promotion status

Promotion threshold (from Canary v1): (1) manifest valid -- met. (2) non-paid fixture canary passes -- met.
(3) **byte-equivalence resolved -- NOW MET** (this slice). (4) explicit-override registry-authoritative run works
-- **NOW MET** (verified). (5) >=1 paid controlled live canary on a real asymmetric game -- **not yet** (the
remaining gate). (6) metrics route appears separately -- pending the paid run. (7) allowlist unchanged until all
met -- held. So the asymmetric route is now technically promotable; only the paid live proof remains.

## paid-call status

**None.** Fixture/dry-run + manifest check only.

## buyer-facing impact

**None.** Change is confined to the FastAPI prompt routing layer (overlay template + slot derivation + tests).
`SportsAnalysisResponse` and the .NET platform are untouched. The asymmetric overlay's bytes now equal the live
prompt's bytes by construction -- no new buyer-visible content; no chain-of-thought or prompt text exposed. The
route remains shadow_only / not allowlisted, so it does not change default model input.

## risks / deferred items

- The asymmetric recipe is now byte-equal to live, so under override it would feed the model the (identical)
  registry bytes -- but it is NOT allowlisted by default, so default behavior is unchanged. Promotion still
  requires the paid live proof + an explicit allowlist change (separate slice).
- Only `starter_asymmetric_market_backed_depth` is byte-aligned; asymmetric + missing/backed remain recipe-less.
- Byte-equivalence is now coupled to the live renderer's exact wording; a future change to the live starter
  block must be mirrored in this overlay (the migration-readiness equivalence tests guard this).

## next recommended slice

**Paid Asymmetric Canary v1** -- now ready (override routes registry-authoritative). Run one controlled paid
analysis on a real asymmetric game (both starters announced, exactly one with season stats) under an explicit
override allowlisting `starter_asymmetric_market_backed_depth`, with the approval gate, to prove: promptSource
registry, artifact body schema unchanged, buyer copy safe, route provenance correct, no assembly_error, metrics
route appears separately -- the final promotion gate (threshold step 5/6) before any DEFAULT_ALLOWLIST change.
Non-blocked alternative: Outcome Reconciliation Follow-up v1 once the 20-run backlog settles.
