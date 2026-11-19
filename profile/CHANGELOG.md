## [SinkSharp.Sinks.Dashboard 1.1.0] — 2026-11-16 — History store, Load older, resilient live stream

### Security
- **Token mode covers the live stream.** `UseToken()` now validates the SignalR connection as well as the stats and history endpoints. Bearer, Basic and IP-allowlist modes are unchanged. Recommended upgrade for everyone using `UseToken()`.

### Added
- **Pluggable history store** — `IDashboardHistoryStore`. The in-memory buffer is the default, so existing setups work as they are. `SinkSharp.Sinks.Dashboard.Redis` adds Redis Streams for history shared across pods and preserved across restarts.
- **Load older** — pages older events in from the history store; the active level, search and correlation filters are applied on the server.
- **`GET {dashboard}/history`** — paging endpoint (`cursor`, `count`, `level`, `q`, `corr`, `from`, `to`), protected the same way as the hub and stats.
- **`GET {dashboard}/health`** and `AddSinkSharpDashboardHealthCheck()` — a reachability check for the configured history store, for monitoring and uptime probes independent of whether a dashboard happens to be open.
- `DashboardOptions.ShowDashboardRequests()`; `DashboardOptions.ConfigureCatchUpPolling(intervalMs, count)`; `DashboardServer.HistoryStore`; stats include `historyStore` and `historyPersistent`.
- `IDashboardHistoryStore.TotalWritten`/`TotalDropped` (nullable, default `null`) — `/stats` and `/health` now report `historyTotalWritten`/`historyTotalDropped` for any store that tracks them.

### Improved
- **Non-blocking live stream** — events are pushed to browsers without holding the logging pipeline, so every other sink keeps its pace whatever the state of the Redis backplane.
- **Always-available dashboard** — while a backplane or history store is temporarily out of reach, the dashboard shows the pod's own recent events, says so plainly in the footer, and reconnects by itself with a gentle back-off (2 s up to 30 s). New events keep appearing on their own too, via a history poll while degraded — a manual refresh is never required.
- **Clean reconnects** — replayed events are de-duplicated and the list stays in time order.
- **Chronological list** — events are ordered by timestamp, including when several pods interleave.
- **Focused view** — below Warning, the dashboard leaves out its own traffic: requests to its endpoints, and SignalR, WebSocket and Kestrel connection logs. Other sinks receive everything. `o.ShowDashboardRequests()` includes it.
- **Event IDs** are unique across pods and restarts (`<instance>-<sequence>`).
- **Startup line** prints the bound address.

---

## [SinkSharp.Sinks.Dashboard.Redis 1.0.0] — 2026-11-16 — Initial release

### Added
- `RedisStreamHistoryStore` — dashboard history in a Redis Stream, shared by every pod that uses the same `StreamKey`, preserved across restarts and deployments.
- `AddSinkSharpDashboardRedisStreams()` for history; `AddSinkSharpDashboardRedis(sink, signalR, streams)` for the SignalR backplane plus history in one call.
- Non-blocking writes: events are queued and written in pipelined batches on a background task.
- Retention by age (`RetentionDays`, Redis 6.2+ or Valkey) and by count (`MaxLength`, default 100,000, any Redis 5.0+).
- `PingAsync()` — a real Redis round-trip with its own short timeout, backing the dashboard's `/health` endpoint and `AddSinkSharpDashboardHealthCheck()`.
- `TotalWritten`/`TotalDropped` implement the new `IDashboardHistoryStore` counters, so `/stats` and `/health` show real write/drop totals for this store instead of `null`.
- Resilient to Redis interruptions: logging and the live stream continue, the dashboard serves each pod's recent events, and shared history resumes automatically.
- Requires `SinkSharp.Sinks.Dashboard` 1.1.0 or later.

---

## [SinkSharp.Core 1.0.10] — 2026-11-13 — Dependency updates

### Changed
- `Microsoft.Extensions.Logging.Abstractions` updated to 10.0.15
- `Microsoft.Extensions.Hosting.Abstractions` updated to 10.0.15
- `Microsoft.Extensions.DependencyInjection.Abstractions` updated to 10.0.15
- `Microsoft.Extensions.Diagnostics.HealthChecks` updated to 10.0.15
- `Microsoft.Extensions.ObjectPool` updated to 10.0.15
- `System.Threading.Channels` updated to 10.0.15
- `Microsoft.AspNetCore.Http.Abstractions` updated to 2.3.14
- All existing functionality unchanged — safe to upgrade from 1.0.9

---

## [SinkSharp.Sinks.OpenTelemetry 1.0.1] — 2026-11-13 — Wire format fix

