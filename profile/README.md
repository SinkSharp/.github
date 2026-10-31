# SinkSharp

**High-performance structured logging for .NET 9 — MEL-compatible, async-first, production-grade.**

[![NuGet](https://img.shields.io/nuget/v/SinkSharp.Core.svg)](https://www.nuget.org/packages/SinkSharp.Core)
[![NuGet Downloads](https://img.shields.io/nuget/dt/SinkSharp.Core.svg)](https://www.nuget.org/packages/SinkSharp.Core)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://github.com/SinkSharp/.github/blob/main/profile/LICENSE)

---
# Dashboard UI (Sample Log Entries)
<img width="1917" height="912" alt="image" src="https://github.com/user-attachments/assets/ce25053c-8719-483c-bf8e-797c24accf2e" />
---


## Why SinkSharp?

| Metric | SinkSharp | Serilog | Result |
|---|---|---|---|
| Hot path no enrichers | 843 ns | 2,672 ns | **3.2× faster** |
| Hot path all enrichers | 795 ns | 2,672 ns | **3.0× faster** |
| P95 mixed load | 1.82 µs | 3.70 µs | **2.0× lower tail** |
| P99 mixed load | 1.33 µs mean | 2.82 µs mean | **2.1× faster** |
| Allocation 0 props | 80 B | 161 B | **50% less** |
| Allocation 3 props | 341 B | 473 B | **28% less** |
| Lock contentions @ 4 threads | ~0 | 1,575 | **Near-zero** |
| Caller blocked by slow sink | No (715 ns) | Yes | **Async isolated** |
| Producer scaling 4 threads | 3.2 ms | 31.1 ms | **9.7× faster** |
| E2E durability 32 producers | 0 events lost | — | **Verified zero loss** |

Measured on .NET 9.0.20, Windows 11, X64 RyuJIT AVX2. Comparison vs Serilog buffered file sink.
See [benchmarks/README.md](benchmarks/README.md) for full methodology and raw numbers.

---

## Packages

| Package | Version | Purpose |
|---|---|---|
| [SinkSharp.Core](src/SinkSharp.Core) | [![NuGet](https://img.shields.io/nuget/v/SinkSharp.Core.svg)](https://www.nuget.org/packages/SinkSharp.Core) | Core pipeline, enrichers, MEL adapter |
| [SinkSharp.Sinks.Console](src/SinkSharp.Sinks.Console) | [![NuGet](https://img.shields.io/nuget/v/SinkSharp.Sinks.Console.svg)](https://www.nuget.org/packages/SinkSharp.Sinks.Console) | Coloured structured console output |
| [SinkSharp.Sinks.File](src/SinkSharp.Sinks.File) | [![NuGet](https://img.shields.io/nuget/v/SinkSharp.Sinks.File.svg)](https://www.nuget.org/packages/SinkSharp.Sinks.File) | Rolling JSONL file sink (CLEF format) |
| [SinkSharp.Sinks.Dashboard](src/SinkSharp.Sinks.Dashboard) | [![NuGet](https://img.shields.io/nuget/v/SinkSharp.Sinks.Dashboard.svg)](https://www.nuget.org/packages/SinkSharp.Sinks.Dashboard) | Embedded real-time web dashboard |
| [SinkSharp.Enrichers.Azure](src/SinkSharp.Enrichers.Azure) | [![NuGet](https://img.shields.io/nuget/v/SinkSharp.Enrichers.Azure.svg)](https://www.nuget.org/packages/SinkSharp.Enrichers.Azure) | Azure context (region, resource group, pod) |

---

## Quick start

```bash
dotnet add package SinkSharp.Core
dotnet add package SinkSharp.Sinks.Console
dotnet add package SinkSharp.Sinks.File
```

```csharp
var logger = SinkSharpBuilder
    .Create()
    .MinimumLevel(LogLevel.Information)
    .MinimumLevelFor("Microsoft", LogLevel.Warning)
    .WithCorrelationId()
    .WithEnvironment("MyApp", "Production")
    .WithPiiMasking()
    .WriteTo(new ColoredConsoleSink(),              LogLevel.Debug)
    .WriteTo(new RollingFileSink("logs/app.jsonl"), LogLevel.Warning)
    .BuildLogger("MyApp");

logger.LogInformation("Order {OrderId} placed by {Customer}", 42, "alice@example.com");
// alice@example.com → automatically PII-masked
// CorrelationId → captured from AsyncLocal, stamped on every event
```

### ASP.NET Core / MEL integration

```csharp
builder.Services.AddSinkSharp(options => options
    .MinimumLevel(LogLevel.Information)
    .MinimumLevelFor("Microsoft", LogLevel.Warning)
    .WithCorrelationId()
    .WithEnvironment("MyApp", builder.Environment.EnvironmentName)
    .WithPiiMasking()
    .WriteTo(new ColoredConsoleSink(),              LogLevel.Debug)
    .WriteTo(new RollingFileSink("logs/app.jsonl"), LogLevel.Warning));

// Web API / Minimal API: reads X-Correlation-Id header, generates if missing,
// sets CorrelationContext (AsyncLocal), writes response header.
// Worker Service / Console App: use CorrelationContext.Use(id) per job instead.
app.UseSinkSharp();
```

---

## Dashboard

SinkSharp includes an embedded real-time log dashboard — free, included with the package, no separate licence required.
Mounts on your existing ASP.NET Core app at the same host and port. No extra process, no extra port.

### Features

- ⚡ Real-time event stream via SignalR WebSocket
- 🕑 Buffer replay — last 500 events shown immediately on connect
- 🔍 Filter by level, service, correlation ID, time range (All / 5m / 30m / 1h)
- 🔎 Full-text search (plain text or regex toggle)
- 🔖 **v1.0.5** — Pin / bookmark individual events to a sticky `Pinned` section above the log list
- 🏷️ **v1.0.5** — Property filter panel: filter by any structured key/value from the current buffer
- 📌 **v1.0.5** — `📌 Only` toolbar button: show only pinned events in the main log list
- ⊞ **v1.0.5** — `⊞ Pinned` toolbar button: full-screen pinned view with Unpin all and Back to logs
- 📋 Properties tab, stack trace tab, JSON copy, CSV export (UTF-8 BOM)
- 📊 Queue stats — pending / processed / dropped (polled every 2s)
- 🔁 Reconnect indicator — shows retry count during SignalR reconnect
- 🌙 Dark / light theme toggle
- 🔗 Copy correlation ID to clipboard — ⧉ button in detail panel
- 🔗 Shareable URL with active filters
- 🔐 Auth: bearer JWT, named policy, shared token, IP allowlist, Basic auth

### Install

```bash
dotnet add package SinkSharp.Core --version 1.0.9
dotnet add package SinkSharp.Sinks.Dashboard --version 1.0.5
```

### Option 1 — Web API / Minimal API (same host, same port)

```csharp
DashboardSink dashSink = null!;

builder.Services.AddSinkSharp(options => options
    .WithDashboard(out dashSink)
    .WriteTo(new ColoredConsoleSink(),              LogLevel.Debug)
    .WriteTo(new RollingFileSink("logs/app.jsonl"), LogLevel.Warning));

builder.Services.AddSinkSharpDashboard(dashSink);

var app = builder.Build();
app.UseSinkSharpDashboard("/sinksharp/dashboard");
// → https://localhost:5001/sinksharp/dashboard
```

### Option 2 — Worker Service / Console App

```csharp
var builder = SinkSharpBuilder.Create()
    .WithDashboard(out var dashSink)
    .WriteTo(new ColoredConsoleSink());

var logger    = builder.BuildLogger("MyApp");
var dashboard = new DashboardServer(dashSink, port: 5341, queue: builder.GetQueue());
await dashboard.StartAsync();
// → http://localhost:5341
```

### Option 3 — Multi-pod / Kubernetes (Redis backplane)

```csharp
builder.Services.AddSinkSharpDashboardRedis(dashSink, signalR =>
    signalR.AddStackExchangeRedis(
        builder.Configuration["Redis:ConnectionString"],
        options => options.Configuration.ChannelPrefix =
            StackExchange.Redis.RedisChannel.Literal("MyApp.SinkSharp")));

app.UseSinkSharpDashboard("/sinksharp/dashboard"); // unchanged
```

---

## Architecture

```
Caller thread (your code)
    │
    ▼ Log()  ← returns immediately, caller never blocks (~850 ns including enrichment)
Striped async channel queue (1 channel per CPU core, max 16)
    │   Bounded (DropOldest) for Info / Debug / Warn
    │   Unbounded for Error / Critical — never dropped
    │
    ▼ Background drainer (thread pool)
SinkSharp pipeline
    ├── Enrichers (CorrelationId, Environment, PII masking, Azure context, user context)
    ├── Formatter (message template rendering)
    └── Sinks (Console, File, Dashboard — all called on drainer thread)
```

**Key properties:**
- Zero lock contention on the hot path — caller enqueues and returns immediately
- CorrelationId captured on caller thread — AsyncLocal invisible on drainer thread
- Object pooling — LogEvent objects reused across calls, reducing GC pressure
- MessageTemplate cached after first parse — zero repeated parsing cost
- Sampler runs inside `IsEnabled()` — dropped events cost nothing (no allocation)

---

## Configuration reference

```csharp
builder.Services.AddSinkSharp(options => options

    // Minimum level for all loggers
    .MinimumLevel(LogLevel.Information)

    // Override level for a specific category (exact match or dot boundary)
    .MinimumLevelFor("Microsoft.AspNetCore", LogLevel.Warning)
    .MinimumLevelFor("System", LogLevel.Error)

    // Sampling (v1.0.9) — drop Debug / Info before enqueue to reduce noise at scale
    // Warning / Error / Critical are never sampled; always pass through
    .WithSampling(rate: 0.10, maxLevel: LogLevel.Debug)   // keep 10% of Debug
    .WithSampling(new SamplingOptions                      // per-level rates
    {
        [LogLevel.Debug]       = 0.10,
        [LogLevel.Information] = 0.50,
    })

    // Built-in enrichers — all project types
    .WithCorrelationId()                          // AsyncLocal correlation ID
    .WithEnvironment("MyApp", "Production")       // AppName, Environment, MachineName
    .WithMachineName()                            // MachineName only
    .WithPiiMasking()                             // email / phone / card / SSN masking

    // HTTP context enrichers — Web API / Minimal API only
    // Option A: fluent builder (no accessor argument — resolved from DI automatically)
    .WithUserContext()                             // UserId (sub), UserName (name), UserRole (role)
    .WithUserContext(claims: ["userId", "role", "tenant"])  // custom JWT claim types

    // Option B: DI registration (equivalent to Option A)
    // builder.Services.AddSinkSharpRequestId();
    // builder.Services.AddSinkSharpRequestPath();
    // builder.Services.AddSinkSharpElapsedTime();
    // builder.Services.AddSinkSharpUserContext(claims: ["userId", "role", "tenant"]);

    // Azure enricher — requires SinkSharp.Enrichers.Azure package
    .WithAzureContext()                           // AzureRegion, ResourceGroup, Slot, Pod, etc.

    // Event filter — drops events before enqueue, zero allocation for dropped events
    .Filter(e => !e.MessageTemplate.Contains("health"))
    .Filter(e => e.Level >= LogLevel.Warning || e.Category.StartsWith("MyApp"))

    // Conditional enrichment — enricher only runs when predicate is true
    .WithEnricher(new MyEnricher(), when: e => e.Level >= LogLevel.Warning)

    // Sinks with per-sink minimum level
    .WriteTo(new ColoredConsoleSink(),                   LogLevel.Debug)
    .WriteTo(new RollingFileSink("logs/app.jsonl"),      LogLevel.Warning)
    .WithDashboard(out dashSink,                         LogLevel.Information)
);

// AddHttpContextAccessor is required for all HTTP context enrichers
builder.Services.AddHttpContextAccessor();

// These are only needed if you did NOT use .WithUserContext() / WithRequestId() etc. above
// builder.Services.AddSinkSharpRequestId();
// builder.Services.AddSinkSharpRequestPath();
// builder.Services.AddSinkSharpElapsedTime();
// builder.Services.AddSinkSharpUserContext(claims: ["userId", "personaId", "solutionName"]);

// Web API / Minimal API: X-Correlation-Id header lifecycle + ElapsedTime stopwatch
app.UseSinkSharp();

// Dynamic level at runtime — no restart needed
SinkSharpLoggerFactory.SetGlobalLevel(LogLevel.Debug);  // enable debug temporarily
SinkSharpLoggerFactory.SetGlobalLevel(LogLevel.None);   // restore startup level
```

### Correlation ID — correct usage per project type

```csharp
// ── Web API / Minimal API ────────────────────────────────────────────────────
// One line in Program.cs after Build().
// Reads X-Correlation-Id header, generates if missing, sets AsyncLocal,
// starts Stopwatch for ElapsedTime enricher, writes response header.
app.UseSinkSharp();

// ── Worker Service / Console App / Background Job ────────────────────────────
// No HTTP pipeline — set CorrelationContext manually per job or operation.
// AsyncLocal flows through every await boundary automatically.
using var _ = CorrelationContext.Use(Guid.NewGuid().ToString("N")[..12]);
// Every logger.Log*() inside this using block gets the CorrelationId.
```

---

## Samples

| Sample | What it shows |
|---|---|
| [SinkSharp.AdvancedEnricherSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.AdvancedEnricherSample) | Sampling, custom claims, filter, conditional enrichment, runtime level |
| [SinkSharp.Sample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.Sample) | Console app, no DI |
| [SinkSharp.WebApiSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.WebApiSample) | Web API with MEL |
| [SinkSharp.WorkerSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.WorkerSample) | Background Worker |
| [SinkSharp.MinimalApiSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.MinimalApiSample) | Minimal API |
| [SinkSharp.MultiPodSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.MultiPodSample) | Redis multi-pod + all auth options |
| [SinkSharp.AzureSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.AzureSample) | Azure context enricher + AKS deploy |
| [SinkSharp.DashboardAuthSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.DashboardAuthSample) | All dashboard auth modes with working code |
| [SinkSharp.MultiSinkSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.MultiSinkSample) | Console + File + Dashboard |
| [SinkSharp.CustomSinkSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.CustomSinkSample) | Custom sink implementation |
| [SinkSharp.CustomEnricherSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.CustomEnricherSample) | Custom enricher |
| [SinkSharp.HealthCheckSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.HealthCheckSample) | ASP.NET health check integration |
| [SinkSharp.ScopedLoggingSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.ScopedLoggingSample) | Scoped logging with ILogger |

Full runnable samples: [SinkSharp-Logging-Samples](https://github.com/SinkSharp/SinkSharp-Logging-Samples)

---

## Links

| | |
|---|---|
| 🌐 Website | [https://www.sinksharp.com](https://www.sinksharp.com) |
| 📦 NuGet | [nuget.org/profiles/sinksharp](https://www.nuget.org/profiles/sinksharp) |
| 🧪 Samples | [SinkSharp-Logging-Samples](https://github.com/SinkSharp/SinkSharp-Logging-Samples) |
| 📧 Support | [support@sinksharp.com](mailto:support@sinksharp.com) |
| 📄 Licence | [Apache 2.0](https://github.com/SinkSharp/.github/blob/main/profile/LICENSE) |
