# MQTT Gateway

Status: ready-for-agent

## Problem Statement

A fleet of devices speaks MQTT. The rest of the organisation does not.

Web front-ends, internal services, scheduled jobs, and operations tooling all speak HTTP. None of
them can hold a long-lived MQTT connection, and none of them should have to: giving every consumer
its own broker client means every consumer independently solves reconnection, session management,
QoS semantics, and credential handling — and most of them solve at least one of those wrongly.

The people who need this data have three problems today.

They cannot **send** a message to a device without embedding an MQTT client. They cannot **read**
what devices have sent, because MQTT is a transport and not a store — a message missed is a message
gone, and nothing retains history. And they cannot **see** what is flowing: there is no way to ask
"is this topic actually producing traffic right now?" without attaching a client and watching.

An operator has a fourth problem. When something stops working, the question "whose fault is it"
has no answer. A service that holds one connection to a broker, one connection to a database, and
one to a cache can fail in each of those directions independently, and unless it reports which,
every outage starts with a guess.

## Solution

A gateway service that holds exactly one authenticated connection to the broker for the lifetime of
the process, and exposes everything a non-MQTT caller needs over HTTP.

Callers publish through a REST endpoint. They manage which topic filters the gateway listens to,
and those subscriptions are durable — they survive a process restart and are re-established on every
reconnect, so a consumer configures a topic once rather than after every deployment. Every message
the gateway receives is persisted and queryable by topic and time range, turning a transport into a
history. Live traffic counts per topic are available over a true rolling window.

The gateway is built to fail in parts rather than as a whole. The broker going down, the database
going down, and the cache going down are three different events, they produce three different
observable states, and only the first two make the service unable to do its job. At any moment the
service can state which condition it is in, and an orchestrator can act on that statement.

Durability is taken seriously in the one place it is usually lost: a message is acknowledged to the
broker only after it has been written down. A gateway that acknowledges on receipt and then fails to
persist has told the broker a lie that nobody can detect afterwards.

## User Stories

### Publishing

1. As an integrating service, I want to publish a message to a topic over HTTP, so that I can
   command a device without embedding an MQTT client.
2. As an integrating service, I want to choose the QoS level for each published message, so that I
   can pay for delivery guarantees only where they matter.
3. As an integrating service, I want to set the retain flag when publishing, so that a device
   connecting later immediately receives the current state rather than waiting for the next update.
4. As an integrating service, I want to publish a binary payload without it being mangled, so that
   I can carry protobuf, images, or any non-text format through the gateway.
5. As an integrating service, I want an invalid topic rejected before it reaches the broker, so
   that I get a clear, immediate error rather than a silent failure or a protocol-level one.
6. As an integrating service, I want to be told explicitly what a successful publish response
   promises, so that I do not assume a delivery guarantee the protocol never gave me.
7. As an integrating service, I want a distinct error when the broker is unreachable, so that I can
   retry rather than treating it as a bad request.
8. As an operator, I want publishing to be rate-limited, so that one misbehaving caller cannot
   exhaust the gateway for everyone else.
9. As an integrating service, I want to know how long to wait when I am rate-limited, so that my
   retry logic does not make the problem worse.

### Subscriptions

10. As an integrating service, I want to register a topic filter over HTTP, so that the gateway
    starts collecting messages I care about without a code change or a deployment.
11. As an integrating service, I want wildcard filters, so that I can collect from a whole device
    class without enumerating every topic.
12. As an integrating service, I want to list the gateway's current subscriptions, so that I can see
    what is being collected.
13. As an integrating service, I want to remove a subscription, so that collection stops when it is
    no longer wanted and the configuration does not accumulate forever.
14. As an integrating service, I want a duplicate filter rejected as a conflict, so that concurrent
    registration attempts cannot produce duplicate collection of the same traffic.
15. As an integrating service, I want each subscription to carry its own QoS, so that a critical
    topic and a chatty telemetry topic can be treated differently.
16. As an operator, I want subscriptions to survive a restart, so that a deployment does not
    silently stop collection.
17. As an operator, I want subscriptions re-established after a broker outage, so that recovery does
    not require anyone to notice and intervene.