### Fixed
- **OTLP binary Protobuf field numbers** corrected to match current `opentelemetry-proto` specification: `severity_number = field 2`, `severity_text = field 3` (field 4 is reserved). Previously used wrong field numbers which caused `wrong wireType = 0 for field SeverityText` errors on all OTLP backends including Grafana Loki, Jaeger, and OTel Collector.
- **gRPC HTTP/2 transport** for plain `http://` endpoints: added `EnableMultipleHttp2Connections = true` to `SocketsHttpHandler` and `HttpVersionPolicy.RequestVersionOrHigher` to the gRPC `HttpRequestMessage`. Without these, .NET downgrades to HTTP/1.1 on non-TLS connections and gRPC fails with a connection error.

### Changed
- Removed `Google.Protobuf` direct package dependency. The Protobuf wire format is now implemented via `ProtoWriter` — a lightweight manual varint encoder. This reduces the dependency surface and eliminates version-specific API behaviour differences in `CodedOutputStream`.

### Verified
- Grafana Loki 3.4.1 — log lines, structured metadata, PII masking, `service_name` and `deployment_environment` labels ✅
- Jaeger 1.65.0 — HTTP request spans, span events, error spans ✅
- OTel Collector 0.164.0 — gRPC ingestion on port 4317 ✅

---

## [1.0.9] — 2026-10-27 — v1.1 feature release

### Added — SinkSharp.Core 1.0.9
- **Pluggable sampling** — `ISampler` interface + `PercentageSampler` implementation. Probabilistically drops Debug / Info / Trace events before enqueue. Warning / Error / Critical are never sampled — always pass through. ~2 ns overhead per `IsEnabled()` call via `Random.Shared` (thread-local, zero contention).
  - `.WithSampling(new SamplingOptions { [LogLevel.Debug] = 0.10 })` — dictionary form, per-level rates
  - `.WithSampling(rate: 0.10, maxLevel: LogLevel.Debug)` — simple single-level overload
  - `.WithSampling(ISampler)` — custom sampler implementation
- **`WithUserContext()` fluent overloads** — two new builder overloads that do not require an `IHttpContextAccessor` argument at builder time (accessor resolved from DI after container is built):
  - `.WithUserContext()` — default claims: `sub` → `UserId`, `name` → `UserName`, `role` → `UserRole`
  - `.WithUserContext(claims: ["userId", "personaId", "solutionName"])` — custom JWT claim types
- **Dashboard property filter UI** — `Props` button in toolbar opens a panel to filter visible events by any structured property key/value. Dynamic key dropdown built from events in the current buffer. Multiple filters AND-ed with existing level/text/time filters. Filter badges with individual remove.
- **Dashboard log bookmarking / pinning** — 🔖 button per row pins the event to a sticky `Pinned` section above the log list. Max 10 pins; oldest auto-dropped when limit reached. Cleared on `Clear All` and on page refresh.

### Fixed — SinkSharp.Core 1.0.9
- **`WithUserContext()` custom claims ignored** — `SinkSharpEnricherStartupService` previously hardcoded `sub`, `name`, `role` regardless of the claims configured in `AddSinkSharpUserContext(claims: [...])` or `.WithUserContext(claims: [...])`. Now delegates to the registered `UserContextEnricher` singleton via a new `DirectBag` per-invocation `IPropertyBag` — thread-safe, no shared state.
- **`SinkSharpBuilderTests` parallel test isolation** — static `_runtimeLevel` leaked between `DynamicLevelTests` and `SinkSharpBuilderTests` when xunit.v3 ran them in parallel. Added `IDisposable` with constructor + `Dispose()` reset in `SinkSharpBuilderTests`.

### Test suite — 1.0.9
- 240 tests, all passing (was 230 before this release)
- New: `SamplingTests` — 7 tests covering rate=0/1/0.5 statistical distribution, Warning bypass, multi-level `SamplingOptions`, custom `ISampler`, invalid rate validation

---

## [SinkSharp.Sinks.Dashboard 1.0.5] — 2026-10-27

### Added
- **Property filter panel** — `Props` toolbar button opens a panel to filter visible events by any structured property key/value. Dynamic key dropdown built from events in the current buffer. Multiple filters AND-ed with existing level/text/time filters. Filter badges with individual remove.
- **Log event bookmarking / pinning** — 🔖 button per row pins the event to a sticky `Pinned` section above the log list. Max 10 pins; oldest auto-dropped when limit reached. Cleared on `Clear All` and page refresh.
- **`📌 Only` toolbar button** — filters the main log list to show only pinned events (non-pinned rows faded to 18% opacity for context). Auto-disables when last pin is removed. Works independently of the Pinned view.
- **`⊞ Pinned` toolbar button** — opens a full-screen dedicated Pinned view that replaces the log list. Sticky header with pin count, `Unpin all`, and `← Back to logs`. Each row shows level badge, timestamp, full message, category, and unpin button. Clicking any row opens the detail panel. Auto-closes when last pin is removed. Clicking `⊞ Pinned` again closes and returns to the log list.
- Both pinned toolbar buttons start disabled and activate as soon as the first event is pinned.

