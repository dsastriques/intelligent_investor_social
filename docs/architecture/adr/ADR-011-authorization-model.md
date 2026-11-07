# ADR-011: Authorization: spaces, roles, per-item shares

- Status: Accepted (v0.4: public space and post visibility per BR-18; club ownership transfer and deletion per BR-08 v0.3)
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
BR-06..BR-11, BR-18, FR-19..FR-26: private spaces visible only to the owner, per-item sharing to community/specific users, club roles (super admin/admin/member) with precise rights, copies on import/mark-as-bought.

## Options
1. Pure RBAC with global roles.
2. Generic ACL tables per item.
3. Resource-based policy: every item belongs to one Space (private or club); decision = space ownership/club role + active Shares; implemented once as a policy function in the shared kernel, enforced in application layer and repeated as query filters.

## Decision
Option 3, with the role/action table in 06 §2.2 and Club invariants in the domain (single super admin, who can eject whom).

v0.4 additions:
- **Space kinds** are now `private`, `club` and **`public`**. There is one platform-wide public (community) space.
- **BR-18:** a post's visibility *is* its space; there is no separate visibility field. Posts in the public space are readable by anyone, **including anonymous visitors** (an `anonymous` principal with read-only access to public-space posts and their comments). Posts in a club space are readable only by that club's members.
- **BR-08 (v0.3):** the super admin may `TransferOwnership` to any other member. They may not `Leave` without a prior transfer or `DeleteClub`. `DeleteAccount` is refused while the account owns any club for which no transfer-or-delete choice has been made (06 §2.2, §4.2).

v0.7: **scripts** and **script runs** are items like tools. A script has `read`/`run`/`copy` on share; pinning another user's script requires `copy` (ADR-024). A script-run embed in a post shows the frozen result and input trace; the formula is shown only if the viewer may read the script (Q-17).

## Consequences
+ simple mental model matching the requirements; one place to test (generated authorization matrix).
+ copy semantics (BR-07 A-R9, BR-10, BR-11) fall out naturally (new item in target space with provenance).
− every list query must apply viewer filters (G-06; IDOR tests).
