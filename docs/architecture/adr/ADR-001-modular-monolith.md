# ADR-001: Modular monolith with hexagonal modules, multiple processes

- Status: Accepted
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
One owner-developer assisted by AI agents (NFR-12) must deliver ten bounded contexts on a low budget (NFR-10), for ≤100 beta users growing to ~10k (NFR-01). Some flows span contexts transactionally (BR-11 mark-as-bought). User tools need isolation (NFR-09) and the collector has a very different workload (NFR-02).

## Options
1. Microservices per bounded context (separate deployables, broker).
2. Classic layered monolith without enforced boundaries.
3. Modular monolith: one package per bounded context, hexagonal inside, boundaries enforced by import-linter, deployed as a few processes (web, worker, sandboxed tool-runner) plus the separate collector.

## Decision
Option 3. Processes are split only where a requirement needs it: `iip-tool-runner` for isolation (NFR-09), `iip-worker` to keep slow/external work off requests (NFR-03), collector for workload/cadence (ADR-002).

## Consequences
+ one deploy, one DB transaction for cross-context invariants (no sagas except account deletion), cheap to run, easy for agents to navigate.
+ contexts can be extracted later (strangler fig) because they talk via facades/events.
− discipline needed to keep boundaries: import-linter contracts and guidelines G-01..G-03 are mandatory CI checks.
− v0.1: single host, vertical scaling. v0.2: web/worker/runner are separate managed containers that scale horizontally; the DB scales vertically (ADR-007, ADR-015).
