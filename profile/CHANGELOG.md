## [1.0.8] — 2026-10-25 — Correctness fixes

### Fixed
- **Data race in LogPipeline** (`_taskBuffer` field shared across concurrent drainer stripes when 2+ sinks configured) — each `ProcessAsync` call now rents a `Task[]` from `ArrayPool<Task>.Shared` and returns it in `finally`
- **Sink exception killed drainer stripe permanently** — per-sink `try/catch` around `EmitAsync`; all non-OCE exceptions increment `SinkFailureCount` and write to stderr; drainer continues; pool return in outer `finally`
- **Pool leak on sink exception** — `ReturnToPool()` moved to outer `finally` block, fires even if every sink throws
- **`TotalDropped` counter always returned 0** — `BoundedChannel.TryWrite` with `DropOldest` always returns `true`; drop detection replaced with atomic per-stripe occupancy counter (`_stripeOccupancy`); race-free under 32 concurrent producers
- **Critical drainer `IndexOutOfRangeException`** — `stripeIdx: _stripeCount` (e.g. 8 on 8-core) caused silent fault on `_rescueQueues[8]` (indices 0–7 only); fixed to `stripeIdx: -1` sentinel; Error/Critical events now correctly drained at shutdown
- **Shutdown race** — replaced plain `_disposed int` with state machine (`StateAccepting=0 / StateDraining=1 / StateStopped=2`) using `Interlocked.CompareExchange`; late producers enqueue safely; `DisposeAsync` is idempotent
- **Category prefix filter false match** — `MinimumLevelFor("Microsoft")` previously matched `"MicrosoftExtra"`; now requires exact match or dot boundary

### Added
- `SinkSharpLogger.ClearCallerCaptures()` — clears static HTTP-context enricher registrations between test instances
- `AsyncLogQueue.SinkFailureCount` — observable counter for sink errors without crashing the drainer
- Benchmark suite: `EndToEndDurabilityBenchmark` now asserts `Processed == Enqueued - Dropped` for all 9 producer/sink combinations (throws `InvalidOperationException` on any violation)

### Test suite
- 230 tests, all passing (was 227 before this release)
- New coverage: critical drainer index sentinel, per-stripe occupancy counter accuracy under 32 concurrent producers, state machine transitions

---

## [SinkSharp.Sinks.Dashboard 1.0.4] — 2026-10-25

### Fixed
- Dashboard in Worker Service and Console App no longer fails to connect — `DashboardServer` was missing `DashboardTokenStore` DI registration causing hub authentication to fail silently
- Dashboard page now loads with the correct auth mode in Worker Service / Console App — `__AUTH_MODE__` placeholder was not replaced at server startup, causing the auth badge to appear and block real-time connections
- CSV export renders special characters and non-ASCII text correctly — UTF-8 BOM added; previously Excel and Windows CSV viewers showed garbled em dashes and accented characters

---

## [SinkSharp.Sinks.File 1.0.2] — 2026-10-25

### Changed
- README updated with SinkSharp.Core 1.0.8 compatibility note and install commands
- No functional changes — safe upgrade from 1.0.1

---

## [1.0.7] — 2026-10-22 — Bug fixes

### Fixed
- **Event delivery guaranteed at all producer counts** — `DrainAndStopAsync()` now awaits all drainer tasks before returning; previously it could return while drainer tasks were still processing events, silently losing the tail of each batch
- **Lock contention eliminated from hot path** — `LogPipeline._enrichers` and `SinkSharpLogger._callerCaptures` previously held locks on every log call; replaced with volatile int count fields written once at startup; contentions reduced from 34+ to near-zero at 8+ concurrent threads

---


## [1.0.7] — 2026-10-22

### SinkSharp.Core 1.0.7

**Bug fixes**
- Event delivery now guaranteed at all producer counts — `DrainAndStopAsync()` previously returned before all drainer tasks finished processing queued events when producers were slow or many. Every enqueued event is now processed before the drain completes.
- Lock contention eliminated from hot path — `LogPipeline._enrichers` and `SinkSharpLogger._callerCaptures` had locks firing on every log call. Both lists are written once at startup and never mutated during logging. Replaced with `volatile int` count fields. Lock contentions reduced from 34+ to near-zero at 8+ threads.

