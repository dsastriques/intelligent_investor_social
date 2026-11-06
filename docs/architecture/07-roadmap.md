# 07 — Phased implementation plan

v0.2 (2026-11-06): M0 unchanged; hosting spike S-1 added to M3; M3 prices use marketstack behind the licence gate; billing milestone M5b added; owner questions updated.

Milestones are ordered by risk: the data asset first (largest unknowns: size, FSN semantics), then the app skeleton, then features. Each milestone ends with a PR (or PRs) reviewed in architect impl mode. Durations are not estimated here (owner/coordinator plan the calendar).

## M0 — Collector sizing test (first milestone)

**Repo:** `financial-data-collector`. **Goal:** measured numbers that confirm or change ADR-003 (store layout), ADR-004 (FSN vs FSDS) and the Hetzner batch VM size and object-storage volume (ADR-015, ADR-018, ADR-019), before any full backfill (NFR-02, research step 6).

Scope:
1. Repo scaffold: `pyproject.toml` (uv, ruff, mypy strict, pytest), hexagonal skeleton per 05 §2, CI (lint, types, tests on fixture zips, no SEC calls).
2. Domain + output contract records; `SourceCollector` port; `SecFsdsSource` and `SecFsnSource` streaming readers; host-wide rate limiter; `EDGAR_USER_AGENT` enforcement.
3. `StoreWriter` for per-company layout **and** single-DB layout (for comparison), staging loader, `collector_run`/`collected_item` tracking, restatement check.
4. `fdc sizing --quarter 2025q2 [--previous 2025q1] --layouts per-company,single --report docs/sizing/M0-report.md`.
5. Run on a machine with ≥ 2 GB RAM and on a 1 GB-limited cgroup (`systemd-run -p MemoryMax=1G`) to prove streaming stays memory-bounded (the existing 1 GB server or a short-lived Hetzner VM; cost < €1).
6. Measure the size distribution of per-company files and the publish delta per quarter (input to ADR-019 cache sizing and object-storage cost).

**Acceptance criteria:**
- AC-1 Report contains, per data set (FSDS, FSN) and layout: zip size, download time, requests, load wall time, peak RSS, staging peak disk, final bytes, rows per table, company-file count and size distribution, p95 query time for a statement view.
- AC-2 Peak RSS ≤ 512 MB under `MemoryMax=1G`.
- AC-3 Rerun of the same quarter loads 0 new rows and records a second `collector_run` with `items_skipped` = all (BR-05).
- AC-4 FSN-vs-FSDS superset check on ≥ 20 sample companies with % facts reproduced.
- AC-5 12-year projection (store size, backfill duration, request count) with stated growth factor.
- AC-6 Restatement check run across two consecutive quarters with a count and five examples reviewed.
- AC-7 Recommendation section: per-company vs single DB; FSN vs FSDS(+FSN on demand); batch VM type (CX43 / CPX32 / AX41) and disk; published-store size and per-quarter delta — handed to the owner for decision where cost is involved.
- AC-8 Unit/integration tests for parsing, idempotency and rate limiter green in CI; no raw zips committed (`.gitignore`, ADR-018).

## M1 — Collector backfill + store client
Full 12-year backfill on the provisioned Hetzner batch VM (DevOps handoff items 2–5 for the batch plane; temporary rescale for the backfill), `store_client` read API (statement as presented, annual, consolidated, notes, revisions, events), `fdc status`, export/import bundles, seed bundle for dev/CI. FR-01, FR-02, FR-05, FR-06, FR-07, FR-08, BR-02..05.

## M2 — Watchers and FFIEC
Feed + per-sector watchers with Arelle per-filing loads (provisional → superseded), daily-index reconcile, NFR-04 SLI; FFIEC call report collector (spike first: automation of CDR bulk download). FR-03, FR-04, BR-15, BR-16.