18. As an operator, I want the gateway to refuse to report itself healthy while its subscriptions
    are not restored, so that a connected-but-deaf gateway cannot pass as working.
19. As an operator, I want a failed subscription registration to leave no trace in the stored
    configuration, so that the stored state never claims something the broker is not doing.

### Receiving and querying messages

20. As a device, I want my published message to be recorded, so that a consumer who was not
    listening at that instant can still act on it.
21. As an integrating service, I want to query received messages by topic, so that I can read a
    device's history without having been subscribed at the time.
22. As an integrating service, I want to filter messages by time range, so that I can investigate a
    specific incident window.
23. As an integrating service, I want results paginated deterministically, so that I can walk a
    large result set without seeing a row twice or missing one.
24. As an integrating service, I want a maximum page size enforced, so that I cannot accidentally
    ask for a response that takes the service down.
25. As an integrating service, I want to retrieve a single message by its identifier, so that I can
    follow a reference from a log line or an alert.
26. As an integrating service, I want to be told whether a payload is text or binary, so that I can
    decode it correctly instead of guessing.
27. As an operator, I want oversized payloads rejected and counted rather than stored, so that one
    bad producer cannot fill the database.
28. As an operator, I want message history to be queryable after a restart, so that a restart is not
    also a data loss event.
29. As an operator, I want to configure how long messages are retained, so that storage growth is a
    decision rather than an accident.

### Durability and backpressure

30. As an operator, I want messages published while the gateway is down to be delivered when it
    returns, so that a restart does not create a hole in the history.
31. As an operator, I want a message acknowledged to the broker only after it is stored, so that the
    broker never believes a message was handled when it was lost.
32. As an operator, I want a slow database to slow collection rather than discard it, so that
    pressure shows up as latency I can see instead of data I cannot recover.
33. As an operator, I want in-flight messages written during a graceful shutdown, so that deploying
    does not drop whatever was queued at that moment.
34. As an operator, I want to see how deep the ingest backlog is, so that I can tell the difference
    between a healthy burst and a developing problem.

### Statistics

35. As an operator, I want live message counts per topic, so that I can confirm a device is actually
    producing traffic right now.
36. As an operator, I want a total throughput figure, so that I can see the overall load at a glance.
37. As an operator, I want the statistics window to mean what it says, so that I am not making
    decisions on a number that silently measures something else.
38. As an operator, I want traffic counted for the concrete topics a wildcard subscription matches,
    so that subscribing broadly does not make the statistics view blind.
39. As an operator, I want statistics to degrade visibly rather than fail the request when the cache
    is down, so that a non-critical subsystem cannot take out a critical one.

### Operating the service

40. As an on-call engineer, I want liveness and readiness to mean different things, so that a
    dependency outage takes the service out of rotation without triggering a restart loop.
41. As an on-call engineer, I want to tell a broker outage from a database outage from a cache
    outage by looking at one endpoint, so that triage starts with a fact instead of a guess.
42. As an on-call engineer, I want the gateway's real connection state, including how many times it
    has reconnected, so that I can spot an unstable link that is currently up.
43. As an on-call engineer, I want to follow one message from the HTTP request through the broker to
    the database and the cache in a single trace, so that I can find where latency is introduced.
44. As an on-call engineer, I want log entries to carry their trace identifier, so that I can move
    between logs and traces without correlating by timestamp.
45. As an on-call engineer, I want logs to carry structured fields, so that I can query them instead
    of reading them.
46. As a security reviewer, I want payload contents kept out of routine logs, so that device data is
    not duplicated into a less protected system.
47. As an operator, I want other MQTT clients to learn when the gateway dies unexpectedly, so that
    dependent systems can react without polling.
48. As an operator, I want a clean shutdown to be distinguishable from a crash by observers, so that
    a planned deployment does not fire failure signals.
49. As an operator, I want the service to start and report itself unready when the broker is down,
    so that an outage in a dependency does not turn into a crash loop.
50. As an operator, I want a bad configuration to stop the service at startup, so that I find out
    immediately rather than on the first request in production.

### Security

51. As a security reviewer, I want the broker to reject anonymous connections, so that an exposed
    port is not an open door.
