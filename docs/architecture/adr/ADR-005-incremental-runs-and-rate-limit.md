# ADR-005: Incremental runs, run tracking, idempotency, SEC rate limit

- Status: Accepted
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
BR-05 (collect only missing data, track last run), FR-07 (run history, health), NFR-06 (collector down a day without loss), NFR-07 (UA, ≤10 req/s), several collector processes (backfill, feed, ~11 watchers, FFIEC) on one host.

## Options
1. Per-process sleeps (as in the screener's EdgarClient).
2. Host-wide token bucket shared via a file-locked state + mandatory UA; `collector_run` and `collected_item` tables; idempotent per item and per `adsh`.

## Decision
Option 2. Default 8 req/s host-wide; backoff on 403/429; one active run per collector+scope; item recorded only after commit; rerun skips items with unchanged checksum/ETag and retries failures; `fdc status` + healthchecks pings.

## Consequences
+ guaranteed fair-access compliance regardless of how many watchers run.
+ crash-safe resumption; reruns are cheap.
− small shared-state component (lock file) to test across processes (04 §6).
