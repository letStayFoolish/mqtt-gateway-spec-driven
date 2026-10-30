# MQTT Gateway — Specification

Version 1.0 · 2026-10-30 · Status: Approved

This document is the single source of truth for what the service does. Code that
disagrees with this document is a bug in one of the two; decide which, then fix that one.

Related documents:

- [`spec-gaps.md`](spec-gaps.md) — every requirement inherited from the original internship
  assignment that was broken, underspecified, or missing, and what this specification does about it.
- [`decisions/`](decisions/) — architecture decision records for choices that were not obvious.
- [`tickets/`](tickets/) — the work, broken into independently deliverable units.

---

## 1. Purpose

**MQTT Gateway** bridges MQTT and HTTP. Clients that do not speak MQTT — a web UI, another
service, a cron job — use the REST API to:

- publish a message to a topic,
- manage which topic filters the gateway subscribes to,
- query the messages the gateway has received,
- read live traffic statistics per topic.

The gateway is a **long-lived stateful service**. It holds one connection to the broker for the
lifetime of the process, and it must survive the broker, the database, and Redis each failing
independently — while being able to state, at any moment, what condition it is in.

The difficulty of this service is not CRUD. It is lifetimes, concurrency, failure isolation,
and operability.

## 2. Scope

### In scope

| Capability | Summary |
|---|---|
| Publish | `POST /api/publish` with topic, payload, QoS, retain flag |
| Subscriptions | CRUD over persisted topic filters, each with its own QoS |
| Subscription restoration | All subscriptions re-established from the database on every (re)connect |
| Message ingest | Every received message persisted to MariaDB, with backpressure |
| Message query | Filter by topic and time range, keyset-paginated |
| Statistics | Per-topic and total traffic over a true rolling 1-hour window, in Redis |
| Last Will and Testament | Gateway registers an LWT; other clients' LWT messages ingest normally |
| Resilience | Reconnect with backoff, failure isolation between broker / database / Redis |
| Health | `/health/live` and `/health/ready` with distinct, meaningful semantics |
| Observability | Structured logs, traces, and metrics via OpenTelemetry |
| Rate limiting | Token-bucket limiter on the publish endpoint |
| Delivery | `docker compose up` brings up the whole system on a clean machine |

### Out of scope

These are deliberate exclusions, not oversights. Each is recorded in the README as a known
limitation so that nobody mistakes absence for ignorance.

| Excluded | Why |
|---|---|
| API authentication (JWT, API keys) | No users, no roles, no caller to authorise. An auth layer here would be decoration with no consumer. The broker itself **is** authenticated — see §9.4. |
| Shared subscriptions (`$share/`) / horizontal scaling | Cannot be demonstrated with a single instance; claiming support without a test proving it is worse than documenting the limit. |
| Outbox / publish idempotency | Solves a problem this service does not yet have. |
| TLS termination | Handled upstream in any realistic deployment. |

## 3. Stack

Every version below was verified against its source on 2026-10-30. Versions are pinned centrally
in `Directory.Packages.props`; container image tags are pinned in `docker-compose.yml`.

| Concern | Choice | Version |
|---|---|---|
| Runtime | .NET | `net10.0` (SDK 10.0.403, runtime 10.0.14) |
| MQTT client | `MQTTnet` | 5.2.0.1603 |
| Broker | `emqx/emqx` | 6.3.2 |
| Database | `mariadb` | 12.3 |
| EF provider | `Microting.EntityFrameworkCore.MySql` | 10.0.14 |
| ORM | `Microsoft.EntityFrameworkCore` | 10.0.14 |
| Cache / stats | `redis` + `StackExchange.Redis` | 8.10.3 / 3.4.29 |
| Validation | `FluentValidation` | 12.2.0 |
| Logging | `Serilog.AspNetCore` | 10.0.0 |
| Resilience | `Polly` | 8.9.0 |
| API document | `Microsoft.AspNetCore.OpenApi` | 10.0.14 |
| API documentation UI | `Scalar.AspNetCore` | 2.18.7 |
| Tests | `xunit.v3` | 4.0.2 |
| Test containers | `Testcontainers` | 4.16.0 |
| Mocking | `NSubstitute` | latest stable |
| Telemetry UI | `mcr.microsoft.com/dotnet/aspire-dashboard` | 10.0 |

