# v1 scope: sports-v1

## current dev slice note
- the live `apps/sports-app/` matchup analyzer currently supports NFL, NBA, and MLB in the thin frontend
- this does not change the commercial v1 priority: NFL and NBA remain the first product focus until brief quality and delivery prove out

## in scope
- NFL games (regular season + playoffs)
- NBA games (regular season + playoffs)
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
- soccer, NHL, college sports (NCAAF, NCAAB)
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
