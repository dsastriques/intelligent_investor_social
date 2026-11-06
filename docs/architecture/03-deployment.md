# 03 — Deployment, infrastructure and operations (input for the devops skill)

Serves NFR-01, NFR-02, NFR-04..NFR-08, NFR-10, NFR-12, FR-07.

## 1. Deployment components

```mermaid
flowchart TB
  user([Browser]) --> cf[Cloudflare: DNS, TLS, WAF, rate limiting, Turnstile, static cache]
  cf -->|Tunnel, no open inbound ports| cfd[cloudflared]
  subgraph VPS[EU VPS prod - Debian 12, systemd]
    cfd --> web[iip-web.service<br/>uvicorn x2-4]
    web <--> appdb[(app.db)]
    web --> mdb[(market.db)]
    web -->|ro| store[(as-reported store)]
    worker[iip-worker.service] <--> appdb
    worker <--> mdb
    worker -->|ro| store
    worker <-->|unix socket| runner[iip-tool-runner.service<br/>sandboxed, no network]
    runner -->|ro| mdb
    timers[systemd timers: prices, plan reviews, reconcile, backups] --> worker
    fdcfeed[fdc-feed.timer] --> store
    fdcw[fdc-watch@sector.service x ~11] --> store
    fdcb[fdc-backfill.service manual/quarterly] --> store
    fdcf[fdc-ffiec.timer] --> store
    ls[litestream.service] --> appdb
  end
  ls --> obj[(EU object storage<br/>backups + raw zips)]
  fdcb --> obj
  worker --> email[(Email provider EU)]
  worker --> llm[(LLM API)]
  worker --> pv[(Price vendor)]
  fdcfeed --> sec[(SEC)]
  hc[(healthchecks.io)] -.-> timers
```

## 2. Environments

| Env | Where | Data | Purpose |
|---|---|---|---|
| dev | developer machine / Claude Code sandbox | `fdc export` seed bundle (~50 companies), synthetic users | development, unit/integration tests |
| ci | GitHub Actions | fixtures only; **no SEC/vendor calls** (C-8) | lint, types, tests, e2e on seed |
| staging | same VPS, separate Unix users, ports, DBs and hostname `staging.<domain>` behind Cloudflare Access (owner only) | seed bundle + anonymised copy option | release rehearsal, migration tests, e2e smoke |
| prod | EU VPS | full store, real users | V1 |

ASSUMPTION A-11: Pro runs on a **new** VPS, not the existing 1 GB server (which keeps running the screener/The Trader). Staging shares the prod VPS to save cost (ASSUMPTION A-20; a second small VPS is the upgrade path).

## 3. Compute choice (ADR-015)

- **One dedicated EU VPS** with systemd services (same operating model the owner already uses), Python via `uv` into a venv per app, no containers required. Rationale: lowest cost and ops load for one developer; SQLite needs local disk; systemd gives sandboxing (runner) and restart policies.
- **Proposed size (validated by M0):** 4 vCPU, 8 GB RAM, 160 GB+ SSD (e.g. Hetzner CPX31/CX-line in Falkenstein/Nuremberg/Helsinki, ≈ €15–20/month) plus a block volume for the store if M0 projects > 100 GB. The 1 GB server is too small for Arelle, uvicorn workers, runner and SQLite page cache together.
- Containers (Docker/Podman) optional later for reproducibility; not required by any NFR.

## 4. Networking, scalability, latency

- DNS + TLS at Cloudflare; **Cloudflare Tunnel** to origin (no inbound ports except SSH restricted by key + firewall); WAF managed rules, rate limiting on `/auth/*` and `/api/*`; Turnstile on sign-up/sign-in.
- Static assets (CSS/JS/images) with long-cache hashed names via Cloudflare cache; HTML not cached (per-user).
- **Scalability path (NFR-01):**
  1. ≤100 users: 2 uvicorn workers, 1 worker process, 2 runner slots.
  2. ~10k users (est. ≤ 5 req/s average, ≤ 50 req/s peak — ASSUMPTION A-21): 4 uvicorn workers, vertical VPS upgrade (8 vCPU/16 GB), SQLite WAL with short write transactions. Writes funnel through short transactions; heavy writes (prices, reconciliation) are in `market.db`, separate from `app.db`.
  3. Switch criteria to Postgres (ADR-007): sustained write lock waits > 50 ms p95, or a second app server needed for availability. Then: managed EU Postgres or Postgres on the VPS; repositories unchanged (SQLAlchemy Core).
