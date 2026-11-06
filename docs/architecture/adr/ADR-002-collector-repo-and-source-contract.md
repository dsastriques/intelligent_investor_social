# ADR-002: Collector in its own repo with a pluggable source contract

- Status: Accepted
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
T9: should the collector live in `financial-data-collector` or inside the product repo? FR-04 requires pluggable collectors per source with one output contract; BR-16 adds FFIEC call reports. The owner created an empty public `financial-data-collector` repo. SEC data is public domain.

## Options
1. Collector as a module inside the product repo.
2. Separate repo/package `fdc` with a published store schema and read-only client; app depends on a pinned tag.

## Decision
Option 2. `fdc` owns BC-1 Filings: `SourceCollector` port (`list_items`, `fetch`, `read` → streaming records `FilingRecord`, `TagRecord`, `FactRecord`, `PresentationRecord`, `TextRecord`, `DimensionRecord`), adapters `sec-fsds`, `sec-fsn`, `sec-filing` (Arelle), `ffiec-cdr`. The store schema (`sql/store_schema_v1.sql`) + `store_client` are the published language; the app pins `fdc` by git tag.

## Consequences
+ independent cadence and CI; reusable by the screener later; public repo fits public data.
+ a hard boundary that enforces BR-02 (the app cannot write the store).
− two repos to version; schema changes need coordinated releases (contract tests, 04 §5).
− no raw data in the repo (ADR-018).
