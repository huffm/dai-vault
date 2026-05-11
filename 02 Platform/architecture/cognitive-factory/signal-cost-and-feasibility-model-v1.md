# signal cost and feasibility model v1

**date:** 2026-05-11
**status:** v1 doctrine — design and documentation slice. no runtime code, no provider integration, no procurement.
**owns:** the rules that decide whether a signal is acquired free, manually, via a low-cost API, via a paid provider, via an enterprise feed, or deferred.

---

## 1. core claim

DAI's cognitive worker doctrine says the factory grades each phase by whether its output improves the artifact. The fallback ladder says each signal must be classified by equivalence. Both are correct and neither has anything to say about **money**.

This slice adds the business layer. Every signal carries a real cost — money, legal exposure, integration complexity, operational risk — and a real value — confidence preserved, posture unblocked, decisions improved. A signal is **feasible** when its value exceeds its cost at the project's current stage. The cognitive factory only ships signals that are feasible right now, and only spends on a signal upgrade when a documented trigger fires.

A signal that the platform cannot afford yet stays visible in the artifact as `missing` with a real reason. It does not get faked, padded, or quietly dropped. Honest absence is a feature, not a defect — it lets discern dampen confidence correctly and lets decide reject aggressive posture.

---

## 2. signal cost tiers

Every signal sits in exactly one tier. Tiers describe *current* cost posture, not aspirations. A signal can move between tiers as the project's revenue and book of customers change.

| tier | name | cost posture | typical examples | legal posture |
|---|---|---|---|---|
| **0** | Free deterministic / internal | $0 marginal cost. Computed from inputs the platform already owns: schedule data, prior retrieved snapshots, derived counters. | days rest, evidence richness count, calibrated confidence, line movement derived from polled odds snapshots | platform-owned data, no third-party terms |
| **1** | Free / public provider | $0 cash cost. Unofficial or public HTTP endpoints that do not require auth or licensing. **Legal posture varies — terms-of-use review required per provider before commercial use.** | ActionNetwork unofficial scoreboard, ESPN scoreboard, statsapi.mlb.com, open-meteo, official NBA injury report PDF | unofficial endpoints carry rate-limit and TOS risk; public stat APIs (statsapi.mlb.com, open-meteo) are explicitly free-to-use |
| **2** | Low-cost odds APIs | $30–$100/mo. Commercial API with documented terms, modest tier ceilings, broad odds coverage. **Splits and exotic signals usually not included.** | The Odds API, OddsJam basic tier, similar consumer-grade aggregators | clean commercial license, attribution typically required |
| **3** | Paid split / stat providers | $100–$1000/mo. Dedicated commercial APIs with structured splits, advanced stats, or richer market data. | SportsDataIO (Trends/Insights tier), Rotowire premium injury feeds, Sports Insights (subject to Better Collective sister-property review) | clean commercial license with redistribution clauses; per-tier feature gating |
| **4** | Enterprise feeds | $50k+/year. Direct integrations with industry data providers. Long sales cycle, contractual minimums. | Sportradar, Genius Sports, Pinnacle commercial partnerships, Don Best | enterprise contracts; legal review per deal |
| **5** | Manual or sponsor-funded special sources | Variable. Human review, one-off CSV uploads, customer-funded data, sponsored access. | manual injury review during playoffs, sponsor-provided model output, partner-supplied feed | per-arrangement |

Note: cost tier and fallback ladder rung are independent. A Tier 3 paid provider could supply a `source_substitution / near` signal or a `fidelity_downgrade / partial` signal — the rung describes what role the signal plays, the tier describes what it costs. Both fields must be recorded on every signal.

---

## 3. signal feasibility fields

Each signal in a niche is described by a feasibility record. This is configuration doctrine, not runtime data. It lives alongside niche config (today: implicit in `CompetitionCatalog`, `SignalQualityEvaluator`, `ActionNetworkClient` decisions). A future slice may extract it into a typed config object. v1 records it as documentation.

