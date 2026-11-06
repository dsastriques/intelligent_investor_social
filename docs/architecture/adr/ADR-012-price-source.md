# ADR-012: EOD price source behind a port: Yahoo for V1 (owner risk acceptance); licensed replacement path marketstack / Tiingo

- Status: **Accepted (owner risk acceptance, 2026-11-06, Q-03b)**. The date to switch to a licensed vendor is open ("decide later").
- Date: 2026-11-06 (v0.3; v0.2 proposed marketstack behind a licence gate)
- Deciders: owner (vendor choice and risk acceptance); architect (port design, mitigations)

## Context
FR-09, BR-13, NFR-05: daily EOD closes for tracked companies, history for alerts. NFR-11: the licence must permit **display to logged-in users of a web app** (private beta ≤100 users, NFR-01; later public). NFR-10: the platform budget is €50–200/month **including** the price vendor. v0.2 excluded Yahoo/yfinance as unlicensed; in v0.3 the owner chose it for V1 knowingly (Decision §2). Exchange fees are not the issue for EOD: data at least one trading day old carries no exchange display fees, only the vendor's licence terms ([marketdata.app licensing guide](https://www.marketdata.app/education/stocks/stock-market-data-licensing/), third-party, Jan 2026). The vendor receives only API keys and tickers, no personal data, so GDPR impact is nil (no DPA needed beyond the account).

## Options (prices checked 2026-11-06; USD→EUR at the ECB reference rate 1.1537 of 16-Sep-2026; excl. VAT)

