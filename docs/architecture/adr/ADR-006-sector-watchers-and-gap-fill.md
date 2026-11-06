# ADR-006: Per-sector watchers and per-filing gap fill

- Status: Accepted
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
BR-15 (one watcher instance per sector), FR-03 (detect new filings, fill gap between bulk postings), NFR-04 (visible within 24 h, ASSUMPTION), NFR-07.

## Options
1. Each sector watcher polls EDGAR independently (N× requests).
2. One shared discovery (`fdc-feed`: latest-filings feed every 15 min + nightly daily-index reconcile) writing a `filing_inbox` with sector; one watcher instance per sector (`fdc-watch@<sector>`) claims and loads its filings from per-filing XBRL (Arelle) as provisional; bulk data later supersedes.

## Decision
Option 2. Sectors from SIC via `sectors.yaml` (A-02).

## Consequences
+ BR-15 satisfied with minimal SEC requests; per-sector isolation of failures and backlogs.
+ < 24 h guaranteed by the daily-index reconcile.
− per-filing XBRL → FSDS-shaped records may differ (R-02); provisional flag shown in UI until bulk supersedes; spike in M2.