```text
SignalFeasibilityRecord {
  signalName             : string   // canonical: market, sharp_public, line_movement, injury_report, ...
  decisionPurpose        : string   // one sentence: what decision does this signal influence?
  sourceType             : string   // "free_unofficial" | "free_public" | "low_cost_api" | "paid_api" | "enterprise" | "manual"
  providerName           : string   // canonical provider id (e.g. "actionnetwork", "sportsdata_io", "the_odds_api")
  costTier               : int      // 0 .. 5
  monthlyCostEstimate    : decimal  // current expected $/mo at observed call volume
  costPerRunEstimate     : decimal  // marginal $ per agent run (often $0 within a flat tier)
  legalUseStatus         : string   // "cleared" | "review_pending" | "scrape_risk" | "needs_license" | "n/a"
  coverageScope          : string[] // ["nba_regular", "nba_playoffs", "nfl_regular", "mlb_regular", ...]
  availabilityRate       : decimal? // observed fraction of runs grounded; null pre-calibration
  freshnessWindow        : string   // "real-time" | "5_minutes" | "intraday" | "daily" | "weekly"
  provenanceRequirement  : string   // "providerName_required" | "providerName_optional"
  fallbackLadderType     : string   // "exact_recovery" | "source_substitution" | "fidelity_downgrade" | "adjacent_proxy" | "lateral_proxy" | "unavailable_with_reason"
  confidencePermission   : string   // confidence_preserved | confidence_mostly_preserved | confidence_dampened | confidence_conservative | confidence_reduced
  posturePermission      : string   // aggressive_allowed | aggressive_allowed_if_corroborated | aggressive_blocked
  upgradeTrigger         : string   // condition under which to escalate this signal a tier (see §5)
  deferReason            : string?  // if not yet acquired, the documented reason
}
```

The four ladder-derived fields (`fallbackLadderType`, `confidencePermission`, `posturePermission`, plus `availabilityRate`) are intentionally duplicated from the runtime `SignalFollowUpRecord`. The duplication keeps the *intended* classification (config doctrine) separate from the *observed* classification (runtime telemetry). When they disagree, calibration must investigate — typically the doctrine record is stale.

---

## 4. provider evaluation rules

A provider is not eligible for production wiring unless **all six** of these gates are passed.

1. **Maps to the intended signal rung.** A provider sold for line-movement cannot satisfy a `sharp_public` substitution. Verify the field set before any discussion of cost or contract.
2. **Acceptable commercial-use terms.** Legal review for: redistribution rights, attribution requirements, scraping prohibitions, integrity-monitoring clauses, NDA scope. A provider with usable data and unusable terms is not usable.
3. **Stable machine-readable access.** A documented API, predictable schema, versioning posture. HTML-only sources are not eligible for production wiring without an explicit scrape-licensing slice and legal sign-off.
4. **Known cost or a sales quote in hand.** "Contact us for pricing" without a quote is not a known cost. Procurement decisions cannot be made on guesses.
5. **Improves at least one recurring decision failure.** Either calibration data shows the signal would have changed decisions on past artifacts, or the calibration report's pattern summary names the missing signal as the dominant gap. Speculative provider purchases are rejected at this gate.
6. **Can record provider provenance per artifact.** The artifact must carry `providerName` on every persisted record so calibration can grade providers independently over time. A provider that obscures its source attribution is not eligible.

A provider that passes all six gates is *eligible*; eligibility does not imply that the budget exists today. The subscription decision rule in §5 then applies.

---

## 5. subscription decision rule

A paid data provider is committed only when at least one of these is documented as true:

1. **Paying users explicitly need that signal.** Customer interviews or product analytics name the signal as a purchase driver.
2. **Missing signal blocks repeated high-value artifacts.** Calibration shows the signal is missing on a majority of the most-trafficked competition / matchup categories, and the gap is suppressing posture or confidence on a meaningful share of runs.
3. **Calibration shows the signal changes decision quality.** Outcome data (`AgentRunOutcome`, `AgentRunEvaluation`) shows that runs with the signal were materially more correct than runs without.
4. **Cost is below the budget gate for the project's stage** (see budget gates below).
5. **Provider unlocks multiple sports or niches.** A provider with broad coverage that solves NBA *and* opens NFL/MLB/crypto/stocks is amortized across niches; the per-niche cost is meaningfully lower.
6. **A customer or sponsor funds the data source.** Bring-your-own-key agreements, sponsor deals, or pilot-funded data flows that do not consume product margin.

