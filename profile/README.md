# SinkSharp

**High-performance structured logging for .NET 9 — MEL-compatible, async-first, production-grade.**

[![NuGet](https://img.shields.io/nuget/v/SinkSharp.Core.svg)](https://www.nuget.org/packages/SinkSharp.Core)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://github.com/SinkSharp/.github/blob/main/profile/LICENSE)
[![.NET](https://img.shields.io/badge/.NET-9.0-purple.svg)](https://dotnet.microsoft.com)

SinkSharp is a drop-in `ILogger<T>` replacement with a striped async channel queue, built-in PII masking, correlation ID propagation, and a free local dashboard. No code changes needed to adopt — just swap the logger factory.

---

## Why SinkSharp?

| | SinkSharp | Serilog |
|---|---|---|
| Caller-side latency | **~670ns–1µs** | ~2.8µs |
| Allocation (0 props) | **92 B** | 161 B |
| Allocation (3 props) | **412 B** | 473 B |
| Lock contentions (32 threads) | **1.3** | 979 |
| Non-blocking callers | ✅ Always | ✅ With async wrapper |
| Built-in PII masking | ✅ | ❌ |
| Auto correlation ID | ✅ | ❌ Manual |
| Per-category log levels | ✅ | ✅ |
| Free local dashboard | ✅ | ❌ ($500/yr Seq) |
| Built-in health check | ✅ | ❌ |
| MEL `ILogger<T>` compatible | ✅ | ✅ |
| Licence | Apache 2.0 | BSD |

> Benchmarks run on .NET 9.0.16, Windows 11, X64 RyuJIT AVX2. Results vary by workload — see [Benchmarks](#benchmarks) for full details.

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
    .MinimumLevelFor("Microsoft", LogLevel.Warning)  // silence ASP.NET noise
    .WithCorrelationId()
    .WithEnvironment("MyApp", "Production")
    .WithPiiMasking()
    .WriteTo(new ColoredConsoleSink())
    .WriteTo(new RollingFileSink("logs", "MyApp"))
    .BuildLogger("MyApp.OrderService");

logger.Log(LogLevel.Information, "Order {OrderId} placed by {UserId}", orderId, userId);
```

---

## ASP.NET Core / Generic Host

```bash
dotnet add package SinkSharp.Core
dotnet add package SinkSharp.Sinks.Console
dotnet add package SinkSharp.Sinks.File
```

```csharp
// Program.cs
builder.Services.AddSinkSharp(options => options
    .MinimumLevel(LogLevel.Information)
    .MinimumLevelFor("Microsoft", LogLevel.Warning)
    .MinimumLevelFor("System",    LogLevel.Error)
    .WithCorrelationId()
    .WithEnvironment(appName, env)
    .WithPiiMasking()
    .WriteTo(new ColoredConsoleSink())
    .WriteTo(new RollingFileSink("logs", appName)));

// Inject as usual — no code changes
public class OrderService(ILogger<OrderService> logger)
{
    public void PlaceOrder(Order order)
    {
        logger.LogInformation("Order {OrderId} placed", order.Id);
    }
}
```

Queue drains automatically on `IHostedService` shutdown — no events lost on graceful exit.

### With health checks

```csharp
builder.Services.AddSinkSharpWithHealth(options => options
    .MinimumLevel(LogLevel.Information)
    .WriteTo(new ColoredConsoleSink())
    .WriteTo(new RollingFileSink("logs", appName)));

app.MapHealthChecks("/health");
// GET /health → 200 Healthy / 200 Degraded / 503 Unhealthy
```

---

## Correlation ID middleware

```csharp
app.Use(async (ctx, next) =>
{
    var id = ctx.Request.Headers["X-Correlation-Id"]
                        .FirstOrDefault()
               ?? Guid.NewGuid().ToString("N")[..12];

    using var _ = CorrelationContext.Use(id);
    ctx.Response.Headers["X-Correlation-Id"] = id;
    await next();
});
```

Every log event within that request automatically carries `CorrelationId` — no manual passing required.

---

## Per-category log levels

Silence framework noise without affecting your own code:

```csharp
.MinimumLevel(LogLevel.Debug)                          // your code: Debug+
.MinimumLevelFor("Microsoft",           LogLevel.Warning)  // ASP.NET: Warning+
.MinimumLevelFor("Microsoft.Hosting",   LogLevel.Information) // startup: Info+
.MinimumLevelFor("System.Net",          LogLevel.Error)    // HTTP client: Error+
```

Longest-prefix match wins — `Microsoft.AspNetCore.Routing` matches `Microsoft.AspNetCore` before `Microsoft`.

---

## PII masking

Automatically masks emails, credit card numbers, SSNs, and phone numbers before any sink writes them:

```csharp
.WithPiiMasking()

// Input:  "User john@example.com placed order"
// Output: "User ***@***.*** placed order"

// Input:  "Card 4111-1111-1111-1111 charged"
// Output: "Card ****-****-****-**** charged"
```

Both property values and the rendered message are masked — PII in template literals is caught too.

---

## Custom sink

```csharp
public sealed class SlackSink : ISink
{
    private readonly SlackClient _client;
    public string Name => "Slack";

    public async ValueTask EmitAsync(LogEvent e, CancellationToken ct = default)
    {
        if (e.Level < LogLevel.Error) return;
        await _client.PostAsync($"[{e.Level}] {e.RenderedMessage}", ct);
    }

    public ValueTask FlushAsync(CancellationToken ct = default)
        => ValueTask.CompletedTask;
}

// Register
builder.WriteTo(new SlackSink(slackClient));
```

---

## Custom enricher

```csharp
public sealed class TenantEnricher : IEnricher
{
    public void Enrich(LogEvent logEvent, IPropertyBag bag)
    {
        var tenantId = TenantContext.Current;
        if (tenantId is not null)
            bag.AddOrUpdate("TenantId", tenantId);
    }
}

// Register
builder.WithEnricher(new TenantEnricher());
```

---

## Local dashboard (preview)

```bash
dotnet add package SinkSharp.Sinks.Dashboard
```

Works for every type of .NET app — with or without an HTTP pipeline.

### ASP.NET Core API / Minimal API / MVC

Mounts on your **existing port and host** — no separate server, no extra port.
Works exactly like Swagger UI.

```csharp
// Program.cs
DashboardSink dashboardSink = null!;

builder.Services.AddSinkSharp(options => options
    .MinimumLevel(LogLevel.Debug)
    .WithCorrelationId()
    .WithDashboard(out dashboardSink)           // ← capture the sink
    .WriteTo(new ColoredConsoleSink()));

builder.Services.AddSinkSharpDashboard(dashboardSink); // ← register for DI

var app = builder.Build();

app.UseSinkSharpDashboard("/sinksharp/dashboard"); // ← mount like Swagger

// → https://localhost:5001/sinksharp/dashboard         (UI)
// → https://localhost:5001/sinksharp/dashboard/hub     (SignalR)
// → https://localhost:5001/sinksharp/dashboard/stats   (JSON stats)
```

### Worker Service / Console App / no HTTP pipeline

Starts its own lightweight Kestrel instance on a dedicated port.

```csharp
var logger = SinkSharpBuilder.Create()
    .MinimumLevel(LogLevel.Debug)
    .WithCorrelationId()
    .WithDashboard(out var sink)
    .WriteTo(new ColoredConsoleSink())
    .BuildLogger("MyWorker");

// Start standalone dashboard server
var server = new DashboardServer(sink, port: 5341);
await server.StartAsync();

// → http://localhost:5341/sinksharp/dashboard
```

No Seq licence needed. Path is configurable — use any prefix that suits your app.

---

## Architecture

```
Your code   Log()          ~500ns–1µs, never blocks
   │           │
   ▼           ▼
ISinkSharpLogger  ──→  Stripe channel (1 per CPU core)
                              │
                              ▼  background thread
                        ┌─ Enrich (correlation, env, PII)
                        ├─ Format (cached template)
                        └─ Route → Sink 1, Sink 2, ...

Error/Critical ──→ Unbounded channel (never dropped)
Info/Warn/Debug ──→ Bounded channel (DropOldest on overflow)
```

**Key properties:**
- Caller threads never block — `Log()` enqueues and returns in ~500ns–1µs
- Striped channels eliminate producer lock contention (754× fewer contentions than Serilog at 32 threads)
- `MessageTemplate` is parsed once and cached — zero parse cost after the first call per template
- `LogEvent` objects are pooled and reused — reduced GC pressure
- Slow sinks do not slow callers — a 10ms sink adds ~50ns to caller latency

---

## Packages

| Package | Version | Description |
|---|---|---|
| `SinkSharp.Core` | 1.0.0 | Queue, pipeline, builder, MEL integration, health check |
| `SinkSharp.Sinks.Console` | 1.0.0 | Coloured console output |
| `SinkSharp.Sinks.File` | 1.0.0 | Rolling daily JSONL file sink |
| `SinkSharp.Enrichers.Azure` | 1.0.0 | Region, resource group, slot name, pod name |
| `SinkSharp.Sinks.Dashboard` | 1.0.0-preview | Embedded localhost:5341 real-time dashboard |

---

## Benchmarks

All benchmarks: .NET 9.0.16 · Windows 11 · X64 RyuJIT AVX2 · BenchmarkDotNet 0.14.0

### Hot path — caller-side latency per `Log()` call

| Method | Mean | Allocation |
|---|---|---|
| Disabled (`IsEnabled` fast-exit) | **3 ns** | 1 B |
| SinkSharp — no enrichers | **1,046 ns** | 385 B |
| SinkSharp — all enrichers (PII + Correlation + Env) | **670 ns** | 391 B |
| SinkSharp — Error (critical channel) | **999 ns** | 378 B |
| Serilog buffered file | 2,810 ns | 401 B |

SinkSharp achieved **2.7×–4.2× lower latency** than the tested Serilog configuration in these workloads.

### Allocation per call

| Properties | SinkSharp | Serilog |
|---|---|---|
| 0 | **92 B** | 161 B |
| 1 | 377 B | 361 B |
| 3 | **412 B** | 473 B |

SinkSharp allocates **43% less** at 0 properties (queue infrastructure amortised by object pooling). At 3 properties, SinkSharp allocates **13% less**.

### P95 latency — mixed load (80% Info / 15% Warn / 5% Error)

30 iterations · captures real GC pauses

| | SinkSharp | Serilog |
|---|---|---|
| Mean | **1,418 ns** | 3,069 ns |
| P95 | **1,942 ns** | 4,301 ns |
| Allocated | **271 B** | 448 B |

### Concurrent producers — N threads × 1,000 logs each

| Producers | SinkSharp | Serilog | Faster | Lock contentions |
|---|---|---|---|---|
| 1 | **790 µs** | 3,390 µs | **4.3×** | 0 vs 0 |
| 4 | **2,935 µs** | 6,052 µs | **2.1×** | 0 vs 135 |
| 8 | **6,285 µs** | 10,785 µs | **1.7×** | 0.25 vs 289 |
| 16 | **14,232 µs** | 24,018 µs | **1.7×** | 0.25 vs 496 |
| 32 | **34,066 µs** | 41,954 µs | **1.2×** | 1.3 vs 979 |

**SinkSharp outperforms the tested Serilog configuration at every producer count from 1 to 32**, with **754× fewer lock contentions** at 32 threads — a structural advantage from striped channels.

### End-to-end durability — Enqueued == Processed

| Variant | Producers | Median | Verified |
|---|---|---|---|
| NullSink | 1 | 2.5 ms | ✅ |
| NullSink | 8 | 8.0 ms | ✅ |
| NullSink | 32 | 31 ms | ✅ |
| 1ms accurate sink | 1 | 1,001 ms | ✅ |
| 1ms accurate sink | 8 | 1,522 ms | ✅ |
| 1ms accurate sink | 32 | 5,545 ms | ✅ |

All enqueued events were processed — zero dropped, zero lost, `Enqueued == Processed` verified across all configurations.

---

## Health check output

```json
{
  "status": "Healthy",
  "results": {
    "sinksharp": {
      "status": "Healthy",
      "description": "Queue healthy. Pending: 42, Processed: 1284921, Dropped: 0"
    }
  }
}
```

- **Healthy** — queue flowing normally
- **Degraded** — >10,000 events pending (sink falling behind)
- **Unhealthy** — any events dropped (queue overflowed)

---

## Requirements

- .NET 9.0+
- `Microsoft.Extensions.Logging.Abstractions` 9.x
- `Microsoft.Extensions.Hosting.Abstractions` 9.x (for DI integration)

---

## Roadmap

| Version | Feature |
|---|---|
| v1.1 | Striped channel improvements for 64+ producer scaling |
| v1.2 | `SinkSharp.AI` — query logs in plain English via Anthropic API |
| v1.3 | `SinkSharp.Sinks.AzureMonitor` — cost-aware routing to App Insights |

---

## Licence

Apache 2.0 — see [LICENSE](https://github.com/SinkSharp/.github/blob/main/profile/LICENSE).

Benchmark comparisons use Serilog (Apache 2.0) as an external reference only. Serilog is not included in any SinkSharp package.
