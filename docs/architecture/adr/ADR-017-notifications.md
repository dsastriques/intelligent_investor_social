# ADR-017: Notifications: in-app + email via EU provider

- Status: Proposed — owner decision Q-07
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
FR-18 in-app and email (ASSUMPTION channels); NFR-08.

## Options
1. In-app only. 2. In-app + email via EU transactional provider behind `EmailSender` port. 3. Add Telegram/push.

## Decision
Option 2; Telegram (used by the screener) can be added as another channel adapter later.

## Consequences
+ reliable delivery via outbox with idempotency keys; per-kind preferences.
− DPA with the email provider; deliverability setup (SPF/DKIM/DMARC).
