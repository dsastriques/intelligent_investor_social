# ADR-013: Async without a broker: outbox, DB job queue, catalog polling

- Status: Accepted
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
FR-12, FR-15, FR-17, FR-18 need reliable reactions to events (tool run done, prices refreshed, filing collected); NFR-10/NFR-12 discourage running a broker.

## Options
1. RabbitMQ/Redis queue.
2. Transactional outbox + job table in `app.db`, worker polling (SKIP-LOCKED-like claim via UPDATE ... RETURNING), systemd timers for schedules; collector events consumed by polling `catalog.filing_event` with a stored cursor.

## Decision
Option 2.

## Consequences
+ exactly-once effects via idempotent consumers; no extra infrastructure; easy to inspect with SQL.
− polling latency (seconds) — fine for EOD/long-term use; revisit with Postgres LISTEN/NOTIFY if switched.
