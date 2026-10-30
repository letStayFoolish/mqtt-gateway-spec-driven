# ADR-0003 — Channel-based ingest with backpressure

Date: 2026-10-30 · Status: Accepted

## Context

MQTTnet invokes the received-message handler **sequentially** for a given client: the handler for
message *n+1* does not begin until the handler for message *n* completes. Three consequences
follow, and the obvious implementations trip over all of them.

Writing to the database inside the handler makes database latency the ceiling on ingestion rate for
*every* topic. Dispatching the write without awaiting it loses the exception, loses ordering, and
runs concurrent operations on a `DbContext` that is not thread-safe. And by default MQTTnet
acknowledges a QoS 1 message as soon as the handler returns — so if the subsequent write fails, the
message is gone while the broker believes it was delivered.

## Decision

```
receive handler → Channel<ReceivedMessage> → batch writer (BackgroundService)
                                               ├─ scope per batch → DbContext → SaveChanges
                                               └─ Redis statistics
```

| Parameter | Value |
|---|---|
| Capacity | 10 000, bounded |
| Full mode | `BoundedChannelFullMode.Wait` |
| `args.AutoAcknowledge` | `false` — acknowledge after the batch is persisted |
| Batch size | 500 |
| Batch window | 100 ms, whichever comes first |
| Shutdown | Drain with a bounded timeout |

The handler's only job is to stamp the receipt time once, build the record, and write it to the
channel.

## Consequences

- **A slow database slows ingestion instead of losing messages.** This is the trade, and it is
  bought deliberately. `Wait` propagates backpressure to the handler, which delays acknowledgement,
  which makes the broker slow down. That is the mechanism MQTT provides; using it is the point.
- QoS 1 means "at least once" all the way to the database, not merely to the process.
- Because acknowledgement is deferred, redelivery after a crash is expected. Ingest must therefore
  tolerate duplicates — they are visible as repeated rows, which is the honest outcome for a message
  log. Deduplication would require a broker-assigned identifier that MQTT 3.1.1 does not provide.
- An unbounded channel was rejected: it converts backpressure into a memory leak that surfaces as
  an out-of-memory kill under exactly the load that needed the signal.
- `DropOldest` was rejected: it keeps the API fast by losing data silently, which is the worst
  property a gateway can have.
- Channel depth, batch size, and ingest latency are exported as metrics. Without them the
  backpressure decision is unobservable and therefore untunable.
- Batching makes per-message latency worse by up to the batch window. For a message log that is an
  acceptable price for an order-of-magnitude improvement in throughput.
