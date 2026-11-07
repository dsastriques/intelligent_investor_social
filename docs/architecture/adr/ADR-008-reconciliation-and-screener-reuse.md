# ADR-008: Reconciliation layer and reuse of the screener core

- Status: Accepted. Packaging decided v0.4 (Q-06: all repos private → private git dependency)
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
FR-10 requires built-in tools ported from `the_intelligent_investor` (rules, guardrail, PE watch, peers, targets, MA alerts). These functions consume `CompanyData`/`AnnualFinancials` (standardized fields), while BR-02 forbids standardizing at storage time. The screener's `edgar_fundamentals.CONCEPTS` already maps us-gaap tags to its fields.

## Options
1. Re-implement the screener rules in the new app.
2. Copy (vendor) screener code into the product repo.
3. Keep screener pure modules (`models`, `metrics`, `rules`, `guardrail`, `ranking`, `prices`, `config`) as a library dependency; build a Reconciliation context (ACL) that derives `AnnualFinancials` from as-reported facts with versioned MappingSets and lineage.

## Decision
Option 3. MappingSet v1 starts from the screener's CONCEPTS table and annual-fact rules; extended for IFRS (20-F), sector branches, derivations. Overlap check against the screener's own EDGAR data is an M3 acceptance gate. Stateful screener parts (storage, providers, Telegram, dashboard, approvals) are not reused.

**v0.7 (scripts, ADR-021/023):** Reconciliation additionally owns the **concept vocabulary** and the point-in-time **input binder** for scripts (BR-21, BR-23, BR-24). MappingSet v1 is extended to quarterly and instant facts with every vintage, published as per-company concept-fact files. `market.std_financials` stays the latest-annual projection that feeds the screener core. Built-in tools remain wrappers around the screener's composite functions (FR-10 unchanged). The per-company calculations are also exposed as built-in **scripts** that call the screener functions or are golden-tested formulas (08 §7).

**Packaging (v0.4, Q-06).** The owner is making `intelligent_investor_social` and `financial-data-collector` **private**, so there is no public/private split any more. The simplest option is chosen: **the private git dependency** `the-intelligent-investor @ git+https://github.com/dsastriques/the_intelligent_investor@vX.Y.Z` (pinned tag) in `pyproject.toml`/`uv.lock`. **No `investor-core` extraction for now:** it would add a repo and a release process without serving a requirement. The import surface is limited by an import-linter contract: Pro may import only `investor.models`, `metrics`, `rules`, `guardrail`, `ranking`, `prices` and `config`, and never `storage`, `providers`, `cli`, Telegram or dashboard code. The screener's extra runtime dependencies (`yfinance`, `jinja2`) are already Pro dependencies (ADR-012, ADR-014). Extraction is reconsidered only if the screener's dependencies start conflicting with Pro's. Access: GitHub Actions uses a fine-grained read-only token (or a deploy key) for the screener repo, passed to `uv sync` and the image build as a BuildKit secret, never baked into images. The batch VM needs no access (it does not run screener code).

## Consequences
+ no logic duplication; screener fixes flow into Pro via version bumps; golden contract tests.
+ BR-02 preserved: standardized data is derived and traceable to filings.
− the screener is a pinned private git dependency; bumping it is a reviewed PR with regenerated golden outputs (04 §5). R-07 is reduced to a credential-hygiene risk (CI read token).
− mapping quality risk (R-04): lineage and coverage shown to users.
