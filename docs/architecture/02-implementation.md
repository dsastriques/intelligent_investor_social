# 02 — Implementation architecture

v0.2 (2026-11-06): managed Postgres (ADR-007), two-plane hosting (ADR-015), store publication (ADR-019), Billing & Entitlements context (ADR-020), price vendor (ADR-012).

## 1. Architecture style (ADR-001)

**Modular monolith** in Python with one **hexagonal module per bounded context**, deployed as a few processes from one codebase, plus the separate collector package (`fdc`, repo `financial-data-collector`).

Why not microservices: one owner-developer + AI agents (NFR-12), ≤100 users growing to ~10k (NFR-01), low budget (NFR-10). 10k users of a long-term (low-frequency) product is a few requests per second at peak — well within one process group on one server. Microservices would add network failure modes, distributed transactions (e.g. BR-11 `MarkAsBought` would need a saga) and ops cost without serving any requirement. Module boundaries + import-linter rules keep the option to extract a context later (strangler fig).

Processes (all from `iip` package unless noted):

| Process | What | Why separate |
|---|---|---|
| `iip-web` | FastAPI (uvicorn), Jinja + HTMX pages, JSON API, payment webhook endpoint (store-and-ack only) | user traffic |
| `iip-worker` | outbox relay, job queue consumers, scheduler (advisory lock), prices, alerts, plan reviews, filing-event consumption, analysis snapshot build, LLM builder calls with metering, webhook processing, email | keeps slow/external work off request path |
| `iip-tool-runner` | DSL evaluator in a private container with no DB role and no secrets (ADR-009, 06 §3.3) | isolation of user-built tools (NFR-09) |
| `iip reconcile` (batch plane) | Reconciliation (BC-3) job on the Hetzner box, writes `market.std_*` | CPU/IO heavy, next to the store of record (ADR-019) |
| `fdc-*` (collector repo) | backfill, feed, watchers, FFIEC, publish | different release cadence, public data repo, CPU/IO heavy |

## 2. Repositories (T9, ADR-002)

| Repo | Visibility | Contains |
|---|---|---|
| `dsastriques/financial-data-collector` | **private** (owner, v0.4) | `fdc` package: collectors, store schema (published language), `store_client`, deploy units for collector jobs, sizing reports |
| `dsastriques/intelligent_investor_social` | **private** (owner, v0.4) | `iip` app: all bounded contexts except Filings, design docs, deploy, e2e tests |
| `dsastriques/the_intelligent_investor` | private | screener; its pure core (`models`, `metrics`, `rules`, `guardrail`, `ranking`, `prices`) consumed as a pinned private git dependency (ADR-008) |

The app depends on `fdc` (store client only) and the screener by git tag in `pyproject.toml` (`fdc @ git+https://github.com/dsastriques/financial-data-collector@v0.x`, `the-intelligent-investor @ git+https://github.com/dsastriques/the_intelligent_investor@vX.Y.Z`). All repos are private, so CI and image builds use a fine-grained read-only token as a BuildKit secret (ADR-008). Raw data never goes to GitHub (ADR-018).

## 3. Technology choices

| Concern | Choice | Reason |
|---|---|---|
| Language/tooling | Python 3.12+, uv, ruff, mypy strict, pytest, pydantic v2 | NFR-12; same as screener so its code is reusable |
| Web | FastAPI + Jinja2 templates + HTMX (+ Alpine.js for small widgets), Tailwind via standalone CLI build | one language end-to-end, server-rendered pages hit p95 < 2 s easily (NFR-03), small JS surface for agents to maintain (ADR-014) |
| Charts | Chart.js / lightweight ECharts loaded from CDN, data via JSON endpoints | statements and price charts |
| DB access | SQLAlchemy 2 **Core** (no ORM in domain) + Alembic migrations, psycopg 3 | ADR-007 |
| App DB | Managed PostgreSQL (EU), schemas `app` and `market` | NFR-08, NFR-10 (managed preferred), FR-29 integrity |
| Background jobs | Postgres job queue (`SKIP LOCKED`, `LISTEN/NOTIFY`) + in-worker scheduler; batch-plane systemd timers (ADR-013) | no broker to run |
| Payments | `PaymentGateway` port; `StripePaymentGateway` adapter (Checkout + webhooks), Paddle as alternative adapter (ADR-020) | FR-29 |
| Packaging | container images (web/worker share one image; minimal runner image) | managed containers (ADR-015) |
| LLM | Anthropic API by default via an `LlmClient` port (ADR-010, Q-05) | |
| Email | `EmailSender` port; EU transactional provider (ADR-017, Q-07) | |
| XBRL per filing | Arelle (in collector) | |
| Observability | structlog JSON logs → journald; Prometheus text `/metrics`; healthchecks.io; Sentry-compatible error tracking (optional, EU) | |