52. As a security reviewer, I want no credential to exist anywhere in version control, so that a
    leak is not permanent the moment it happens.
53. As a security reviewer, I want every unbounded input — page size, payload size, publish rate —
    to have a ceiling, so that a caller cannot exhaust the service by asking politely.

### Building and running it

54. As a developer, I want to bring the whole system up with one command on a clean machine, so that
    onboarding does not start with a day of setup.
55. As a developer, I want to run the application on my host against containerised dependencies, so
    that I can use a debugger without rebuilding an image.
56. As a developer, I want database migrations applied reproducibly as part of bringing the system
    up, so that a fresh environment is not an empty schema.
57. As a developer, I want the application to wait for its dependencies to be genuinely ready, so
    that the first start is not a guaranteed crash.
58. As a developer, I want an interactive API reference, so that I can exercise an endpoint without
    reading the source.
59. As a developer, I want each endpoint's failure responses documented alongside its success
    response, so that I can write a correct client the first time.
60. As a developer, I want editing a source file not to invalidate the dependency restore layer, so
    that rebuilding takes seconds.
61. As a developer, I want the shipped image to contain no build tooling and no source, so that what
    runs in production is small and has little to attack.
62. As a reviewer, I want every non-obvious decision recorded with its reasoning and its cost, so
    that the next person can tell a deliberate trade from an accident.

## Implementation Decisions

### Platform and runtime

- .NET 10 (`net10.0`), which is the current LTS. The originating assignment specified .NET 8, which
  reaches end of support within weeks of this specification being written.
- Package versions are managed centrally so that a version appears in exactly one place.
- Container image tags are pinned. No `latest` anywhere — notably, the database's `latest` is a
  rolling release that reaches end of life this quarter.

### Architecture

- **A single application project, organised by feature**, not Clean Architecture. This service has
  no domain logic worth isolating from infrastructure — it *is* infrastructure. Its entities are
  records of things that happened, and its hard parts are lifetimes, concurrency, failure isolation,
  and operability, none of which a layer boundary helps with. Recorded as ADR-0002.
- A feature owns its endpoint, its request and response types, its validation, its entity, and its
  persistence configuration. Only genuinely shared, impersonal concerns are factored out.
- **No repository or unit-of-work abstraction** over the ORM, which already is both. An interface
  with exactly one implementation and no boundary crossed is an abstraction introduced to satisfy a
  single caller.
- Duplication between features is tolerated until it hurts. That is the trade vertical slices make.

### Lifetimes

The single most common defect in a service shaped like this is a long-lived component capturing a
request-scoped database context. The rules are therefore explicit rather than emergent:

- The database context is request-scoped and is **never** held by a singleton.
- The broker connection, the connection lifecycle worker, the ingest worker, the cache multiplexer,
  and the statistics reader and writer are singletons.
- Any background component that needs the database creates a **scope per unit of work** — per batch,
  not per message and not per process — through the framework's scope factory. One mechanism, used
  consistently.
- Scope validation is enabled in development, so a violation fails at startup rather than under load.

### HTTP API

- MVC controllers with the API controller convention.
- Resource-oriented routes: a publish action, a subscriptions collection with create, list, and
  delete, a messages collection with a by-identifier read, statistics, a connection state resource,
  and separate liveness and readiness endpoints.
- **RFC 9457 problem details for every failure, without exception.** One shape for all errors;
  validation failures use the validation variant.
- Services return a result type carrying the outcome; controllers translate it and contain no
  exception handling of their own. A controller that catches is deciding policy it does not own.
- **A broker outage and a database outage are both "service unavailable" and are never the same
  problem type.** Merging them is the defect this rule exists to prevent.
- **The statistics topic filter is a query parameter, not a path segment.** A topic contains slashes;
  binding it as a single path segment cannot work, and this is the single most common way this API
  shape is broken.
- **A successful publish means "accepted for delivery to the broker" and nothing stronger.** QoS 0 is
  never acknowledged by any broker, so no stronger promise is available in every case, and returning
  two different meanings from one endpoint would make the response a lie in one of them.
