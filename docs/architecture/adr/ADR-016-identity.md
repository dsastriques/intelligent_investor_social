# ADR-016: Identity: self-managed accounts and sessions

- Status: **Accepted** (owner, 2026-11-06, Q-08; FR-27 v0.3)
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
FR-27 sign-up/in, profile, deletion; NFR-08 GDPR (minimise sub-processors); NFR-10 cost.

## Options
1. Hosted IdP (Auth0/Clerk/Cognito).
2. Self-managed email + password (argon2id), email verification, reset, optional magic link and TOTP, server-side sessions.

## Decision
Option 2. Sign-in is **email + password** (argon2id) with **mandatory email verification** before the first sign-in completes. **Passwordless magic link** is an option on the sign-in page. Magic links are single-use, valid 15 minutes, signed (`itsdangerous`), stored hashed in `id_login_token` and rate-limited per email and IP; using one also verifies the email. Password reset uses the same token mechanism. TOTP is optional for the operator role only. Social login is out of V1. Emails go through `EmailSender` (ADR-017).

## Consequences
+ no extra sub-processor or cost; full control of deletion.
− security-sensitive code to maintain: use vetted libraries (argon2-cffi, itsdangerous), rate limits, Turnstile, tests.
