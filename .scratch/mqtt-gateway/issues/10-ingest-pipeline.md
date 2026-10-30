# 10 — Channel-based ingest with backpressure

Status: ready-for-agent
Depends on: 07, 09

## Goal

Every received message is persisted exactly as described in [ADR-0003](../../../docs/adr/0003-ingest-pipeline.md),
without the receive handler ever touching the database.

## Scope

- Receive handler: stamp `DateTime.UtcNow` **once**, build the record, write to the channel. Nothing
  else.
- `Channel<ReceivedMessage>`, bounded at 10 000, `BoundedChannelFullMode.Wait`.
- `args.AutoAcknowledge = false`; acknowledge after the batch is persisted.
- `MessageIngestWriter : BackgroundService` draining up to 500 messages or 100 ms, whichever comes
  first; one `IServiceScopeFactory` scope — and therefore one `DbContext` — per batch.
- Payloads above `Ingest:MaxPayloadBytes` are rejected at the handler, counted, and logged at
  `Warning`. They never enter the channel and never stall it.
- Graceful shutdown drains in-flight messages with a bounded timeout.
- No `Task.Run`. No `Task.WhenAll` over a database and a Redis call.

## Acceptance criteria

- [ ] 1000 messages at QoS 1 produce at least 1000 rows, with no exception in the logs
- [ ] A deliberately slow database slows ingestion and does not lose messages or throw
      `A second operation was started on this context instance`
- [ ] Shutdown mid-flight persists the queued messages rather than dropping them
- [ ] An oversized payload is rejected and counted, and the next message is processed normally
- [ ] No singleton holds a `DbContext`; scope validation is enabled in Development and passes

## Gaps closed

GAP-01 (captive `DbContext`), GAP-02 (concurrent `SaveChanges`), GAP-10 (thread-unsafe in-memory
list), GAP-24 (`Task.Run` over async I/O), GAP-25 (swallowed `WhenAll` exception), GAP-28
(timestamp read twice), GAP-30 (payload bytes).

## Notes

Deferred acknowledgement means redelivery after a crash is expected and duplicate rows are
possible. That is the honest outcome for a message log; MQTT 3.1.1 provides no broker-assigned
identifier to deduplicate against.