- Validation uses a rules library wired through an explicit filter rather than the framework
  integration package, which is deprecated and never supported anything but one hosting model.
- **Topic names and topic filters are different grammars validated by different rules.** Wildcards
  are legal in a filter and illegal in a name; a multi-level wildcard is legal only as the final
  level. These do not share a validation routine.

### Persistence

- Relational storage for messages and subscriptions.
- **The ORM provider is a maintained community fork**, not the long-standing default, which has had
  no commit in fourteen months, caps the ORM at the previous major version, and carries no
  deprecation marker on the package registry — so it installs cleanly into a current project and
  silently pins it to the old version. The fork's API is identical; the reasoning, the risk, and the
  exit path are recorded as ADR-0001.
- The server version is declared explicitly rather than auto-detected. Auto-detection opens a
  connection during service registration, which makes the application unable to start before its
  database.
- **Messages** carry topic, raw payload bytes, payload size, QoS, retain flag, and a receipt
  timestamp. A covering index supports the only query shape the API exposes; no speculative indexes.
- **Subscriptions** carry a topic filter, QoS, and a creation timestamp, with a **unique index on the
  filter**. Uniqueness is enforced by the database, not by a check-then-insert, which is a race. The
  resulting constraint violation maps to a conflict response, never an internal error.
- Topic columns have an explicit maximum length. Without one the provider emits an unbounded text
  type, which cannot carry an efficient or unique index — the index is the reason the length matters.
- Timestamps are UTC with microsecond precision. The alternative offset-aware type has no native
  column type in this database and is converted anyway, so storing UTC is the honest representation.
  A single event is timestamped **once**, at receipt, and that value is carried through the pipeline.
- **Payloads are stored as bytes, not text.** MQTT carries no encoding contract, and a gateway that
  assumes UTF-8 corrupts every binary payload passing through it. The API reports which encoding it
  used, so a caller is never guessing.
- A configurable retention worker deletes aged rows in bounded batches, defaulting to disabled. The
  mechanism exists and is tested, so the limit is a setting rather than an omission.
- Migrations are applied by a dedicated one-shot step, not by the application at startup, which
  races when more than one instance starts.

### Broker connection

- **A fixed client identifier, a non-clean start, and a session expiry interval.** Together these
  make the broker queue messages while the gateway is down and deliver them on reconnect. A gateway
  that loses everything published during a restart is not a gateway. Recorded as ADR-0005.
- The direct cost is stated rather than discovered: **one running instance per client identifier.** A
  second process using the same identifier causes a session takeover, and if both reconnect they
  evict each other indefinitely. This is asserted by a test.
- The protocol version is pinned explicitly rather than inherited from a client-library default.
- The connection is owned by a background worker. It is never established in a request path and
  never awaited during startup: the application starts whether or not the broker is reachable, and
  reports that it is not ready.
- Reconnection uses exponential backoff with full jitter and a ceiling. A cancellation token flows
  through the entire chain including the reconnect loop, or shutdown hangs.
- **Subscription restoration is a barrier, not a best effort.** On every connect, every stored
  subscription is re-established before readiness is reported; until then readiness is unhealthy and
  the connection resource says so. A failed restoration disconnects and retries the whole set —
  partial restoration is not a state worth preserving, because an operator cannot tell which half is
  missing. A connected gateway with zero subscriptions, zero messages, and zero signals is the worst
  kind of failure, and this requirement exists to make it impossible.
- A last will is registered at connect and published to a status topic, retained. A clean shutdown
  publishes the offline state explicitly and disconnects normally, so the will does **not** fire —
  it covers only abnormal termination.
- Persisting a subscription happens before registering it at the broker, and is rolled back if the
  broker call fails. The stored configuration must never claim a subscription the broker does not
  have; the opposite ordering leaves an orphan no restart reconciles.

### Ingest pipeline

The client library invokes the receive handler **sequentially** per connection. Doing database work
inside it makes database latency the ceiling on reception for every topic; dispatching the work
without awaiting it loses exceptions, loses ordering, and runs concurrent operations against a
context that is not thread-safe. Recorded as ADR-0003.

- The handler stamps the receipt time once, builds a record, and writes it to a **bounded in-memory
  channel**. Nothing else.
