# 03 — Broker connection

**What to build:** The gateway holds exactly one authenticated connection to the broker for the
lifetime of the process, recovers from broker outages on its own, and can state its real connection
condition on request. Other MQTT clients learn when it dies unexpectedly — and are not misled when
it shuts down cleanly.

**Blocked by:** 02

**Status:** ready-for-agent

- [ ] A connection-state request reports the current condition, the client identity, the broker
      endpoint, how long the connection has been up, and how many times it has reconnected
- [ ] The application starts and reports itself not ready when the broker is unreachable, rather
      than failing to start
- [ ] Starting the broker afterwards connects without restarting the process
- [ ] Reconnection uses exponential backoff with jitter and a ceiling; the schedule is unit-tested
      as monotonic, capped, and jittered within bounds
- [ ] Cancellation propagates through the whole connection chain, including the reconnect loop, so
      shutdown completes within its timeout rather than hanging
- [ ] The protocol version is pinned explicitly rather than inherited from a library default
- [ ] A durable session is used, with a bounded expiry interval
- [ ] A last will is registered and retained on a status topic
- [ ] Integration test: severing the connection without a protocol-level disconnect delivers the
      last will to a second client
- [ ] Integration test: a clean shutdown publishes the offline state explicitly and does **not**
      fire the will
- [ ] Integration test: two instances sharing a client identity produce an observable session
      takeover — the documented cost of the durable session
- [ ] Reported state never relies solely on the client library's cached connection flag, which
      reflects last known state rather than a live probe