### SinkSharp.Sinks.Dashboard 1.0.4

**Bug fixes**
- Dashboard now connects and streams logs in Worker Service and Console App — `DashboardServer` was missing `DashboardTokenStore` DI registration, causing the SignalR hub to fail on every connection attempt
- Dashboard page loads with correct auth mode in Worker Service and Console App — `__AUTH_MODE__` placeholder was not replaced in `DashboardServer`, causing the auth badge to show and block all connections
- CSV export renders special characters correctly — UTF-8 BOM prepended so Excel and Windows CSV viewers decode em dashes and other non-ASCII log message characters correctly

---

## [1.0.6] — 2026-10-21

### SinkSharp.Core 1.0.6

**New features**
- `AddSinkSharpRequestId()` — DI helper that stamps `RequestId` from `HttpContext.TraceIdentifier` on every HTTP request log event. Requires `builder.Services.AddHttpContextAccessor()`.
- `AddSinkSharpUserContext()` — DI helper that stamps `UserId`, `UserName`, `UserRole` from the authenticated `ClaimsPrincipal`
- `AddSinkSharpRequestPath()` — DI helper that stamps `HttpMethod` and `RequestPath` on every HTTP request log event.
- `AddSinkSharpElapsedTime()` — DI helper that stamps `ElapsedMs` (milliseconds since request start) on every log event within an HTTP request. Requires `app.UseSinkSharp()` in the pipeline.
- `app.UseSinkSharp()` extended — in addition to `X-Correlation-Id` lifecycle, now starts the request stopwatch used by `AddSinkSharpElapsedTime()`.

---

## [1.0.1] — File — 2026-10-21

### SinkSharp.Sinks.File 1.0.1

**Bug fix**
- Log events now appear on disk immediately after being written. Previously Info, Warning, and Debug events were buffered by the OS and only flushed to disk when an Error or Critical event occurred, or when the application stopped. Every write is now flushed immediately.

---

## [1.0.5] — 2026-10-20

### SinkSharp.Core 1.0.5

**New features**
- Added `WithRequestId(accessor)` enricher — stamps `RequestId` from `HttpContext.TraceIdentifier`
  on every log event. Captures on the caller thread (safe — `HttpContext` is not accessible on the
  drainer thread). Works alongside `WithCorrelationId()` — both properties coexist on the same event.
  Requires `builder.Services.AddHttpContextAccessor()`. Register via `AddSinkSharpRequestId()` in DI.
- Added `WithUserContext(accessor)` enricher — stamps `UserId` (sub), `UserName` (name), `UserRole`
  (role) from the authenticated `ClaimsPrincipal`. Safe for anonymous and unauthenticated requests —
. Custom claim types via `WithUserContext(accessor, claims: new[] { "sub", "email" })`.
  Register via `AddSinkSharpUserContext()` in DI.
- Added `app.UseSinkSharp()` middleware — handles the full `X-Correlation-Id` lifecycle in one call:
  reads from incoming header, generates a 12-char GUID if absent, sets `CorrelationContext.Current`
  (flows through all awaits), writes `X-Correlation-Id` to the response. Replaces the manual
  middleware block that every sample previously required. Extension method on `IApplicationBuilder`
  via `SinkSharp.Core.Middleware` namespace.
- Added `WriteTo(sink, minLevel)` overload — per-sink minimum log level independent of the global
  minimum. `LogLevel.None` (the default) means no per-sink guard. Compatible with all sinks including `WithDashboard(out sink, minLevel)`.
- Added `AddSinkSharpRequestId()` and `AddSinkSharpUserContext()` DI registration helpers in
  `ServiceCollectionExtensions` — resolve `IHttpContextAccessor` from the built container after
  `AddSinkSharp()` completes, avoiding the early-container build timing issue.
- Added `WithMachineName()` enricher — stamps `MachineName` standalone, without requiring the
  full `WithEnvironment(app, env)` block. Useful in containerised environments where you want
  the pod hostname only. Reads `Environment.MachineName` once at startup — zero runtime cost.
- Added `WithRequestPath(accessor)` enricher — stamps `HttpMethod` and `RequestPath` from
  `HttpContext.Request` on every log event. Invaluable for correlating logs to specific API routes.
  Register via `AddSinkSharpRequestPath()` in DI.