- The channel **blocks when full** rather than dropping. This propagates genuine backpressure: the
  handler slows, acknowledgement is delayed, the broker throttles. That is the mechanism the protocol
  provides, and using it is the point. An unbounded channel converts backpressure into a memory leak;
  dropping keeps the API fast by losing data silently, which is the worst property a gateway can have.
- **Automatic acknowledgement is disabled; a message is acknowledged after its batch is persisted.**
  This is what makes at-least-once delivery mean at-least-once all the way to storage.
- A worker drains up to a batch size or a time window, whichever comes first, using one scope and
  therefore one database context per batch.
- Shutdown drains in flight with a bounded timeout.
- Oversized payloads are rejected at the handler, counted, and logged. They never enter the channel
  and never stall it.
- The accepted consequences, stated rather than hidden: a slow database slows ingestion instead of
  losing messages; deferred acknowledgement means redelivery after a crash is expected, so duplicate
  rows are possible, which is the honest outcome for a message log given that the protocol version in
  use provides no broker-assigned identifier to deduplicate against.
- No thread-pool offloading of work that is already asynchronous I/O, and no parallel composition of
  a database write with a cache write — the two are sequential steps in one batch, which is both
  simpler and correct.

### Statistics

Recorded as ADR-0004.

- **Per-topic, per-minute counter keys with a slightly-over-an-hour expiry.** Reading a topic sums
  the last sixty buckets in a single multi-key fetch.
- This exists because the obvious implementation — increment a counter and refresh its expiry on
  every message — does **not** produce a rolling window. Each increment pushes expiry forward, so
  the counter accumulates from the first message ever seen and only expires after an hour of
  *silence*. For an actively publishing topic, which is the only interesting case, the reported
  figure is wrong and the stated window is false.
- Increment and expiry are issued as one atomic unit, with expiry set only on creation. Issued
  separately, a crash between them leaves a key with no expiry, forever.
- **Active concrete topics are tracked in a set.** A wildcard subscription produces counters for the
  concrete topics it matches, so enumerating stored subscriptions would report zero for everything
  that actually has traffic. Scanning the keyspace in a loop is not used.
- The response states its window length, so a caller never infers it.
- **The cache is derived state.** When it is unavailable, ingest continues, message queries are
  unaffected, and the statistics endpoint succeeds with a degraded marker and zeroed counts. It never
  fails the request. The resulting divergence between the store and the cache after a partial failure
  is named rather than papered over; there is no distributed transaction and there will not be one.

### Health, observability, and limits

- **Liveness checks nothing but the process.** A liveness probe that fails during a dependency outage
  causes restart loops that make the outage worse.
- **Readiness reflects the ability to do the job**: unhealthy when the broker is disconnected, when
  subscriptions are not restored, or when the database is unreachable; degraded when the cache is
  unreachable. Every check is cheap and time-limited — a health check that runs an expensive query is
  itself an availability risk.
- Structured logging with named fields throughout. Interpolated log templates defeat structured
  logging and are treated as a defect. Payload contents are never logged at informational level.
- Distributed tracing and metrics exported to a local telemetry dashboard that is part of the
  development stack, so that an end-to-end trace is visible with no additional setup. Custom metrics
  cover ingest channel depth, batch size, ingest latency, reconnect count, rejected payloads, and
  rate-limit rejections — the backpressure trade above is only defensible if it is observable.
- Logs are enriched with trace identifiers.
- **A token-bucket rate limiter on publishing**, chosen over a fixed window because bursty publishing
  is the normal shape of gateway traffic, and a fixed window either rejects legitimate bursts or
  permits double the intended rate across a boundary. Rejections carry a retry hint.

### Configuration and security

- All configuration is bound to typed options and **validated at startup**. A configuration error
  must stop the process, not surface on the first request in production.
- Configuration objects are injected as options, never as a raw configuration abstraction.
- Environment variables override file configuration, which is how container deployment supplies
  everything.
- **The broker rejects anonymous connections.** There is no API-level authentication by decision, but
  that is not the same as an open broker.
