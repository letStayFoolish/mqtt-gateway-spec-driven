# 05 — MQTT connection lifecycle, reconnect, LWT, `/api/connection`

Status: ready-for-agent
Depends on: 03, 04

## Goal

The gateway holds one connection to the broker for the life of the process, recovers from broker
outages without a restart, and reports its real state.

## Scope

- `IMqttConnection` singleton wrapping one `IMqttClient` created from `MqttClientFactory`
  (MQTTnet 5 — `MqttFactory` no longer exists).
- `MqttConnectionService : BackgroundService` owning connect, reconnect, and shutdown. Connection is
  never established in a request path and never awaited during startup.
- Protocol version pinned to MQTT 5.0.0 explicitly rather than relying on the v5 default.
- Fixed `ClientId` from options, `CleanStart = false`, `SessionExpiryInterval` from options.
- Reconnect: exponential backoff with full jitter, capped at 30 s. `CancellationToken` threaded
  through the entire loop.
- Last Will: `gateway/{clientId}/status`, payload `offline`, QoS 1, retained. On graceful shutdown,
  publish `offline` explicitly and send DISCONNECT so the will does not fire.
- On connect, publish `online` retained to the same topic.
- `GET /api/connection` returning state, client id, broker endpoint, `subscriptionsRestored`,
  active subscription count, connected-since, reconnect count.
- Unit tests for the backoff schedule — monotonic, capped, jittered within bounds.

## Acceptance criteria

- [ ] The application starts with the broker **down** and reports `Disconnected`
- [ ] Starting the broker afterwards connects without a process restart
- [ ] `Ctrl+C` disconnects cleanly and the process exits within the shutdown timeout
- [ ] `IsConnected` is never used as the sole source of reported state — it reflects last known
      state, not a live probe

## Gaps closed

GAP-19 (no API reconnect), GAP-12 (LWT never implemented), GAP-33 (clean session never considered),
GAP-37 (MQTTnet v4 API against a v5 package).

## Traps

`MQTTnet.Extensions.ManagedClient` is frozen at v4 and incompatible with v5 — `ManagedMqttClient`
was removed permanently. Reconnect is written here, not imported.
