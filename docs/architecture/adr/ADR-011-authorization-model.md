# ADR-011: Authorization: spaces, roles, per-item shares

- Status: Accepted
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
BR-06..BR-11, FR-19..FR-26: private spaces visible only to the owner, per-item sharing to community/specific users, club roles (super admin/admin/member) with precise rights, copies on import/mark-as-bought.

## Options
1. Pure RBAC with global roles.
2. Generic ACL tables per item.
3. Resource-based policy: every item belongs to one Space (private or club); decision = space ownership/club role + active Shares; implemented once as a policy function in the shared kernel, enforced in application layer and repeated as query filters.

## Decision
Option 3, with the role/action table in 06 §2.2 and Club invariants in the domain (single super admin, who can eject whom).

## Consequences
+ simple mental model matching the requirements; one place to test (generated authorization matrix).
+ copy semantics (BR-07 A-R9, BR-10, BR-11) fall out naturally (new item in target space with provenance).
− every list query must apply viewer filters (G-06; IDOR tests).
