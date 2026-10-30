# Specification Gaps

Every defect, omission, and contradiction inherited from the original internship assignment
(`~/init-technologies/Backend Praksa/`), and what this project does about it.

The original assignment is **not** modified. It is the input; this table is the diff.

Each gap has a stable ID. Where a gap is closed by a behaviour that can be asserted, the test
carrying the fix references the ID in its name, so the link between a requirement and its proof
survives refactoring:

```csharp
[Fact]
public async Task Statistics_RollingWindowExpiresByMinuteBucket_GAP21()
```

Legend — **Status**: `Fixed` · `Declined` (not implemented, with a reason) · `N/A` (an artefact of
the original teaching format with no bearing on this codebase).

---

## A. Defects that prevent the application from running

| ID | Original requirement | Problem | Resolution | Status |
|---|---|---|---|---|
| GAP-01 | `AddSingleton<MqttService>()` (Day 6) combined with `AppDbContext` injected into its constructor (Day 7) | `AddDbContext` registers `DbContext` as **Scoped**. A singleton capturing it is a captive dependency; ASP.NET Core's scope validation throws at startup: `Cannot consume scoped service 'AppDbContext' from singleton`. The two workarounds an AI will suggest — `ValidateScopes = false`, or a single `CreateScope()` in the constructor — produce one `DbContext` living forever, with an ever-growing change tracker, used concurrently from a type that is not thread-safe | Background components take `IServiceScopeFactory` and create **one scope per batch**. No `DbContext` is ever held by a singleton. See spec §4.1 | Fixed |
| GAP-02 | `Task.Run(async () => { /* DB write */ })` running concurrently with the Redis write (Day 8) | Concurrent `SaveChangesAsync` on a shared `DbContext` throws `A second operation was started on this context instance before a previous operation completed`. Guaranteed under the Day 10 "send 100 messages quickly" scenario | Single-writer batch consumer draining a `Channel<T>`. One `DbContext` per batch, never concurrent. Spec §7 | Fixed |
| GAP-03 | Route `GET /api/mqtt/stats/{topic}`, tested with `GET /api/mqtt/stats/test/api` | A single `{topic}` route segment cannot bind a value containing `/`. Since essentially every real MQTT topic contains `/`, the endpoint is unusable as specified — it 404s | Topic filter is a **query parameter**: `GET /api/stats/topic?filter=`. Spec §8.1 note 1 | Fixed |
| GAP-04 | `EXPOSE 80`, compose maps `5000:80` (Day 9) | .NET 8 and later container images listen on **8080** (`ASPNETCORE_HTTP_PORTS=8080`). The published port reaches nothing and `http://localhost:5000/swagger` never responds | Container listens on 8080; compose maps `8080:8080`. Spec §11 | Fixed |
| GAP-05 | `dotnet new webapi` plus a hand-written `Controllers/MqttController.cs` (Day 6) | Since .NET 7 the template emits Minimal APIs. Without `--use-controllers`, `AddControllers()`, and `MapControllers()`, the controller is never routed and all five endpoints 404 | Controllers are registered and mapped explicitly | Fixed |
| GAP-06 | Day 10: "Mock `AppDbContext`", "Mock `RedisService`", "Mock `MqttService`" | None of these has an interface; `RedisService`'s constructor opens a live connection and `MqttService`'s methods are non-virtual. Moq cannot substitute non-virtual members — `NotSupportedException`. The prescribed unit tests cannot be written | Interfaces exist where a seam is genuinely needed, introduced at the point of need rather than retrofitted for tests. The database is never mocked — integration tests use a real MariaDB via Testcontainers. Spec §10 | Fixed |
| GAP-07 | `depends_on` without health conditions, plus `ServerVersion.AutoDetect(connectionString)` | `AutoDetect` opens a connection during service registration, and plain `depends_on` waits only for container start. MariaDB needs several more seconds; the API crashes on first `compose up` | Explicit `MariaDbServerVersion` — no connection at registration — plus `depends_on: condition: service_healthy`. Spec §11 | Fixed |
| GAP-08 | No migration step anywhere in the compose stack | Migrations are only ever applied from the developer's machine. A fresh `docker compose up` yields an empty schema and the API fails on first query | Dedicated one-shot `migrator` service. Not `Database.Migrate()` at API startup, which races across instances. Spec §11 | Fixed |
| GAP-09 | `builder.Services.AddSingleton(new RedisService("localhost:6379"))` | Hardcoded endpoint never rewired to the `RedisConnection` configuration the Docker day introduces, and `ConnectionMultiplexer.Connect` is synchronous inside a constructor executed at startup — it blocks, and throws if Redis is not yet up | `IConnectionMultiplexer` registered as a lazily-connected singleton, endpoint from typed options | Fixed |
| GAP-10 | `List<string> _messages` in a singleton, written from the MQTT receive thread and read by HTTP requests | Not thread-safe. Intermittent `InvalidOperationException: Collection was modified` during serialisation, and unbounded growth — a memory leak | No in-memory message store exists. The database is the only source of truth. Spec §8.1 | Fixed |
| GAP-11 | Config key `MqttBroker` consumed by `ConnectAsync(string broker, int port)` | One configuration value, two parameters. The format of `MqttBroker` — host, `host:port`, or URI — is never defined and the parsing never shown | Typed `MqttOptions` with separate `Host` and `Port`, validated at startup | Fixed |

