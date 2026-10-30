# MG-018 — Dockerfile and application in compose

Depends on: MG-013

## Goal

`git clone` → `docker compose up -d` → working system, on a machine with nothing but Docker.

## Scope

- Multi-stage Dockerfile: `mcr.microsoft.com/dotnet/sdk:10.0` to build,
  `mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled` to run.
- `Directory.Packages.props`, `Directory.Build.props`, and `.csproj` files copied **before** the
  rest of the source, so the restore layer survives source edits.
- Non-root user. No `EXPOSE 443`, no certificate handling in the container — TLS terminates
  upstream.
- The application listens on **8080**. .NET 8 and later images do not listen on 80.
- `.dockerignore` excluding `bin/`, `obj/`, `.git/`, `.vs/`, test output, and `*.log`.
- `app` service added to compose with `depends_on: condition: service_healthy` for EMQX, MariaDB,
  and Redis, and for the `migrator` to have completed.
- All configuration through environment variables; `__` maps to `:`. No secret in the compose file.
- Hostnames inside the network are service names — `emqx`, `mariadb`, `redis`. `localhost` inside a
  container is that container.

## Acceptance criteria

- [ ] `docker compose up -d` from a clean checkout yields a working system with no manual step
- [ ] `GET http://localhost:8080/health/ready` returns `200` once dependencies are healthy
- [ ] An end-to-end publish → ingest → query → statistics cycle works entirely inside compose
- [ ] `docker compose down && up -d` preserves messages and subscriptions; statistics do not persist,
      which is correct for derived state
- [ ] Editing a `.cs` file and rebuilding does not re-run `restore`
- [ ] The final image contains no SDK and no source

## Gaps closed

GAP-04 (port 80 versus 8080), GAP-07 (`depends_on` without readiness), GAP-08 (no migration path),
GAP-41 (`:latest` tags), GAP-44 (secrets in compose).