## 4. Module layout (hexagonal)

```
intelligent_investor_social/
  pyproject.toml            # package "iip"; deps incl. fdc (git tag), the-intelligent-investor (git tag, ADR-008)
  alembic/                  # migrations for Postgres schemas app and market (two version tables)
  src/iip/
    shared/                 # shared kernel: ids, Money, Clock, errors, Principal, SpaceId, Policy protocol, events
    identity/               # BC-9
    market/                 # BC-2
    reconciliation/         # BC-3
    tools/                  # BC-4 (+ builtins/ adapters around screener core, dsl/)
    portfolio/              # BC-5
    watchlists/             # BC-6
    social/                 # BC-7
    clubs/                  # BC-8
    notifications/          # BC-10
    billing/                # BC-11 Billing & Entitlements (ADR-020)
    platform/               # composition root: settings, DI wiring, outbox relay, job runner, web app factory
    web/                    # templates, static, HTMX partials, page routers (thin; call application services)
    runner/                 # tool-runner entry point (imports only tools.dsl + read-only data readers)
  tests/{unit,integration,contract,e2e,nfr}/
  deploy/                   # Dockerfiles, infra/ (OpenTofu: Scaleway + Hetzner), batch/ (systemd units, install.sh, pg_dump), runbooks/
  docs/architecture/
```

Each context package:

```
<context>/
  domain/          # aggregates, entities, value objects, domain events, domain services. stdlib + pydantic only.
  application/     # use cases (commands/queries), ports (Protocols): repositories, gateways, policies
  adapters/
    in_web/        # FastAPI routers / HTMX endpoints for this context (or in web/ calling these use cases)
    in_jobs/       # job handlers, event subscribers
    out_db/        # SQLAlchemy Core repositories, table definitions
    out_*/         # external gateways (price vendor, email, LLM, store client)
  config.py        # settings slice for this context
```

**Dependency rule (enforced by import-linter in CI):** `domain` imports nothing outside `domain` and `iip.shared.domain`; `application` imports `domain` and its own ports; `adapters` import `application`/`domain`; contexts talk to each other only through another context's `application` public API (facade) or events, never its `adapters` or tables.

## 5. Key ports (Python Protocols)

```python
# shared
class Clock(Protocol):            def now(self) -> datetime: ...
class UnitOfWork(Protocol):       # transaction + collected domain events -> outbox on commit
    def __enter__(self) -> Self: ...; def commit(self) -> None: ...; def add_events(self, *e: DomainEvent) -> None: ...
class Policy(Protocol):           def check(self, principal: Principal, action: Action, resource: ResourceRef) -> None: ...  # raises Forbidden

# market
class PriceProvider(Protocol):    def eod_bulk(self, exchange: str, on: date) -> Iterator[DailyClose]: ...
class FxProvider(Protocol):       def rates(self, since: date) -> Iterator[FxRate]: ...
class FilingStore(Protocol):      # wraps fdc.store_client
    def statement(...); def statements(...); def notes(...); def revisions(...); def events(after_seq: int) -> Iterator[FilingEvent]: ...

# reconciliation
class StandardizedFinancialsRepo(Protocol): def get(self, company_id: str) -> CompanyFinancials: ...; def save(...): ...

# tools
class ToolRunner(Protocol):       def run(self, spec: ToolSpec, inputs: RunInputs, limits: Limits) -> RunResult: ...
class LlmClient(Protocol):        def complete(self, messages: list[Msg], schema: JsonSchema, budget: TokenBudget) -> LlmReply: ...
class BuiltinTool(Protocol):      id: str; def run(self, universe: Sequence[CompanyData], params: Mapping[str, Any]) -> ToolOutput: ...

# billing (BC-11) - facade used by tools
class Entitlements(Protocol):     def can_use(self, principal: Principal, feature: Feature) -> EntitlementDecision: ...
class Metering(Protocol):
    def reserve(self, account: AccountId, session: SessionId, worst_case: Money, idem_key: str) -> ReservationId: ...  # raises InsufficientCredit
    def settle(self, reservation: ReservationId, usage: LlmUsage) -> UsageRecord: ...
    def release(self, reservation: ReservationId) -> None: ...
class PaymentGateway(Protocol):   # adapters: paddle / stripe / mollie / fake
    def create_checkout(self, account: AccountId, product: ProductRef, idem_key: str) -> CheckoutUrl: ...
    def parse_webhook(self, raw: bytes, headers: Mapping[str, str]) -> ProviderEvent: ...   # verifies signature
    def refund(self, payment_ref: str, amount: Money, idem_key: str) -> None: ...

# portfolio / watchlists / clubs / social: one Repository per aggregate root
# notifications
class EmailSender(Protocol):      def send(self, to: str, template: str, ctx: Mapping[str, Any], idem_key: str) -> None: ...
```

