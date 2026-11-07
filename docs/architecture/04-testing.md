# 04 — Testing strategy (input for the qa skill)

## 1. Strategy and pyramid

| Level | Scope | Tooling | Runs |
|---|---|---|---|
| Unit | domain aggregates/value objects, application use cases with fake ports, DSL parser/evaluator, policies, collector record parsing, restatement logic, reconciliation mapping | pytest, hypothesis (DSL, restatement tolerances) | every commit |
| Integration | SQLAlchemy repositories on real Postgres (CI service container / testcontainers), Alembic up/down, outbox relay, job queue (`SKIP LOCKED`), billing ledger/reservations under concurrency, webhook inbox, collector loaders on fixture zips, store_client + snapshot reader, tool-runner over HTTP | pytest + Postgres + tmp SQLite store files | every PR |
| Contract | (a) store schema v1: app's `FilingStore` adapter vs `fdc` published `store_client` + schema version (pinned tag); (b) tool-runner request/response JSON schema; (c) `PriceProvider` adapters (Yahoo V1; marketstack/Tiingo replacement) vs recorded vendor fixtures, all against one port contract suite; (e) `StripePaymentGateway` vs recorded Stripe webhook payloads + `Stripe-Signature` fixtures; (d) screener-core wrapper vs pinned screener version (golden outputs) | pytest, JSON Schema | every PR |
| E2E | user journeys UC-01..UC-13 in a browser against app on seed bundle | Playwright (Python) | every PR (smoke subset) + nightly full |
| NFR | performance (NFR-03), load (NFR-01), sandbox security (NFR-09), authZ matrix (NFR-08/BR-06), backup/restore (NFR-06), collector memory (NFR-02), rate limit (NFR-07) | locust/k6, pytest, `/usr/bin/time -v`, ZAP baseline | nightly / pre-release |

Target shape: many unit tests, fewer integration, a focused e2e set (one happy path + key negative path per UC).

## 2. E2E setup

- **App under test:** `iip-web` + `iip-worker` + `iip-tool-runner` started by a pytest fixture (`uv run iip dev-up --seed tests/seed/bundle`) on random ports; LLM, email, price vendor and payment provider replaced by **fakes** selected via env (`IIP_LLM=fake` with deterministic token usage, `IIP_EMAIL=capture`, `IIP_PRICES=fixture`, `IIP_PAYMENTS=fake`).
- **Test data:** `tests/seed/bundle/` = small `fdc export` (≈ 10 companies: industrial, bank, insurer, REIT, 20-F filer, a company with a restatement, one FFIEC bank) + fixture prices (2 years) + factory for users/clubs (`tests/factories.py`). Each test resets schema `app` from a Postgres template database (`CREATE DATABASE … TEMPLATE`, fast); schema `market`, the store files and the analysis snapshot are read-only fixtures.
- **CI:** GitHub Actions job installs Playwright Chromium, runs smoke e2e on PRs and full e2e nightly; artefacts: traces/screenshots on failure. No network to SEC/vendors (C-8).

## 3. Commands (also in `architect.config.md`)

| Purpose | Command |
|---|---|
| build | `uv sync --frozen --all-groups` |
| lint | `uv run ruff check . && uv run ruff format --check . && uv run mypy && uv run lint-imports` |
| unit | `uv run pytest tests/unit` |
| integration | `uv run pytest tests/integration tests/contract` |
| e2e | `uv run pytest tests/e2e --browser chromium` |
| regression | `uv run pytest tests/unit tests/integration tests/contract tests/e2e -m "not slow"` + `uv run pytest -m regression` |

## 4. Business rules coverage matrix

Status: all **planned** (no code yet). Test locations are the agreed names the developer creates.

