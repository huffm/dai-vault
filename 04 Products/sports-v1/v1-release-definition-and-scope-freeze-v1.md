---
title: "Sports-v1 V1 Release Definition and Scope Freeze v1"
type: "product"
date: "2026-07-14"
status: "active"
project: "DAI"
slice: "V1 Release Definition and Critical Path v1"
repos:
  dai: "unchanged (read-only audit)"
  dai-vault: "docs-only"
tags:
  - product
  - release
  - scope
related:
  - "04 Products/sports-v1/buyer-validation-brief-v1.md"
  - "04 Products/sports-v1/v1-scope.md"
  - "04 Products/sports-v1/buyer-copy-safety-v1.md"
  - "06 Execution/plans/v1-release-critical-path-2026-07-14-v1.md"
  - "06 Execution/reports/hardened-regime-baseline-measurement-2026-07-11-v1.md"
  - "06 Execution/reports/gate4-evidence-readout-v2-day2-2026-07-11-v1.md"
---

# sports-v1 v1 release definition and scope freeze v1

**This document freezes the V1 pilot product. Where it conflicts with the broader
2026-04-19 `v1-scope.md` (multi-sport, delivery tiers, webhook alerts), THIS document
governs the V1 pilot; the older scope remains the long-range map.** It extends, and does
not replace, the ACTIVE `buyer-validation-brief-v1.md` (target buyer, $29/mo concierge
offer, claim-safe copy, outreach plan) -- V1 is that brief made deliverable on a schedule.

## 1. the product

A **private MLB decision brief**: one compact, structured, traceable pre-game read per
selected matchup, delivered to a paying pilot buyer, followed by a settled-outcome recap
after the game completes.

Each brief contains, per the buyer output contract (section 3):

- a directional lean OR an explicit no-position read
- evidence strength as a qualitative, evidence-gated band (never a numeric probability)
- market context (whether the read agrees or diverges from the books, in plain language)
- key supporting evidence (starters, availability, situational factors -- grounded only)
- material uncertainty and counter-case
- stable game identity (teams, date, gamePk) and generation timestamp
- after completion: the final score, and what the read said versus what happened

**The V1 promise is structured, traceable decision support.** It is a workflow product
(replaces 30-45 minutes across five tabs with a one-minute read), not a picks product.

## 2. target buyer, offer, access, payment

Unchanged from `buyer-validation-brief-v1.md`:

- **buyer:** the serious recreational bettor who does their own pregame research.
- **offer:** $29/month, month-to-month, cancel anytime, first read free.
- **delivery:** concierge -- the operator generates each read on the local stack and
  delivers the rendered buyer-safe brief by email; the settled recap follows the next
  morning. No hosted deployment, no account creation, no login for pilot buyers.
- **payment:** manual Stripe payment link (stripe = truth). Entitlement = the paid
  Stripe receipt plus the operator's delivery ledger. No checkout code, no billing
  integration, no entitlement schema in V1.
- **access proof for the release gate:** at least one Stripe-paid pilot user receiving
  briefs on the delivery ledger.

## 3. buyer-facing output contract