## B. Required, graded, and never implemented

| ID | Original requirement | Problem | Resolution | Status |
|---|---|---|---|---|
| GAP-12 | Last Will and Testament (Day 2; part of the 20 %-weighted MQTT criterion) | Taught and graded, never implemented, configured, or tested anywhere in the ten days | LWT registered at connect; delivery on abnormal termination asserted by an integration test. Spec §6.4 | Fixed |
| GAP-13 | "Integration testovi" (Day 10 heading and both deliverable lists) | Named repeatedly; no integration test, test-host strategy, or test-database approach is ever described | Nine mandatory integration scenarios against real dependencies. Spec §10 | Fixed |
| GAP-14 | Retain flag (taught Day 2, stored in the entity, printed in the log format) | `PublishRequest` carries only topic, payload, and QoS. Retain can be received and stored but never *sent* — the feature is half-wired | `retain` accepted on `POST /api/publish` | Fixed |
| GAP-15 | Per-subscription QoS (Day 3 requires user-selectable subscribe QoS) | `SubscribeAsync(string topic)` has no QoS parameter and the `Subscription` entity has no QoS column, so the QoS used on restore is undefined | `qos` is a column on `subscriptions` and part of the create request | Fixed |
| GAP-16 | Unsubscribe | Subscriptions are persisted and replayed forever with no way to remove one. The table grows monotonically | `DELETE /api/subscriptions/{id}` removes the row and unsubscribes at the broker | Fixed |
| GAP-17 | `GET /api/mqtt/messages` | Returns the in-memory list, never the database. After a restart it is empty while the table holds thousands of rows. No filtering, no paging | Reads from the database, filtered by topic and time range, keyset-paginated. Spec §8.1 | Fixed |
| GAP-18 | "Redis caching" (listed as a deliverable) | `mqtt:{topic}:{ticks}` keys are written and expire, but `RedisService` has no method to read them and nothing consumes them. There is no cache read path anywhere in the system — only the counter is read | The write-only key space is removed. Redis holds statistics only, which is the one thing it is actually used for. A cache with no reader is not a cache | Fixed |
| GAP-19 | API reconnect | The three-attempt reconnect exists only in the Day 3 console application. The API connects once at startup; if the broker drops, nothing reconnects and nothing reports it | `BackgroundService` with exponential backoff and jitter, plus subscription restoration. Spec §6.2, §6.3 | Fixed |
| GAP-20 | Health, readiness, metrics ("monitoring" appears as a bonus criterion) | None exist | `/health/live`, `/health/ready` with distinct semantics, plus OpenTelemetry traces and metrics. Spec §9.2, §9.3 | Fixed |

## C. Semantic defects — code runs, behaviour is wrong