| # | Vendor / plan | USD/month | ≈ EUR/month | Display to logged-in users of our app | Verified? |
|---|---|---|---|---|---|
| 1 | **marketstack** Basic / **Professional** / Business (annual: 8.99 / 43.99 / 127.99) | 9.99 / **49.99** / 149.99 | 8.7 / **43.3** / 130.0 | Paid plans list "Commercial use"; free plan "Non-Commercial Use". Pricing page, FAQ and service agreement **say nothing about display or redistribution**. US data "licensed and sourced from Tiingo, Inc." | price yes; display **no**, needs written confirmation. Sources: [pricing](https://marketstack.com/pricing), [FAQ](https://marketstack.com/faq), [agreement](https://marketstack.com/agreement) |
| 2 | Tiingo Power / Commercial | 30 / 50 | 26 / 43 | **No**: "Internal Use Only" | yes, [tiingo.com/about/pricing](https://www.tiingo.com/about/pricing) |
| 3 | **Tiingo Redistribution**, Start-ups / Enterprise | 250 / 500 | 217 / 433 | **Yes**: a redistribution plan that includes EOD prices (and 24x5, corporate actions, security master). Eligibility and limits for "Start-ups" not published | price yes, terms no. [EOD product page](https://www.tiingo.com/products/end-of-day-stock-price-data) |
| 4 | Twelve Data Venture (≈17 % off with annual billing) | 499 | 433 | **Yes**: "External display data access"; individual plans cannot display to third parties; attribution required | yes. [pricing-business](https://twelvedata.com/pricing-business), [usage article](https://support.twelvedata.com/en/articles/5332349-commercial-and-personal-usage) |
| 5 | marketdata.app Commercial | 333 (months 1–6), 666 (7–12), 999 after | 289 → 866 | **Yes**: "Public Display to Your Users", no per-user fees; display only (no programmatic redistribution); data deleted on termination | yes. [pricing](https://www.marketdata.app/pricing/), [commercial addendum](https://www.marketdata.app/terms/commercial-use-addendum/) |
| 6 | Intrinio start-up display tier | 333–999 | 289–866 | Yes, by order form; the $150 individual plan has no display right | **no**, third-party only ([SentiSense, 2026-10-28](https://sentisense.ai/blog/market-data-api-commercial-use/)); [Intrinio EOD page](https://intrinio.com/products/stock-prices-eod) gives no terms |
| 7 | EODHD Commercial | internal use 399; display: custom quote "from $399" | ≥ 346 | Internal-use plan: display **not** permitted; display needs a custom quote | yes. [commercial pricing](https://eodhd.com/commercial-pricing), [licence page](https://eodhd.com/financial-apis/commercial-vs-personal-license-use) |
| 8 | FMP | quote only | — | Only with a separate "Data Display and Licensing Agreement", whether or not users pay | yes, no price. [FMP article](https://site.financialmodelingprep.com/insights/platform/can-you-use-fmp-data-in-a-public-app-website-or-client-dashboard) |
| 9 | Massive (ex-Polygon) Stocks Business | 2,499 | 2,166 | Yes; Business needed "as soon as anyone other than you can see the data" | yes. [business-stocks](https://massive.com/business-stocks), [KB](https://massive.com/knowledge-base/article/which-plan-do-i-need-to-show-massive-data-in-my-app) |
| 10 | Alpha Vantage | premium 49.99–249.99 personal; commercial by contact | — | Not without a written agreement | yes. [terms](https://alphavantage.co/terms_of_service) |
| 11 | EODData commercial feed | ≈750 (US exchanges) | ≈650 | A commercial licence is required to republish, charts included | yes. [EODData](https://eoddata.com/Products/DataFeed.aspx) |
| 0 | **Yahoo/yfinance (V1 choice, owner risk acceptance)** | 0 | 0 | **No**: the terms forbid automated collection, commercial use and building data feeds (§2(d)(ix), §2(d)(x), §2(h)) | yes, [Yahoo ToS](https://legal.yahoo.com/us/en/yahoo/terms/otos/index.html) |

Observations (v0.2 analysis, kept for the replacement path):
- Among licensed options, only marketstack (row 1) fits NFR-10. Its licence for our use is unconfirmed.
- Of the vendors that explicitly license display, Tiingo Start-ups (row 3) is the cheapest at about €217/month. That alone is above the €200 ceiling.
- Twelve Data, marketdata.app and Intrinio are licensed but cost 1.5–4× the whole budget.

## Decision (v0.3)
1. Keep the **`PriceProvider` port** with a batched EOD method. Every price row stores `source` (G-19), so changing vendor means writing a new adapter and nothing else.
2. **V1 adapter: Yahoo** (`YahooPriceProvider`, through `yfinance`, the library the screener already uses). It is free, so the price-vendor line in 03 §10 drops to €0. The **owner chose this with the risk known**. Yahoo's terms ([legal.yahoo.com Terms of Service](https://legal.yahoo.com/us/en/yahoo/terms/otos/index.html), checked 2026-11-06) forbid it:
   - **§2(d)(ix)**: no access to or collection of data "using any automated means" without Yahoo's express prior permission;
   - **§2(d)(x)**: Yahoo data may not be used "to create any database, archive, mobile application, data feed, widget or any other aggregated data source" that competes with or substitutes for its services;
   - **§2(h)**: the services may not be exploited "for any commercial purposes" without written permission.
   Yahoo has no official API; `yfinance` scrapes. This decision does not change **NFR-11** (licensed sources). It is recorded as an owner decision that runs against it (00 §5 traceability, R-03).
3. **Mitigations (required for the Yahoo adapter):**
   - **Isolation:** Yahoo is used only inside its adapter. No other module imports `yfinance` (import-linter contract).
   - **Resilience:** fetches run only in the worker, at low frequency (once per trading day for tracked instruments, weekly for the universe, A-22), throttled, with retry/backoff and a circuit breaker. If a fetch fails, the **last cached price** stays and is shown with its date and a **staleness indicator** (warning when older than 2 trading days). Alerts based on stale prices are suppressed and flagged instead of firing.
   - **Health:** a "price feed broken" alert (refresh missed, or more than 5 % of tracked symbols empty) and runbook RB-03 (switch adapter).
   - **Attribution and disclaimer:** price displays carry "Prices: Yahoo Finance, unofficial, may be delayed or incorrect; not investment advice" (wording to be confirmed by the owner). Raw Yahoo price series are **not** offered for download or through the public API. Only on-page display and derived values are shown.
   - **Stored history:** prices are kept for alerts (FR-09). If the vendor changes, the history is re-sourced from the new vendor, so Yahoo rows are not kept as the long-term record (backfill job per adapter).
4. **Licensed replacement path** (comparison above, unchanged): **marketstack Professional** ($49.99 ≈ €43/month, pending written confirmation that it allows display to logged-in users), then **Tiingo Start-up Redistribution** ($250 ≈ €217, explicit redistribution plan), then Twelve Data Venture. Their adapters are written against the same port contract tests (04 §5) and can be switched with a flag.
5. **Recommendation (not a requirement):** review the switch to a licensed vendor **before public launch, or before the first paid premium top-up (M5b)**, whichever comes first. Charging users makes the use of Yahoo data clearly commercial (§2(h)). It also adds the platform's own revenue to the exposure.
6. The **licensed-source test and the `docs/licences/` gate are removed.** They are replaced by **"source recorded per price row"** (test `test_every_price_row_has_source`), and the UI shows the source and date.

## Consequences
+ €0 for prices in V1, so hosting option A drops to about €98/month (03 §10). It uses the code path the owner already knows.
+ Swapping vendors is an adapter plus a flag, and contract tests cover every adapter.
− **Legal exposure:** the use breaches Yahoo's terms (automated collection, commercial use, data feed). Possible consequences are blocking, takedown requests or claims. The owner has accepted this risk (R-03).
− **Operational fragility:** the feed can change, rate-limit or block us without notice, and its data quality is unguaranteed (R-13). The cache, staleness indicator and alert suppression limit the damage but do not stop prices going stale.
− NFR-11 is not met in V1 until the switch is made.
