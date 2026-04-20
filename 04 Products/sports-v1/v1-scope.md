# v1 scope: sports-v1

## current dev app frontend

the dev app at `apps/sports-app/` (port 4201) has a thin angular shell with three routed pages. this documents what is real vs mocked as of 2026-04-19.

**real and wired:**
- matchup analyzer at `/analyzer` — sport/level/team/date selection, live api calls, real analysis pipeline
- top header with 3-item nav — Matchup Analyzer, History, Account — route-driven, responsive
- lazy-loaded routing for all three pages

**mocked (client-side only, no backend):**
- history page at `/history` — styled page with six sample reads; filter between All / Saved; save toggle is in-memory only; no reads are persisted; no real run history fetched from the api
- account page at `/account` — static placeholder for profile, plan, delivery, and security sections; no real auth, billing, or delivery wiring

**not yet built:**
- real run history api and history page data binding
- real account management, auth, or billing integration
- saved reads persistence
- any delivery, scheduling, or subscription flows

this section describes the dev app frontend only. the backend v1 product scope is below.

---

## current dev slice note
- the live `apps/sports-app/` matchup analyzer is now competition-aware
- currently supported combinations:
  - football + pro → NFL
  - football + college → NCAAF
  - basketball + pro → NBA
  - basketball + college → NCAAMB
  - baseball + pro → MLB
- baseball + college is intentionally visible as unsupported
- this does not change the commercial v1 delivery priority: pro competitions remain the first commercial focus until brief quality and delivery prove out

## in scope
- NFL games (regular season + playoffs)
- NCAAF games
- NBA games (regular season + playoffs)
- NCAAMB games
- MLB games (regular season + playoffs)
- pre-game brief per game, covering:
  - current spread and total
  - line movement summary (opening vs current)
  - sharp vs public money split
  - injury and availability summary
  - weather flag (outdoor games)
  - situational context (rest, travel, home/away)
  - aggregate signal score
  - 3-5 sentence narrative
- email delivery for starter tier
- slack/webhook delivery for pro tier
- line movement re-alert if spread moves 2+ points post-delivery (pro only)

## out of scope for v1
- soccer, NHL
- college baseball
- player prop coverage
- in-game or live betting signals
- outcome tracking or model calibration
- brokerage or book integrations

## key decisions for v1 build
- data sources: the odds api (lines), actionnetwork (sharp/public split), rotowire (injuries), open-meteo (weather)
- trigger timing: 10 AM local on game day for standard Sunday games; 24h before kickoff for Thursday/Monday night games
- brief format: fixed markdown template rendered to email and/or webhook payload

## definition of done for v1
- brief produces correctly for all games in a test NFL week
- brief produces correctly for all games in a test NBA week
- delivery works via email and webhook for both sports
- at least one paying subscriber before expanding to soccer or other sports
