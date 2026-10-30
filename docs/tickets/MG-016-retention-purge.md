# MG-016 — Configurable message retention

Depends on: MG-011

## Goal

The message table has a bounded size when an operator wants one.

## Scope

- `RetentionPurgeService : BackgroundService` deleting rows older than `Retention:Days` in bounded
  batches, on a configurable interval.
- Default `Retention:Days = 0`, meaning disabled — preserving the original assignment's behaviour.
  The mechanism exists and is tested, so the limit is a setting rather than an omission.
- Deletion is batched and throttled so a purge of a large backlog does not lock the table against
  ingest.
- Rows deleted exported as a metric.

## Acceptance criteria

- [ ] With retention enabled, rows older than the threshold are removed and newer rows are not
- [ ] With the default of `0`, nothing is ever deleted
- [ ] A purge of a large backlog does not block ingest beyond the batch duration
- [ ] The service survives a database outage and resumes

## Gaps closed

GAP-34 (message table grows without bound).