| ID | Original requirement | Problem | Resolution | Status |
|---|---|---|---|---|
| GAP-21 | "Live statistika (1h rolling window)", implemented as `INCR` + `EXPIRE` per message, and reported as `"timeWindow": "1 hour"` | Resetting the TTL on every increment produces a **sliding expiry**, not a rolling window: the counter accumulates from the first message until an hour of *silence*. For an actively publishing topic the reported window is simply false. The Day 8 test asserting the count resets after an hour fails for exactly this reason | Per-minute buckets with a 70-minute TTL; a read sums the last 60. Spec §9.1 | Fixed |
| GAP-22 | `INCR` followed by `EXPIRE` | Two commands, not atomic. A crash between them leaves a key with no TTL, forever | A single pipelined transaction; `EXPIRE` is issued only when `INCR` returns 1 | Fixed |
| GAP-23 | `GET /api/mqtt/stats` enumerating the `Subscriptions` table | A wildcard subscription such as `test/#` produces counters keyed by *concrete* topics (`stats:test/a:count`). Looking up `stats:test/#:count` always returns 0. The stats view silently loses data | Active concrete topics tracked in a Redis set, enumerated from there. Spec §9.1 | Fixed |
| GAP-24 | `Task.Run(...)` + `Task.WhenAll(...)`, justified as "faster than sequential" | `Task.Run` adds a thread-pool hop to work that is already asynchronous I/O; it buys nothing. The guidance teaches the anti-pattern and then asks the student to explain the difference | No `Task.Run` anywhere. Persistence and statistics are sequential steps inside one batch, which is both simpler and correct | Fixed |
| GAP-25 | `Task.WhenAll(dbTask, redisTask)` | When both fault, `WhenAll` surfaces only the first exception; the second is swallowed. More importantly, a Redis failure would fail the database write | Statistics failures are isolated from persistence failures by construction. Redis is derived state and degrades independently. Spec §9.1 | Fixed |
| GAP-26 | Check-then-insert for duplicate subscriptions | A race. Eight concurrent `POST`s for the same filter produce eight rows | Unique index on `topic_filter`; the provider's unique violation maps to `409`. Spec §5.2 | Fixed |
| GAP-27 | `Topic` as an unconstrained string | Without an explicit length the provider emits `longtext`, which cannot carry an efficient or unique index — the index is the whole reason the length matters | `varchar(512)` on both `topic` and `topic_filter` | Fixed |
| GAP-28 | `ReceivedAt = DateTime.Now` | Local time in a store read across timezones, and two `Now` reads for one event give two different values | `DateTime.UtcNow`, read once at receipt. `DateTime.Now` is banned. Spec §5.3 | Fixed |
| GAP-29 | Auto-subscribe from the database at startup only | Runs once. After an MQTT reconnect the subscriptions are gone again — the Day 3 defect returns through the back door | Restoration runs on **every** connect and gates readiness. Spec §6.3 | Fixed |
| GAP-30 | Payload handled as `string` throughout | MQTT payloads are bytes with no encoding contract. A gateway that assumes UTF-8 corrupts every binary payload it forwards, and throws on some | Stored as `varbinary`; the API reports `utf8` or `base64` explicitly. Spec §5.4 | Fixed |
| GAP-31 | "Sve metode su async" alongside `List<string> GetMessages()` | Self-contradictory within one section | Query paths are asynchronous end to end | Fixed |
| GAP-32 | Every endpoint returns `Ok(...)` | No `400` for an invalid QoS or empty topic, no `503` when the broker is down, no error contract — while "error handling" is a graded component of the REST API criterion | `Result<T>` plus RFC 9457 `ProblemDetails` on every path, with broker and database failures distinguishable. Spec §8 | Fixed |
| GAP-33 | Clean session never discussed | The assignment asks "what happens to messages published while the subscriber is offline?" as a comprehension question and never resolves it in the design. With the implied clean session, the answer is "they are lost" | `CleanStart = false` with an MQTT 5 session expiry interval, so the broker queues during downtime. The cost — one instance per `ClientId` — is documented and tested. Spec §6.1 | Fixed |
| GAP-34 | Message table with no retention policy | Grows without bound | Configurable retention purge, disabled by default. Spec §5.5 | Fixed |
| GAP-35 | `try/catch (Exception) { }` and broad catches around the whole program | Merges unrelated causes into one response and hides failures | Exceptions are caught where there is something to do about them. Broad catches that merge two causes into one answer are forbidden | Fixed |

## D. Version and tooling decay

