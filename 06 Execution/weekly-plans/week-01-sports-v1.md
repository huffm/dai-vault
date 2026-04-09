# week 01: sports-v1

**dates:** week of 2026-04-09
**phase:** 1 — research and foundation
**goal:** data sources confirmed, tenant model implemented, collector working for one game

---

## focus for this week
do not write application logic until the data is confirmed. the most common early mistake is building the pipeline before knowing what the sources actually return. this week is about reading before writing.

---

## monday — data source audit

**morning**
- [ ] sign up for the odds api, run the NFL odds endpoint, inspect the response shape
- [ ] document: which books are returned, what movement history looks like, what the update frequency is
- [ ] confirm the `historical/odds` endpoint is accessible on your tier

**afternoon**
- [ ] test open-meteo with 3 NFL outdoor stadium lat/lon coordinates (e.g. Arrowhead, Lambeau, MetLife)
- [ ] confirm hourly forecast returns wind speed, wind direction, precipitation, and temperature
- [ ] start a `data-source-notes.md` scratch file: one section per source, what it returns and what it doesn't

---

## tuesday — data source audit continued

**morning**
- [ ] test actionnetwork: confirm which page or endpoint returns sharp/public split per game
- [ ] note whether this is a scrape or an api — document the approach and fragility risk
- [ ] spot check 3 recent games: do the percentages look plausible?

**afternoon**
- [ ] sign up for rotowire, pull the NFL injury report for the current week
- [ ] confirm: player name, team, position, injury status, last update timestamp are all present
- [ ] compare one team's rotowire injury report against the official NFL injury report for the same week — confirm freshness

**end of day**
- [ ] data-source-notes.md complete with confirmed field shapes and any gaps or risks noted

---

## wednesday — platform primitives

**morning**
- [ ] implement the tenant record: tenant_id, niche, tier, stripe fields, delivery config, niche config
- [ ] implement niche config for sports-analytics: sport list, delivery timing hours, alert threshold
- [ ] write a simple test: create a tenant, read it back, confirm fields persist

**afternoon**
- [ ] implement the brief envelope schema: brief_id, sections[], trigger, generated_at, delivery_status
- [ ] confirm the schema accepts the section types defined in output-brief-schema.md
- [ ] set up stripe: create the three products (free, starter $29, pro $79) in test mode

---

## thursday — collector first pass

**morning**
- [ ] write the collector for the odds api: fetch one specific NFL game, return structured lines object
- [ ] add line movement: opening spread, current spread, delta
- [ ] confirm structured output matches the game data object spec in collector-prompt.md

**afternoon**
- [ ] add rotowire collector: fetch injury report for both teams in the test game
- [ ] add open-meteo collector: fetch forecast for the venue (outdoor only, skip if indoor)
- [ ] add actionnetwork collector: fetch sharp/public split for the same game
- [ ] write stale data detection: flag any field where the source timestamp is older than threshold

**end of day**
- [ ] collector returns a complete, well-structured game data object for one known NFL game
- [ ] all four sources integrated and returning data

---

## friday — integration test and review

**morning**
- [ ] run the collector end-to-end on 3 different NFL games from the current or most recent week
- [ ] inspect output: are all fields present? are stale flags firing correctly?
- [ ] identify any source that returned unexpected nulls or malformed data — fix or flag

**afternoon**
- [ ] review the evaluator prompt doc (prompts/evaluator-prompt.md) against actual collector output
- [ ] note any signal categories where the collector output doesn't give the evaluator enough to score
- [ ] update collector output fields if gaps are found (do not change the prompt yet — note the gap)
- [ ] write up a brief end-of-week note: what's confirmed, what needs follow-up before building the evaluator

---

## week 01 exit criteria
- [ ] all four data sources confirmed working with real data
- [ ] tenant record and niche config implemented and tested
- [ ] brief envelope schema implemented
- [ ] stripe products created in test mode
- [ ] collector produces complete structured output for at least 3 NFL games

---

## not doing this week
- evaluator, synthesizer, delivery — do not start until collector output is stable
- prompt finalization — review and note gaps, finalize next week
- billing gate — stripe is set up but not wired into the pipeline yet