## M3 — Hosting spike, app skeleton, identity, market data, reconciliation, company pages
**S-1 hosting spike first** (03 §3): deploy a hello-world web/worker/runner on Scaleway Serverless Containers + DB-DEV-S; measure cold start, always-on cost for 7 days, cache-miss latency for a published company file, private runner auth, DB ACL from the Hetzner box, backup retention/PITR → confirm option A or switch to C. Then `iip` scaffold (hexagonal modules, import-linter, Alembic on Postgres, UoW/outbox/job queue), Identity (sign-up/in, sessions, consents), Market Data (instruments, **marketstack adapter + 30-instrument probe**, FX), `fdc publish` + `FilingStore` object-storage adapter (ADR-019), Reconciliation MappingSet v1 + overlap check vs screener, company pages (UC-01). Staging + prod deploy (DevOps handoff complete). **Go-live gate:** written marketstack display-licence confirmation archived (ADR-012) before any non-owner user sees prices. FR-08, FR-09, FR-27 (partial), FR-28, BR-13, NFR-11.

## M4 — Portfolio, plans, watchlists, alerts, notifications
UC-04..UC-07: portfolio with reasons, plans and reviews, watchlists, alert rules, in-app + email. BR-01, BR-12, FR-13..FR-18.

## M5 — Tools
Built-in tools (screener core wrappers, golden tests), ToolSpec DSL + sandboxed runner (private container, analysis snapshot), conversational builder with LLM per Q-05 behind the `Entitlements`/`Metering` facade (fake billing until M5b). FR-10..FR-12, NFR-09.

## M5b — Billing & Entitlements (premium builder)
BC-11 per ADR-020: billing account, ledger, reservations, usage records, price table, `PaymentGateway` adapter for the provider chosen in Q-13 (sandbox first), top-up checkout + webhook inbox, usage page, operator reconciliation view (RB-09), deletion-saga step. **Gate before `billing_live`:** owner's business/VAT set-up decided (Q-15), terms of sale + withdrawal-right wording, price table and margin set (Q-14). BR-17, FR-29.

## M6 — Social and clubs
Sharing, posts/comments, moderation; clubs with roles, club space, import tool, mark as bought. BR-06..BR-11, FR-19..FR-26.

## M7 — Hardening and beta launch
GDPR export/delete saga, NFR suites (performance, load, sandbox, ZAP), restore drill, runbooks, privacy notice/terms, beta invitations. NFR-01, NFR-03, NFR-06, NFR-08.

## Dependencies
M0 → M1 → (M2 ∥ M3) → M4 → M5 → M5b → M6 → M7. M3 can start on the M1 seed bundle. Q-02b (hosting option) before batch VM provisioning (M1) and S-1 (M3); Q-03b (vendor + licence answer) by M3 go-live; Q-05 (LLM) by M5; Q-13/14/15 by M5b; Q-06 (screener code packaging) by M3/M5.

## Owner questions (options and architect recommendation)

Answered 2026-11-06 (owner): **Q-01** private beta ≤100 invited users (design for ~10k) · **Q-02** €50–200/month excl. LLM, managed preferred, Hetzner for heavy batch · **Q-03** architect proposes the price vendor → ADR-012 · **Q-04** record-only clubs, no personalised advice confirmed.