| ID | Original requirement | Problem | Resolution | Status |
|---|---|---|---|---|
| GAP-36 | .NET 8 throughout | .NET 8 reaches end of support in November 2026 — within weeks of this writing | `net10.0` (LTS, supported to 2028-11-14) | Fixed |
| GAP-37 | `dotnet add package MQTTnet` with no version | Every code reference — `MqttFactory`, `MqttApplicationMessageBuilder`, `byte[]` payload — is the **v4** API. An unversioned add resolves to v5, where `MqttFactory` does not exist as a type and `Payload` is a `ReadOnlySequence<byte>`. The instructions do not compile, on the day with the least context to debug it | `MQTTnet` pinned to 5.2.0.1603; v5 API used throughout, including an explicit protocol version — v5 silently defaults to MQTT 5.0.0 | Fixed |
| GAP-38 | `Pomelo.EntityFrameworkCore.MySql`, unversioned | Latest stable is 9.0.0 (2025-08-17), capped at EF Core 9, with no commit to `main` in fourteen months, an EF Core 10 pull request open for eleven, and no deprecation banner on NuGet. It targets `net8.0`, so it installs cleanly into a `net10.0` project and silently pins EF Core to 9 | `Microting.EntityFrameworkCore.MySql` 10.0.14 — a maintained fork with an identical API. [ADR-0001](decisions/0001-ef-core-provider.md) | Fixed |
| GAP-39 | Moq | Unable to substitute the concrete classes it is asked to (GAP-06), and carrying the SponsorLink controversy | NSubstitute | Fixed |
| GAP-40 | `dotnet new xunit`; no test framework version | The .NET 10 SDK template still scaffolds xunit v2, which is **deprecated** on NuGet, with a `Microsoft.NET.Test.Sdk` four minor versions stale. Taking xunit.v3 4.x instead fails under `dotnet test` on .NET 10 unless the runner is switched — xUnit closed that as Won't Fix | `xunit.v3` 4.0.2 with the Microsoft.Testing.Platform runner selected in `global.json`. [ADR-0006](decisions/0006-test-platform.md) | Fixed |
| GAP-41 | `emqx/emqx:latest`, `mariadb:latest`, `redis:latest` | Unreproducible builds. `mariadb:latest` in particular is a rolling 13.0.x that reaches end of life this quarter, and MariaDB 11.4 is no longer the current LTS | Pinned: `emqx/emqx:6.3.2`, `mariadb:12.3`, `redis:8.10.3` | Fixed |
| GAP-42 | Swagger at `https://localhost:5001/swagger`, and in Docker at `http://localhost:5000/swagger` | Swashbuckle was removed from the template in .NET 9; ports 5000/5001 are a .NET Core 3.1-era default superseded by randomised ports in `launchSettings.json`; and the template guards the UI with `IsDevelopment()` while the compose service sets no `ASPNETCORE_ENVIRONMENT`, so the documented URL would 404 even if the rest worked | `Microsoft.AspNetCore.OpenApi` for the document, Scalar for the UI, enabled in Development, documented at its real address | Fixed |
| GAP-43 | EMQX dashboard credentials `admin` / `public` | EMQX 5.x forces a password change on first login, so the documented steps cannot be followed as written | Dashboard password supplied from `.env`; broker runs with `allow_anonymous = false`. Spec §9.5 | Fixed |
| GAP-44 | Root database password in plaintext in `appsettings.json`, in the compose file, and in the README | Once pushed, a credential stays in git history | No secret in the repository. `dotnet user-secrets` locally, environment variables in containers, `.env` git-ignored. Spec §9.5 | Fixed |

## E. Declined

| ID | Original requirement | Reason for declining | Status |
|---|---|---|---|
| GAP-45 | `CompositeMessageLogger` / `IMessageLogger` / `FileMessageLogger` (Day 5) | A composite logger is a teaching exercise for the Composite pattern and dependency injection, and it was the right exercise. In this codebase it would be a hand-rolled abstraction with one real implementation sitting beside Serilog, which already provides multiple sinks, structured output, and file rolling. Building it would mean introducing an abstraction to satisfy a single caller. File logging itself **is** provided — as a Serilog sink | Declined |
| GAP-46 | "Dodaj XML comments na sve public metode" (Day 10) | Comments that restate the method signature add maintenance cost and no information. Comments are used only where the logic is genuinely non-obvious, which is rare in this codebase, and the reasoning that would have gone into prose lives in `decisions/` instead. Note that .NET 10 *does* ingest XML comments into the OpenAPI document — so where a comment would be read by a caller, it is written | Declined |
| GAP-47 | API authentication | No users, no roles, no caller to authorise. See spec §2. The broker is authenticated regardless (§9.5) | Declined |
| GAP-48 | Console application (Days 1–5) | Out of scope by instruction; this project starts at the Web API | N/A |
| GAP-49 | Plan v2.0 / checklist v1.0 divergences — differing commit messages, EMQX port lists, `.dockerignore` contents, test names, review cadence | Artefacts of a checklist that was never regenerated when the plan moved to 2.0. They describe the teaching process, not the system | N/A |