## 6. Application services and transactions

- One use case = one class/function with an explicit command (pydantic model), executed inside a `UnitOfWork`: authorize (Policy) → load aggregate(s) → call domain → save → events to outbox → commit.
- **Cross-context writes in one transaction are allowed only via application facades inside the same DB** (both contexts in schema `app`). Used exactly once by design: `MarkAsBought` (BR-11) calls `portfolio.application.record_change` and `portfolio.application.copy_plan` / `clubs.application.copy_analysis` within one UoW. No saga needed (benefit of the monolith).
- **Plan review (FR-15):** scheduled job finds plans with `review_date <= today` and no review → creates `PlanReview` with machine-evaluated outcomes (prices from schema `market`, standardized fields from BC-3) and notifies the user to complete manual expectations.
- **Account deletion (FR-27):** see `06-security-authz-sandbox.md` §4.2.
- **Builder call with metering (BR-17, FR-29, ADR-010/020):** worker job `builder.message`: (1) `Entitlements.can_use` → (2) count input tokens, compute worst case from `PriceTable` → (3) `Metering.reserve` in its own short transaction (row lock on billing account) → (4) call `LlmClient` (outside any DB transaction) → (5) `Metering.settle` with vendor usage + save BuilderSession message in one UoW → on error `release`. A sweeper job expires reservations older than 15 min.
- **Top-up (FR-29):** web `POST /billing/checkout` → `PaymentGateway.create_checkout` → user pays on provider page → provider webhook `POST /billing/webhook` → web verifies signature, inserts `bl_webhook_inbox` (unique event id), returns 200 → worker job `HandleProviderEvent` → ledger `top_up` + `CreditToppedUp` event.

## 7. Patterns applied (each traced to a requirement)

| Pattern | Where | Serves |
|---|---|---|
| Hexagonal ports & adapters | every module, collector | NFR-12 (agent-friendly, testable), FR-04 (pluggable sources) |
| Aggregate + invariants in domain | Portfolio, Club, InvestmentPlan, Tool, Filing | BR-01, BR-08, BR-12, BR-03 |
| Repository + Unit of Work | all aggregates | consistency, testability |
| Strategy / plug-in | `SourceCollector`, `PriceProvider`, `BuiltinTool`, alert `Condition` evaluators | FR-04, FR-10, FR-17, NFR-11 |
| Anti-corruption layer | FFIEC adapter, Reconciliation, price vendor adapter, screener wrapper | BR-02, BR-16, FR-10, NFR-11 |
| Published language / open host | store schema v1 + `store_client` + `filing_event` feed | FR-05, FR-08, BR-02 |
| Specification | ToolSpec filters/rules; alert conditions | FR-11, FR-17 |
| Interpreter (DSL) | ToolSpec evaluator | FR-11, NFR-09 |
| Domain events + transactional outbox | `app.outbox` table, relay in worker (LISTEN/NOTIFY wake-up) | FR-12, FR-17, FR-18 reliable delivery |
| Idempotent consumer | `processed_event(consumer, event_id)`; email idempotency keys; collector loads keyed by `adsh` | BR-05, FR-18 |
| Retry with backoff + circuit breaker | SEC, price vendor, LLM, email gateways | NFR-06, NFR-07 |
| Token bucket rate limiter | collector (host-wide) | NFR-07 |
| Policy object (resource-based authZ) | `Policy` per context | BR-06..BR-11 |
| CQRS-lite (read models) | company page read model in schema `market`, portfolio valuation view | NFR-03 |
| Copy-on-share/import (prototype) | tool copy, club import, mark-as-bought copies | BR-07, BR-10, BR-11 |
| Saga (local, orchestrated) | account deletion across contexts (job steps, resumable) | FR-27, NFR-08 |
| Ledger (append-only) + reservation (two-phase hold) | Billing credit ledger, reserve/settle per LLM call | BR-17, FR-29 |
| Idempotent webhook inbox | `bl_webhook_inbox` keyed by provider event id | FR-29 |
| Published immutable snapshot | store/analysis snapshots on object storage (ADR-019) | NFR-03, NFR-06, NFR-09 |
| Bulkhead | separate containers for web, worker, runner (own CPU/memory limits); collector on a separate plane | NFR-06, NFR-09 |

