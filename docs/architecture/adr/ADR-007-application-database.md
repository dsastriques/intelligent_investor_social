# ADR-007: Application database: managed PostgreSQL (EU) from day one

- Status: **Accepted** (v0.2, follows the owner's managed-services preference, NFR-10). Provider: Scaleway, per ADR-015 option A (accepted by the owner 2026-11-06).
- Date: 2026-11-06 (v0.1 chose SQLite WAL + Litestream with switch criteria; superseded)
- Deciders: architect (design mode); owner for provider and cost

## Context
- v0.2 NFR-10: the owner prefers managed services if the budget allows, and it does (ADR-015, 03 §10).
- The serving plane runs on managed containers with no shared persistent disk, so SQLite cannot be the transactional store there.
- NFR-08: personal data must be backed up reliably in the EU.
- NFR-01: ~10k users without re-architecture.
- New in v0.2: BR-17/FR-29 add a billing ledger. It needs strict transactional integrity and row locking for credit reservations (ADR-020).

## Options
1. SQLite WAL + Litestream on a VM (v0.1).
2. Self-run Postgres on the VM.
3. **Managed PostgreSQL in the EU**: Scaleway DB-DEV-S (2 vCPU/2 GB) ≈ €11.4/month plus 20 GB block storage ≈ €2 ([Scaleway pricing](https://www.scaleway.com/en/pricing/managed-databases); hourly price × 730 h); HA node +≈ €9.9.
4. Other managed EU Postgres: Sliplane (Berlin, from €19, PITR included; [vendor blog](https://sliplane.io/blog/managed-postgres-providers-with-point-in-time-recovery)); Ubicloud on Hetzner hobby-1 $19.54, per-minute restore within 7 days ([Ubicloud pricing](https://www.ubicloud.com/docs/about/pricing)).

## Decision
Option 3, co-located with the serving plane: Scaleway, same region. One instance with two logical schemas and least-privilege roles:
- `app` holds identity, portfolio, watchlists, tools, social, clubs, notifications and billing. Table prefixes stay per context (02 §11).
- `market` holds instruments, prices, FX, standardized financials, lineage, filing events and read models.
- Roles: `iip_web` and `iip_worker` (read/write `app`, read `market`, write prices); `iip_batch` (batch plane: write `market.std_*` and `market.filing_event`, nothing in `app`); `iip_ro` (operator read-only). The tool runner gets **no** DB role (06 §3.3).
- Staging is a separate database (`iip_staging`) on the same instance (A-20).
- SQLAlchemy Core + Alembic are unchanged. Postgres-specific features are now allowed where they serve a requirement: `FOR UPDATE SKIP LOCKED` and `LISTEN/NOTIFY` (ADR-013), row locks for credit reservations (ADR-020), partial indexes.
- The as-reported store stays SQLite files (ADR-003); it is public, rebuildable and file-portable (FR-05).
- **Backups (A-08):** provider automated backups (on by default; retention and PITR **not verified**, see S-1), plus a nightly `pg_dump` from the batch box, encrypted with `age`, to Hetzner Object Storage with 35-day retention (a second provider guards against account loss). If S-1 shows no PITR, the beta accepts RPO ≤ 24 h for logical errors (R-14). **Standby node: accepted by the owner (2026-11-06, NFR-06), part of the planned set-up from the beta.** Synchronous hot standby with automatic failover, ≈ +€9.9/month. It covers node failure (RPO ≈ 0) but not logical errors, which replicate to the standby; dumps/PITR cover those (R-14).
- Option 4 is the fallback if Scaleway backups or PITR fall short, or if option C/D hosting moves elsewhere.

## Consequences
+ No DB patching or backup scripts to run. Concurrent writers and row locking (billing, job queue). The 10k-user path is vertical node upgrades and a read replica, not a migration.
+ v0.1's switch criteria and planned SQLite→Postgres migration disappear (R-08 closed).
− About €13–23/month, and every DB call crosses the network (≈ 1 ms in-region). Mitigated by connection pooling (SQLAlchemy pool per container) and per-request query budgets (NFR-03).
− Integration tests need Postgres: a GitHub Actions service container, and `testcontainers` locally (04 §1).
