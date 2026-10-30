# ADR-0005 — Session strategy and subscription restoration

Date: 2026-10-30 · Status: Accepted

## Context

Two questions decide whether this service deserves the name "gateway": what happens to messages
published while it is down, and what happens to its subscriptions after a reconnect.

With a clean session — the implicit default, and what the original assignment produces — the answer
to the first is "they are lost" and to the second "they are silently gone." The second is the more
dangerous: after reconnecting, `IsConnected` is `true`, the logs are quiet, and not a single
message arrives. Nothing in the system reports a fault.

## Decision

**Session.** A fixed `ClientId` from configuration, `CleanStart = false`, and an MQTT 5
`SessionExpiryInterval` (default one hour). Subscriptions are established at the QoS stored with
them. MQTTnet 5 negotiates MQTT 5.0.0 by default, so session expiry is available — and the protocol
version is pinned explicitly rather than relied upon.

**Restoration as a barrier.** On every successful connect, every subscription in the database is
re-established *before* the gateway reports readiness.

- While restoration is incomplete: `/health/ready` is `Unhealthy`, and `GET /api/connection`
  reports `subscriptionsRestored: false`.
- If restoration fails, the gateway disconnects and retries the entire set.

**Reconnect.** Exponential backoff with full jitter, capped at 30 seconds, driven by a
`BackgroundService`. The connection is never established in a request path and never awaited during
application startup — the application starts whether or not the broker is reachable, and says it is
not ready. A `CancellationToken` flows through the whole chain including the reconnect loop.

**Last Will.** Registered at connect: `gateway/{clientId}/status`, payload `offline`, QoS 1,
retained. A clean shutdown publishes `offline` explicitly and sends a DISCONNECT, so the will does
not fire — it covers only abnormal termination.

## Consequences

- The broker queues messages at QoS 1 while the gateway is down and delivers them on reconnect.
  Restarting the service no longer loses traffic.
- **One running instance per `ClientId`.** A second process with the same identifier triggers a
  session takeover, and if both reconnect they evict each other indefinitely. This is the direct
  cost of a durable session, it is asserted by an integration test, and it is why horizontal
  scaling is out of scope until shared subscriptions are implemented.
- A queued session consumes broker-side storage. `SessionExpiryInterval` bounds it: after the
  interval the broker discards the session and the queue with it.
- Partial restoration is deliberately not a supported state. Restoring three of five filters and
  carrying on would leave an operator unable to tell which two are missing; disconnecting and
  retrying the set is recoverable and visible.
- Readiness now depends on the broker. A broker outage takes the service out of rotation, which is
  correct — it cannot do its job — while `/health/live` stays green so nothing restarts the
  container in a loop.