Not used, deliberately: message broker, event sourcing, service mesh, Kubernetes (no requirement needs them).

## 8. Market data, prices and alerts (T7, ADR-012, ADR-013)

- **Instruments:** built from catalog companies + SEC `company_tickers.json` (ticker → CIK); daily sync job. Instrument key `XNYS:KO`-style plus vendor symbol mapping.
- **Price refresh (FR-09, BR-13, NFR-05):** job `prices.refresh_eod` runs Mon–Fri after US close (default 23:30 Europe/Amsterdam, plus a 07:30 catch-up) — fetches EOD closes via `PriceProvider` (V1 adapter Yahoo, owner risk acceptance; throttled, circuit breaker; ADR-012), upserts `market.prices_daily(instrument_id, date, close, adj_close, volume, currency, source)`, then emits `PricesRefreshed`. **Tracked** instruments (in any portfolio, watchlist or alert rule) daily; the rest of the listed universe weekly (`prices.universe_refresh`, A-22) to stay within the vendor quota; price date, source and a staleness indicator (> 2 trading days) shown on results; alerts are not fired from stale prices; raw price series are not exported (ADR-012 §3).
- **Alerts:** `PricesRefreshed` → evaluate all enabled price/value rules in batches (vectorised per instrument; rules grouped by instrument so each series is read once). `FilingCollected`/`RestatementDetected` (consumed from `market.filing_event`, appended by `fdc publish`, cursor stored in schema `app`) → evaluate filing rules. Triggered → `AlertTriggered` → Notifications.
- **FX:** ECB daily reference rates (free, permitted reuse) for portfolio base-currency valuation; the screener's `fx.py` logic reused.
- Page loads read latest stored price only; no live calls in request path (BR-13).

## 9. Reconciliation and built-in tools (T5, ADR-008)

### 9.1 Reconciliation (BC-3)
- Input: as-reported facts via `store_client`; output (written by `iip reconcile` on the batch plane, role `iip_batch`): `market.std_financials(company_id, fiscal_year, mapping_version, data_json, coverage)` + `std_lineage(company_id, fiscal_year, field, filing_key, tag, version, ddate, qtrs, rule)`.
- **MappingSet v1** starts from the screener's `providers/edgar_fundamentals.CONCEPTS` alias table (same us-gaap tags, same "first alias with data wins", annual facts only from 10-K/20-F/40-F with `qtrs=4` (flows) or `qtrs=0` (stocks), consolidated only, latest filed value for the period), extended with: IFRS tags for 20-F filers, derivations (e.g. gross profit = revenue − COGS), sector branches (bank/insurer/REIT), and extension-tag fallbacks via `pre` label/position heuristics only where explicitly configured.
- Runs incrementally on `filing_collected` events (recompute the affected company) and fully when the mapping version changes.
- **Overlap check:** for US companies also present in the screener's `financial.db`, compare M1 output with the screener's EDGAR-derived values; report mismatches (acceptance gate in M3).
- Builds the screener's `CompanyData` (`Company`, `AnnualFinancials`, `Market`, `EventRecord`): Company kind from SIC (bank/insurer/REIT/utility), market from latest price × shares (dei `EntityCommonStockSharesOutstanding`), events from filing metadata (`NT 10-K` late filings, 8-K 4.02 non-reliance, restatements from `fact_revision`).

### 9.2 Built-in tools (FR-10)
Wrappers implementing `BuiltinTool` around the screener's pure functions (no copy of logic):

