# ADR-015: Hosting: EU VPS, systemd, Cloudflare, continuous backups

- Status: Proposed — owner decision Q-02 / Q-11
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
NFR-06 (99 %), NFR-08 (EU hosting), NFR-10 (low cost), NFR-12 (owner already runs systemd + Cloudflare), NFR-02 (store size), existing 1 GB server is too small and hosts other apps.

## Options
1. PaaS (Fly.io/Render) + managed Postgres.
2. Kubernetes.
3. Single EU VPS (4 vCPU/8 GB, size confirmed by M0) with systemd, Cloudflare DNS/WAF/Tunnel, Litestream + object storage backups.

## Decision
Option 3.

## Consequences
+ ≈ €25–55/month before price vendor and LLM; familiar operating model.
− single point of failure (accepted by 99 % SLO; rebuild runbook); vertical scaling only until ADR-007 criteria trigger.
