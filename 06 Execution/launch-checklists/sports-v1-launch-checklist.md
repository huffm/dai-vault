# launch checklist: sports-v1

**product:** NFL pre-game brief
**date:** 2026-04-09

check each item only when it is confirmed working — not when it is assumed working.

---

## data sources

- [ ] the odds api key active in production, returns NFL games and line movement for current week
- [ ] actionnetwork returns sharp/public split for at least 10 games in a test week
- [ ] rotowire api key active, injury report data confirmed fresh (within 2 hours of official report)
- [ ] open-meteo returns correct forecast for 3 outdoor NFL venue lat/lon coordinates
- [ ] collector produces complete output for all 16 games in a test NFL week
- [ ] stale data handling confirmed: collector flags missing fields, does not pass nulls silently

---

## platform

- [ ] tenant record creates and persists correctly with sports-analytics niche config
- [ ] niche config validation rejects malformed input (wrong sport, missing fields)
- [ ] brief envelope schema validates on all collector-to-delivery test runs
- [ ] stripe products created: free, starter ($29/month), pro ($79/month)
- [ ] stripe webhook configured in production and receiving subscription events
- [ ] subscription status check gates brief dispatch correctly (active = send, else skip)
- [ ] tier enforcement confirmed: free = 1 game, starter = all games, pro = all games + alert

---

## agent pipeline

- [ ] collector output passes schema validation for a full NFL week
- [ ] evaluator scores all five signal categories — no null scores on complete input
- [ ] aggregate score and confidence flag compute correctly from evaluator output
- [ ] synthesizer produces a readable 3–5 sentence narrative for 10 test games — no hallucinated player or team details
- [ ] compliance agent runs on each brief and confirms no constraint violations
- [ ] brief schema validation passes end-to-end before delivery step

---

## delivery

- [ ] email brief arrives in test inbox with correct formatting and readable signal table
- [ ] webhook brief delivers valid JSON to test endpoint
- [ ] slack brief delivers correctly to test workspace (pro tier)
- [ ] scheduled trigger fires at correct time for Sunday games (10 AM local)
- [ ] scheduled trigger fires 24 hours before kickoff for Thursday and Monday night games
- [ ] delivery status recorded on brief envelope after every dispatch attempt
- [ ] failed dispatch retries once and marks `failed` with log on second failure

---

## line movement re-alert (pro tier)

- [ ] poller runs every 15 minutes during game-day window (8 AM to kickoff)
- [ ] alert fires when spread moves 2+ points from brief-time value
- [ ] alert brief labeled as update, alert-reason section present
- [ ] only pro tier tenants receive the alert

---

## observability

- [ ] every workflow run produces a log entry: tenant_id, niche, workflow, trigger, start/end time
- [ ] every agent step produces a log entry: role, duration, output summary
- [ ] at least one full workflow run reviewed in logs manually before go-live
- [ ] at least one failure scenario tested and confirmed to produce a usable error log

---

## content quality

- [ ] 10 briefs reviewed manually across at least 2 different NFL weeks
- [ ] signal scoring spot-checked against manually verified game context for 5 games
- [ ] brief reads naturally and does not recommend picks or make outcome predictions
- [ ] delivery timing confirmed against real clock — brief arrives when expected relative to kickoff

---

## go / no-go sign-off

| area | confirmed | notes |
|---|---|---|
| data sources clean for a full week | | |
| platform and billing wired correctly | | |
| agent pipeline quality acceptable | | |
| delivery working in all three channels | | |
| line movement alert tested end-to-end | | |
| one full end-to-end run with a real test account | | |