| Built-in tool | Screener code reused | Output |
|---|---|---|
| Value screen (rules + ranking) | `rules.evaluate`, `rules.peers.build_peers`, `ranking.score_universe` | ranked table with points, gates, reasons; optional watchlist |
| Accounting guardrail | `guardrail.run_guardrail`, `guardrail.scores`, `guardrail.enron` | verdict, risk score, checks with reasons |
| PE watch | `ranking.pe_watch.select_pe_watch` / `qualifies` | list with P/E and distance to P/E 15 |
| Peer comparison | `rules.peers` + `metrics` | company vs industry medians |
| Price/value targets | `prices.intrinsic_value` (+ user's own target price, as the screener's `targets` store does) | intrinsic value, Graham number, EPV, margin of safety |
| Moving-average alerts | `prices.check_alert`, `prices.sma` | alert level; also used by the `value_signal` alert condition |

Settings: the screener's `Settings` (config.yaml) become **tool parameters** with the screener defaults; users can override within validated ranges (copying a built-in into their space as a parameterised user tool).
Stateful screener parts (watchlist selection history, guardrail approvals, Telegram, dashboard, EODHD/Yahoo providers, `storage`) are **not** reused; their Pro equivalents are Watchlists, Notifications, and the web UI.

## 10. API contracts

- **Web UI:** server-rendered routes (`/companies/{id}`, `/portfolios/{id}`, `/tools/{id}`, `/clubs/{id}/…`), HTMX partials under `/hx/...`. CSRF tokens on all unsafe methods.
- **JSON API** `/api/v1/...` (OpenAPI generated by FastAPI) used by HTMX widgets and charts; versioned by path prefix; breaking changes → `/api/v2`. Outline:
  - `GET /api/v1/companies?query=` · `GET /api/v1/companies/{id}/statements?stmt=IS&annual=1&consolidated=1` · `GET .../filings` · `GET .../notes/{filing_key}` · `GET .../revisions` · `GET .../prices?from=`
  - `POST /api/v1/portfolios` · `POST /api/v1/portfolios/{id}/changes` (body requires `reason`) · `GET /api/v1/portfolios/{id}`
  - `POST/PUT /api/v1/plans` · `POST /api/v1/plans/{id}/reviews/{rid}/complete`
  - `POST /api/v1/watchlists/{id}/entries` · `POST /api/v1/alert-rules`
  - `POST /api/v1/tools` · `POST /api/v1/tools/{id}/runs` · `POST /api/v1/tool-builder/sessions` · `POST .../messages`
  - `POST /api/v1/shares` · `DELETE /api/v1/shares/{id}` · `POST /api/v1/posts` · `POST /api/v1/comments` · `POST /api/v1/reports`
  - `POST /api/v1/clubs` · `POST /api/v1/clubs/{id}/invitations` · `POST .../members/{uid}/role` · `DELETE .../members/{uid}` · `POST .../watchlist/{entry}/mark-as-bought`
  - `GET /api/v1/me/export` · `DELETE /api/v1/me`
- **Internal async contract:** `filing_event` rows (published by collector), outbox event envelope `{id, type, version, occurred_at, aggregate, payload}` (AsyncAPI-style schema in `docs/architecture/events/` when implemented).
- **Tool-runner contract:** JSON over HTTPS to the private runner container (worker token): `{spec, inputs, limits, snapshot_url, snapshot_seq}` → `{status, rows, metrics, usage}` (`runner_protocol: 1`).
- **Billing endpoints:** `GET /api/v1/billing` (balance, available, usage by session/month), `POST /api/v1/billing/checkout` (top-up product → provider URL), `POST /billing/webhook` (provider → inbox; no session auth, signature-verified).

## 11. Data storage per context

| Store | Engine | Owner | Contents | Backup |
|---|---|---|---|---|
| Postgres schema `app` | Managed PostgreSQL (ADR-007) | Identity, Portfolio, Watchlists, Tools, Social, Clubs, Notifications, **Billing** (table prefixes per context) | personal data, billing ledger | provider backups + nightly encrypted dump to Hetzner (03 §6) |
| Postgres schema `market` | same instance | Market Data, Reconciliation (written by batch role) | instruments, prices, FX, std_financials, lineage, filing_event, company_store_version, read models | same (rebuildable) |
| as-reported store | SQLite catalog + per-company files on the batch VM; published immutable copies on Scaleway Object Storage (ADR-019) | Filings (collector) | public filings | raw zips in object storage + weekly `fdc export` bundle (rebuildable) |
| analysis snapshot | SQLite file on object storage, rebuilt after reconcile/price refresh | Tools (read-only input to runner) | public data only | rebuildable |
| raw archive | files + EU object storage | Filings | SEC/FFIEC zips | object storage is the copy |

