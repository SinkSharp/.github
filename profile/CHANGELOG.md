# Changelog

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
  - `opts.AllowFrom("10.0.0.0/8")` — IP allowlist, no user auth needed
  - `opts.UseBasicAuth("user", "pass")` — HTTP Basic auth, no `AddAuthentication()` needed
  - `opts.ConfigureEndpoints(ep => ...)` — full control escape hatch for custom policies,
    rate limiting, CORS, or any other ASP.NET endpoint configuration
- All auth options apply to all three dashboard endpoints (UI, `/hub`, `/stats`) consistently
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

**Performance** (vs Serilog, .NET 9.0.16 Windows 11 X64)
- 4.0×–4.5× lower hot-path latency
- Zero lock contentions at 32 concurrent producers
- `Enqueued == Processed` verified end-to-end
