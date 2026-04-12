# data-sources: sports-analytics

## primary sources

### the odds api
- endpoint: odds, lines, books, movement history
- v1 sports: NFL (`americanfootball_nfl`), NBA (`basketball_nba`)
- also covers: MLB, NHL, NCAAF, NCAAB — not used in v1
- cost: tiered by request volume
- use: core line data and sport schedule for both NFL and NBA

### actionnetwork (scrape or api)
- endpoint: public betting percentages, sharp money indicators, line movement history
- cost: free scrape; paid api available
- use: sharp vs public split — most actionable signal after raw line data

### rotowire api
- endpoint: structured injury reports with position and depth impact
- covers: NFL and NBA (confirm basketball injury feed is available on the plan tier used)
- cost: paid, reasonable for the data quality
- use: availability signals for both NFL and NBA
- note: prefer over espn unofficial scraping — espn has ToS risk and no stable api contract

### open-meteo or weather.gov
- endpoint: hourly forecast by lat/lon
- cost: free
- use: weather signals for NFL outdoor games only — skip entirely for NBA (indoor arenas)

## secondary sources (v1 candidates)
- pro football reference: NFL historical matchup and situational data
- basketball reference: NBA historical matchup, back-to-back schedule data, rest differential
- fantasypros: secondary injury aggregation if rotowire is not yet licensed for both sports

## data freshness requirements
| signal | refresh frequency |
|---|---|
| lines | every 15 minutes during window |
| injury reports | daily, plus on-demand before game |
| sharp/public split | every 15 minutes during window |
| weather | 6 hours before game, once |
| historical | static per season |

## access pattern
all data pulled by collector agent at workflow trigger time. no streaming or websocket in v1. results cached per game until brief is dispatched.
