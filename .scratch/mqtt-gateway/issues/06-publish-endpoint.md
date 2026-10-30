# 06 — `POST /api/publish` and topic-name validation

Status: ready-for-agent
Depends on: 04, 05

## Goal

A caller that does not speak MQTT can publish a message, and is told honestly what the response
means.

## Scope

- `POST /api/publish` accepting `topic`, `payload`, `payloadEncoding` (`utf8` default, or `base64`),
  `qos`, `retain`.
- `202 Accepted`, documented as "accepted for delivery to the broker" and nothing stronger. QoS 0 is
  never acknowledged by a broker, so no stronger promise is available for every case, and two
  meanings for one status code would be a lie in one of them.
- `503` with a broker-specific problem type when the connection is down. Never `200`.
- **Topic name** validation — a separate rule set from topic filters: `+` and `#` rejected, empty
  rejected, `$`-prefixed rejected, length capped at 512 bytes.
- QoS bound to `MqttQualityOfServiceLevel`, not a raw `int`.
- Payload decoded per `payloadEncoding`; size checked against `Ingest:MaxPayloadBytes`.
- Publish is **not** retried by Polly: it is not idempotent from a subscriber's point of view, and
  a retry after an ambiguous failure can deliver twice.

## Acceptance criteria

- [ ] Publishing `sensors/room1/temp` is observed by an external subscriber (MQTTX or a test client)
- [ ] `fabrika/#` as a publish topic returns `400`, not a broker error
- [ ] `qos: 7` returns `400` with a field-level message
- [ ] With the broker stopped, publish returns `503` with the broker problem type
- [ ] A base64 payload arrives byte-identical at the subscriber
- [ ] Unit tests cover topic-name validation including the wildcard asymmetry

## Gaps closed

GAP-14 (retain never publishable), GAP-30 (payload assumed UTF-8), GAP-32 (no error contract).
