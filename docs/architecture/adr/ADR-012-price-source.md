# ADR-012: EOD price source behind a port

- Status: Proposed — owner decision Q-03
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
FR-09, BR-13, NFR-05: daily EOD prices with history for alerts. NFR-11: the source must permit display to multiple users of a public product; the screener's Yahoo (yfinance) source is unofficial and unlicensed.

## Options
1. Yahoo/yfinance (unlicensed — rejected for Pro).
2. EODHD EOD plan with appropriate display terms.
3. Tiingo commercial tier.
4. Polygon (Massive)/Nasdaq Data Link EOD with display licence.

## Decision
`PriceProvider` port with a bulk-EOD method; adapter chosen after a licence-terms check and a 30-instrument probe. Only tracked instruments stored. Vendor choice is the owner's (cost).

## Consequences
+ vendor swap is an adapter change; prices tagged with `source` (G-19).
− launch blocked until a licensed vendor is chosen (R-03).