Three ecosystem traps this stack deliberately avoids — all three are live as of today and all
three silently produce a broken project:

1. **`Pomelo.EntityFrameworkCore.MySql` is unmaintained** and caps EF Core at 9.x, with no
   deprecation banner on NuGet. See [ADR-0001](decisions/0001-ef-core-provider.md).
2. **`xunit` v2 is deprecated**, and `xunit.v3` 4.x fails under `dotnet test` on the .NET 10 SDK
   unless the Microsoft.Testing.Platform runner is selected in `global.json`.
   See [ADR-0006](decisions/0006-test-platform.md).
3. **`FluentValidation.AspNetCore` is deprecated** and never supported anything but MVC
   auto-validation. Validation is wired through an explicit action filter instead.

## 4. Architecture

One project, organised by feature. Not Clean Architecture: this service has no domain logic worth
isolating from infrastructure — it *is* infrastructure. Layers here would buy nothing and cost
indirection. See [ADR-0002](decisions/0002-architecture.md).

```
src/MqttGateway.Api/
  Program.cs
  Features/
    Publishing/        controller, request/response, validator, service
    Subscriptions/     controller, entity, EF configuration, service
    Messages/          controller, entity, EF configuration, query service
    Statistics/        controller, Redis reader and writer
    Diagnostics/       connection endpoint, health checks
  Infrastructure/
    Mqtt/              connection lifecycle, ingest, publishing primitives
    Persistence/       GatewayDbContext, Migrations/
    Redis/             multiplexer registration
    Http/              Result<T>, ToActionResult, ProblemDetails, validation filter
    Options/           typed configuration with ValidateOnStart
tests/
  MqttGateway.UnitTests/
  MqttGateway.IntegrationTests/
```

Rules:

- A feature owns its controller, its DTOs, its validator, its entity, and its EF configuration.
- `Infrastructure/` holds only what is genuinely shared and impersonal.
- Duplication between features is permitted until it hurts. That is the trade vertical slice makes.
- No repository or unit-of-work abstraction over EF Core. `DbContext` already is both, and a
  repository interface with exactly one implementation is an abstraction without a second caller.

### 4.1 Lifetimes

The single most common defect in this kind of service is a captive dependency: a singleton
holding a scoped `DbContext`. The rules are therefore explicit.

| Component | Lifetime | Note |
|---|---|---|
| `GatewayDbContext` | Scoped | Never injected into a singleton |
| `IMqttConnection` | Singleton | Owns the one long-lived client |
| `MqttConnectionService` (`BackgroundService`) | Singleton | Connection lifecycle, reconnect, resubscribe |
| `MessageIngestWriter` (`BackgroundService`) | Singleton | Resolves a **new scope per batch** via `IServiceScopeFactory` |
| `IConnectionMultiplexer` | Singleton | Connected lazily, never synchronously at startup |
| `IStatisticsWriter` / `IStatisticsReader` | Singleton | Stateless over the multiplexer |
| Feature services | Scoped | Injected into controllers |

Any background component needing the database takes `IServiceScopeFactory` and creates a scope
per unit of work. Never `IDbContextFactory` *and* `IServiceScopeFactory` — one mechanism, used
consistently.

## 5. Data model

Database `mqtt_gateway`, character set `utf8mb4`.

### 5.1 `messages`

| Column | Type | Notes |
|---|---|---|
| `id` | `bigint` PK, identity | |
| `topic` | `varchar(512)` not null | Concrete topic name, never a filter |
| `payload` | `varbinary(max)` not null | Raw bytes; see §5.4 |
| `payload_size` | `int` not null | Denormalised for querying without reading the blob |
| `qos` | `tinyint` not null | 0, 1, 2 |
| `retain` | `bool` not null | As received |
| `received_at` | `datetime(6)` not null | UTC, assigned once at receipt |

Indexes: `ix_messages_topic_received_at` on `(topic, received_at desc, id desc)` — covers the
only query shape the API exposes. No speculative indexes.

### 5.2 `subscriptions`

