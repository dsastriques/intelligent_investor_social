# ADR-020: Billing and entitlements: prepaid credit ledger, per-call LLM metering, payment provider behind a port

- Status: **Proposed.** Metering design accepted. The payment provider, plan model and margin are owner decisions (Q-13, Q-14); business set-up and VAT are the owner's with an accountant (Q-15).
- Date: 2026-11-06 (new in v0.2)
- Deciders: architect (design mode); owner for provider, prices and business set-up

## Context
- BR-17 / FR-29 (owner, 2026-11-06): the conversational tool builder (FR-11) is **premium**. Users pay for the LLM tokens they consume, through a subscription and/or top-up. Usage is metered per user and per tool session, shown to the user, and charged as a pass-through with an optional margin (owner decision). Users without credit cannot start builder sessions. Running built-in and already-built tools stays free (ASSUMPTION).
- NFR-08: payment data and the payment provider are GDPR-relevant (sub-processor). NFR-10: payment fees are paid out of revenue, not the platform budget (ASSUMPTION A-26).
- BR-14 and G-20 forbid holding or moving *investment* money. Charging for the platform's own service is FR-29 and is not affected. The guideline is narrowed so that only the Billing context may call a payment provider, and only for the platform's own fees (G-20, G-23).
- EU VAT: selling electronically supplied services to consumers in other EU countries generally makes VAT due at the customer's member-state rate. The OSS (One-Stop Shop) scheme lets a NL seller declare it quarterly in NL. A €10,000/year EU-wide threshold for cross-border B2C sales allows home-country VAT below it. Prepaid credits may fall under the EU voucher rules (single- vs multi-purpose voucher decides when VAT is due), and EU consumer law adds price display incl. VAT and the 14-day withdrawal right for digital services/content (waivable with express consent when performance starts). **These are general rules stated from architect knowledge and not verified for this case. The owner decides the business set-up with an accountant (Q-15).** The design supports both a merchant-of-record (MoR) provider, where the provider is the seller and handles VAT, and a plain payment processor, where the owner is the seller.

## Options

### Plan model (Q-14)
| | Model | For | Against |
|---|---|---|---|
| P1 | **Prepaid credit top-ups** (e.g. €10/€25/€50), premium = "has a billing account with credit" | no debt and no dunning; simplest to meter (reserve → settle); matches "block when no credit" | per-transaction fees hurt small top-ups; credits may count as vouchers for VAT |
| P2 | Monthly subscription with included credit + top-ups | predictable revenue; "premium plan" wording | unused-credit rules, proration, cancellations; more provider features needed |
| P3 | Post-paid metered billing (invoice usage monthly) | pay only for what was used | credit risk, dunning, can't "block when no credit" cleanly |

