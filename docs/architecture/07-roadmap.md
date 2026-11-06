# 07 — Phased implementation plan

Milestones are ordered by risk: the data asset first (largest unknowns: size, FSN semantics), then the app skeleton, then features. Each milestone ends with a PR (or PRs) reviewed in architect impl mode. Durations are not estimated here (owner/coordinator plan the calendar).

## M0 — Collector sizing test (first milestone)

**Repo:** `financial-data-collector`. **Goal:** measured numbers that confirm or change ADR-003 (store layout), ADR-004 (FSN vs FSDS) and the VPS size (ADR-015), before any full backfill (NFR-02, research step 6).

Scope:
1. Repo scaffold: `pyproject.toml` (uv, ruff, mypy strict, pytest), hexagonal skeleton per 05 §2, CI (lint, types, tests on fixture zips, no SEC calls).
2. Domain + output contract records; `SourceCollector` port; `SecFsdsSource` and `SecFsnSource` streaming readers; host-wide rate limiter; `EDGAR_USER_AGENT` enforcement.
3. `StoreWriter` for per-company layout **and** single-DB layout (for comparison), staging loader, `collector_run`/`collected_item` tracking, restatement check.
4. `fdc sizing --quarter 2025q2 [--previous 2025q1] --layouts per-company,single --report docs/sizing/M0-report.md`.
5. Run on a machine with ≥ 2 GB RAM and on a 1 GB-limited cgroup (`systemd-run -p MemoryMax=1G`) to prove the 1 GB constraint.

**Acceptance criteria:**
- AC-1 Report contains, per data set (FSDS, FSN) and layout: zip size, download time, requests, load wall time, peak RSS, staging peak disk, final bytes, rows per table, company-file count and size distribution, p95 query time for a statement view.
- AC-2 Peak RSS ≤ 512 MB under `MemoryMax=1G`.
- AC-3 Rerun of the same quarter loads 0 new rows and records a second `collector_run` with `items_skipped` = all (BR-05).
- AC-4 FSN-vs-FSDS superset check on ≥ 20 sample companies with % facts reproduced.
- AC-5 12-year projection (store size, backfill duration, request count) with stated growth factor.
- AC-6 Restatement check run across two consecutive quarters with a count and five examples reviewed.
- AC-7 Recommendation section: per-company vs single DB; FSN vs FSDS(+FSN on demand); VPS disk size — handed to the owner for decision where cost is involved.
- AC-8 Unit/integration tests for parsing, idempotency and rate limiter green in CI; no raw zips committed (`.gitignore`, ADR-018).

## M1 — Collector backfill + store client
Full 12-year backfill on the provisioned VPS (after DevOps handoff items 1–5, 7), `store_client` read API (statement as presented, annual, consolidated, notes, revisions, events), `fdc status`, export/import bundles, seed bundle for dev/CI. FR-01, FR-02, FR-05, FR-06, FR-07, FR-08, BR-02..05.

## M2 — Watchers and FFIEC
Feed + per-sector watchers with Arelle per-filing loads (provisional → superseded), daily-index reconcile, NFR-04 SLI; FFIEC call report collector (spike first: automation of CDR bulk download). FR-03, FR-04, BR-15, BR-16.

## M3 — App skeleton, identity, market data, reconciliation, company pages
`iip` scaffold (hexagonal modules, import-linter, Alembic, UoW/outbox/job queue), Identity (sign-up/in, sessions, consents), Market Data (instruments, price provider adapter per Q-03, FX), Reconciliation MappingSet v1 + overlap check vs screener, company pages (UC-01). Staging + prod deploy (DevOps handoff complete). FR-08, FR-09, FR-27 (partial), FR-28, BR-13.

## M4 — Portfolio, plans, watchlists, alerts, notifications
UC-04..UC-07: portfolio with reasons, plans and reviews, watchlists, alert rules, in-app + email. BR-01, BR-12, FR-13..FR-18.

## M5 — Tools
Built-in tools (screener core wrappers, golden tests), ToolSpec DSL + sandboxed runner, conversational builder with LLM per Q-05, quotas. FR-10..FR-12, NFR-09.