| Column | Type | Notes |
|---|---|---|
| `id` | `int` PK, identity | |
| `topic_filter` | `varchar(512)` not null | May contain `+` and `#` |
| `qos` | `tinyint` not null | Requested QoS |
| `created_at` | `datetime(6)` not null | UTC |

Indexes: `ux_subscriptions_topic_filter` — **unique**. Uniqueness is enforced by the database, not
by a check-then-insert in application code, which is a race. A unique-violation from the provider
maps to `409 Conflict`; it never surfaces as `500`.

`varchar(512)` rather than an unbounded string is not cosmetic: without an explicit length the
provider emits `longtext`, which cannot carry an efficient or unique index.

### 5.3 Timestamps

`DateTime` in UTC, stored as `datetime(6)`. Not `DateTimeOffset`: MySQL and MariaDB have no
timezone-aware column type, so the provider converts anyway — storing UTC explicitly is the
honest representation. `DateTime.Now` is forbidden.

A single received message is timestamped **once**, at the point of receipt, and that value is
carried through the pipeline. Two `UtcNow` reads for one event produce two different values.

### 5.4 Payload representation

MQTT payloads are bytes. The protocol carries no encoding contract, and a gateway that assumes
UTF-8 corrupts every binary payload that passes through it.

- Stored as `varbinary`, with a configurable maximum (`Ingest:MaxPayloadBytes`, default 262144).
  Messages above the limit are rejected at ingest, counted, and logged — not silently truncated.
- Returned by the API as:
  ```json
  { "payload": "temperature=21.5", "payloadEncoding": "utf8" }
  { "payload": "3q2+7w==",         "payloadEncoding": "base64" }
  ```
  `utf8` when the bytes decode as valid UTF-8, `base64` otherwise. The caller is always told which.
- Accepted on publish in the same shape, with `payloadEncoding` defaulting to `utf8`.

### 5.5 Retention

`messages` grows without bound otherwise. A `RetentionPurgeService` deletes rows older than
`Retention:Days` in bounded batches. Default is `0`, meaning disabled — preserving the original
assignment's behaviour — but the mechanism exists and is tested, so the limit is a setting rather
than an omission.

## 6. MQTT behaviour

### 6.1 Session

- **Fixed `ClientId`** from configuration (`Mqtt:ClientId`), not a random GUID.
- **`CleanStart = false`** with MQTT 5 `SessionExpiryInterval` (`Mqtt:SessionExpirySeconds`,
  default 3600).
- Subscriptions are established at QoS 1 unless the stored subscription says otherwise.

Together these mean the broker queues messages while the gateway is down and delivers them on
reconnect. A gateway that loses everything published during a restart is not a gateway.

The cost is explicit: **one running instance per `ClientId`.** A second process using the same
`ClientId` causes a session takeover and the two will evict each other indefinitely if both
reconnect. This is tested, not assumed. See [ADR-0005](decisions/0005-mqtt-session.md).

### 6.2 Connection lifecycle

Connection is owned by a `BackgroundService`, never established in the request path and never
awaited during application startup. The application starts whether or not the broker is reachable;
it reports that it is not ready.

Reconnect uses exponential backoff with full jitter, capped at 30 seconds. A `CancellationToken`
flows through the entire chain including the reconnect loop — otherwise shutdown hangs.

### 6.3 Subscription restoration is a barrier

On every successful connect, every subscription in the database is re-established **before** the
gateway reports readiness.

- While restoration is incomplete, `/health/ready` is `Unhealthy` and
  `GET /api/connection` reports `subscriptionsRestored: false`.
- If restoration fails, the gateway disconnects and retries the whole set. Partial restoration is
  not a state worth preserving: the operator cannot tell which half is missing.

`IsConnected == true` with zero subscriptions, zero messages, and zero signals is the worst kind
of failure — the silent kind. This requirement exists to make it impossible.

### 6.4 Last Will and Testament

The gateway registers an LWT at connect:

- Topic: `Mqtt:LastWill:Topic`, default `gateway/{clientId}/status`
- Payload: `offline`
- QoS 1, `retain = true`

