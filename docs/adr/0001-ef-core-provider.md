# ADR-0001 — EF Core provider for MariaDB

Date: 2026-10-30 · Status: Accepted

## Context

The service persists to MariaDB through EF Core on `net10.0`. The default choice for MySQL and
MariaDB has been `Pomelo.EntityFrameworkCore.MySql` for years, and it is what the original
assignment specifies.

Verified on 2026-10-30:

| Signal | Finding |
|---|---|
| Latest stable | 9.0.0, published 2025-08-17 |
| Declared dependency | `Microsoft.EntityFrameworkCore.Relational >= 9.0.0 && <= 9.0.999` |
| Latest prerelease | `9.0.0-rc.1.efcore.9.0.0`, 2025-07-28 — older than the stable release |
| Last commit to `main` | 2025-08-17 |
| PR #2019 "Upgrade to EF Core 10.0.0" | Opened 2025-11-15, still a draft after eleven months. Requests for a preview build and an ETA went unanswered |
| Issue #2050 "Compatibility with .NET 10 / EF Core 10" | Open, no maintainer response |
| Issue #2051 "Has the project stopped maintenance?" | Opened 2026-10-19, no maintainer response |
| Microsoft's provider list | Pomelo listed for "EF Core 8, 9" only |
| NuGet deprecation banner | **None** |

The absence of a deprecation banner is what makes this a trap rather than a signpost. The package
targets `net8.0`, so it installs into a `net10.0` project without a warning and silently caps the
project at EF Core 9.

## Options

**Pomelo 9.0.0 with EF Core 9 on `net10.0`.** Works. The one real C# 14 regression — first-class
spans changing overload resolution for `Contains` over enum arrays — was backported and fixed in
November 2025. But it means starting a new project, today, on a provider with no maintainer
response to two direct questions about whether it is abandoned.

**`Microting.EntityFrameworkCore.MySql` 10.0.14** (published 2026-10-29). A direct fork of Pomelo.
The API is identical: `UseMySql(connectionString, serverVersion)`, `MariaDbServerVersion`,
`MySqlServerVersion`, and `ServerVersion.AutoDetect()` all keep their names. Only the package ID,
the assembly name, and the namespace change. Approximately 1.5 million downloads, released within
days of each EF Core patch, and CI-tested against MariaDB 10.5 through 11.6. EF Core Power Tools
adopted it for EF Core 10 in July 2026 for the same reason.

**`Doka.EntityFrameworkCore.MySql`.** Written from scratch, explicitly not a Pomelo drop-in.
First release August 2026, roughly 6 800 downloads. Too young to depend on.

**Oracle's `MySql.EntityFrameworkCore`.** Documents MySQL 8.0 or later and never mentions MariaDB.
Different API surface.

**PostgreSQL with Npgsql.** The best-supported EF Core provider, and the default in our own
standards. Rejected: MariaDB is a requirement of the assignment, and swapping the database to
avoid a provider problem is avoiding the problem rather than solving it.

## Decision

`Microting.EntityFrameworkCore.MySql` 10.0.14 with EF Core 10.0.14.

## Consequences

- EF Core 10 (LTS, supported to 2028-11-10) rather than a version line already superseded.
- A dependency on a fork rather than the canonical package. The fork's own description calls it a
  fast-moving maintenance vehicle with "no intentions for this package to implement new features."
  For this service that is sufficient — we need EF Core patch tracking, not new provider features.
- No spatial companion package is published. Not needed here.
- Migrating back to Pomelo, should it revive, is a package reference and a namespace change.
- Provider-specific behaviour is confined to `Infrastructure/Persistence/`, so the surface exposed
  to this risk is small and known.
