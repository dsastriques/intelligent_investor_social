# ADR-017: Notifications: in-app + email via an EU provider (Scaleway TEM recommended)

- Status: Proposed. Owner decision Q-07; v0.2 recommends Scaleway TEM, which follows ADR-015.
- Date: 2026-11-06 (v0.2)
- Deciders: architect (design mode); owner for items marked Proposed

## Context
FR-18: in-app and email (ASSUMPTION on channels). NFR-08: EU processing. NFR-10: email is inside the platform budget.

## Options
1. In-app only. 2. In-app + email via an EU transactional provider behind the `EmailSender` port. 3. Add Telegram/push.
Providers for 2: **Scaleway Transactional Email** (300 emails/month free, then €0.25 per 1,000, pay-as-you-go; [Scaleway](https://scaleway.com/en/pricing/managed-services)); Brevo or Mailjet (EU, prices not re-verified in v0.2).

## Decision
Option 2 with Scaleway TEM: the same provider and DPA as the serving plane. Beta volume (≤100 users × alerts, digests, verification mails, an estimated ≤ 5k/month) costs about €1/month. Telegram (used by the screener) can be added later as another channel adapter.

## Consequences
+ Reliable delivery via the outbox with idempotency keys; per-kind preferences; one sub-processor fewer.
− Deliverability set-up (SPF/DKIM/DMARC). TEM's sending limits and reputation for a new domain need checking in M4.
