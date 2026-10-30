# 02 — Infrastructure compose: EMQX, MariaDB, Redis, telemetry

Status: ready-for-agent
Depends on: 01

## Goal

`docker compose up -d` brings up a healthy broker, database, Redis, and telemetry dashboard. The
application is not containerised yet (18).

## Scope

- Compose project name `mqtt-gateway-spec` — deliberately distinct, so volumes are never shared
  with another project using the same ports.
- Services, all with healthchecks and pinned tags:
  - `emqx` — `emqx/emqx:6.3.2`, ports 1883 and 18083, `allow_anonymous = false`, gateway
    credentials and dashboard password from `.env`, healthcheck `emqx ctl status`
  - `mariadb` — `mariadb:12.3`, port 3306, database `mqtt_gateway`, healthcheck
    `healthcheck.sh --connect --innodb_initialized`
  - `redis` — `redis:8.10.3`, port 6379, healthcheck `redis-cli ping`
  - `aspire-dashboard` — `mcr.microsoft.com/dotnet/aspire-dashboard:10.0`, OTLP ingest plus UI
- Named volumes for EMQX data and MariaDB data. Redis has none — statistics are derived state, and
  losing them on restart is correct.
- `.env.example` committed with placeholders; `.env` git-ignored. Every variable uses the
  `${VAR:?message}` form so a missing value fails loudly instead of defaulting to empty.
- Ports bound to `127.0.0.1`.

## Acceptance criteria

- [ ] `docker compose up -d` reaches healthy on all four services from a clean state
- [ ] MQTTX connects to 1883 **only** with credentials; anonymous connection is refused
- [ ] `docker compose down && up -d` preserves database contents
- [ ] No credential appears in any committed file

## Gaps closed

GAP-41 (`:latest` tags), GAP-43 (EMQX anonymous access and default dashboard credentials),
GAP-44 (secrets in the repository).
