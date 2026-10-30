# 06 — Subscription restoration

**What to build:** A connected gateway with no subscriptions is never reported as working. Every
stored filter is re-established on every connect — not once at startup — before the service claims
it is ready.

**Blocked by:** 05

**Status:** ready-for-agent

- [ ] Integration test: killing the broker and restarting it re-establishes every subscription and
      resumes delivery, with no process restart
- [ ] Restoration runs on every connect, not only the first
- [ ] While restoration is incomplete, readiness is unhealthy and the connection-state resource says
      so explicitly
- [ ] Integration test: a forced restoration failure disconnects and retries the whole set rather
      than serving traffic in a half-restored state
- [ ] Registering a filter, restarting the process, and publishing delivers the message without
      re-registering through the API
- [ ] Each subscription is restored at the QoS stored with it