- Stateless web: sessions in DB, so more workers/hosts are possible.
- Queue-based load levelling: tool runs, LLM calls, emails, plan reviews, alert evaluation are jobs.
- Connection handling: one SQLite connection per worker thread, `busy_timeout=5000`, `synchronous=NORMAL` in WAL.

## 5. Availability (NFR-06: 99 % monthly ≈ 7.3 h downtime budget)

- systemd `Restart=on-failure`, `WatchdogSec` for web/worker; Cloudflare health check on `/health` (DB reachable, disk free > 10 %, last price refresh < 36 h on trading days, collector last run age) → email/Telegram alert to operator.
- Single VPS is a SPOF accepted by the 99 % target (R-10); mitigated by documented rebuild (`deploy/install.sh` idempotent) + Litestream restore in < 1 h for app.db; the store restored from the latest `fdc export` bundle or rebuilt from raw zips.
- Collectors may be down a day without data loss: incremental, resumable, daily-index reconcile (NFR-06, BR-05).
- Deploys: brief restart (< 10 s) of web via systemd socket activation or two uvicorn instances behind cloudflared with sequential restart (rolling on one host).

## 6. Backups and disaster recovery (A-08)

| Data | Method | RPO | RTO |
|---|---|---|---|
| app.db | Litestream continuous replication to EU object storage (S3-compatible), 35-day retention; nightly `sqlite3 .backup` snapshot + `PRAGMA integrity_check` | ≤ 1 min (target ≤ 15 min) | ≤ 1 h restore, ≤ 4 h full host rebuild |
| market.db | nightly snapshot to object storage (rebuildable from vendor + store) | 24 h | 4 h |
| as-reported store | raw zips in object storage (source of truth) + weekly `fdc export --all` bundle | 24 h | ≤ 2 days rebuild from raw, hours from bundle |
| config/secrets | env files in an encrypted offline copy held by owner (password manager) | — | — |

