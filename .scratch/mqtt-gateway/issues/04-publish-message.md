# 04 — Publish a message

**What to build:** A caller that does not speak MQTT can publish a message over HTTP and have it
arrive at the broker — with the retain flag and QoS of its choosing, and with binary payloads
passing through intact. This slice also establishes the error contract every later endpoint shares:
one problem shape for every failure, and a broker outage that cannot be mistaken for a bad request.

**Blocked by:** 03

**Status:** ready-for-agent

- [ ] Publishing through the API is observed by an external MQTT subscriber
- [ ] The caller chooses QoS and the retain flag per message
- [ ] A binary payload arrives byte-identical; it is neither assumed to be text nor corrupted
- [ ] A payload larger than the configured ceiling is rejected
- [ ] A wildcard in a publish topic is rejected with a validation error, not forwarded to the broker
- [ ] Topic **names** and topic **filters** are validated by separate rules and do not share a
      routine — the grammars differ
- [ ] An out-of-range QoS is rejected with a field-level message
- [ ] Every failure response uses the same problem shape; no endpoint returns a bare string or an
      untyped object
- [ ] A broker outage and a bad request are distinguishable by status and by problem type
- [ ] The success response documents what it promises — acceptance for delivery to the broker,
      nothing stronger, because the lowest QoS is never acknowledged by any broker
- [ ] No controller contains exception handling of its own
- [ ] Validation is wired through an explicit filter rather than the deprecated framework
      integration package
- [ ] Publishing is not retried automatically; it is not idempotent from a subscriber's perspective
- [ ] Unit tests cover topic-name validation, including the wildcard asymmetry and the length ceiling