---

## [SinkSharp.Sinks.OpenTelemetry 1.0.1] — 2026-11-10 — Dependency updates

### Changed
- `Google.Protobuf` updated from 3.30.1 to 3.36.2
- `OpenTelemetry.Extensions.Hosting` updated from 1.9.0 to 1.20.1

---

## [SinkSharp.Sinks.OpenTelemetry 1.0.0] — 2026-11-03 — Initial release

### Added
- **`OtlpLogSink`** — SinkSharp `ISink` that exports fully-enriched, PII-masked log events to any OpenTelemetry-compatible backend via binary Protobuf OTLP. Receives events from the SinkSharp drainer thread after all enrichers have run — PII is masked in-process before any data reaches the OTLP endpoint.
- **`OtlpLogSinkOptions`** — unified configuration for endpoint, protocol, API key headers, service name/version/environment, batch size, flush interval, export timeout, and retry count. One config object shared between `OtlpLogSink` and `AddSinkSharpTracing`.
- **`OtlpProtocol` enum** — `Grpc` (binary Protobuf over HTTP/2, port 4317) and `HttpProtobuf` (binary Protobuf over HTTP/1.1, port 4318).
- **`AddSinkSharpTracing()`** — registers OpenTelemetry distributed tracing via a `configureTracing` callback. Caller adds instrumentation libraries (AspNetCore, HttpClient, SqlClient) and the OTLP trace exporter — keeping the package independent of specific instrumentation packages.
- **`OtlpSuppressingHandler`** — `DelegatingHandler` that suppresses `Activity.Current` during each OTLP export call, preventing `AddHttpClientInstrumentation` from creating child spans for the sink's own export requests.
- **Binary Protobuf wire format** — implemented from scratch using `Google.Protobuf` primitives (`CodedOutputStream`, `WireFormat`, `ByteString`). No third-party proto package required.
  - gRPC path: Protobuf bytes wrapped in 5-byte gRPC frame (`Content-Type: application/grpc+proto`, HTTP/2)
  - HTTP path: raw Protobuf bytes (`Content-Type: application/x-protobuf`, POST to `/v1/logs`)
- **LogEvent → OTel LogRecord mapping** per the OTel Log Data Model specification:
  `SeverityNumber`, `SeverityText`, `Body`, `log.record.original`, `code.namespace`,
  `correlation.id`, `trace.id`, `span.id`, `exception.type`, `exception.message`, `exception.stacktrace`
- **Batching** — configurable `BatchSize` (default 512) and `FlushInterval` (default 5 seconds). Error and Critical events flush immediately.
- **Retry with exponential back-off** — 1s → 2s → 4s on transient failures (429, 503, network errors). Non-transient failures (400, 401, 403) are reported and dropped.
- **Graceful shutdown drain** — `DisposeAsync` flushes remaining buffered events before exit. Zero events lost at graceful shutdown.
- **`NOTICE.txt`** — Apache 2.0 attribution for opentelemetry-proto wire format specification bundled in the NuGet package.

### Verified backends
- Grafana Loki — all log levels, structured metadata, PII masking confirmed ✅
- Jaeger — HTTP request spans, span events in Logs tab, error spans confirmed ✅
- OTel Collector → Loki + Jaeger (Grafana profile) confirmed ✅

### Sample project
- `samples/SinkSharp.OpenTelemetrySample` — complete working sample with Docker Compose for Jaeger and Grafana + Loki + OTel Collector profiles. Demonstrates all endpoints, PII masking, span events, error recording.

### Dependencies
- `Google.Protobuf` 3.30.1
- `Grpc.Net.Client` 2.67.0
- `OpenTelemetry.Extensions.Hosting` 1.9.0

---

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
- Benchmark suite: `EndToEndDurabilityBenchmark` now asserts `Processed == Enqueued - Dropped` for all 9 producer/sink combinations

### Test suite
- 230 tests, all passing (was 227 before this release)

---

## [SinkSharp.Sinks.Dashboard 1.0.4] — 2026-10-25