Rendered brief fields (single deterministic export per run; the buyer app remains the
operator's console):

| field | source | rule |
|---|---|---|
| matchup + game identity | persisted run (teams, gameDate, gamePk) | echoed from the API, never from form state; gamePk and generated-at timestamp always present |
| read stance | posture (play/pass/monitor/wait/compare/avoid) + lean | lean absent -> explicit "no clear lean / no position" rendering (already implemented) |
| evidence strength | evidence-gated qualitative band (existing Signal Summary band: High/Medium/Low + Confirmation strength) | NEVER the numeric confidence; NEVER the 0.70/0.45 threshold "Strong/Mixed/Weak" labels |
| market context | persisted market fields (agreement, consensus side, book count) | plain language ("the books lean the same way" / "this read diverges from the books"); no implied superiority |
| supporting evidence | sanitized buyer fields (summary, factors) | claim-safe per `buyer-copy-safety-v1.md`; grounded signals only |
| uncertainty / counter-case | counterCase, watchFor, whatWouldChangeTheRead | rendered when populated |
| settled recap | evaluation + outcome after finals | final score, what the read said, what happened; per-read traceability only -- no aggregate record claims |

**Numeric confidence decision: OMIT from every buyer surface.** The 15-row v2 evidence
shows inverted discrimination (0.80 band underperforms 0.75); any numeric or
monotone-implying label ("75%", "Strong") is an unsupported claim. The number remains an
internal diagnostic on operator/dev surfaces only. Buyer-facing strength language is the
existing evidence-sufficiency band, which is grounded in evidence depth (a fact), not in
outcome prediction (an unproven claim).

## 4. competition, cadence, responsibilities

- **competition:** MLB only. (NBA remains buyer-ready in the app but is off-season and
  out of the pilot promise.)
- **cadence:** one delivery batch per covered slate day, up to 4 briefs/day, operator
  selects matchups; settled recaps the following morning. Coverage days announced to the
  buyer a day ahead; no guaranteed daily coverage (doubleheaders, break days, operator
  availability).
- **manual (operator):** matchup screening and selection, run generation (registry v2
  hardened route via the process-scoped flag), brief rendering and email delivery,
  settlement, recap delivery, payment links, delivery ledger, failure recovery.
- **automated (platform):** retrieval, analysis, persistence, identity resolution,
  attribution guards, prompt-trace observability, settlement idempotency and residue.

## 5. non-goals (V1 must not)

Claims discipline -- the product must not claim or imply:

- guaranteed or profitable picks; "edge", "best bet", "lock", "value" language
  (enforced by `buyer-copy-safety-v1.md` sanitization)
- statistically validated predictive advantage
- calibrated win probability (no numeric confidence on any buyer surface)
- superiority over market consensus
- any aggregate performance record (the 9/6 v2 record is n=15 noise; Gate 4 forbids
  conclusions)

Scope discipline -- V1 excludes:

- hosted deployment, Entra production registration, self-serve accounts, login for
  buyers, CI/CD (the operator console runs locally; deployment shape exists for later)
- payment/entitlement/billing code (manual Stripe link only)
- NFL/NBA/NCAA coverage, props, live betting, line-move re-alerts
- real history page, saved reads, account page (both remain mocked; recap delivery
  covers outcome traceability)
- dashboards, schedulers, delivery automation, webhooks
- prompt/model/threshold/confidence/scoring changes of any kind (locked layers; the
  15-row sample tunes nothing)
- doubleheader capture operation (independent evidence activity under its own
  authorization; not a release dependency)
- identity-status refinement, WI-0002 chip alignment, WI-0003 shared module (dormant/
  post-V1)

## 6. outcome interpretation (what the evidence supports)

From the 2026-07-11 cadence closeouts (15 active settled v2 rows, 9 correct / 6
incorrect; attribution Pass 14 / FAIL 0 / Unclear 1; Gate 4 conclusionsAllowed=false;
discrimination inverted -0.1486; market-opposed n=8 unreadable):

**Supported and shippable:** pipeline reliability (16/16 captures, 0 guard FAIL, 0
identity errors); v2 market-attribution hardening (no attribution mismatch ever on the
v2 route; the one deliberate divergence named the market-favored team); settlement
integrity (idempotent, full residue, correct-run attachment proven); an existing,
working abstention path (lean-null "no clear lean" + pass/avoid postures); cost
~$0.0007 model spend per read.

**Not supported:** any accuracy, calibration, discrimination, or market-beating claim.
The 0.80 confidence band underperforms the 0.75 band -- numeric confidence is
anti-informative at this sample and must not reach buyers.

**Fields safe for buyers:** stance/lean, no-position, sanitized prose fields,
evidence-gated band, signal availability aggregate, source depth, game identity,
timestamps, plain-language market agreement, per-read settled outcome.

**Fields requiring revised semantics before buyer exposure:** numeric confidence and
the threshold-derived "Signal quality: Strong/Mixed/Weak" labels (currently rendered
from the raw POST response) -- removed from buyer surfaces by WI-0011.

**No release-blocking outcome-pattern correction exists.** The blocking work is
presentation conformance, not model behavior. Nothing is tuned from n=15.

## 7. release acceptance criteria (RC gate, 2026-07-31)

The release candidate must demonstrate, end-to-end on the real local stack:

1. the Stripe payment-link and entitlement-ledger workflow has been successfully
   dry-run using a non-production or explicitly marked test transaction and recorded
   on the delivery ledger (corrected 2026-07-14 under the WI-0011 authorization; the
   FIRST REAL paid entitlement and Stripe receipt remain required for pilot
   validation, not by RC -- and per the 2026-07-15 operator decision, 2026-08-07 is
   the earliest planning target for a paid private pilot if commercial activation is
   separately authorized, not an active commitment while outreach, buyer contact, and
   real delivery remain deferred)
2. an upcoming MLB matchup is selected from the live schedule
3. one analysis generates with no duplicate active run (creation guard + runbook check)
4. the rendered buyer brief is comprehensible and matches the section-3 contract
5. zero unsupported claims on the brief (claim-safe checks pass; no numeric confidence)
6. game identity (teams/date/gamePk) is identical across persisted run, prompt-trace,
   and rendered brief
7. a forced failure (source outage or model error) is observable and recoverable per
   runbook
8. the model call and external-source usage for the run are recorded (cost log with
   correct pricing for the configured model)
9. settlement attaches to the correct run with full residue
10. the settled recap renders and is deliverable to the buyer
11. an operator executes the entire daily workflow from the runbook alone, including
    one recovery path
12. the whole drill is documented as the RC record in the vault

## 8. pilot success metrics (evaluated 2026-08-21)

Primary (economic):

- **>= 1 paying pilot customer** (Stripe receipt; money is the only validation)
- **revenue per unit of operator attention:** $/operator-hour across the pilot
  (operator minutes logged per delivery day on the ledger; target: a clear trend toward
  <= 30 operator minutes per delivery day by week 2)

Secondary (behavioral):

- repeat usage: the buyer reads briefs on >= 3 distinct slate days per week (email
  opens/replies or stated usage at check-in)
- delivery success: >= 95% of promised briefs delivered before first pitch
- cost to serve: model + odds spend per delivered brief (basis ~$0.0007 model/read +
  odds calls; tracked from the cost log)
- manual intervention: unplanned interventions per delivery day (target: declining
  week over week)
- buyer comprehension: at check-in, the buyer can restate a brief's stance and its main
  caveat without operator explanation
- retention: the buyer is still paying (not canceled) at the 2026-08-21 review

**Non-validation:** signups, page views, verbal interest, free-sample consumption
without conversion.
