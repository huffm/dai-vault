# observability

**date:** 2026-04-13
**status:** reflects what is implemented today — not a target state

---

## what is currently in place

### correlation id flow

every `.NET` → FastAPI call carries an `X-Correlation-Id` header. the value is `Activity.Current?.Id` (ASP.NET Core distributed trace ID) falling back to a fresh `Guid` if no trace is active. FastAPI logs it on every request.

- `.NET` side: `FastApiClient.cs` adds the header per call using `HttpRequestMessage` / `SendAsync`; typed `HttpClient` singletons share no state across calls
- Python side: `app/routes/sports.py` reads `request.headers.get("x-correlation-id", "none")` and logs it at INFO alongside sport, home, away, and game date

### structured logging in .NET

`ILogger<T>` is used throughout `AgentRunService` and `FastApiClient`. log levels:
- INFO: normal request flow, cache hits/misses, schedule results
- WARNING: soft failures (no odds API key mapping for a sport slug)
- ERROR: non-2xx from FastAPI (with truncated response body), Odds API failures, missing API key

### structured logging in FastAPI

`app/routes/sports.py` logs at INFO on every inbound analysis request:
```
sports analyze request: correlation_id=... sport=... home=... away=... date=...
```
errors also include `correlation_id`.

### schedule lookups (OddsScheduleClient)

- cache hit/miss logged at INFO with cache key
- zero-match result logged at INFO with a note to verify team names
- API errors logged at ERROR with status code and response body
- missing API key logged at ERROR

---

## what is not in place yet

- Application Insights / Azure Monitor not configured in `Program.cs`
- no distributed trace parent propagation — correlation IDs are point-to-point headers, not a trace context
- no per-run timing breakdown at the step level (only total `DurationMs` on the `AgentRun` row)
- no tenant-scoped log filtering
- every workflow run does not yet log: tenant_id, niche, trigger start/end (see `sports-v1-roadmap.md` phase 3.5)
- every agent step does not yet log: role, duration, output summary

---

## when to add more

add Application Insights before the first real deployment. the existing `ILogger<T>` wiring means adding `AddApplicationInsightsTelemetry` in `Program.cs` is the only code change required.

do not add distributed tracing infrastructure (OpenTelemetry, Jaeger) until there are multiple services producing enough traces to make it worth the operational cost.