Restore drill quarterly (runbook RB-05), verified by staging restore. No backups in GitHub (unlike the screener's quarterly push; personal data must not go to a public/US repo).

## 7. Observability and troubleshooting

- **Logs:** structlog JSON → journald (30-day retention, `SystemMaxUse`), correlation ids (`request_id`, `job_id`, `run_id`, `tool_run_id`, `collector_run_id`).
- **Metrics:** `/metrics` (Prometheus text, bound to localhost) — request latency histograms per route, job queue depth/age, tool run durations, LLM tokens, email failures, price refresh age, collector counters (from `collector_run`). V1: scraped by a free Grafana Cloud agent (EU region) **or** simply the `/ops` page — owner choice (low cost).
- **Health checks:** healthchecks.io per timer (start/success/fail, as in the screener); `/health` for Cloudflare.
- **Errors:** Sentry-compatible SDK to an EU-hosted error tracker (or GlitchTip self-host) — optional, owner choice.
- **SLIs/SLOs:** availability 99 % (Cloudflare health checks); page p95 < 2 s (NFR-03) from server timing; filings visible < 24 h (NFR-04) = `now − accepted` of newest filing per sector; prices refreshed by 08:00 CET next trading day (NFR-05).
- **Alerts:** health fail > 5 min; price refresh missed; collector stale; queue age > 15 min; disk > 80 %; backup replication lag > 15 min; LLM budget 80 %.
- **Runbooks** (`deploy/runbooks/`):
  - RB-01 SEC 403/429 or rate-limit block — check UA, lower `FDC_RATE`, resume.
  - RB-02 Backfill crashed / disk full — `fdc status`, free staging, resume (idempotent).
  - RB-03 Price refresh failed — vendor status, rerun job for date, alerts re-evaluate.
  - RB-04 SQLite "database is locked" spikes — identify long transaction via logs, check worker batch sizes, consider ADR-007 switch.
  - RB-05 Restore app.db from Litestream; restore store from bundle.
  - RB-06 Tool runner saturation / runaway — check limits, kill runs, adjust quotas.
  - RB-07 LLM budget exhausted — kill switch, inform users.
  - RB-08 GDPR request — export/delete commands and verification.

## 8. CI/CD

- **CI (GitHub Actions, per PR):** `uv sync` → `ruff check` + `ruff format --check` → `mypy` → `lint-imports` (import-linter contracts) → unit → integration (SQLite) → contract (store schema from `fdc` tag, runner protocol) → e2e (Playwright against app on seed bundle) → `pip-audit`. Required checks for merge.
- **Collector repo CI:** same lint/type/tests on fixture zips; never downloads from SEC (C-8).
- **CD (owner-triggered):** tag `vX.Y.Z` on main → GitHub Action builds a wheel and release notes (no deploy credentials in GitHub, matching the screener's "data on server only" rule) → on the server `deploy/install.sh vX.Y.Z` (pull tag, `uv sync --frozen`, `alembic upgrade head` against staging first, smoke e2e, then prod, restart units). Optionally a GitHub deployment via SSH key with a restricted forced command — owner choice.
- **Strategy:** staging → prod; rolling restart; rollback = reinstall previous tag; DB migrations expand/contract so the previous version runs on the new schema; Litestream point-in-time restore for disasters.
- **IaC:** `deploy/install.sh` (idempotent: users, dirs, venvs, units, cloudflared, litestream config) + `deploy/provision.md` for the VPS (Hetzner Cloud CLI or console). Terraform optional later.
- **Secrets in pipeline:** none needed for CI; deploy secrets live only on the server.

## 9. Collector operations

| Unit | Schedule | Notes |
|---|---|---|
| `fdc-backfill.service` | manual start (initial) then quarterly timer for new data sets | `MemoryMax=768M`, `Nice=10`, `IOSchedulingClass=idle` |
| `fdc-feed.timer` | every 15 min (+ 06:00 daily-index reconcile) | one SEC request per poll |
| `fdc-watch@<sector>.service` | long-running (or 15-min timer) per sector | shares host-wide limiter |
| `fdc-ffiec.timer` | quarterly + monthly check for amendments | |
| `fdc-export.timer` | weekly | bundle to object storage |

Unix user `fdc` owns the store; `iip` and `iip-runner` are in group `fdc-read` with read-only access.

## 10. Cost notes (owner decides ceiling, Q-02)

| Item | Estimate (EUR/month) | Notes |
|---|---|---|
| VPS 4 vCPU/8 GB EU | 15–20 | Hetzner-class pricing; verify current price |
| Extra volume 100–200 GB (if M0 needs) | 5–10 | |
| Object storage (raw zips + backups, ~100–300 GB) | 2–10 | Hetzner Object Storage / Backblaze B2 EU / Cloudflare R2 (EU jurisdiction) |
| Cloudflare (free plan: DNS, Tunnel, WAF basic, Turnstile) | 0 | |
| Email (EU provider, free/low tier for beta volumes) | 0–15 | |
| healthchecks.io free | 0 | |
| Price vendor EOD with multi-user display licence | **owner decision** (≈ 0 – 100+) | see ADR-012 |
| LLM (tool builder) | **owner decision**: ≈ 1–5 per active builder user/month at proposed quotas | see ADR-010 |
| **Total excl. price/LLM** | **≈ 25–55** | |

Figures are rough planning estimates to be verified at purchase time.

## 11. DevOps handoff checklist

1. Provision EU VPS (size per M0 result; default 4 vCPU/8 GB/160 GB) with Debian 12, SSH key-only, firewall (deny inbound except SSH from owner IPs), unattended security upgrades.
2. Create Unix users `iip`, `iip-runner`, `fdc`, group `fdc-read`; directories `/opt/iip`, `/srv/iip/{app.db,market.db}`, `/srv/fdc/{store,staging,raw}` with permissions per §9 and 06 §3.3.
3. Install uv, Python 3.12, sqlite3 CLI, litestream, cloudflared.
4. Write `deploy/install.sh` (idempotent) + `deploy/iip.env.example` + `/etc/iip/iip.env`, `/etc/fdc/fdc.env` (incl. `EDGAR_USER_AGENT`).
5. systemd units: `iip-web`, `iip-worker`, `iip-tool-runner` (hardening per 06 §3.3), timers (prices, plan reviews, reconcile, analytics rollups, backups), `fdc-*` units per §9, `litestream`; all jobs wrapped with healthchecks pings.
6. Cloudflare: domain/zone, Tunnel to `iip-web`, WAF managed rules, rate-limit rules on auth/API, Turnstile keys, Access policy for `staging.` and `/ops`.
7. Object storage bucket(s) in EU with lifecycle rules (35 days backups; raw zips kept), credentials scoped per bucket.
8. Litestream replication for `app.db`; nightly snapshot jobs for `market.db`; weekly `fdc export` upload; restore runbook RB-05 tested once.
9. Observability: journald limits, `/metrics` localhost, `/health` endpoint wired to Cloudflare health check + alert email; healthchecks.io checks per timer.
10. CI workflows for both repos (§8) with required checks; release workflow building wheels; no data calls in CI.
11. Staging environment on the same VPS (separate users/DBs/hostname) with seed bundle import.
12. Runbooks RB-01..RB-08 in `deploy/runbooks/`.
13. Cost tracking note: actual monthly cost recorded in `docs/architecture/03-deployment.md` §10 after provisioning.