On a clean shutdown the gateway publishes `online` → `offline` explicitly and disconnects
normally, so the will is *not* fired. The will covers only abnormal termination: killed process,
severed network, expired keep-alive. An integration test kills the connection without a DISCONNECT
packet and asserts the will is delivered.

### 6.5 Topic validation

Topic names and topic filters are different grammars and are validated by different rules. They do
not share a validation function.

| | Topic name (publish) | Topic filter (subscribe) |
|---|---|---|
| `+` / `#` | Rejected | Allowed |
| `#` position | n/a | Last level only |
| Empty | Rejected | Rejected |
| Length | ≤ 512 bytes | ≤ 512 bytes |
| `$`-prefixed | Rejected | Rejected |

## 7. Ingest pipeline

The MQTTnet receive handler is invoked **sequentially** per client. Doing database work inside it
throttles reception of every message. Doing that work fire-and-forget loses messages and corrupts
a shared `DbContext`.

```
receive handler  →  Channel<ReceivedMessage>  →  batch writer (BackgroundService)
                                                   ├─ scope per batch → DbContext → SaveChanges
                                                   └─ Redis statistics
```

| Decision | Value | Reason |
|---|---|---|
| Channel capacity | 10 000 | Bounded; unbounded is a memory leak with extra steps |
| Full mode | `Wait` | Real backpressure. The broker slows down; nothing is dropped |
| `AutoAcknowledge` | `false` | Acknowledge after persistence, not before |
| Batch size | 500 messages | |
| Batch window | 100 ms | Whichever comes first |
| Shutdown | Drain with timeout | In-flight messages are written, not abandoned |

`AutoAcknowledge = false` is what makes QoS 1 mean "at least once" all the way to the database. With
the default, the broker is told the message was handled the instant it arrives; if the insert then
fails, the message is gone and the broker believes it was delivered.

The trade this buys, stated plainly: **a slow database now slows ingestion instead of losing
messages.** Channel depth is exported as a metric so the condition is visible rather than inferred.

See [ADR-0003](decisions/0003-ingest-pipeline.md).

## 8. HTTP API

ASP.NET Core MVC controllers with `[ApiController]`. Errors use RFC 9457 `ProblemDetails`
**without exception** — one shape for every failure. Validation failures use
`ValidationProblemDetails`.

Services return `Result<T>`; controllers translate with `ToActionResult()` and contain no
`try`/`catch`. A controller that catches exceptions is deciding policy it does not own.

### 8.1 Endpoints

| Method | Route | Success | Failures |
|---|---|---|---|
| `POST` | `/api/publish` | `202 Accepted` | `400` invalid topic/payload/QoS · `429` rate limited · `503` broker unavailable |
| `GET` | `/api/subscriptions` | `200` | — |
| `POST` | `/api/subscriptions` | `201` + `Location` | `400` · `409` filter already exists · `503` broker unavailable |
| `DELETE` | `/api/subscriptions/{id:int}` | `204` | `404` · `503` |
| `GET` | `/api/messages?topic=&from=&to=&cursor=&pageSize=` | `200` | `400` invalid parameters |
| `GET` | `/api/messages/{id:long}` | `200` | `404` |
| `GET` | `/api/stats` | `200` | — |
| `GET` | `/api/stats/topic?filter=` | `200` | `400` |
| `GET` | `/api/connection` | `200` | — |
| `GET` | `/health/live` | `200` | `503` |
| `GET` | `/health/ready` | `200` | `503` |

Design notes, each resolving a specific defect in the original assignment:

1. **Topics contain `/`.** `GET /api/stats/{topic}` cannot bind `sensors/room1/temp` — the slash
   starts a new route segment and the request 404s. The filter is therefore a **query parameter**,
   not a path segment.
2. **`202` on publish means "accepted for delivery to the broker", and nothing more.** For QoS 0
   the broker never acknowledges, so no stronger promise is available; returning `200` for QoS 0
   and `202` for QoS 1 would be two contracts for one endpoint. The documented meaning is the weaker
   one, uniformly, and `GET /api/connection` is how a caller checks the link is healthy.
3. **`503` from the broker and `503` from the database are different failures** and carry different
   `type` and `title` values. They are never merged by a `catch (Exception)`.
