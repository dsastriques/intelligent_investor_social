# ADR-003: As-reported store: catalog + one SQLite file per company

- Status: Accepted, to be validated by M0
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
T3: owner suggested SQLite, single file or per company. Data: ~6–8k filers × 12 years; FSDS quarterly zips 60–120 MB, notes much larger (NFR-02). Must be exportable to seed new DBs (FR-05), keep everything as reported (BR-02), and serve company pages quickly (NFR-03) on a small server.

## Options
1. Store as-reported facts inside the app transactional DB.
2. One large SQLite (or Postgres) DB for all facts.
3. `catalog.db` (companies, filing index, runs, collected items, event feed) + one SQLite file per company.
4. Columnar files (Parquet per quarter) + DuckDB.

## Decision
Option 3, behind `StoreWriter`/`store_client` ports so the layout can change. M0 measures options 2 and 3 side by side; switch to 2 if per-file overhead > 30 % or file count makes operations painful. Option 4 is kept as a future analytics read model, not the store of record.

## Consequences
+ portable file sets (FR-05), small hot files for company pages, isolated corruption, easy partial rebuilds, simple backups of a rebuildable asset.
+ app DB stays small for Litestream (ADR-007).
− cross-company queries on raw facts are expensive; mitigated by Reconciliation's standardized table in market.db.
− ~8–10k files to manage; catalog must stay consistent (loads write company file first, then catalog, idempotently).
