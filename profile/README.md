# SinkSharp

**High-performance structured logging for .NET 9 — MEL-compatible, async-first, production-grade.**

[![NuGet](https://img.shields.io/nuget/v/SinkSharp.Core.svg)](https://www.nuget.org/packages/SinkSharp.Core)
[![NuGet Downloads](https://img.shields.io/nuget/dt/SinkSharp.Core.svg)](https://www.nuget.org/packages/SinkSharp.Core)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://github.com/SinkSharp/.github/blob/main/profile/LICENSE)

---
## Dashboard UI

<img width="1917" height="905" alt="SinkSharp_Dashboard" src="https://github.com/user-attachments/assets/3e9438c1-b5c9-4ca8-b025-6872b3ccad12" />


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

See [benchmarks/README.md](benchmarks/README.md) for methodology and raw numbers.

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

// One-line correlation middleware — ASP.NET Core Web API / Minimal API only.
// Reads X-Correlation-Id header, generates if missing, sets CorrelationContext,
// writes response header. Replaces the manual app.Use() block.
// Worker Services and Console Apps: use CorrelationContext.Use(id) directly in your code.
app.UseSinkSharp();
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
- 📥 Export visible events as JSON or CSV
- 🔡 Regex search toggle — switch between plain text and RegExp search
- 🔁 Reconnect indicator — shows retry count during SignalR reconnect
- 🌙 Dark / light theme toggle
- 🌐 Works with `UsePathBase()`, reverse proxy, Kubernetes ingress
- 📋 Copy correlation ID to clipboard — ⧉ button in detail panel
- 🔗 Shareable URL with filters — paste pre-filtered view to a colleague
- 🔐 Auth: bearer JWT, shared token, IP allowlist, Basic auth

### Install

```bash
dotnet add package SinkSharp.Core --version 1.0.6
dotnet add package SinkSharp.Sinks.Dashboard --version 1.0.1
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

// Role check — inline, no named policy needed
app.UseSinkSharpDashboard("/sinksharp/dashboard", opts =>
    opts.RequireRole("Admin", "DevOps"));

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
`accessTokenFactory` → ASP.NET validates JWT → events stream in real time without a page refresh.
Token expiry is detected from the JWT `exp` claim — the token bar reappears automatically 60s before expiry.

> See [src/SinkSharp.Sinks.Dashboard/README.md](src/SinkSharp.Sinks.Dashboard/README.md)
> for full architecture, multi-pod setup, path prefix config, and auth details.
>
> See [SinkSharp.DashboardAuthSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.DashboardAuthSample)
> for a runnable project showing all auth modes with working code.

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

    // Built-in enrichers — work in all project types
    .WithCorrelationId()                          // AsyncLocal correlation ID
    .WithEnvironment("MyApp", "Production")       // AppName, Environment, MachineName
    .WithMachineName()                            // MachineName only (skip WithEnvironment)
    .WithPiiMasking()                             // email/phone/card masking
    .WithAzureContext()                           // region, resource group, pod

    // HTTP context enrichers — Web API / Minimal API only
    .WithRequestId(accessor)                      // RequestId from HttpContext.TraceIdentifier
    .WithUserContext(accessor)                    // UserId, UserName, UserRole (authenticated only)
    .WithRequestPath(accessor)                    // HttpMethod + RequestPath
    .WithElapsedTime(accessor)                    // ElapsedMs since request start

    // Structured filter — drops events before enqueue, zero allocation for dropped events
    .Filter(e => !e.MessageTemplate.Contains("health"))  // drop health check noise
    .Filter(e => e.Level >= LogLevel.Warning ||
                 e.Category.StartsWith("MyApp"))         // composite condition

    // Conditional enrichment — enricher only runs when predicate returns true
    // Example: only stamp the current UTC timestamp on Warning+ events (skip Debug/Info)
    .WithEnricher(new TimestampEnricher(), when: e => e.Level >= LogLevel.Warning)
    // TimestampEnricher is defined in SinkSharp.AdvancedEnricherSample — implement IEnricher
    // and add bag.AddOrUpdate("UtcTimestamp", DateTime.UtcNow.ToString("O"));

    // Sinks with per-sink minimum level
    .WriteTo(new ColoredConsoleSink(),                   LogLevel.Debug)
    .WriteTo(new RollingFileSink("logs/app.jsonl"),      LogLevel.Warning)
    .WithDashboard(out dashSink,                         LogLevel.Information)
);

// One-line middleware (Web API / Minimal API): X-Correlation-Id + ElapsedTime stopwatch
app.UseSinkSharp();

// HTTP context enrichers — add AddHttpContextAccessor() first, then register what you need
builder.Services.AddHttpContextAccessor();   // required — register this before any enricher below
builder.Services.AddSinkSharpRequestId();   // stamps RequestId
builder.Services.AddSinkSharpRequestPath(); // stamps HttpMethod + RequestPath
builder.Services.AddSinkSharpElapsedTime(); // stamps ElapsedMs since request start
builder.Services.AddSinkSharpUserContext(); // stamps UserId, UserName, UserRole

// Dynamic level at runtime — no restart needed
SinkSharpLoggerFactory.SetGlobalLevel(LogLevel.Debug);  // enable debug temporarily
SinkSharpLoggerFactory.SetGlobalLevel(LogLevel.None);   // restore startup level
```

---

## Samples

| Sample | What it shows |
|---|---|
| [SinkSharp.AdvancedEnricherSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.AdvancedEnricherSample) | All v1.1 enrichers, filter, conditional enrichment, runtime level |
| [SinkSharp.Sample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.Sample) | Console app, no DI |
| [SinkSharp.WebApiSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.WebApiSample) | Web API with MEL |
| [SinkSharp.WorkerSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.WorkerSample) | Background Worker |
| [SinkSharp.MinimalApiSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.MinimalApiSample) | Minimal API |
| [SinkSharp.MultiPodSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.MultiPodSample) | Redis multi-pod + all auth options |
| [SinkSharp.AzureSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.AzureSample) | Azure context enricher + AKS deploy |
| [SinkSharp.CustomSinkSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.CustomSinkSample) | Custom sink implementation |
| [SinkSharp.CustomEnricherSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.CustomEnricherSample) | Custom enricher |
| [SinkSharp.MultiSinkSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.MultiSinkSample) | Console + File + Dashboard |
| [SinkSharp.HealthCheckSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.HealthCheckSample) | ASP.NET health check integration |
| [SinkSharp.ScopedLoggingSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.ScopedLoggingSample) | Scoped logging with ILogger |
| [SinkSharp.AspNetSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.AspNetSample) | Full ASP.NET Core integration |
| [SinkSharp.DashboardAuthSample](https://github.com/SinkSharp/SinkSharp-Logging-Samples/tree/main/SinkSharp.DashboardAuthSample) | All dashboard auth modes |

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
