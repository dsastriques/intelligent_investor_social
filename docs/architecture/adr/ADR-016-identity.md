# ADR-016: Identity: self-managed accounts and sessions

- Status: Proposed — owner decision Q-08
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
FR-27 sign-up/in, profile, deletion; NFR-08 GDPR (minimise sub-processors); NFR-10 cost.

## Options
1. Hosted IdP (Auth0/Clerk/Cognito).
2. Self-managed email + password (argon2id), email verification, reset, optional magic link and TOTP, server-side sessions.

## Decision
Option 2.

## Consequences
+ no extra sub-processor or cost; full control of deletion.
− security-sensitive code to maintain: use vetted libraries (argon2-cffi, itsdangerous), rate limits, Turnstile, tests.