| BR | Level | Test (location::name) | Status |
|---|---|---|---|
| BR-01 | unit | `tests/unit/portfolio/test_portfolio.py::test_change_without_reason_rejected`, `::test_blank_reason_rejected`, `::test_each_change_kind_requires_reason` | planned |
| BR-01 | integration | `tests/integration/portfolio/test_repo.py::test_db_check_rejects_empty_reason` | planned |
| BR-01 | e2e | `tests/e2e/test_portfolio.py::test_add_position_requires_reason` | planned |
| BR-02 | integration | `fdc: tests/integration/test_fsds_load.py::test_extension_tags_labels_line_order_segments_preserved`, `::test_values_exact_text` | planned |
| BR-02 | contract | `tests/contract/test_store_schema.py::test_app_never_writes_store` (store opened ro) | planned |
| BR-03 | integration | `fdc: tests/integration/test_amendments.py::test_amendment_does_not_overwrite_original`, `::test_reload_same_adsh_replaces_atomically` | planned |
| BR-04 | unit | `fdc: tests/unit/test_window.py::test_only_last_12_years_selected` | planned |
| BR-05 | integration | `fdc: tests/integration/test_incremental.py::test_rerun_collects_nothing_new`, `::test_failed_item_retried`, `::test_last_run_recorded`, `::test_concurrent_run_refused` | planned |
| BR-06 | unit | `tests/unit/shared/test_authz_matrix.py::test_private_items_invisible_to_others` (generated matrix) | planned |
| BR-06 | integration | `tests/integration/test_idor.py::test_every_route_denies_foreign_item` | planned |
| BR-06 | e2e | `tests/e2e/test_privacy.py::test_other_user_cannot_open_portfolio_url` | planned |
| BR-07 | unit | `tests/unit/social/test_share.py::test_share_community`, `::test_share_specific_user`, `::test_revoke`, `::test_recipient_can_copy_not_edit_tool` | planned |
| BR-07 | e2e | `tests/e2e/test_sharing.py::test_share_tool_and_recipient_runs_copy` | planned |
| BR-08 | unit | `tests/unit/clubs/test_club.py::test_creator_is_single_super_admin`, `::test_only_super_admin_grants_admin`, `::test_admin_cannot_eject_admin`, `::test_admin_ejects_member`, `::test_super_admin_cannot_be_ejected`, `::test_member_cannot_edit_documents` , `::test_transfer_ownership_to_any_member`, `::test_transfer_to_non_member_rejected`, `::test_only_super_admin_transfers`, `::test_former_super_admin_becomes_member`, `::test_super_admin_leave_rejected_without_decision`, `::test_delete_club_only_by_super_admin` | planned |
| BR-08 | e2e | `tests/e2e/test_clubs.py::test_roles_lifecycle`, `::test_transfer_ownership_then_leave`, `::test_delete_club_removes_club_items_keeps_member_copies` | planned |
| BR-09 | unit | `tests/unit/shared/test_authz_matrix.py::test_club_members_read_and_comment_all_documents` | planned |
| BR-18 | unit | `tests/unit/shared/test_authz_matrix.py::test_public_space_post_readable_by_anonymous`, `::test_anonymous_cannot_comment`, `::test_club_post_hidden_from_non_members_and_anonymous`, `::test_anonymous_denied_everything_else`; `tests/unit/social/test_post.py::test_post_space_immutable`, `::test_member_cannot_create_club_post` | planned |
| BR-18 | e2e | `tests/e2e/test_posts.py::test_visitor_reads_public_post`, `::test_club_post_only_for_members` | planned |
| BR-19 | unit | `tests/unit/social/test_anonymise.py::test_posts_and_comments_kept_with_deleted_user_author`, `::test_no_link_to_former_account_retained`, `::test_audience_unchanged_after_anonymisation`, `::test_embedded_private_items_render_placeholder` | planned |
| BR-19 | integration | `tests/integration/identity/test_gdpr.py::test_delete_saga_keeps_posts_deletes_private_items`, `::test_anonymisation_idempotent_on_retry` | planned |
| BR-19 | e2e | `tests/e2e/test_account_deletion.py::test_deleted_users_post_shows_deleted_user` | planned |
| BR-19 + FR-30 | integration | `tests/integration/identity/test_gdpr.py::test_anonymisation_applies_only_to_remaining_content_after_purge` | planned |
| BR-10 | unit | `tests/unit/tools/test_tool.py::test_import_to_club_creates_copy_with_provenance`, `::test_tool_built_in_club_space` | planned |
| BR-11 | integration | `tests/integration/clubs/test_mark_as_bought.py::test_creates_position_with_reason_and_copies`, `::test_copies_editable_originals_unchanged`, `::test_atomic_on_failure` | planned |
| BR-11 | e2e | `tests/e2e/test_clubs.py::test_mark_as_bought_flow` | planned |
| BR-12 | unit | `tests/unit/portfolio/test_plan.py::test_plan_requires_horizon_and_measurable_expectation` | planned |
| BR-13 | integration | `tests/integration/market/test_prices.py::test_eod_upsert_latest_shown`, `tests/unit/web/test_no_live_price_calls.py` | planned |
| BR-14 | unit (architecture) | `tests/unit/test_architecture.py::test_no_broker_or_payment_dependencies` (dependency allow-list), e2e `test_disclaimer_visible` | planned |
| BR-15 | unit | `fdc: tests/unit/test_sectors.py::test_sic_maps_to_one_sector`, `tests/integration/test_watch.py::test_watcher_claims_only_its_sector` | planned |
| BR-16 | integration | `fdc: tests/integration/test_ffiec.py::test_call_report_loaded_with_source_ffiec` | planned |
| BR-17 | unit | `tests/unit/billing/test_ledger.py::test_reserve_rejected_when_available_below_worst_case`, `::test_settle_never_exceeds_reservation`, `::test_release_restores_available`, `::test_reservation_single_terminal_transition`, `::test_balance_never_negative` (hypothesis) | planned |
| BR-17 | unit | `tests/unit/tools/test_builder_session.py::test_session_start_requires_entitlement`, `::test_llm_not_called_when_reservation_fails`, `tests/unit/tools/test_run_tool.py::test_running_tools_not_gated_by_billing` | planned |
| BR-17 | integration | `tests/integration/billing/test_concurrency.py::test_parallel_reservations_cannot_overspend`, `tests/integration/billing/test_webhooks.py::test_top_up_granted_once_per_event`, `::test_invalid_signature_rejected`, `::test_return_url_grants_nothing` | planned |
| BR-17 | e2e | `tests/e2e/test_premium.py::test_no_credit_blocks_builder_but_runs_tools`, `::test_top_up_then_build_and_see_usage` | planned |