- No secret exists in any committed file or in history. Local development uses the platform's secret
  store; containers use environment variables; the compose environment file is ignored by version
  control and shipped only as a placeholder template.

### Delivery

- Multi-stage image build: full SDK to build, a minimal runtime image to ship, running as a non-root
  user. Dependency manifests are copied before source so that editing code does not invalidate the
  restore layer.
- **The container listens on 8080.** Current runtime images do not listen on port 80, and assuming
  they do produces a published port that reaches nothing.
- Dependencies are started with health conditions, not merely start ordering — the database needs
  several seconds beyond container start, and ordering alone produces a reliable first-boot crash.
- Within the container network, hosts are service names. A loopback address inside a container is
  that container.
- No TLS termination or certificate handling inside the container; that belongs upstream.

## Testing Decisions

### What makes a good test here

A test asserts **externally observable behaviour** and nothing else. It does not reach into private
state, does not assert that a particular collaborator was called, and does not break when the same
behaviour is implemented differently.

The decisive question for every test in this project is: *if I delete the body of the method under
test, does this test fail?* A test that still passes is measuring coverage, not behaviour.

Two specific prohibitions follow from the nature of this service. **Dependency failures are tested by
stopping the real dependency**, never by substituting something that throws — the interesting part is
how the service behaves when a real connection dies, and a substitute cannot reproduce that. And
**the database is never substituted**: the defects worth catching here are schema, index, constraint,
and concurrency defects, all of which a substitute hides by construction.

### Seams

This is a greenfield repository, so there is no prior art to follow; these are the seams, and the
intent is to keep the count at this number.

**One integration seam: the process boundary.** The gateway is observed only from outside, through
the two ports it already has — HTTP and MQTT. The test host runs the real application against a real
broker, a real database, and a real cache in containers, and the test itself acts as both an HTTP
client and an MQTT client, exactly as an operator with a desktop MQTT tool would. A publish goes in
over HTTP and is observed on the broker; a device publishes to the broker and the result is read back
over HTTP.

**One unit surface: pure functions.** Everything with no I/O — topic name and topic filter validation,
payload encoding detection, the backoff schedule, pagination cursor encoding and decoding, and the
statistics bucket arithmetic across hour and midnight boundaries. These are tested directly because
they are decision-dense and cheap to exercise exhaustively.

**No seam is introduced for the benefit of tests.** There are no service interfaces whose only
implementation is the real one and whose only consumer is a substitute. The mocking library is a
dependency of last resort and is expected to be nearly unused.

### What is tested at the integration seam

The scenarios below are load-bearing: each one closes a defect that the originating assignment either
contained or left undefined, and several of them are the only proof that a claim in this document is
true rather than intended.

1. A message published through the API reaches the broker, is ingested, and becomes visible through
   the message query endpoint.
2. Concurrent registration of the same topic filter yields exactly one stored subscription and a
   conflict response for the rest.
3. Killing the broker makes the gateway report itself unready; restarting it reconnects, restores
   subscriptions, and resumes ingestion with no process restart.
4. Stopping the cache leaves ingestion and message queries unaffected, and the statistics endpoint
   succeeds in a degraded state rather than failing.
5. Severing the connection without a protocol-level disconnect delivers the last will to another
   client; a clean shutdown does not.
6. A thousand messages at least-once QoS produce at least a thousand stored rows, and the statistics
   agree.
7. Two instances sharing a client identifier produce an observable session takeover.
8. A non-UTF-8 payload round-trips without corruption and is reported as binary.
9. An oversized payload is rejected and counted without stalling the pipeline.
10. Restoration failure disconnects and retries rather than serving traffic in a half-restored state.
11. Pagination across rows sharing a timestamp returns every row exactly once.
12. Statistics age out progressively rather than all at once, demonstrating a rolling window rather
    than a sliding expiry.

### Mechanics

- Containers are shared per test collection, not created per class, so start-up cost is paid once.
- Test configuration overrides connection settings **before** the dependency container is built.
- Asynchronous outcomes are awaited by polling with a timeout. Fixed sleeps are banned: they are
  simultaneously too slow and too flaky.
- No test depends on execution order or on state left by another test.
- Continuous integration runs the whole suite and therefore requires a container runtime.