- Table prefix per context in schema `app` (`id_`, `pf_`, `wl_`, `tl_`, `so_`, `cl_`, `nt_`, `bl_`) — schema ownership; no cross-context foreign keys except to `id_account`/`id_space`.
- Migrations: Alembic, forward-only in prod, each migration with a tested downgrade for dev; expand/contract for breaking changes.
- Consistency: strong within schema `app` (incl. billing ledger); eventual between collector → publish → `market` → alerts (minutes).
- Caching: in-process LRU for company read models and statements (keyed by `filing_event.seq`), HTTP caching via Cloudflare for static assets only (pages are per-user).
- Retention: prices and filings kept indefinitely; tool run results 90 days (ASSUMPTION A-14) except the last per tool; notifications 180 days; logs per Cockpit retention (03 §7); billing ledger and usage records for the statutory period, pseudonymised after account deletion (A-27).
- Time: all timestamps UTC ISO-8601 in storage; display in the user's time zone; trading dates as `date`.
- Money/quantities: `Decimal` in domain, stored as TEXT; currencies ISO-4217.

## 12. Performance budget (NFR-03)

Company page: catalog lookup + one company file + std read model → server time ≤ 300 ms p95; portfolio page: ≤ 50 positions × latest price ≤ 200 ms. Pages are server-rendered with deferred HTMX loads for heavy panels (notes, revisions). Measured by the NFR suite (04 §6).

## 13. Configuration, secrets, flags

- 12-factor: settings via pydantic-settings from env; serving plane env injected from Scaleway Secret Manager; batch plane `/etc/fdc/fdc.env` (0640); never in git; `deploy/iip.env.example` documents every key.
- Secrets: `SESSION_SECRET`, `DATABASE_URL` (per role), `EDGAR_USER_AGENT`, `PRICE_API_KEY`, `LLM_API_KEY`, `EMAIL_API_KEY`, `S3_*` (scoped per bucket), `RUNNER_TOKEN`, `PAYMENT_API_KEY`, `PAYMENT_WEBHOOK_SECRET`, `DUMP_AGE_RECIPIENT` (public key only on the box), `HEALTHCHECK_URL_*`, `SENTRY_DSN`.
- Feature flags: table `platform_flag(name, enabled, audience)` read with a short cache: `tool_builder_enabled`, `billing_live` (off = sandbox provider), `signup_open`, `clubs_enabled`, `email_enabled`, `llm_kill_switch`.
- Tool run quotas and LLM safety limits in config (A-09); `billing.price_table` (versioned YAML: per-model USD/Mtok, margin, min charge), `billing.session_min_credit`, `billing.top_up_amounts`, `billing.credit_expiry_months` — owner sets values (Q-14).

## 14. Authentication and authorization (summary; detail in 06)

- AuthN (ADR-016, accepted): email + password (argon2id), mandatory email verification, optional passwordless magic link (single-use, 15 min, hashed token), password reset, optional TOTP for operator; server-side sessions (`id_session` table, opaque cookie `HttpOnly; Secure; SameSite=Lax`); rate-limited sign-in; Cloudflare Turnstile on sign-up.
- AuthZ (ADR-011): policies in the **application layer**; repositories take the `Principal` and apply visibility filters (defence in depth). Service-to-service: worker → runner via private-container token; each process has its own least-privilege Postgres role (ADR-007); batch plane writes only `market` tables via `iip_batch`; published store objects are read-only for the serving plane.

## 15. Cross-cutting