4. **Paging is keyset**, over `(received_at, id)` — a composite cursor, because `received_at` alone
   is not unique and offset paging degrades on deep pages. `pageSize` defaults to 50 and is capped
   at 200.

### 8.2 Representative payloads

```jsonc
// POST /api/publish
{ "topic": "sensors/room1/temp", "payload": "21.5", "payloadEncoding": "utf8", "qos": 1, "retain": false }

// POST /api/subscriptions
{ "topicFilter": "sensors/#", "qos": 1 }

// GET /api/messages?topic=sensors/room1/temp&pageSize=50
{
  "items": [
    { "id": 8831, "topic": "sensors/room1/temp", "payload": "21.5",
      "payloadEncoding": "utf8", "qos": 1, "retain": false,
      "receivedAt": "2026-10-30T18:22:04.512831Z" }
  ],
  "nextCursor": "MjAyNi0xMC0zMFQxODoyMjowNC41MTI4MzFaOjg4MzE=",
  "pageSize": 50
}

// GET /api/connection
{
  "state": "Connected",
  "clientId": "mqtt-gateway-1",
  "brokerEndpoint": "emqx:1883",
  "subscriptionsRestored": true,
  "activeSubscriptions": 3,
  "connectedSince": "2026-10-30T17:04:11.000000Z",
  "reconnectCount": 2
}

// GET /api/stats
{
  "windowMinutes": 60,
  "totalMessages": 18422,
  "topics": [ { "topic": "sensors/room1/temp", "messageCount": 3600 } ],
  "degraded": false
}
```

## 9. Cross-cutting behaviour

### 9.1 Statistics

A true **rolling 1-hour window**, not a sliding expiry.

- Counter key per topic per minute: `stats:{topic}:{yyyyMMddHHmm}`, TTL 70 minutes.
- Reading a topic is one `MGET` across the last 60 bucket keys, summed.
- Active concrete topics are tracked in a Redis set `stats:topics` (also TTL-maintained), because
  a wildcard subscription such as `sensors/#` produces counters for *concrete* topics. Enumerating
  the `subscriptions` table would therefore report zero. `SCAN` in a loop is not used.

`INCR` + `EXPIRE` on every message — the obvious implementation — does not produce a rolling
window. It pushes expiry forward on every increment, so the counter means "everything since the
first message, until an hour of silence". See [ADR-0004](decisions/0004-redis-rolling-window.md).

**Redis is derived state.** If Redis is unavailable: ingest continues, `/api/messages` works,
`/api/stats` responds `200` with `"degraded": true` and zeroed counters, and `/health/ready`
reports `Degraded`. It never returns `500`. Statistics are best-effort by construction, and the
resulting inconsistency between the database and Redis after a failure is named in the README
rather than papered over.

### 9.2 Health

| Endpoint | Meaning | Unhealthy when |
|---|---|---|
| `/health/live` | The process is alive and not deadlocked | Never, unless the host is wedged — an orchestrator restarts the container on failure |
| `/health/ready` | The service can do its job | Broker disconnected · subscriptions not restored · database unreachable |
| | `Degraded` | Redis unreachable |

The distinction is load-bearing: a liveness probe that fails on a dependency outage causes restart
loops that make the outage worse. Checks are cheap and time-limited; a health check that runs an
expensive query is itself an outage risk.

### 9.3 Observability

- **Serilog**, structured. Named placeholders only — `{Topic}`, `{MessageId}`, `{SubscriptionId}`.
  String interpolation in a log template is a defect. Payloads are never logged at `Information`.
- **OpenTelemetry** traces and metrics over OTLP to an Aspire Dashboard container, so the path
  `POST /api/publish` → broker → ingest → batch insert → Redis is visible without any additional
  configuration.
- Custom metrics: channel depth, batch size, ingest latency, reconnect count, rejected-payload
  count, rate-limit rejections.

### 9.4 Rate limiting

A token-bucket limiter on `POST /api/publish` — token bucket rather than fixed window because
bursty publishing is the normal shape of gateway traffic. Rejections return `429` with
`ProblemDetails` and a `Retry-After` header. Limits come from configuration.

### 9.5 Security posture

