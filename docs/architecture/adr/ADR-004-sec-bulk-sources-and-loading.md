# ADR-004: SEC bulk sources and per-accession loading policy

- Status: Accepted; FSN-vs-FSDS scope gated by M0
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
FR-01 (face statements, 12 years), FR-02 (notes), BR-02 (as reported incl. extensions, labels, order, segments), BR-03 (keep amendments, no overwrite), BR-04. The 2026-11-03 research recommends SEC Financial Statement Data Sets (sub/num/tag/pre) over companyfacts (which drops extension tags); Financial Statement and Notes data sets add notes.

## Options
1. companyfacts API per company (drops extension tags and presentation; rejected by BR-02).
2. FSDS only, notes later per filing.
3. FSN (Financial Statement and Notes) only.
4. Both readers; FSN default if it is a superset and fits the budget, else FSDS full window + FSN recent/on demand.

## Decision
Option 4. Loading policy: data sets oldest → newest; filter forms 10-K/10-Q/20-F/40-F and /A at stream time; facts keyed by accession (`adsh`) — each filing's facts stored under its own key, never merged across filings; reload of the same `adsh` replaces that filing atomically (A-04); amendments are separate filings.

## Consequences
+ one pass can give both FR-01 and FR-02; decision based on measured data.
+ restatement history preserved by construction (BR-03).
− FSN volume risk (R-01); mitigated by M0 thresholds and the fallback.
− exact column sets must be verified against the current SEC readme in M0.
