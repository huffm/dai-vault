# v1 scope: sports-v1

## in scope
- NFL games only (regular season + playoffs)
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
- NBA, MLB, college sports
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
- delivery works via email and webhook
- at least one paying subscriber before expanding to NBA