- **Error model:** domain errors (`ReasonRequired`, `Forbidden`, `NotFound`, `InvariantViolation`, `QuotaExceeded`, `InsufficientCredit` → 402 with top-up link) mapped to HTTP 400/403/404/409/429/402 with problem+json body; UI shows the message. Unknown errors → 500 with correlation id.
- **Logging/tracing:** structlog JSON with `request_id`/`job_id`/`run_id` correlation ids propagated to jobs and tool runs; no personal data or financial values of users in logs (only ids).
- **Idempotency:** `Idempotency-Key` header on POST create endpoints (portfolio change, mark-as-bought) stored 24 h; event consumers record processed ids.
- **Pagination:** cursor-based (`?after=`), max 100.
- **i18n:** English UI V1 (ASSUMPTION A-15); strings in templates via gettext-ready helpers; number/date formatting per locale.
- **Analytics:** product events (sign-up, tool run, share, club created) as rows in `app.analytics_event` without free-text content, aggregated by SQL; no third-party trackers (NFR-08). PII limited to user id; deleted with the account.
- **Disclaimer:** every analysis/tool page shows "record-keeping and analysis, not investment advice" (BR-14).

## 16. Implementation guidelines checklist (reviewed in impl mode)

- **G-01** Dependency rule: `domain/` imports only stdlib, pydantic and `iip.shared.domain`; enforced by import-linter contracts in CI.
- **G-02** Contexts communicate via application facades or events; never import another context's `adapters` or query its tables.
- **G-03** Ports are Protocols owned by the `application` layer of the consuming context; adapters implement them.
- **G-04** Invariants (BR-xx) live in aggregates/value objects, each with a unit test for the passing **and** failing path; DB constraints duplicate critical ones (BR-01 reason, BR-08 single super admin).
- **G-05** Names follow the ubiquitous language (01 §1): `PortfolioChange`, `Reason`, `InvestmentPlan`, `Expectation`, `Share`, `Club`, `Membership`, `FactRevision`… no synonyms.
- **G-06** Authorization is checked in every application command/query via `Policy.check`; adapters never decide access; every query that lists items applies the viewer filter.
- **G-07** Every state change runs in a `UnitOfWork`; domain events leave only via the outbox.
- **G-08** Consumers and loads are idempotent (event ids, `adsh`, `Idempotency-Key`).
- **G-09** No secrets, tokens or personal data in code, logs, fixtures or git; settings from env.
- **G-10** Config thresholds (rules, guardrail, quotas, tolerances) in config files/flags, not code (screener rule 5).
- **G-11** Pydantic models frozen for value objects; Decimal for money/quantities; UTC timestamps.
- **G-12** No network calls in the request path except to own DB/files; external calls go through gateways with timeout, retry, circuit breaker.
- **G-13** As-reported data is never modified or renamed by the app; standardization only in `reconciliation` (BR-02).
- **G-14** Every portfolio change requires a reason end-to-end (form validation, API schema, domain, DB) (BR-01).
- **G-15** Migrations: Alembic, reversible in dev, expand/contract for breaking changes; no manual DDL.
- **G-16** User-built tools only via the ToolSpec DSL and the `ToolRunner` port; never `eval`/`exec`/`pickle` of user input anywhere.
- **G-17** Tests: unit for domain/application (≥ 90 % line coverage on `domain/`), integration for adapters with real Postgres (service container / testcontainers) and real SQLite store files, contract tests for store schema and tool-runner protocol, e2e for each UC touched.
- **G-18** ruff, ruff format, mypy strict, import-linter, pytest green before every commit; CI never fetches market/SEC data (C-8).
- **G-19** Every external data source carries `source` on stored rows; the licence (or the owner's risk acceptance, ADR-012) of each source is recorded in an ADR (NFR-11).
- **G-20** No code that places orders, connects to brokers, holds or moves investment money, or gives personalised advice (BR-14). The only payment integration allowed is BC-11 charging for the platform's own premium usage (FR-29), via `PaymentGateway`.
- **G-26** Post visibility is derived from the post's space only (BR-18); no code path may expose club-space posts outside the club or private items to `anonymous`.
- **G-21** Comments and docstrings explain *why*; every user-visible automated change (watchlist from tool, alert) carries a plain-English reason (screener rule 4).
- **G-22** Design docs updated in the same PR when implementation changes the design; new significant decisions get an ADR.
- **G-23** Money in Billing: integer EUR micros, never floats; every ledger change via `Metering`/`Billing` commands in a UoW; no `UPDATE`/`DELETE` on `bl_ledger_entry`; credit granted only from verified idempotent provider events; no card data stored (hosted checkout).
- **G-24** No LLM call without a prior successful reservation (BR-17); `LlmClient` adapters are only invoked from the metered builder use case (import-linter contract).
- **G-25** The tool runner image contains no DB driver, cloud SDK or secrets; it reads only the snapshot URL passed per request (NFR-09).