There is no API authentication, by decision (§2). What this explicitly does **not** mean is an
open broker:

- EMQX runs with `allow_anonymous = false`; credentials come from `.env`, which is git-ignored.
- No secret appears in `appsettings.json`, in `docker-compose.yml`, or in git history. Local
  development uses `dotnet user-secrets`; containers use environment variables.
- `pageSize` is capped, payload size is capped, and the publish endpoint is rate-limited — the
  three unbounded inputs a caller could otherwise use to exhaust the service.

### 9.6 Configuration

All configuration is bound to typed options with `ValidateOnStart()`. A configuration error must
fail the process at startup, not surface on the first request in production.

Sections: `Mqtt`, `Ingest`, `Retention`, `Redis`, `RateLimiting`, `ConnectionStrings`.
Environment variables override `appsettings.json`; `__` maps to `:`.

## 10. Testing

| Kind | Target | Mechanism |
|---|---|---|
| Unit | Topic name and filter validation, payload encoding detection, backoff schedule, cursor encode/decode, rolling-window bucket arithmetic | xunit.v3, NSubstitute |
| Integration | The whole chain against real dependencies | `WebApplicationFactory` + Testcontainers |

Integration tests run against **EMQX**, the same broker as production, built with a hand-rolled
`ContainerBuilder` and an `emqx ctl status` wait strategy. `Testcontainers.Mosquitto` exists and
would be less work, but LWT and session semantics are precisely where brokers differ, and those are
under test here. Testing them on a different broker than the one deployed would prove nothing.

Mandatory scenarios:

1. Publish via the API → message arrives at the broker → ingested → visible via `GET /api/messages`.
2. Duplicate subscription filter → `409`, exactly one row in the database.
3. Kill the broker container → gateway reports not ready → restart it → gateway reconnects,
   restores subscriptions, resumes ingesting, all without a process restart.
4. Stop Redis → `/api/stats` returns `200` with `"degraded": true`; ingest and `/api/messages`
   unaffected; `/health/ready` reports `Degraded`.
5. Terminate the connection without a DISCONNECT packet → LWT delivered to a second client.
6. Publish 1000 messages at QoS 1 → exactly 1000 rows persisted and statistics report 1000.
7. Two gateway instances with the same `ClientId` → session takeover observed and asserted.
8. A non-UTF-8 payload round-trips as `base64` without corruption.
9. A payload above `MaxPayloadBytes` is rejected and counted, and does not stall the pipeline.

No `Thread.Sleep`. Asynchronous outcomes are awaited by polling with a timeout.

## 11. Delivery

`git clone` → `docker compose up -d` → working system, on a machine with nothing but Docker.

- Multi-stage Dockerfile: `sdk:10.0` to build, `aspnet:10.0-noble-chiseled` to run, non-root user.
  `.csproj` and `Directory.Packages.props` are copied before the rest of the source so the restore
  layer survives source edits.
- The container listens on **8080**. .NET 8 and later images do not listen on 80.
- `depends_on` uses `condition: service_healthy` for all three dependencies. Plain `depends_on`
  waits for container start, not readiness, and the database needs several seconds more than that.
- Inside the compose network, hosts are service names — `emqx`, `mariadb`, `redis`. `localhost`
  inside a container is that container.
- Migrations are applied by a dedicated one-shot `migrator` service, not by the API at startup.
  Migrating from application startup races when more than one instance starts.
- `.dockerignore` excludes `bin/`, `obj/`, `.git/`, and test output.
- A `.http` file covers every endpoint.

## 12. Definition of Done

- [ ] `git clone` + `docker compose up -d` on a clean machine yields a working system
- [ ] Every endpoint in §8.1 behaves as specified and returns `ProblemDetails` on every failure
- [ ] Subscriptions and messages survive a restart; subscriptions are restored on reconnect
- [ ] Broker, database, and Redis each fail independently without taking down the service, and each
      failure is distinguishable from the others
- [ ] All nine integration scenarios in §10 pass in CI
- [ ] No secret in git history
- [ ] Every item in [`spec-gaps.md`](spec-gaps.md) is either implemented or explicitly declined with a reason
- [ ] README is sufficient for someone who is not the author to run the project