### Fixed
- Dashboard in Worker Service and Console App no longer fails to connect — `DashboardServer` was missing `DashboardTokenStore` DI registration
- Dashboard page now loads with the correct auth mode in Worker Service / Console App — `__AUTH_MODE__` placeholder not replaced at server startup
- CSV export renders special characters and non-ASCII text correctly — UTF-8 BOM added

---

## [SinkSharp.Sinks.File 1.0.2] — 2026-10-25
- README updated with Core 1.0.8 compatibility note. No functional changes.

---

## [1.0.7] — 2026-10-22

### Fixed
- Event delivery guaranteed at all producer counts — `DrainAndStopAsync()` now awaits all drainer tasks before returning
- Lock contention eliminated from hot path — `_enrichers` and `_callerCaptures` replaced with volatile int count fields written once at startup

---

## [1.0.6] — 2026-10-20

### Added
- `AddSinkSharpRequestId()` — stamps `RequestId` from `HttpContext.TraceIdentifier` on every HTTP request event; requires `AddHttpContextAccessor()`
- `AddSinkSharpUserContext()` — stamps `UserId` (sub), `UserName` (name), `UserRole` (role) from JWT claims; silently skips anonymous/unauthenticated requests; requires `AddHttpContextAccessor()`
- `AddSinkSharpRequestPath()` — stamps `HttpMethod` + `RequestPath` on every HTTP request event; requires `AddHttpContextAccessor()`
- `AddSinkSharpElapsedTime()` — stamps `ElapsedMs` since request start; reads `Stopwatch` from `HttpContext.Items` set by `app.UseSinkSharp()`; requires `AddHttpContextAccessor()`
- `SetGlobalLevel()` / `GetGlobalLevel()` on `SinkSharpLoggerFactory` — change minimum log level at runtime without restart

---

## [1.0.5] — 2026-10-18

### Added
- `WriteTo(sink, minLevel)` — per-sink minimum level; `WithDashboard(out sink, minLevel)` overload
- `WithMachineName()` — standalone MachineName enricher without AppName/Environment labels
- `.Filter(predicate)` — drops events before enqueue; zero allocation for filtered events (returned to pool immediately)
- `.WithEnricher(enricher, when: predicate)` — conditional enrichment; predicate receives the `LogEvent`; typical use: level gate or category prefix
- `app.UseSinkSharp()` — `SinkSharpCorrelationMiddleware`: reads `X-Correlation-Id` header, generates 12-char ID if missing, sets `CorrelationContext` (`AsyncLocal`), starts `Stopwatch` in `HttpContext.Items` for `WithElapsedTime`, writes `X-Correlation-Id` response header
- Dashboard: Copy correlation ID to clipboard (⧉ button in detail panel)
- Dashboard: Shareable URL with active filters — hash encodes level/service/corr filters
- Dashboard: Regex search toggle — `.* ` button in search bar
- Dashboard: Reconnect indicator — shows retry count during SignalR reconnect
- Dashboard: Buffer capacity display in stats bar

---

## [1.0.4] — 2026-10-15

### Fixed
- `DisposeAsync()` is now idempotent — double-dispose no longer throws `ObjectDisposedException`; uses `Volatile.Read` + `Interlocked.Exchange` guard
- Phone number PII masking regex corrected — `(555) 123-4567` format now matched correctly

---

## [1.0.1] — 2026-10-12

### Fixed
- Email PII masking regex corrected for edge cases (subdomains, plus-addressing)

---

## [1.0.0] — 2026-10-10

### Initial release
- Striped async channel queue — 1 `BoundedChannel` per logical CPU (max 16); bounded `DropOldest` for Info/Debug/Warn; unbounded for Error/Critical
- `ISink`, `IEnricher`, `IFormatter` abstractions — full extensibility
- MEL `ILogger<T>` and `ILoggerFactory` compatible — zero changes to existing log call sites
- `LogEvent` object pool — reduces GC pressure under sustained load
- `MessageTemplate` parsing and caching — zero repeated parse cost
- `CorrelationIdEnricher` — `AsyncLocal<string?>` propagation
- `EnvironmentEnricher` — stamps `AppName`, `Environment`, `MachineName` at startup
- `PiiMaskingEnricher` — email, credit card (16-digit), SSN, phone (compiled regex)
- `AzureContextEnricher` (`SinkSharp.Enrichers.Azure`) — reads 8 Azure/AKS env vars once at startup; zero per-event cost
- `ColoredConsoleSink` — level-coloured structured output
- `RollingFileSink` — daily JSONL/CLEF rolling files
- `DashboardSink` — SignalR real-time stream, 500-event circular buffer replay
- `SinkSharpHealthCheck` — Healthy/Degraded/Unhealthy based on drop rate and queue pressure
- `SinkSharpBuilder` — fluent builder, no DI required
