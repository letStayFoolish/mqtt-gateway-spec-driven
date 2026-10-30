# 05 — Subscriptions

**What to build:** A caller registers a topic filter over HTTP and the gateway starts listening to
it — durably, so that the configuration survives a restart. Filters can be listed and removed.
Registering the same filter twice is a conflict, guaranteed by the database rather than hoped for by
a check. This slice introduces persistence and migrations.

**Blocked by:** 04

**Status:** ready-for-agent

- [ ] Registering a filter returns a created response with a location, and the gateway begins
      receiving matching messages
- [ ] Filters can be listed and removed; removal stops delivery, verified by publishing afterwards
- [ ] Each subscription carries its own QoS
- [ ] Wildcard filters are accepted; a multi-level wildcard anywhere but the final level is rejected
- [ ] Integration test: eight concurrent registrations of the same filter produce exactly one stored
      row — one created response and seven conflicts
- [ ] Uniqueness is enforced by a database constraint, and the resulting violation maps to a
      conflict response rather than an internal error
- [ ] Topic columns have an explicit length so the column type can carry an index; the live schema
      is verified, not assumed
- [ ] Timestamps are stored in UTC with sub-second precision
- [ ] The database server version is declared explicitly; the application does not open a connection
      during service registration and therefore starts even when the database is down
- [ ] A registration that fails at the broker leaves no stored row — the configuration never claims
      something the broker is not doing
- [ ] Migrations are applied by a dedicated step, not by the application at startup; applying them
      twice is a no-op
- [ ] The generated migration is reviewed before it is applied
- [ ] A database outage and a broker outage produce different problem types on this endpoint