| Q | Question | Options | Recommendation | Blocks |
|---|---|---|---|---|
| Q-02b | Hosting option within €50–200 (03 §10, ADR-015) | (A) Scaleway managed serving plane + Hetzner batch ≈ €121–164 (plan €141); (B) everything on one Hetzner VM ≈ €88–111; (C) Scaleway VM + managed Postgres + Hetzner batch ≈ €114–160; (D) all Scaleway ≈ €145–165 | **(A)**, matches "managed where the budget allows" with €36–79 headroom; S-1 spike confirms, (C) is the fallback | M1 batch VM, M3 |
| Q-03b | Confirm price vendor and fallback budget | (a) marketstack Professional ≈ €43 after written display confirmation, fallback Tiingo Start-up Redistribution ≈ €217 (exceeds €200 → budget exception); (b) go straight to a licensed vendor (Tiingo $250 / Twelve Data $499) and raise the ceiling; (c) beta without displayed prices (not recommended: FR-09/FR-17 need prices) | **(a)**; owner sends the one-line licence question to marketstack/apilayer now (text in ADR-012) and pre-approves either the Tiingo fallback (+≈ €175/month) or a delay | M3 go-live |
| Q-05 | LLM vendor and limits for the tool builder | (a) Anthropic API (Haiku-class default, larger model for repairs), (b) other hosted vendor, (c) EU-hosted open model | (a) with structured output, prompt caching, DPA + zero/short retention; prices verified at M5. Limits are now safety limits only (users pay, BR-17) | M5 |
| Q-06 | How the public product repo consumes the private screener code | (a) private git dependency, (b) extract pure core into a private `investor-core` package, (c) make the core public | (b); note: with container images built in GitHub Actions, CI needs a read token for the private package (registry/deploy key) | M3, M5 |
| Q-07 | Transactional email provider | (a) Scaleway TEM (same provider/DPA as hosting, 300 free then €0.25/1k), (b) Brevo/Mailjet (EU), (c) US provider with SCCs | **(a)** if Q-02b = A/C/D; else (b) | M4 |
| Q-08 | Sign-in method | (a) email + password (+ verification), (b) passwordless magic link, (c) social login | (a) + (b) as option; (c) later | M3 |
| Q-09 | Posts visibility (A-R11) | (a) members-only, (b) public read, (c) author chooses | (c) with default members-only | M6 |
| Q-10 | GDPR deletion of social content and clubs | posts: (a) delete, (b) anonymise; super admin leaves/deletes: (a) club deleted, (b) transfer to an admin | posts (b) for comments in others' threads, delete own posts; clubs (b) — changes BR-08, needs owner approval. v0.2 addition: billing records are pseudonymised, not deleted (A-27) | M6, M7 |
| Q-11 | Object storage | (a) Hetzner Object Storage for batch (raw, bundles, dumps) + Scaleway one-zone for the published store, (b) one provider for both, (c) Cloudflare R2 | **(a)** (follows ADR-015 option A; ADR-018/019) | M1 |
| Q-12 | Task source for architect impl mode | (a) GitHub issues, (b) Trello, (c) manual | (a) | impl mode |
| Q-13 | Payment provider (FR-29; owner decision) | (A) Paddle (MoR, ≈ 5 % + $0.50), (B) Stripe Managed Payments (MoR, Stripe fees + 3.5 %; NL eligibility unverified), (C) Stripe Payments+Billing+Tax (processor; owner registers for VAT/OSS), (D) Mollie (NL processor, iDEAL €0.29, cards 1.8 % + €0.25; owner handles VAT), (E) Lemon Squeezy / Creem (MoR) | **(A) Paddle** for the beta: MoR removes VAT/OSS filing, invoicing and most chargeback work for a one-person operation; move to (C)/(D) when volume or a company set-up justifies in-house VAT. Fees are third-party sourced; verify on the provider's site before signing | M5b |
| Q-14 | Premium plan model, margin, credit rules | plan: (P1) prepaid top-ups, (P2) subscription with included credit + top-ups, (P3) post-paid metered; margin: 0 % pass-through / 10–30 %; credit expiry: none / 12 months; unused credit on account deletion: refund / forfeit after notice | **P1** prepaid top-ups (€10/€25/€50, min €10 to keep fees ≤ ~10 %); margin covering payment fees + metering error (e.g. 15–20 %, owner's call); 12-month expiry; refund unused credit on request within the legal window | M5b |
| Q-15 | Business set-up and tax for selling premium credits (owner + accountant) | (a) sell as a private individual/sole trader (eenmanszaak) via an MoR; (b) company (BV) via an MoR; (c) own VAT registration + OSS with a processor | Architect does not decide. Design supports all three via the `PaymentGateway` port. Points to check with the accountant: EU VAT on electronically supplied services (customer's country rate, OSS, €10k threshold), whether prepaid credits are single- or multi-purpose vouchers, consumer withdrawal right for digital services, bookkeeping retention (A-27), and that credits are not e-money | M5b gate |