### FR/NFR acceptance tests (selection)

| Req | Test |
|---|---|
| FR-03 / NFR-04 | `fdc: test_watch.py::test_provisional_then_superseded_by_bulk`; SLI check in ops |
| FR-05 | `fdc: test_export.py::test_export_import_roundtrip_checksums` |
| FR-06 | `fdc: test_restatement.py::test_changed_value_flagged`, `::test_rounding_within_tolerance_not_flagged` |
| FR-07 | `fdc: test_status.py::test_status_reports_last_success_and_failures` |
| FR-08 | `tests/integration/market/test_statements.py::test_as_presented_order`, `::test_annual_only`, `::test_consolidated_only` |
| FR-10 | `tests/contract/test_screener_core.py::test_builtin_outputs_match_golden` |
| FR-11 | `tests/integration/tools/test_builder.py::test_fake_llm_spec_validated_dry_run_saved` |
| FR-12 | `tests/integration/tools/test_tool_to_watchlist.py` |
| FR-15 | `tests/integration/portfolio/test_plan_review.py::test_machine_expectations_evaluated` |
| FR-17/18 | `tests/integration/alerts/test_alerts.py::test_price_move_fires_once_per_state_change`, `::test_email_and_inapp_delivered_idempotently` |
| FR-27 | `tests/integration/identity/test_gdpr.py::test_export_contains_all_contexts`, `::test_delete_saga_removes_personal_data_resumable`, `::test_billing_records_pseudonymised_not_deleted`, `tests/unit/identity/test_delete_account.py::test_refused_until_decision_for_each_owned_club`, `::test_decisions_executed_first_in_saga`; `tests/integration/identity/test_magic_link.py::test_single_use_and_expiry`, `::test_unverified_email_cannot_sign_in` |
| FR-30 | unit `tests/unit/social/test_content_purge.py::test_requires_valid_confirmation_token`, `::test_rejected_without_recent_reauth`, `::test_only_author_can_purge_own_content`, `::test_one_active_purge_per_account`; integration `tests/integration/social/test_content_purge.py::test_deletes_all_posts_and_comments_public_and_every_club`, `::test_includes_clubs_user_has_left`, `::test_club_documents_untouched`, `::test_private_items_kept_shares_revoked`, `::test_others_replies_kept_under_post_deleted_placeholder`, `::test_account_and_private_space_intact`, `::test_idempotent_rerun_deletes_nothing`, `::test_resumes_after_crash_mid_batch`, `::test_content_created_after_cutoff_is_kept` (concurrent posting during job), `::test_account_deletion_waits_for_running_purge`, `::test_notifications_for_deleted_content_removed`; e2e `tests/e2e/test_settings.py::test_delete_all_my_content_confirmation_text_and_result` |
| FR-29 | `tests/integration/billing/test_metering.py::test_usage_record_per_call_with_price_table_version`, `::test_charge_uses_vendor_reported_tokens_and_margin`, `::test_expired_reservations_released`; `tests/unit/billing/test_price_table.py` |
| NFR-11 (owner decision: Yahoo V1) | `tests/integration/market/test_prices.py::test_every_price_row_has_source`, `::test_stale_price_flagged_and_alert_suppressed`; `tests/unit/test_architecture.py::test_yfinance_imported_only_by_yahoo_adapter`; e2e `test_price_attribution_visible` |

