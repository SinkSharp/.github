# SinkSharp.Core

Fast structured logging pipeline for .NET 9.

[![NuGet](https://img.shields.io/nuget/v/SinkSharp.Core.svg)](https://www.nuget.org/packages/SinkSharp.Core)

---
# Dashboard UI
<img width="1917" height="907" alt="SinkSharp_Dashboard" src="https://github.com/user-attachments/assets/06edb3ad-d089-492e-bfdd-6e997999b370" />

---

## Installation

```bash
dotnet add package SinkSharp.Core --version 1.0.8
```

---

## Quick start

```csharp
var builder = SinkSharpBuilder.Create()
    .WriteTo(new ColoredConsoleSink(),              LogLevel.Debug)
    .WriteTo(new RollingFileSink("logs/app.jsonl"), LogLevel.Warning);

var logger = builder.BuildLogger("MyApp");
logger.LogInformation("Order {OrderId} placed by {User}", 42, "alice");
```

### With ASP.NET Core (MEL integration)

```csharp
builder.Services.AddSinkSharp(options => options
    .WriteTo(new ColoredConsoleSink())
    .WriteTo(new RollingFileSink("logs/app.jsonl")));
```

---

## Enrichers

```csharp
builder.Services.AddSinkSharp(options => options
    // All project types
    .WithCorrelationId()                  // CorrelationId from AsyncLocal
    .WithEnvironment("MyApp", "Prod")     // AppName, Environment, MachineName
    .WithMachineName()                    // MachineName only (no app/env needed)
    .WithPiiMasking()                     // email/card/SSN/phone masking
    // .WithAzureContext()                // Azure App Service / AKS context
    //                                     // requires SinkSharp.Enrichers.Azure package

    // HTTP enrichers: use DI helpers below instead (v1.0.6+)

    // Structured filter — zero allocation for dropped events
    .Filter(e => !e.MessageTemplate.Contains("health"))

    // Conditional enrichment — enricher runs only when predicate is true
    // Conditional enrichment — implement IEnricher and register with a predicate:
    // .WithEnricher(new TimestampEnricher(), when: e => e.Level >= LogLevel.Warning)
    // TimestampEnricher example: bag.AddOrUpdate("UtcTimestamp", DateTime.UtcNow.ToString("O"))

    // Per-sink minimum level
    .WriteTo(new ColoredConsoleSink(),              LogLevel.Debug)    // everything
    .WriteTo(new RollingFileSink("logs", "MyApp"), LogLevel.Warning)); // disk: warnings+

// HTTP context enrichers — register before Build() (Web API / Minimal API only)
// AddHttpContextAccessor() must come before any enricher below
builder.Services.AddHttpContextAccessor();
builder.Services.AddSinkSharpRequestId();    // stamps RequestId on every request
builder.Services.AddSinkSharpRequestPath();  // stamps HttpMethod + RequestPath
builder.Services.AddSinkSharpElapsedTime();  // stamps ElapsedMs since request start
builder.Services.AddSinkSharpUserContext();  // stamps UserId, UserName, UserRole (auth only)

var app = builder.Build();

// Web API / Minimal API only — one line after Build():
// X-Correlation-Id header + starts ElapsedTime stopwatch
app.UseSinkSharp();

// Console App / Worker Service — no HTTP pipeline
// Use CorrelationContext.Use() per operation and Stopwatch for elapsed time
using (CorrelationContext.Use("job-001"))
{
    var sw = System.Diagnostics.Stopwatch.StartNew();
    // ... do work ...
    sw.Stop();
    logger.Log(LogLevel.Information,
        "Job {JobId} complete — {Count} items in {ElapsedMs}ms", jobId, count, sw.ElapsedMilliseconds);
    // ElapsedMs appears in console properties line, dashboard Properties tab, and CSV export
}

// Runtime level — no restart needed
SinkSharpLoggerFactory.SetGlobalLevel(LogLevel.Debug);  // enable debug temporarily
SinkSharpLoggerFactory.SetGlobalLevel(LogLevel.None);   // restore startup level
```

---

## Performance

Benchmarks run on .NET 9.0.20, Windows 11, X64 RyuJIT AVX2, BenchmarkDotNet v0.14.0.
Single producer thread for hot-path and allocation benchmarks. NullSink (in-memory, no I/O) to
isolate queue and pipeline cost. Serilog comparison uses buffered file sink.
Ratios are more meaningful than absolute numbers — run on your own hardware to reproduce.

| Metric | SinkSharp | Serilog | Result |
|---|---|---|---|
| Hot path no enrichers | **843 ns** | 2,672 ns | **3.2× faster** |
| Hot path all enrichers (PII + Corr + Env) | **795 ns** | 2,672 ns | **3.4× faster** |
| Hot path Error (critical channel) | **835 ns** | 2,672 ns | **3.2× faster** |
| IsEnabled fast-exit (below min level) | **6 ns** | — | zero allocation |
| P99 mean — mixed load (80% Info/15% Warn/5% Error) | **1.33 µs** | 2.82 µs | **2.1× faster** |
| P95 tail — mixed load | **1.82 µs** | 3.70 µs | **2.0× lower tail** |
| Allocation 0 props | **80 B** | 161 B | **50% less** |
| Allocation 1 prop | **299 B** | 361 B | **17% less** |
| Allocation 3 props | **341 B** | 473 B | **28% less** |
| Lock contentions @ 4 threads (ConcurrentThroughputBenchmark) | **~0** | 1,575 | **Near-zero** |
| Backpressure — caller with 10ms SlowSink | **715 ns** | blocked | **Caller never waits** |
| Concurrent producers 4 threads × 1,000 events | **3,098 µs** | 4,682 µs | **1.5× faster** |
| Concurrent producers 8 threads × 1,000 events | **6,657 µs** | 8,747 µs | **1.3× faster** |
| E2E durability — events lost (all 9 sink/producer combinations) | **0** | — | **Zero loss verified** |

---

## Architecture

- Striped async channel queue — one channel per CPU core (max 16)
- Bounded `DropOldest` for Info/Debug/Warn; unbounded for Error/Critical — critical events are never dropped
- Background drainer thread — caller enqueues and returns in ~850ns (includes enrichment; sink writes happen on drainer thread)
- LogEvent object pool — reduced allocation and GC pressure under sustained load
- `MessageTemplate` parsed and cached on first use

---

## Release notes

### v1.0.8
- Fixed: `_taskBuffer` data race in `LogPipeline` when 2+ sinks configured — each `ProcessAsync` now uses `ArrayPool<Task>` per-invocation
- Fixed: sink exception permanently killed drainer stripe — per-sink `try/catch`; `SinkFailureCount` counter added; drainer continues
- Fixed: `ReturnToPool()` now in `finally` — pool leak on sink exception eliminated
- Fixed: `TotalDropped` always returned 0 — replaced `BoundedChannel.TryWrite` drop detection with atomic per-stripe occupancy counter; race-free under 32 concurrent producers
- Fixed: critical drainer `IndexOutOfRangeException` — `stripeIdx: _stripeCount` → `stripeIdx: -1` sentinel; Error/Critical events now correctly processed at shutdown
- Fixed: shutdown race — replaced `_disposed int` with state machine (`Accepting/Draining/Stopped`) using `Interlocked.CompareExchange`
- Fixed: category prefix filter false match — `MinimumLevelFor("Microsoft")` no longer matches `"MicrosoftExtra"`; requires exact match or dot boundary
- Added: `SinkSharpLogger.ClearCallerCaptures()` for test teardown
- Added: `AsyncLogQueue.SinkFailureCount` — observable counter for sink errors
- 230 tests, all passing

### v1.0.7
- Event delivery guaranteed at all producer counts — drain waits for all events to be processed
- Lock contention eliminated from hot path — near-zero contentions at 8+ threads

### v1.0.6
- `AddSinkSharpRequestId()` — stamps `RequestId` on every HTTP request event
- `AddSinkSharpUserContext()` — stamps `UserId`, `UserName`, `UserRole` (authenticated only)
- `AddSinkSharpRequestPath()` — stamps `HttpMethod` + `RequestPath`
- `AddSinkSharpElapsedTime()` — stamps `ElapsedMs` since request start (requires `app.UseSinkSharp()`)

### v1.0.5
- `WithMachineName()` — standalone machine/pod name enricher
- `WithRequestId(accessor)` — stamps `RequestId` from `HttpContext.TraceIdentifier`
- `WithUserContext(accessor)` — stamps `UserId`, `UserName`, `UserRole`; safe for anonymous endpoints
- `WithRequestPath(accessor)` — stamps `HttpMethod` and `RequestPath`
- `WithElapsedTime(accessor)` — stamps `ElapsedMs` since request start; needs `app.UseSinkSharp()`
- `WriteTo(sink, minLevel)` — per-sink minimum log level independent of global minimum
- `.Filter(predicate)` — drop events before enqueue; zero allocation for dropped events
- `.WithEnricher(enricher, when: predicate)` — conditional enrichment by level or category
- `SinkSharpLoggerFactory.SetGlobalLevel(level)` — runtime level change without restart
- `app.UseSinkSharp()` — one-line middleware: X-Correlation-Id + ElapsedTime stopwatch
- `AddSinkSharpRequestId()`, `AddSinkSharpUserContext()`, `AddSinkSharpRequestPath()`, `AddSinkSharpElapsedTime()` DI helpers

### v1.0.4
- Fixed: `DisposeAsync()` is now idempotent — calling it multiple times no longer
  throws `ObjectDisposedException`. Uses `Volatile.Read` / `Interlocked.Exchange` guard.

### v1.0.3
- Performance: `EnsureCapacity()` called before property extraction

### v1.0.2
- Fixed email regex in `PiiMaskingEnricher`
- Fixed `CorrelationId` captured on caller thread
- Fixed `PendingCount` counter arithmetic
- Fixed `DefaultFormatter` safe property lookup
- Fixed `LogBuffer` counter on eviction
- Fixed phone number masking for `(555) 123-4567` format

### v1.0.1
- Bug fix: email PII regex corrected

### v1.0.0
- Initial release
