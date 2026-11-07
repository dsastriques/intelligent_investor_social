# ADR-023: Concept vocabulary and a point-in-time input binder in BC-3 Reconciliation

- Status: Proposed (architect, design v0.7; default basis is owner question Q-16, total_debt definition Q-20)
- Date: 2026-11-07
- Deciders: architect (design mode); owner for Q-16, Q-20

## Context
BR-21: script inputs name financial concepts (`total_debt`, `cash_and_equivalents`, `diluted_eps_ttm`, `price`). Binding values from as-reported data for a company and date is a separate component, the reconciliation layer of BR-02 (ADR-008). BR-23: series and required lengths. BR-24: the binder supplies normalised values (units, scale, currency). BR-03: all filings are kept and later values never overwrite earlier ones, so a past date must be evaluable with the data known then. FR-34: price at the chosen date is an input supplied by the caller, and the trace must show values, periods and source filings. ADR-008 today produces only `market.std_financials`: the latest annual value per fiscal year, used by the screener core.

## Options
1. Bind from `market.std_financials` (annual, latest-filed). No quarters (no TTM), no vintages (no point-in-time), no per-fact lineage. Fails BR-23 TTM, BR-03 point-in-time and FR-34 traces.
2. Bind directly from the as-reported store per request (tags → concepts at run time). It reuses the store, but puts mapping logic in the request path and duplicates the MappingSet.
3. **Extend Reconciliation:** a versioned **concept vocabulary**, MappingSet v1 extended to quarterly and instant facts, and per-company **concept-fact files with every vintage** (`std/<company_id>/<hash>.sqlite`, published immutably like ADR-019). A pure **binder** domain service plus a `BindScriptInputs` application service reads those files and BC-2 market data (prices, splits, FX).
4. Option 3 but with concept facts in Postgres. Simpler queries, but roughly 4–10k rows per company × ~8–10k filers ≈ 30–100 M rows in the managed DB (**not measured**) against NFR-10.

## Decision
Option 3 (08 §4):
- **Vocabulary v1** owned by BC-3 (`config/concepts/v1.yaml`). It starts from the screener's `AnnualFinancials` fields and `CONCEPTS` aliases, renamed to the brief's concept names where they differ (`cash_and_equivalents`, `preferred_equity`, `diluted_eps`, `diluted_shares`), and adds `minority_interest`, `shares_outstanding`, `price`, and the derived `total_debt`. Users never extend it. Storage tags appear only in MappingSets (G-28).
- **Binder vs script line:** the binder only standardises (tag choice, summing components into a standard concept, sign conventions, TTM/Q4 assembly, unit/scale/split/currency normalisation). All analytics are scripts (market cap is a built-in script).
- **Bases:** `at_date` (price, last close ≤ as_of within 5 trading days), `latest`, `fy`, `fq`, `ttm` (4 discrete quarters if newer than the last FY, else FY), and series of these. A shorter or gapped series → `missing_input` naming the parameter and the gap.
- **Point in time:** default `as_known`, which means facts from filings accepted on or before the as-of date; the latest-accepted vintage wins and restated values are flagged with their prior value. Option `latest` uses today's knowledge. Recorded in every run (Q-16).
- **Currency:** one evaluation currency per run (default: reporting currency), ECB rates (stocks at period end, flows at period average, price at price date), recorded per input. Every money value carries its currency tag for the kernel's mixed-currency check.
- **Storage:** concept-fact files written by `iip reconcile` on the batch plane and published to object storage. `market.company_std_version` points to the current file. `market.std_financials` stays as the latest-annual projection for the screener core and universe tools (ADR-008 unchanged).

## Consequences
+ Scripts stay pure, and every figure is traceable to filing, tag, period and acceptance time (FR-34, BR-02).
+ Past-date evaluation is stable under later restatements (BR-03), which is useful for plan reviews (FR-15) and later back-tests.
+ The managed Postgres stays small. Reads reuse the ADR-019 cache path.
− Reconciliation grows: quarterly differencing, Q4 derivation and vintage handling. Mapping errors (R-04) now also affect quarters, so coverage metrics are kept per concept and basis.
− File size and rebuild time of concept-fact files are not measured (R-19). Measured in M3 on the seed and a full sector, with a fallback to fewer vintages (only the latest per filing-year) if needed.