If none of these is true, the signal stays in its current tier — usually deferred or honestly recorded as `missing`.

### budget gates

These are the spending ceilings the chief architect approves without escalation. Exceeding them requires a documented business case with revenue, customer, or sponsor backing.

| stage | gate | rationale |
|---|---|---|
| **pre-revenue** (current state) | $0–$30/month total recurring provider spend | Discovery phase. Every dollar comes from owner's pocket. Free unofficial endpoints, public APIs, and one trial-grade subscription are the budget. |
| **early validation** (first paying customer through ~10 customers) | $30–$100/month total | Cost is small enough to absorb against early MRR while validating that customers actually return for the product. |
| **post-revenue** (recurring MRR established) | data provider spend ≤ 10–15% of MRR | Maintains gross margin. Above 15% means cost of goods sold becomes the dominant constraint and product economics start to leak. |
| **enterprise feeds** (Tier 4) | only after established revenue, a partnership, or a fully funded pilot | Sportradar / Genius Sports / Pinnacle commercial deals require six-figure annual commitments. Never enter that contract on speculation. |

The gates are ceilings, not targets. The smallest spend that satisfies the decision rule is the right spend.

---

## 6. signal ROI ledger (runtime telemetry contract)

Calibration cannot grade signals without per-run telemetry. The existing `SignalAvailabilityRecord` and `SignalFollowUpRecord` already carry most of what is needed; the ROI ledger names the full target shape so future additive fields land in the right place.

Per signal per run, the artifact must capture:

| field | source today | gap |
|---|---|---|
| signal requested | `SignalAvailability[].Signal` | none |
| signal grounded vs missing | `SignalAvailability[].Status` | none |
| missing reason if any | `SignalAvailability[].Reason` | none |
| provider used | implicit in `SignalAvailability[].Source` | **gap: not yet a typed providerName field on the record** |
| cost tier | n/a today | **gap: add `CostTier` field — additive optional, can default null on existing records** |
| source updated time | `SharpPublicContext.UpdatedAt` for sharp_public; not present on most other signals | **gap: standardize across all signals** |
| retrieved time | per-run `StartedUtc` proxy today | **gap: per-signal retrieval timestamp** |
| effect on confidence | `SignalAvailability[].ConfidenceEffect` (today's "effect" enum); `SignalFollowUpRecord.ConfidencePermission` (ladder) | partial — two parallel notions of effect |
| effect on posture | `SignalFollowUpRecord.PosturePermission` | observational only — not yet enforced by `SportsEvaluator` |
| outcome result | `AgentRunOutcome`, `AgentRunEvaluation` (run-level, not signal-level) | **gap: signal-level outcome attribution** |
| calibration finding | aggregated in calibration report PS1 | not stored on the artifact itself |
| user usefulness feedback | not collected | **gap: future user feedback surface** |

**Sequencing rule:** none of these gaps are filled in this slice. They are sequenced for future slices when the upstream data — outcome records, customer feedback, calibration grades — is rich enough to justify the storage. The doctrine names the target so future additive fields land coherently.

---

## 7. low-cost acquisition strategy

Practical patterns that produce useful signal at Tier 0–1 without provider spend.

### 7a. keep what works free

ActionNetwork's NBA regular-season `sharp_public` retrieval is confirmed working (8 of 8 regular-season games on 2025-01-20 returned populated splits at `bookId=15`). That path stays in production for regular-season runs. The playoff null pattern is a real gap, not a reason to abandon the regular-season working path. The same holds for ESPN basketball schedule, statsapi.mlb.com, and the existing Odds API spread retrievals — all Tier 1 and all producing usable signal.

### 7b. record absence honestly

Every missing signal is persisted with a structured reason (`SignalAvailabilityRecord.Reason`, `Detail`). The artifact never silently drops a missing signal. Discern dampens confidence; decide downgrades posture; calibration sees the pattern. Honest absence is cheaper than fake presence and produces better long-term decisions.

### 7c. derive signals you already pay for

The Odds API is already wired for current spread retrieval (`OddsMarketClient`). If the platform polls the same matchup more than once over the day (it does, via caching expiry and reruns), the snapshots can be persisted with timestamps. Computing `line_movement` from those persisted snapshots is **Tier 0 deterministic** — no new provider, no new cost. This is the cheapest realistic path to a usable `line_movement` signal and turns a Tier 1+ external signal into a Tier 0 derived signal. The slice that does this is "Line Movement from Polled Snapshots v1" — not on the recommended next-slice list yet, but documented here as the cheapest upgrade.

### 7d. use public injury reports where terms allow

The NBA publishes an official injury report PDF, and ESPN publishes injury status pages. Both are Tier 1. Both have terms-of-use posture that needs review before automated commercial scraping. A manual review pass during playoffs (Tier 5, human reviewer) is cheaper short-term than building a scraping pipeline; the manual record can populate `injury_context` for high-value runs while a longer-term Tier 1/2 source is negotiated. **Track injury feasibility as its own line item** — for NBA playoffs, a single star scratch (Tatum out, Giannis questionable) has higher decision impact than the sharp/public split. See §8.

### 7e. allow manual review for early calibration

A "manual override" surface in the artifact (today this would mean a developer with database access; later a real internal tool) lets a human-grounded signal land for high-stakes individual runs. This is Tier 5 by definition. It is appropriate during discovery and early validation; it does not scale and does not become production policy.

### 7f. bring-your-own-key for power users

Once the product has paying customers, individual power users may already have SportsDataIO, Rotowire, or other paid subscriptions. A "bring your own key" surface lets them route their own credentials through the platform for their own runs. This adds Tier 3 signal quality without adding Tier 3 cost to the platform. Architecturally it requires the same provider-provenance fields the discovery slice already specified.

### 7g. defer premium providers until a revenue gate is hit

Tier 3 and Tier 4 providers do not enter production until the subscription decision rule fires. Speculative procurement is rejected — even when the product appears stuck on a missing signal. The honest-absence path is always cheaper than a speculative subscription.

---

## 8. sports v1 concrete recommendations

Per-signal feasibility for the current sports assembly line. Costs are current observed or quoted; legal posture is current best understanding subject to formal review before any commercial action.

| signal | current source | cost tier | est. $/mo | legal status | ladder rung | upgrade trigger | defer reason |
|---|---|---|---|---|---|---|---|
| `market` | Odds API via `OddsMarketClient` | 1 | $0 (current free tier) | cleared via published terms | exact_recovery when grounded | call volume exceeds free tier | n/a |
| `sharp_public` | ActionNetwork unofficial | 1 | $0 | unofficial endpoint, no published commercial-use terms; live with risk for v1 | exact_recovery when grounded; unavailable_with_reason during NBA playoffs | sustained NBA playoff null pattern + paid-key Phase A verification of SportsDataIO returns populated splits; budget gate clears | wait for Phase A verification |
| `rest_schedule` | ESPN scoreboard | 1 | $0 | public endpoint, unofficial use | exact_recovery | n/a | n/a |
| `starting_pitching` | statsapi.mlb.com | 1 | $0 | official MLB public API | exact_recovery | n/a | n/a |
| `line_movement` | not implemented | 0 (derived) or 1 (provider) | $0 if derived from polled odds; $0–$30 if from low-cost odds API | depends on derivation path | adjacent_proxy / adjacent regardless of source | sharp_public substitution definitively fails *and* line movement adds enough confidence to justify the slice | adjacent_proxy on the ladder; not an equivalent substitute |
| `injury_report` | not implemented | 1 (NBA report / ESPN HTML) → 3 (Rotowire $9.99–$100/mo) | $0–$100 | NBA injury report PDF likely public-use-safe; ESPN scrape posture needs review; Rotowire requires commercial license | source_substitution candidate for a different decision purpose | calibration shows injury status changes decision quality on NBA / NFL runs more than sharp_public does | track separately as injury feasibility sub-slice |
| `weather` | open-meteo | 1 | $0 | open data license, attribution required | exact_recovery (NFL outdoor) | n/a | NBA / MLB indoor or roofed contexts |
| `situational` | derived (rest, divisional, back-to-back) | 0 | $0 | platform-owned | exact_recovery | n/a | n/a |

### specific calls

1. **Keep ActionNetwork as primary free sharp_public path where it works.** The NBA regular-season retrieval is confirmed working. The playoff null pattern is a known degradation, not a reason to abandon the working regular-season path. Production should not change in this slice.
2. **Do not implement SportsDataIO until Phase A verifies populated NBA playoff splits with an unscrambled key.** Per the verification slice. The data dictionary qualifies the schema; the runtime availability is the open question. Free-trial verification is insufficient because the trial data is scrambled by design.
3. **Do not automate VSiN / DraftKings scrapes without legal review.** The data is verified live and publicly visible. Commercial-use posture is unclear. Treat as discovery-tier only until a sponsor funds the legal pass.
4. **Use low-cost odds APIs only for market-side signals, not sharp_public.** A $30/mo Odds API tier upgrade buys broader market coverage and line movement, not splits. Splits are not a Tier 2 product.
5. **Track injury signal feasibility separately.** For NBA playoffs, the missing signal that probably dominates decision quality is injury status (which star is out tonight?), not sharp/public. The cost / feasibility model for injury context is its own sub-investigation: NBA injury report PDF (free) vs ESPN injury page (scrape risk) vs Rotowire premium ($9.99–$100/mo, commercial license). The ROI is plausibly higher than chasing sharp/public further.
6. **Keep missing high-cost signals visible in the artifact.** Discern dampens confidence; decide downgrades posture; the user sees an honest read with a documented gap. This is the cheapest correct behavior.

---

## 9. relationship to existing doctrine

This slice does not replace any prior doctrine. It adds a business-feasibility layer on top.

- **Cognitive Worker Doctrine** still defines phase ownership and what the model may or may not do.
- **Cognitive Skill Pack Architecture v1** still defines the skill / worker-pack split.
- **Sharp/Public Fallback Ladder v1** still defines equivalence classification per follow-up. The new `costTier` field is orthogonal to the ladder rung; both are recorded.
- **Provider investigations and verifications** continue to live under `04 Products/sports-v1/provider-investigations/`. This doctrine doc references them but does not absorb them.

Calibration always updates the responsible pack. If `availabilityRate` drifts below the documented `confidencePermission` floor for a signal, the fix lands in the ladder-and-feasibility config for that signal, not in `SportsEvaluator` clamps. The pipeline does not change without a deliberate slice.

---

## 10. scope guard

v1 does:

- name six cost tiers
- name the feasibility-record fields
- name the six provider-evaluation gates
- name the six subscription-decision conditions and four budget gates
- name the ROI ledger target fields and identify gaps versus today's artifact shape
- name seven low-cost acquisition patterns
- record current sports v1 per-signal feasibility status

v1 does not:

- implement any provider client (SportsDataIO, VSiN, Rotowire, alternate odds APIs)
- add `CostTier` or `providerName` fields to runtime contracts in this slice
- modify `SportsEvaluator`, `SignalQualityEvaluator`, or `SignalFollowUpEvaluator`
- implement Line Movement Proxy v1 or any line-movement derivation
- implement injury retrieval
- enter procurement on any paid provider

---

## 11. references

- `cognitive-skill-pack-architecture-v1.md` — the architecture this slice extends
- `signal-fallback-ladder.md` — ladder rung classifications; orthogonal to cost tier
- `cognitive-worker-doctrine.md` — phase ownership and platform vs model responsibilities
- `decision-intelligence-model.md` — the strategic frame this slice operationalizes
- `04 Products/sports-v1/provider-investigations/sharp-public-provider-investigation-v1.md`
- `04 Products/sports-v1/provider-investigations/sharp-public-provider-verification-results.md`
- `04 Products/sports-v1/provider-investigations/sharp-public-alternate-provider-discovery-v1.md`
- `04 Products/sports-v1/provider-investigations/sportsdataio-betting-splits-verification-v1.md`