## 5. Contract tests between components
- `fdc` store schema v1 ↔ app `FilingStore` adapter: the app's CI installs the pinned `fdc` tag, builds a store from its fixture zips, and runs the adapter tests. Schema major bump requires an ADR + app change.
- Tool-runner protocol: request/response JSON Schemas versioned (`runner_protocol: 1`); both sides tested against the schema.
- Screener core: golden files produced by the pinned screener version for the seed companies; upgrade of the pin regenerates them in a reviewed PR.

## 6. NFR tests
- **Performance (NFR-03):** locust scenario (company page, portfolio page, statements API) on staging with full store: p95 < 2 s at 50 req/s; server-timing budget 300 ms.
- **Load/scale (NFR-01):** synthetic 10k users / 30k portfolios / 300k changes in schema `app`; run alert evaluation and page tests; record DB CPU, connection use and slow queries (sizing input for the DB node, ADR-007).
- **Sandbox (NFR-09):** DSL fuzzing (hypothesis) – no crash/timeout beyond limits; runner image contains no DB driver/SDK/secrets (image inspection test, G-25); runner endpoint rejects calls without the worker token; memory bomb spec rejected by cost estimate; container limits enforced (timeout test).
- **Security:** OWASP ZAP baseline against staging; authZ matrix; pip-audit; CSRF/XSS tests for Markdown posts.
- **Resilience:** kill worker mid-job → job retried, no duplicate email; kill backfill mid-quarter → rerun completes with no duplicates (BR-05).
- **Collector memory (NFR-02):** M0 measurement and a regression test that streams a 200 MB synthetic data set with RSS < 512 MB (nightly, `-m slow`).
- **Rate limit (NFR-07):** unit test of token bucket across processes; integration with fake server counting requests/second.
- **Backup/restore and failover (NFR-06):** weekly automated dump-restore test, a quarterly drill RB-05, and one recorded standby failover test in staging or prod before beta (connection retry works, no data loss).
- **Billing (FR-29):** monthly reconciliation RB-09 (usage records vs LLM invoice ≤ 1 % gap).

## 7. Regression suite
`-m regression` marks: BR matrix tests, authZ matrix, golden screener outputs, e2e smoke per UC. Command in §3. Must be green before every merge to main and every release tag.

## 8. Quality gates
- Required CI checks green (lint, types, import contracts, unit, integration, contract, smoke e2e, pip-audit).
- Coverage: ≥ 90 % lines on `*/domain/` and `tools/dsl/`, ≥ 80 % overall (`pytest --cov`); collector ≥ 85 %.
- Every BR row in §4 must be `passing` before V1 release.
- No high/critical pip-audit findings; ZAP baseline no high alerts.

## 9. QA handoff checklist
1. Create `tests/{unit,integration,contract,e2e,nfr}` skeletons and pytest markers (`regression`, `slow`, `e2e`).
2. Build the seed bundle (with collector M1) and fixture prices; factories for users/clubs/portfolios.
3. Implement the generated authorization matrix test from 06 §2.
4. Playwright harness with `dev-up` fixture, fakes for LLM/email/prices, trace artefacts in CI.
5. Golden-output contract tests for built-in tools against the pinned screener core.
6. NFR suites: locust scenario, 10k synthetic data generator, sandbox fuzz and isolation tests, ZAP baseline job.
6a. Billing: fake `PaymentGateway` + fake LLM with fixed usage; concurrency tests for reservations; recorded webhook fixtures per chosen provider.
7. Wire `regression` command and CI required checks; update §4 statuses as tests land.
