# ADR-012: EOD price source behind a port — vendor: marketstack (licence-gated), Tiingo redistribution as fallback

- Status: **Accepted — pending owner** (Q-03 answered 2026-11-06: "architect proposes the vendor"; owner confirms choice and the licence gate, Q-03b)
- Date: 2026-11-06 (v0.2; v0.1 left the vendor open)
- Deciders: architect (design mode); owner for cost and the written licence confirmation

## Context
FR-09, BR-13, NFR-05: daily EOD closes for tracked companies, history for alerts. NFR-11: the licence must permit **display to logged-in users of a web app** (private beta ≤100 users, NFR-01; later public). NFR-10: the platform budget is €50–200/month **including** the price vendor. Yahoo/yfinance stays excluded (unlicensed). Exchange fees are not the issue for EOD: data at least one trading day old carries no exchange display fees, only the vendor's licence terms ([marketdata.app licensing guide](https://www.marketdata.app/education/stocks/stock-market-data-licensing/), third-party, Jan 2026). The vendor receives only API keys and tickers, no personal data, so GDPR impact is nil (no DPA needed beyond the account).

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
| — | Yahoo/yfinance | — | — | No licence. **Excluded** | — |

Observations:
- The only option that fits NFR-10 is marketstack (row 1). Its licence for our use is unconfirmed.
- Of the vendors that explicitly license display, Tiingo Start-ups (row 3) is the cheapest at about €217/month. That alone is above the €200 ceiling.
- Twelve Data, marketdata.app and Intrinio are licensed but cost 1.5–4× the whole budget.

## Decision
1. Keep the **`PriceProvider` port** with a bulk/batched EOD method. Rows carry `source` (G-19) so the vendor can be swapped with an adapter change.
2. **Vendor: marketstack Professional** ($49.99/month, ≈ €43; annual $43.99 ≈ €38). **Gate (before M3 goes live with users):** the owner obtains written confirmation from apilayer/marketstack that the plan permits displaying EOD closes and derived values (portfolio values, alerts, charts) to logged-in users of our web app. The request is one email, as was done for EODHD's storage terms. The reply is archived in `docs/licences/`. If they say yes but a higher tier is needed, pick the tier they name. Business ($149.99) still fits only with the lean hosting variant (03 §10).
3. **Fallback if marketstack declines or stays silent:** Tiingo Start-up Redistribution ($250/month), after confirming its start-up eligibility and limits with Tiingo sales. This needs an NFR-10 budget exception of about +€175/month (owner decision Q-03b). Twelve Data Venture is the second fallback.
4. **Quota-aware refresh (ASSUMPTION A-22):** daily EOD only for **tracked** instruments (in any portfolio, watchlist or alert rule, FR-09). The rest of the listed universe used by built-in screens is refreshed **weekly** (config `prices.universe_refresh: weekly|daily`). Result pages show the price date. Estimated requests at one request per ticker (marketstack request counting **not verified**; docs page was not reachable): beta ≈ 1,000 tracked × 21 days + 6,000 universe × 4.3 weeks ≈ 47k/month, inside Professional's 100k. If multi-symbol requests count as one, Basic suffices.
5. **Probe before go-live** (like the screener's EODHD probe): 30 instruments covering NYSE, Nasdaq, an ADR (20-F filer), a bank and a recent split. Compare closes and adjustments against a second source for 60 days. Record coverage of the CIK→ticker map.

## Consequences
+ Fits the €200 ceiling. US data comes from the same upstream as Tiingo, and the vendor can be swapped.
+ Licensing risk is handled explicitly: there is a written gate and a priced fallback, and nothing goes live without a licence (NFR-11).
− Launch with real users depends on the licence answer (R-03, R-13). If the fallback is needed, the budget ceiling is broken unless the owner raises it.
− Weekly universe prices make full-universe P/E screens up to a week stale. Tracked companies stay daily (FR-09).