### Payment provider (Q-13). Fees are as found 2026-11-06; see the verification column
| | Provider | Role | Fee (indicative) | iDEAL/SEPA | Verified? |
|---|---|---|---|---|---|
| A | **Paddle** | MoR: seller of record, collects and remits VAT, issues invoices | 5 % + $0.50 per checkout, no monthly fee | via Paddle checkout (not verified) | third-party only ([Creem blog, 2026-10-29](https://www.creem.io/blog/honest-payment-fee-breakdown-creem-vs-stripe-paddle-ls); [Dodo](https://dodopayments.com/blogs/paddle-fees-explained)); paddle.com/pricing was blocked |
| B | Stripe Managed Payments | MoR | Stripe processing + 3.5 % (+ Billing 0.5–0.7 %) | yes | third-party only ([Dodo, 2026-05](https://dodopayments.com/blogs/stripe-managed-payments-fees-explained)); NL eligibility **not verified** |
| C | Stripe Payments + Billing + Tax | processor, owner is the seller (OSS registration) | EEA cards ≈ 1.5 % + €0.25 (**not verified**, stripe.com was blocked) + Billing/Tax add-ons | yes | no |
| D | Mollie (NL) | processor, owner is the seller | iDEAL €0.29; EU cards 1.8 % + €0.25; SEPA DD €0.25 + 0.4 %; no monthly fee; Subscriptions API | yes | third-party only ([comparepsp](https://comparepsp.pages.dev/providers/mollie/)); mollie.com was blocked |
| E | Lemon Squeezy / Creem | MoR | 5 % + $0.50 (+1.5 % intl) / 3.9 % + $0.40 | partly | vendor blogs only |

## Decision
1. **New bounded context BC-11 Billing & Entitlements** (01 §5.9). It owns `BillingAccount`, the append-only `CreditLedger`, `Reservation`, `UsageRecord`, the versioned `PriceTable`, `Entitlement` and `PaymentIntent`. Tools asks it through a facade (`entitlements.can_use`, `metering.reserve/settle/release`) and never touches its tables (G-02).
2. **Ledger, not a mutable balance:** `bl_ledger_entry(account, kind ∈ {top_up, subscription_grant, settlement, refund, adjustment, expiry}, amount_eur_micros, ref, at)`. The balance is the sum of the ledger. `bl_reservation(id, account, session, amount, status ∈ {open, settled, released, expired}, expires_at)`. The available balance is the balance minus open reservations. Reserving takes a row lock on the account (Postgres, ADR-007), so concurrent calls cannot overspend. Invariants are in 01 §5.9 (BR-17).
3. **Metering per LLM call** (ADR-010): reserve the worst case, call, then settle with the vendor-reported tokens × `PriceTable(version)` × (1 + margin), converted to EUR at the ECB daily rate. Each settlement writes a `UsageRecord` (user, session, tool, model, tokens by type, vendor request id, cost USD, charged EUR, table version). Users see a usage page and a running session cost (FR-29). Operator view: revenue vs vendor cost per month (RB-09 reconciliation against the LLM invoice).
4. **Payments behind a `PaymentGateway` port** (`create_checkout(account, product, idem_key)`, `parse_webhook(raw, signature)`, `refund(...)`, `customer_portal_url(...)`). Credits are granted **only** from a verified, idempotent webhook (`bl_webhook_inbox`, unique provider event id). Return URLs never grant credit. No card data touches our systems (hosted checkout; PCI scope SAQ-A).
5. **Recommendation:** **P1 prepaid top-ups** for the beta, with a minimum top-up of €10 so a 5 % + $0.50 fee stays ≤ ~10 %. Pair it with **provider A (Paddle, MoR)**. With one owner-developer and no accountant-run VAT process yet, MoR removes OSS registration and filing, per-country VAT rates, invoices and most chargeback handling, and costs ≈ €0.95 on a €10 top-up. Switch to C or D (processor, lower fees) once volume or a company set-up justifies doing VAT in-house. B is equivalent to A if the owner prefers the Stripe ecosystem and Stripe confirms NL eligibility. P2 can be added later as a `subscription_grant` source without changing metering.
6. **Credits are restricted:** usable only for the platform's own builder tokens; not transferable, not withdrawable, refundable only through provider refunds within the withdrawal period; an optional expiry of 12 months (config, owner decision). This keeps credits a prepayment for our service rather than stored value. **Legal check advised** (R-15).
7. **Free paths stay free:** `RunTool` (built-in, own, shared or club tools) has **no** entitlement check, only the existing run quotas (06 §3.6). A club admin who builds a tool in the club space with the builder pays from their **own** account (ASSUMPTION A-25).
8. **GDPR and retention:** the provider is listed as a sub-processor. The ledger, usage records and invoices referenced by provider id are kept for the statutory bookkeeping period (NL: 7 years, **to be confirmed by the accountant**, A-27) after account deletion, pseudonymised (account id → hashed reference, no conversation content). The deletion saga (06 §4.2) gets a Billing step that pseudonymises rather than deletes. Unused credit at deletion: refund policy per Q-14.

## Consequences
+ BR-17 holds by construction: no reservation, no LLM call. The platform budget carries no LLM cost. Usage is transparent to users.
+ Swappable provider (port), MoR now and processor later. Idempotent webhooks and a ledger make reconciliation auditable.
− An MoR fee of ≈ 5 % + $0.50 per top-up, and the margin decision is the owner's.
− A new context to build and test (M5b in 07). Billing bugs touch real money, so ledger invariants are unit-tested on passing and failing paths, and property tests check that balances never go negative (04 §4).
− Legal and tax set-up (VAT/OSS, vouchers, withdrawal right, terms of sale, e-money exclusion) must be settled before the first paid top-up (gate in M5b).