## M6 — Social and clubs
Sharing, posts/comments, moderation; clubs with roles, club space, import tool, mark as bought. BR-06..BR-11, FR-19..FR-26.

## M7 — Hardening and beta launch
GDPR export/delete saga, NFR suites (performance, load, sandbox, ZAP), restore drill, runbooks, privacy notice/terms, beta invitations. NFR-01, NFR-03, NFR-06, NFR-08.

## Dependencies
M0 → M1 → (M2 ∥ M3) → M4 → M5 → M6 → M7. M3 can start on the M1 seed bundle. Q-03 (price vendor) needed by M3; Q-05 (LLM) by M5; Q-06 (screener code packaging) by M3/M5; Q-02 (budget) before VPS provisioning (M1).

## Owner questions (options and architect recommendation)

| Q | Question | Options | Recommendation | Blocks |
|---|---|---|---|---|
| Q-01 | V1 audience and scale | (a) private beta ≤100 invited users, (b) public launch | (a) — design already scales to 10k; beta lets data quality (R-04) and price licensing (Q-03) settle | M7 |
| Q-02 | Monthly running-cost ceiling (hosting + LLM + prices) | (a) ≤ €50, (b) ≤ €150, (c) higher | (b) for beta: ≈ €25–55 infra + price vendor + LLM capped by quotas/kill switch; (a) is feasible only with a free/cheap price source and builder limited to the owner | M1 provisioning, M3, M5 |
| Q-03 | EOD price vendor licensed for multi-user display (NFR-11) | (a) EODHD EOD plan with commercial/display terms, (b) Tiingo (commercial/redistribution tier), (c) Polygon/Massive or Nasdaq Data Link EOD with display licence, (d) Yahoo (unofficial, no licence; not acceptable for a public product) | Shortlist (a)/(b)/(c) by written licence terms for "display to logged-in users of a web app" and price; adopt the cheapest that permits it. Run a 30-instrument probe (like the screener's EODHD probe). Do **not** use Yahoo in Pro | M3 |
| Q-04 | Confirm record-only clubs and no personalised advice (BR-14) | (a) confirm, (b) want pooled money/advice later | (a); any change needs a legal check (regulated activity) and a new ADR | M6 |
| Q-05 | LLM vendor and limits for the tool builder | (a) Anthropic API (Claude Haiku-class default, Sonnet-class for repairs), (b) other hosted vendor, (c) EU-hosted open model | (a) with structured output + per-user quotas + global cap; confirm DPA/retention settings. Model ids/prices verified at implementation | M5 |
| Q-06 | How the public product repo consumes the private screener code | (a) private git dependency (deploy key on server, token in CI), (b) extract pure core into a new private `investor-core` package, (c) make the screener (or its core) public | (b): smallest surface, clean versioning, keeps IP private; (a) acceptable short-term | M3, M5 |
| Q-07 | Transactional email provider | (a) EU provider (e.g. Brevo, Mailjet, Scaleway TEM), (b) US provider with SCCs (Postmark, SES eu-central) | (a) for GDPR simplicity | M4 |
| Q-08 | Sign-in method | (a) email + password (+ verification), (b) passwordless magic link, (c) social login (Google/Apple) | (a) + (b) as option; (c) later (adds US sub-processors) | M3 |
| Q-09 | Posts visibility (A-R11) | (a) members-only, (b) public read, (c) author chooses | (c) with default members-only | M6 |
| Q-10 | GDPR deletion of social content and clubs | posts/comments: (a) delete, (b) anonymise ("deleted user"); super admin deletes account or leaves: (a) club deleted, (b) mandatory ownership transfer to an admin | posts: (b) anonymise comments in others' threads, delete own posts; clubs: (b) transfer to longest-serving admin, else delete — this changes BR-08 ("creator = super admin") so needs owner approval | M6, M7 |
| Q-11 | Object storage for raw zips/backups | (a) Hetzner Object Storage (EU), (b) Backblaze B2 EU, (c) Cloudflare R2 (EU jurisdiction) | (a) if VPS is Hetzner (same provider, low latency), else (c) | M1 |
| Q-12 | Task source for architect impl mode | (a) GitHub issues in `intelligent_investor_social`, (b) Trello board, (c) manual | (a) — keeps tasks next to PRs | impl mode |