- Added `WithElapsedTime(accessor)` enricher — stamps `ElapsedMs` (milliseconds since request
  start) on every log event. Requires `app.UseSinkSharp()` to start the stopwatch at request entry.
  Shows exactly when in the request lifecycle each log event occurred — no timestamp arithmetic.
  Web API / Minimal API only. Register via `AddSinkSharpElapsedTime()` in DI.
- Added `.Filter(predicate)` — structured event filter. Events for which the predicate returns
  `false` are dropped before enqueue — zero allocation for dropped events (returned to pool).
  Multiple filters are AND-ed. Example: `.Filter(e => !e.MessageTemplate.Contains("health"))`.
- Added `.WithEnricher(enricher, when: predicate)` — conditional enrichment. The predicate runs
  on the drainer thread; the enricher only runs when it returns `true`. Useful for expensive
  enrichers that should only run for specific levels or categories.
  Example: `.WithEnricher(new ExpensiveEnricher(), when: e => e.Level >= LogLevel.Warning)`.
- Added `SinkSharpLoggerFactory.SetGlobalLevel(LogLevel)` — changes the minimum log level at
  runtime without restarting the application. Backed by a `volatile` field — thread-safe, no lock.
  Set to `LogLevel.None` to restore normal level resolution from startup config.
  Example: call from a management endpoint to temporarily enable debug logs in production.
- Added `AddSinkSharpRequestPath()` and `AddSinkSharpElapsedTime()` DI registration helpers.
- `app.UseSinkSharp()` now also starts a `Stopwatch` in `HttpContext.Items` at request entry
  for use by `WithElapsedTime()` — all per-request SinkSharp concerns in one middleware call.

---

## [1.0.1] — Dashboard — 2026-10-20

### SinkSharp.Sinks.Dashboard 1.0.1

**New features**
- Copy correlation ID to clipboard — a ⧉ button now appears inline next to the correlation ID in
  the detail panel. One click copies the raw ID string. Button shows ✓ for 1.5s to confirm. Does
  not trigger row selection (`stopPropagation`). Falls back to `execCommand` on non-HTTPS connections.
- Shareable URL with active filters — active filters (level, service, correlation, search, time) are
  encoded into the URL hash on every filter change using `history.replaceState`. Format:
  `#level=ERR&service=OrderApi&corr=abc123&q=payment&time=60`. Filters are restored from the hash
  on page load before connecting to the hub — share a pre-filtered view with a colleague by copying
  the browser URL. Hash is cleared when ❌ Clear all is clicked.
- Regex search toggle — `.*` button next to search input switches between plain substring match
  and `RegExp` search. Invalid regex falls back silently and shows a red border on the input.
- Buffer size visible in stats bar — buffer capacity now read from the server stats endpoint
  (`bufferCapacity` field) rather than hardcoded to 500. Reflects the actual `bufferSize` configured
  in `WithDashboard(out sink, bufferSize: N)`.
- CSV export alongside JSON — Export button now offers JSON (existing) and CSV. CSV format:
  `Timestamp,Level,Service,CorrelationId,Message` with proper quoting and escaping. Pure client-side.
- Reconnect indicator with retry count — topbar now shows `Reconnecting (attempt N)…` during
  SignalR automatic reconnect, and flashes `Reconnected` for 2 seconds on successful reconnect,
  before returning to `Live`. Uses SignalR's `onreconnecting` / `onreconnected` events.

---

# Changelog

## [1.0.0] — Dashboard stable — 2026-10-18

### SinkSharp.Sinks.Dashboard 1.0.0

First stable release of the dashboard package. API surface unchanged from `1.0.6-preview`.

**Bug fixes**
- Fixed: Live logs now stream correctly in bearer JWT auth mode without requiring a page refresh —
  `accessTokenFactory` now reads the JWT live from `sessionStorage` on every connect and reconnect
  attempt. Previously a stale closure caused reconnects after token refresh or network hiccup to
  silently stop receiving events until the browser was manually refreshed
- Fixed: JS syntax error (`Uncaught SyntaxError: Unexpected token ';'` at dashboard:751) — a
  misplaced `return` statement inside `selectRow()` caused the entire script block to fail to parse,
  preventing SignalR from connecting in all auth modes

---

## [1.0.4] — 2026-10-18

### SinkSharp.Core 1.0.4

