# Changelog

All notable changes to SinkSharp packages are documented here.

## [1.0.0] — 2026-10-08

### SinkSharp.Core 1.0.0
- Striped async channel queue (one per CPU core) — 891× less lock contention vs Serilog at 32 threads
- MessageTemplate caching — zero parse cost after first call per unique template
- LogEvent object pool — 43% less allocation at 0 properties vs Serilog
- Built-in PII masking (email, credit card, SSN, phone numbers)
- Correlation ID auto-propagation via AsyncLocal — captured on caller thread
- Per-category minimum level overrides (longest-prefix match)
- MEL `ILogger<T>` compatible — no application code changes needed
- ASP.NET Core DI integration with graceful shutdown drain
- Built-in health check (Healthy / Degraded / Unhealthy)
- Two-channel design: bounded for Info/Debug/Warn (DropOldest), unbounded for Error/Critical (never dropped)

### SinkSharp.Sinks.Console 1.0.0
- Coloured console output using ANSI escape codes
- Writes on background drainer thread — never blocks callers

### SinkSharp.Sinks.File 1.0.0
- Rolling daily JSONL (Compact Log Event Format) output
- Error/Critical events flush immediately
- Cached JsonSerializerOptions — zero overhead per serialisation
- Pre-sized property dictionary

### SinkSharp.Sinks.Dashboard 1.0.0-preview
- Middleware-style mount for Web API (`app.UseSinkSharpDashboard("/sinksharp/dashboard")`)
- Standalone `DashboardServer` for Worker Service / Console App
- Real-time SignalR event stream
- Buffer replay on connect — see history immediately
- Level filter, full-text search, correlation ID trace
- Queue stats endpoint at `/stats`
- PII values correctly masked in dashboard
- SignalR event name fixed (`log` not `onLog`)
- `PendingCount` uses safe counter math (no `ChannelReader.Count` which throws on some channel types)

### SinkSharp.Enrichers.Azure 1.0.0
- Reads Azure environment variables once at startup — zero cost per event
- App Service: AzureSiteName, AzureSlot, AzureRegion, AzureResourceGroup
- AKS / Container Apps: K8sPodName, K8sNamespace
- Missing env vars silently omitted — no errors when running locally

## Performance (final benchmarks)

.NET 9.0.16 · Windows 11 · X64 RyuJIT AVX2 · BenchmarkDotNet 0.14.0

| Benchmark | SinkSharp | Serilog | Faster |
|---|---|---|---|
| Hot path (no enrichers) | 1,046ns | 2,810ns | 2.7× |
| Hot path (all enrichers) | 670ns | 2,810ns | 4.2× |
| Allocation (0 props) | 92B | 161B | 43% less |
| P95 mixed load | 1,942ns | 4,301ns | 2.2× |
| 32-thread scaling | 34,066µs | 41,954µs | 1.2× |
| Lock contentions (32 threads) | 1.3 | 979 | 891× less |
| E2E NullSink drain (32 producers) | 31ms | — | verified |
| E2E 1ms sink drain (32 producers) | 5,545ms | — | verified |
