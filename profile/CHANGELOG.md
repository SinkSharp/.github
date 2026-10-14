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

## [1.0.3] — 2026-10-12

### SinkSharp.Core 1.0.3

**Performance — Hybrid inline property store (SmallPropertyBag)**

Replaced `Dictionary<string,object?>` backing store with a hybrid inline array
following the ASP.NET Core `RouteValueDictionary` pattern:

- First 4 properties stored as inline struct fields on `LogEvent` — zero heap allocation
- 5th+ properties spill to overflow `Dictionary` (rare in practice — typical log call has 1-3)
- `LogEvent.Properties` public API unchanged — returns `Dictionary<string,object?>` as before
  (built lazily from inline slots on first external access — never allocated in the hot path)

Expected improvements vs v1.0.2:
- Allocation at 0 props: 87B → ~0B
- Allocation at 1 prop:  377B → ~32B  (was WORSE than Serilog 361B — now 91% less)
- Allocation at 3 props: 425B → ~96B
- Gen2 GC at 8 threads:  45.9 → near zero (no Entry[] array to survive GC cycles)
- Gen2 GC at 16 threads: 103.5 → ~10-20

**Internals**
- Added `SmallPropertyBag` struct (`Pipeline/SmallPropertyBag.cs`)
- `PropertyBag` now wraps `LogEvent._bag` directly — enricher writes skip Dictionary
- `DefaultFormatter` reads `_bag` directly — hot drainer path skips Dictionary
- `CorrelationId` promoted to direct `LogEvent` field — zero allocation capture on caller thread
- `PooledLogEvent.ReturnToPool()` calls `_bag.Clear()` — O(1) struct field zeroing vs Dict.Clear()

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