**Bug fixes**
- Fixed: `DisposeAsync()` is now idempotent — calling it multiple times no longer throws
  `ObjectDisposedException`. Uses `Volatile.Read` / `Interlocked.Exchange` guard so the
  `CancellationTokenSource` is only cancelled and disposed once regardless of call count

---

## [1.0.6-preview] — 2026-10-17

### SinkSharp.Sinks.Dashboard 1.0.6-preview

**New features**
- Real project logo embedded in dashboard header (replaces placeholder S# text)
- Theme toggle — dark/light mode switch in topbar (🌙/☀️)
- Time filter — All / 5m / 30m / 1h quick filter in search bar (client-side, no server change)
- Export button — downloads visible events as JSON file

**UX improvements**
- Topbar restructured as 3-column grid — auth pills in centre, actions never overflow
- Service column widened (128px) with `minmax` on message — full names now visible
- Properties panel uses CSS grid (130px key column) — aligned two-column layout
- Detail panel taller (260px) — more property rows visible without scrolling
- Dark mode contrast improved — text, borders, sidebar labels clearly readable
- Critical level added as separate sidebar filter (was grouped with Error)

**Bug fixes**
- Message column blank — JS was reading `renderedMessage`, server sends `message`
- `[object Object]` for HttpMethod — now unwraps structured log objects:
  `{Value:"GET"}`, `{value:"GET"}`, `{method:"GET"}` (single-key) all → `GET`
- `null` JS errors on renderAll/clearAll — empty-state no longer a persistent DOM node
- `appendChild` errors — clearAll, filterByCorr, setLevel all fixed
- CRT (Critical) now shown when Error level filter is active
- Service filter uses exact short name match (was `.includes()`, too loose)
- Auto-scroll re-enables when user scrolls back to top
- Export URL properly revoked after download (memory leak fix)
- Copy JSON falls back to `execCommand` on HTTP (non-HTTPS) connections

---

## [1.0.5-preview] — 2026-10-13

### SinkSharp.Sinks.Dashboard 1.0.5-preview

**New features**
- Added `DashboardOptions` — auth and access control for the dashboard via `UseSinkSharpDashboard()` overload:
  - `opts.RequireAuthorization()` — any authenticated user (uses your app's existing auth)
  - `opts.RequireAuthorization("PolicyName")` — named policy you define in `AddAuthorization()`
  - `opts.RequireRole("Admin", "DevOps")` — inline role check, no named policy needed
  - `opts.UseToken("secret")` — shared password, no `AddAuthentication()` needed
  - `opts.AllowFrom("10.0.0.0/8")` — IP allowlist, no user auth needed
  - `opts.UseBasicAuth("user", "pass")` — HTTP Basic auth, no `AddAuthentication()` needed
  - `opts.ConfigureEndpoints(ep => ...)` — full control escape hatch for custom policies,
    rate limiting, CORS, or any other ASP.NET endpoint configuration
- All auth options apply to hub and stats endpoints consistently
- Default behaviour unchanged — no auth when `configure` is not passed

---

## [1.0.4-preview] — 2026-10-12

### SinkSharp.Sinks.Dashboard 1.0.4-preview

**New features**
- Added `AddSinkSharpDashboardRedis()` — reuse your existing Redis instance as a
  SignalR backplane for multi-pod / Kubernetes deployments. All replicas stream logs
  to the same dashboard session. Safe to share with IDistributedCache, session,
  rate limiting — uses isolated `SinkSharp` channel prefix by default.
  Requires `Microsoft.AspNetCore.SignalR.StackExchangeRedis` NuGet package.

**Bug fixes**
- Fixed: Dashboard hub and stats URLs now derived from `window.location.pathname` at runtime
  instead of being injected server-side — works correctly with `app.UsePathBase()`,
  reverse proxies, and Kubernetes path prefixes (e.g. `/labor`).
  Previous server-side `__SINKSHARP_PATH__` injection caused 404 on stats and SignalR
  hub when deployed with a path prefix regardless of `UsePathBase` ordering in `Program.cs`

---

## [1.0.3] — 2026-10-12

### SinkSharp.Core 1.0.3

**Performance**
- `EnsureCapacity()` called before `ExtractFromTokens` — pre-sizes the property dictionary to the exact
  number of tokens in the template, eliminating the resize on first `Add()` for every log event
- All v1.0.2 bug fixes included

---

## [1.0.2] — 2026-10-12

### SinkSharp.Sinks.Dashboard 1.0.2-preview

**Bug fixes**
- Fixed: Dashboard hub and stats URLs now derived from `window.location.pathname` at runtime
  instead of being injected server-side — works correctly with `app.UsePathBase()`,
  reverse proxies, Kubernetes path prefixes (e.g. `/labor`), and any `UsePathBase` ordering.
  Previous server-side injection caused 404 on `/sinksharp/dashboard/stats` when
  deployed with a path prefix because the PathBase was not included in the injected URL

### SinkSharp.Core 1.0.2

**Bug fixes**
- Fixed email regex: `[A-Z|a-z]` corrected to `[A-Za-z]` in `PiiMaskingEnricher`
  (`|` inside `[]` is a literal character, not alternation — no functional impact
  since no real email TLD contains `|`, but technically incorrect)
- Fixed `CorrelationId` captured on caller thread in `SinkSharpLogger.BuildEvent()`
  (`AsyncLocal` is not visible on the drainer thread — correlation ID was blank on log events)
- Fixed `PendingCount` uses safe counter arithmetic (`TotalEnqueued - TotalProcessed - TotalDropped`)
  (`ChannelReader.Count` throws `NotSupportedException` on striped channel configurations)

---

## [1.0.1] — 2026-10-11

### SinkSharp.Sinks.Console 1.0.1

**New features**
- Added `showFullCategory` parameter — show full namespace (e.g. `MyApp.Services.OrderService`)
  like Serilog `{SourceContext}`; default `false` (short name only)
- Added `showSourceContext` parameter — include full category in the properties output line;
  default `false`

### SinkSharp.Sinks.Dashboard 1.0.1-preview

**Bug fixes**
- Fixed: SignalR event name `onLog` → `log` — events were not appearing in the dashboard
- Fixed: `LevelShort` switch — `Warning` produced `"War"` instead of `"WRN"`, breaking the level filter
- Fixed: Hardcoded `/logs/hub` and `/logs/stats` paths replaced with `__SINKSHARP_PATH__` placeholder —
  dashboard failed when mounted on any path other than `/logs`
- Fixed: `SetQueue()` now correctly wired in both `UseSinkSharpDashboard()` and `DashboardServer` —
  queue stats showed `0 total` for Worker and Console apps
- Fixed: `DashboardServer` accepts queue via constructor parameter — separate DI container
  could not resolve the queue from the host DI container

---

## [1.0.0] — 2026-10-06

### Initial release

**Core**
- Striped async channel queue (one channel per CPU core) — eliminates producer lock contention
- `MessageTemplate` caching — zero parse cost after first call per unique template
- `LogEvent` object pool — reduced GC pressure under sustained load
- Two-channel design: bounded for Info/Debug/Warn (DropOldest), unbounded for Error/Critical (never dropped)
- MEL `ILogger<T>` compatible — no application code changes needed

**Enrichers**
- `WithCorrelationId()` — propagates correlation ID via `AsyncLocal`, set from middleware
- `WithEnvironment(appName, env)` — adds app name, environment, machine name to every event
- `WithPiiMasking()` — masks emails, credit card numbers, SSNs, phone numbers in properties and rendered message
- Per-category minimum level overrides via `MinimumLevelFor(prefix, level)`

**Sinks**
- `ColoredConsoleSink` — ANSI-coloured structured output
- `RollingFileSink` — daily rolling JSONL files (Compact Log Event Format compatible)
- `DashboardSink` — embedded Kestrel + SignalR real-time dashboard at localhost:5341 (preview)

**Integration**
- `AddSinkSharp()` — one-call DI registration, drains on shutdown
- `AddSinkSharpWithHealth()` — includes ASP.NET Core health check
- `SinkSharpHealthCheck` — Healthy/Degraded/Unhealthy based on queue pressure
- `AzureContextEnricher` — region, resource group, slot name, pod name

**Performance** (vs Serilog, .NET 9.0.20 Windows 11 X64)
- 4.0×–4.5× lower hot-path latency
- Zero lock contentions at 32 concurrent producers
- `Enqueued == Processed` verified end-to-end
