# ADR-007: Application database: SQLite WAL, Postgres-ready

- Status: Accepted
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
NFR-10/NFR-12 favour SQLite (owner already uses it); NFR-01 needs a path to 10k users; NFR-08 needs reliable backups of personal data.

## Options
1. Managed Postgres from day one.
2. SQLite (WAL) for `app.db` and `market.db`, SQLAlchemy Core + Alembic, Litestream continuous backup, with explicit switch criteria to Postgres.

## Decision
Option 2. Switch to Postgres when: write lock waits p95 > 50 ms sustained, or a second app host is required, or DB > ~20 GB. Repositories use SQLAlchemy Core with portable SQL; no SQLite-only features in `app.db` except pragmas.

## Consequences
+ zero DB ops cost, very fast local reads, simple dev/test (copy a file).
+ market data in a separate file keeps heavy writes away from user writes.
− single writer per DB; mitigated by short transactions and job-based writes.
− migration to Postgres is a planned, tested step (expand/contract, data copy) if criteria trigger.
