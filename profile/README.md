# SinkSharp

**High-performance structured logging for .NET 9 — MEL-compatible, async-first, production-grade.**

[![NuGet](https://img.shields.io/nuget/v/SinkSharp.Core.svg)](https://www.nuget.org/packages/SinkSharp.Core)
[![NuGet Downloads](https://img.shields.io/nuget/dt/SinkSharp.Core.svg)](https://www.nuget.org/packages/SinkSharp.Core)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://github.com/SinkSharp/.github/blob/main/profile/LICENSE)

---

## Why SinkSharp?

| Metric | SinkSharp | Serilog | Result |
|---|---|---|---|
| Hot path no enrichers | 435 ns | 1,742 ns | **4.0× faster** |
| Hot path all enrichers | 473 ns | 1,742 ns | **3.7× faster** |
| P99 mixed load | 444 ns | 2,009 ns | **4.5× faster** |
| Allocation 3 props | 412 B | 473 B | **13% less** |
| Lock contentions 32 threads | 0 | 207 | **Zero vs 207** |
| Caller blocked by slow sink | No (430 ns) | Yes | **Async isolated** |
| Sustained 8 threads | 2,522 μs | 4,371 μs | **1.7× faster** |

See [benchmarks/README.md](benchmarks/README.md) for full details.

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
    .WriteTo(new ColoredConsoleSink())
    .WriteTo(new RollingFileSink("logs/app.jsonl"))
    .BuildLogger("MyApp");

logger.LogInformation("Order {OrderId} placed by {Customer}", 42, "alice@example.com");
// alice@example.com → automatically PII-masked
// CorrelationId → captured from AsyncLocal, stamped on every event in the request
```

### ASP.NET Core / MEL integration

```csharp
builder.Services.AddSinkSharp(options => options
    .MinimumLevel(LogLevel.Information)
    .MinimumLevelFor("Microsoft", LogLevel.Warning)
    .WithCorrelationId()
    .WithEnvironment("MyApp", builder.Environment.EnvironmentName)
    .WithPiiMasking()
    .WriteTo(new ColoredConsoleSink())
    .WriteTo(new RollingFileSink("logs/app.jsonl")));
```

---

## Dashboard

SinkSharp includes an embedded real-time log dashboard — **free alternative to Seq ($500/yr)**.
Mounts on your existing ASP.NET Core app at the same host and port. No extra process, no extra port.

### Features

- ⚡ Real-time event stream via SignalR WebSocket
- 🕑 Buffer replay — last 500 events shown immediately on connect
- 🔍 Filter by level (Error / Critical / Warning / Info / Debug)
- 🔎 Full-text search by message, service, or correlation ID
- 🔗 Click correlation ID to trace a complete request
- 📋 Properties tab with structured key/value display
- 📊 Queue stats — pending / processed / dropped (polled every 2s)
- 🛡️ PII masked values displayed correctly
- ⏱️ Time filter — All / last 5m / 30m / 1h
- 📥 Export visible events as JSON
- 🌙 Dark / light theme toggle
- 🌐 Works with `UsePathBase()`, reverse proxy, Kubernetes ingress
- 🔐 Auth: bearer JWT, shared token, IP allowlist, Basic auth

### Install

```bash
dotnet add package SinkSharp.Core
dotnet add package SinkSharp.Sinks.Dashboard --prerelease
```

### Option 1 — Single pod / local dev (Web API / Minimal API)

```csharp
DashboardSink dashSink = null!;

// 1. Register SinkSharp with dashboard sink
builder.Services.AddSinkSharp(options => options
    .WithDashboard(out dashSink)
    .WriteTo(new ColoredConsoleSink())
    .WriteTo(new RollingFileSink("logs/app.jsonl")));

// 2. Register dashboard DI (in-memory SignalR — single pod)
builder.Services.AddSinkSharpDashboard(dashSink);

var app = builder.Build();

// 3. Mount dashboard on your app — same host, same port
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

```bash
dotnet add package Microsoft.AspNetCore.SignalR.StackExchangeRedis
```

```csharp
// Replace AddSinkSharpDashboard() with AddSinkSharpDashboardRedis()
// Safe to reuse your existing Redis — SinkSharp uses its own channel prefix
builder.Services.AddSinkSharpDashboardRedis(dashSink, signalR =>
    signalR.AddStackExchangeRedis(
        builder.Configuration["Redis:ConnectionString"],
        options => options.Configuration.ChannelPrefix =
            StackExchange.Redis.RedisChannel.Literal("MyApp.SinkSharp")));

app.UseSinkSharpDashboard("/sinksharp/dashboard"); // unchanged
```

```
Without Redis:  Browser → Pod A only  (Pod B invisible)
With Redis:     Browser → Pod A + Pod B merged  ✅  (no sticky sessions needed)
```

### Dashboard authentication

```csharp
// No auth — default (local dev, internal tools behind VPN)
app.UseSinkSharpDashboard("/sinksharp/dashboard");

// Bearer JWT — page always loads, token bar shown, user pastes JWT
// Hub and stats protected — ASP.NET validates JWT from browser
// Supports custom policies (PermissionAttribute, claims, roles)
app.UseSinkSharpDashboard("/sinksharp/dashboard", opts =>
    opts.RequireAuthorization("AdminOnly"));  // your policy in AddAuthorization()

// Shared token — simple password, no auth middleware needed
app.UseSinkSharpDashboard("/sinksharp/dashboard", opts =>
    opts.UseToken(builder.Configuration["Dashboard:Token"]));

// IP allowlist — no user auth needed
app.UseSinkSharpDashboard("/sinksharp/dashboard", opts =>
    opts.AllowFrom("10.0.0.0/8", "192.168.0.0/16"));

// Basic auth — browser native prompt, no AddAuthentication() needed
app.UseSinkSharpDashboard("/sinksharp/dashboard", opts =>
    opts.UseBasicAuth("admin", "secret123"));

// Full control — any ASP.NET endpoint configuration
app.UseSinkSharpDashboard("/sinksharp/dashboard", opts =>
    opts.ConfigureEndpoints(ep => ep.RequireRateLimiting("fixed")));
```

**How bearer mode works:** The dashboard page always loads — no 401, no blocked page.
Without a valid token the page opens but shows no log events (same "Waiting…" state).
User pastes their JWT once → stored in `sessionStorage` → SignalR connects with
`accessTokenFactory` → ASP.NET validates JWT → events stream. Token expiry is detected
from the JWT `exp` claim — the token bar reappears automatically 60s before expiry.

> See [src/SinkSharp.Sinks.Dashboard/README.md](src/SinkSharp.Sinks.Dashboard/README.md)
> for full architecture, multi-pod setup, path prefix config, and auth details.

---

## Architecture

```
Caller thread (your code)
    │
    ▼ Log()  ← returns immediately, caller never blocks
Striped async channel queue (1 channel per CPU core, max 16)
    │   Bounded (DropOldest) for Info/Debug/Warn
    │   Unbounded for Error/Critical — never dropped
    │
    ▼ Background drainer (thread pool)
SinkSharp pipeline
    ├── Enrichers (CorrelationId, Environment, PII masking, Azure context)
    ├── Formatter (message template rendering)
    └── Sinks (Console, File, Dashboard — all called on drainer thread)
```

**Key properties:**
- Zero lock contention on the hot path (caller thread enqueues and returns)
- CorrelationId captured on caller thread — AsyncLocal invisible on drainer
- Object pooling — LogEvent objects reused across calls
- MessageTemplate cached after first parse — zero repeated parsing cost

---

## Configuration reference

```csharp
builder.Services.AddSinkSharp(options => options

    // Minimum level for all loggers
    .MinimumLevel(LogLevel.Information)

    // Override level for a specific category (prefix match)
    .MinimumLevelFor("Microsoft.AspNetCore", LogLevel.Warning)
    .MinimumLevelFor("System", LogLevel.Error)

    // Built-in enrichers
    .WithCorrelationId()                          // AsyncLocal correlation ID
    .WithEnvironment("MyApp", "Production")       // AppName + EnvironmentName
    .WithPiiMasking()                             // email/phone/card masking
    .WithAzureContext()                           // region, resource group, pod

    // Sinks
    .WriteTo(new ColoredConsoleSink())
    .WriteTo(new RollingFileSink("logs/app.jsonl"))
    .WithDashboard(out dashSink)                  // real-time web dashboard
);
```

---

## Samples

| Sample | What it shows |
|---|---|
| [SinkSharp.Sample](samples/SinkSharp.Sample) | Console app, no DI |
| [SinkSharp.WebApiSample](samples/SinkSharp.WebApiSample) | Web API with MEL |
| [SinkSharp.WorkerSample](samples/SinkSharp.WorkerSample) | Background Worker |
| [SinkSharp.MinimalApiSample](samples/SinkSharp.MinimalApiSample) | Minimal API |
| [SinkSharp.MultiPodSample](samples/SinkSharp.MultiPodSample) | Redis multi-pod + all auth options |
| [SinkSharp.AzureSample](samples/SinkSharp.AzureSample) | Azure context enricher + AKS deploy |
| [SinkSharp.CustomSinkSample](samples/SinkSharp.CustomSinkSample) | Custom sink implementation |
| [SinkSharp.CustomEnricherSample](samples/SinkSharp.CustomEnricherSample) | Custom enricher |
| [SinkSharp.MultiSinkSample](samples/SinkSharp.MultiSinkSample) | Console + File + Dashboard |
| [SinkSharp.HealthCheckSample](samples/SinkSharp.HealthCheckSample) | ASP.NET health check integration |
| [SinkSharp.ScopedLoggingSample](samples/SinkSharp.ScopedLoggingSample) | Scoped logging with ILogger |
| [SinkSharp.AspNetSample](samples/SinkSharp.AspNetSample) | Full ASP.NET Core integration |

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
