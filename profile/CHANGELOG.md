# Changelog

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
- 2.7×–4.2× lower hot-path latency
- 43% less allocation at 0 properties
- 754× fewer lock contentions at 32 concurrent producers
- `Enqueued == Processed` verified end-to-end
