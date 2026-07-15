---
title: "Sports-v1 Buyer Validation Brief v1"
type: "product"
date: "2026-07-06"
status: "RETAINED, OUTREACH DEFERRED -- validation plan kept ready; outreach and buyer contact NOT authorized (operator decision 2026-07-15); activation requires a separate explicit operator decision"
project: "DAI"
related:
  - "04 Products/sports-v1/product-brief.md"
  - "04 Products/sports-v1/target-customer.md"
  - "04 Products/sports-v1/value-proposition.md"
  - "04 Products/sports-v1/v1-scope.md"
  - "03 Niches/sports-analytics/positioning.md"
---

# sports-v1 buyer validation brief v1

## 1. purpose

this is a first-subscriber validation brief, not a launch plan. it defines the smallest
honest test of whether one person will pay for the sports-v1 decision artifact on
workflow value alone, before any further platform machinery is built. money is
validation; nothing else on this page is. it sells the sports-v1 read, never the dai
platform.

## 2. target buyer

one primary segment only (from target-customer.md): the serious recreational bettor --
bets nfl/nba (and in-season mlb) regularly at meaningful stakes, does their own pregame
research today, has paid $30-$100/mo for a tout service and found it low-accountability,
and wants context to make their own call rather than someone else's conclusion.
secondary segments (dfs players, sports media writers) are out of scope for this test.

## 3. pain statement

the pain is time and fragmentation, not the absence of winners: 30-45 minutes per game
across five tabs (odds, injuries, weather, movement, situational context), re-checked
under time pressure, with the synthesis left entirely to the buyer. the result is
decision fatigue and the nagging feeling of having missed one thing. this brief does
NOT frame the pain as needing guaranteed winners or market-beating selections; a buyer
who only has that pain is not this product's buyer (see stop criteria).

## 4. claim-safe value proposition

workflow value only: turn a 30-45 minute pregame review into one compact structured
read you can scan in about a minute -- line move since open, availability, rest/travel,
where the signals agree and where they conflict, all timestamped, with the reasoning
visible. the read shows a stance (play, pass, monitor, wait, compare, avoid) and why;
it does not tell the buyer what to bet, and it makes no claim about being right more
often than the market or anyone else.

## 5. one concrete offer

$29/month starter (the doctrine's indicative starter anchor), run as manual concierge:
- one compact read per covered game-day slate, delivered by email.
- starts on live mlb reads now (mlb is inside v1-scope.md's in-scope list; nfl/nba are
  off-season as of 2026-07); nfl/nba coverage when their seasons start.
- month-to-month, cancel anytime, first read free as the sample.
- payment collected manually (invoice/payment link handled by the founder); no stripe
  implementation, no checkout flow, no account creation required for subscriber 1.

## 6. one acquisition channel

direct one-to-one outreach in a single existing community where serious recreational
bettors already discuss their own research (one subreddit or one discord; founder
selects the specific venue and follows its self-promotion rules). no ads, no content calendar,
no platform marketing, no second channel until subscriber 1 completes the loop.
conversations start by offering one free sample read for a game the prospect already
cares about -- mirroring the conversion trigger in target-customer.md.

## 7. claim-safe outreach copy

send-ready message (adjust the game reference to the prospect's context):

> i'm building a pregame brief for people who do their own research. one game, one
> compact read: line move since open, availability, rest and travel, where the signals
> agree and where they conflict -- timestamped, reasoning visible, scannable in about a
> minute. it doesn't tell you what to bet; it shows you the picture so you can make
> your own call, instead of spending 30-45 minutes across five tabs. i'm looking for
> one serious person to trial it at $29/month by email, cancel anytime. want a free
> sample read for tonight's slate to see if it's useful?

## 8. forbidden claims list

banned in all buyer-facing language for this test (and until gate 5 unlocks
performance claims, most of it permanently):

- pick / picks (as product language; the ui word is read stance)
- lock / locks
- guaranteed / guarantee
- edge
- beat the market / market-beating
- profitable / profit
- win rate / accuracy / hit rate / any percentage of correctness
- sharp (as in "sharp plays" / self-description)
- model says / the model likes
- ai prediction / ai-powered picks
- any sportsbook-router or "bet now" language, deep links, or promo-code framing

## 9. manual fulfillment path for subscriber 1

no new machinery. serve subscriber 1 with what exists:

- generation: the existing artifact flow (matchup analyzer pipeline) produces the read;
  founder triggers it manually per slate. spend for these runs follows the existing
  approval-gated posture and is the founder's call outside this document.
- delivery: founder sends the read by email manually (the fixed brief format from
  v1-scope.md); no scheduler, no delivery automation.
- account/payment: manual invoice or payment link; a spreadsheet row is the account
  system; no auth slice, no stripe code, no dashboard.
- feedback: founder asks two questions after each week -- "what did you check anyway?"
  (residual tab-opening = value gap) and "would you pay again next month?" -- and logs
  answers manually in this folder.
- buyer-facing sanitization note: reads sent to the buyer use the buyer surface only;
  nothing from calibration internals, confidence plumbing, or gate machinery appears.

## 10. success criterion

one paying subscriber completes the full loop:
interest -> pays $29 -> receives reads for a real slate week -> gives feedback ->
would pay again (says so, or actually renews). partial credit (interest without
payment, payment without engagement) is signal, not success.

## 11. stop criteria

stop or revise the test if any of these hold:

- no willingness to pay after the free sample across a reasonable number of genuine
  prospects (workflow value alone is not clearing $29).
- prospects ask only for picks, locks, or performance guarantees -- wrong buyer or
  wrong framing; do not chase them by drifting toward performance claims.
- the value proposition only converts if we make locked gate-5 performance claims --
  then validation is blocked on evidence, not marketing, and the test ends.
- fulfillment turns out to require new platform machinery before payment -- then the
  premise of this brief is false and it returns for revision.

## 12. what this does not authorize

- no platform marketing
- no stripe implementation
- no auth/account work
- no dashboard
- no tuning
- no buyer performance claims
- no new sports scope (mlb concierge is existing v1 scope, not an expansion)
- no new capture
- no reconciliation
