# ADR-008: Reconciliation layer and reuse of the screener core

- Status: Accepted; packaging per owner question Q-06
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
FR-10 requires built-in tools ported from `the_intelligent_investor` (rules, guardrail, PE watch, peers, targets, MA alerts). These functions consume `CompanyData`/`AnnualFinancials` (standardized fields), while BR-02 forbids standardizing at storage time. The screener's `edgar_fundamentals.CONCEPTS` already maps us-gaap tags to its fields.

## Options
1. Re-implement the screener rules in the new app.
2. Copy (vendor) screener code into the public repo.
3. Keep screener pure modules (`models`, `metrics`, `rules`, `guardrail`, `ranking`, `prices`, `config`) as a library dependency; build a Reconciliation context (ACL) that derives `AnnualFinancials` from as-reported facts with versioned MappingSets and lineage.

## Decision
Option 3. MappingSet v1 starts from the screener's CONCEPTS table and annual-fact rules; extended for IFRS (20-F), sector branches, derivations. Overlap check against the screener's own EDGAR data is an M3 acceptance gate. Stateful screener parts (storage, providers, Telegram, dashboard, approvals) are not reused.

## Consequences
+ no logic duplication; screener fixes flow into Pro via version bumps; golden contract tests.
+ BR-02 preserved: standardized data is derived and traceable to filings.
− private code in a public product: must be consumed as a private dependency (Q-06), never vendored (R-07).
− mapping quality risk (R-04): lineage and coverage shown to users.
