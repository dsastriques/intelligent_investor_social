# ADR-013: Async without a broker: outbox, Postgres job queue, filing-event table

- Status: Accepted (v0.2: moved from SQLite to managed Postgres, ADR-007)
- Date: 2026-11-06
- Deciders: architect (design mode)

## Context
FR-12, FR-15, FR-17, FR-18 need reliable reactions to events: tool run done, prices refreshed, filing collected. FR-29 needs metering settlements and payment webhooks processed exactly once. NFR-10 and NFR-12 discourage running a broker. v0.2 hosting splits a serving plane (managed containers) from a batch plane (Hetzner, ADR-015). The two planes share only the Postgres instance and object storage.

## Options
1. Managed broker (e.g. Scaleway Messaging/Queues, NATS) or Redis.
2. Transactional outbox + job table in Postgres. Workers claim jobs with `SELECT ... FOR UPDATE SKIP LOCKED`, and `LISTEN/NOTIFY` wakes them. The scheduler runs in the worker behind a Postgres advisory lock (one active scheduler even with two worker instances). Collector events reach the app through a `market.filing_event` table that the batch plane appends to after it publishes a snapshot (ADR-019); the worker consumes it with a stored cursor.

## Decision
Option 2. Scheduled jobs keep their v0.1 definitions (prices 23:30 Europe/Amsterdam + 07:30 catch-up, plan reviews, alert evaluation, analytics roll-ups, snapshot builds). Batch-plane schedules stay systemd timers on the Hetzner box. Payment webhooks are written to `bl_webhook_inbox` (unique provider event id) and processed as jobs (ADR-020).

## Consequences
+ Exactly-once effects through idempotent consumers (`processed_event`, unique keys). No extra infrastructure. Everything can be inspected with SQL.
+ `LISTEN/NOTIFY` removes v0.1's polling latency for interactive jobs such as tool runs.
− Job throughput is bounded by Postgres, more than enough for NFR-01. A broker can be added later behind the same `JobQueue` port.