## Out of Scope

Each of these is a decision, not an oversight, and each is disclosed rather than discovered.

- **API authentication.** There are no users, no roles, and no caller to authorise; an authentication
  layer here would be decoration with no consumer. This explicitly does not extend to the broker,
  which is authenticated.
- **Horizontal scaling.** Shared subscriptions would be required for more than one instance to
  consume without duplicating every message, and the durable-session decision above already fixes the
  service at one instance per client identifier. Claiming support without a test proving it would be
  worse than documenting the limit.
- **Publish idempotency and an outbox.** Solves a problem this service does not yet have.
- **TLS termination.** Belongs upstream in any realistic deployment.
- **A pluggable logging abstraction with a composite implementation.** This was a teaching exercise in
  the originating assignment, and a good one, but here it would be a hand-rolled abstraction with one
  real implementation sitting beside a logging library that already provides multiple sinks,
  structured output, and file rolling. File logging itself is provided.
- **Documentation comments on every public member.** Comments that restate a signature cost
  maintenance and carry no information. They are written where the runtime surfaces them to a caller
  in the API document, and where logic is genuinely non-obvious; the reasoning that would otherwise
  have gone into prose lives in the decision records.
- **The introductory console application** from the originating assignment.

## Further Notes

### Provenance

This specification derives from a backend internship assignment: a ten-day plan, a daily checklist,
and a mentor's review guide. The functional requirements are inherited faithfully. The
implementation is not: an audit of those documents produced **forty-nine defects**, and this
specification resolves or explicitly declines every one.

They fall into five groups.

**Eleven would prevent the application from running at all** — a singleton capturing a scoped
database context, concurrent writes through a context that is not thread-safe, a statistics route
that cannot bind a topic containing a slash, a container port that no longer matches the runtime, a
controller that is never routed because the project template changed, prescribed unit tests that
cannot substitute the concrete types they name, dependency ordering that waits for container start
rather than readiness, no migration path inside the container stack, a hardcoded cache endpoint
connected synchronously at startup, a thread-unsafe in-memory collection shared across threads, and a
configuration value whose format is never defined.

**Nine were graded but never implemented** — last will, integration tests, publishing with the retain
flag, per-subscription QoS, unsubscribing, reading messages from storage rather than memory, a
"caching" feature whose keys nothing ever reads, reconnection in the API, and any health or metrics
surface at all.

**Fifteen are semantic defects** where the code runs and the behaviour is wrong — most importantly a
sliding expiry presented as a rolling window and reported with a hardcoded window label, a
non-atomic increment-and-expire pair, wildcard counters invisible to the statistics view,
thread-pool offloading of asynchronous I/O presented as a performance technique, local time in a
store read across zones, a check-then-insert race, an unbounded text column that cannot be indexed,
subscriptions restored only at startup, and payloads assumed to be text.

**Nine are version and tooling decay** — a runtime weeks from end of support, unversioned client
library references whose documented API belongs to the previous major version, an ORM provider that
is unmaintained but carries no warning, a deprecated test framework that the current tooling still
scaffolds by default, floating image tags, a documentation UI removed from the template two versions
ago, and credentials committed in three places.

**Five are declined**, listed under Out of Scope above.

### Known limitations

These are consequences of decisions above, repeated here so that no reader has to reconstruct them:

- One running instance per client identifier. Scaling out requires shared subscriptions first.
- The store and the cache can disagree after a partial failure. There is no distributed transaction.
- Deferred acknowledgement means redelivery after a crash can produce duplicate rows.
- Statistics have one-minute resolution, and the window edge moves in one-minute steps.
- There is no API authentication.
- The broker's current major version is licensed under a source-available licence that is free for
  non-production and single-node production use — relevant if this is ever deployed in a cluster.
- The ORM provider is a community fork. The reasoning and the exit path are in ADR-0001.

### Decision records

Six decisions were non-obvious enough to warrant their own record, and each states its cost as well
as its benefit: the ORM provider choice, the architecture, the ingest pipeline and its backpressure
trade, the statistics window design, the broker session strategy, and the test platform and the
choice of which broker to test against.
