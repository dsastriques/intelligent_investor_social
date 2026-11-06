# 06 — Security, authorization, tool sandbox and GDPR (topics T4, T6, part of T8)

Serves BR-06..BR-11, FR-11, FR-19..FR-27, NFR-08, NFR-09, NFR-10.

## 1. Threat model (summary)

| Threat | Asset | Control |
|---|---|---|
| Another user reads my private portfolio/plans/tools | private space (BR-06) | policy checks in application layer + viewer-filtered queries; authZ test matrix (04 §4) |
| Broken access on embeds/shares | posts, shares | embed resolution checks read permission per viewer |
| User-built tool reads other users' data or attacks host | app DB, host | DSL only, separate sandboxed runner with no app DB access and no network (§3) |
| Prompt injection through tool builder | LLM → spec | LLM output is only a **ToolSpec JSON** validated against a schema; no tool calling with side effects; user confirms before save |
| Account takeover | accounts | argon2id, rate limits, Turnstile, email verification, session rotation, optional TOTP, Cloudflare WAF |
| XSS/CSRF | web | Jinja autoescape, Markdown sanitised (bleach/nh3 allow-list), CSP, CSRF tokens, SameSite cookies |
| Data loss | app.db | Litestream continuous backup + restore drills (03 §6) |
| SEC block for over-fetching | collector | host-wide rate limiter, UA (NFR-07) |
| Regulatory exposure | platform | record-only (BR-14, G-20), disclaimers |

## 2. Authorization model (T6, ADR-011)

### 2.1 Concepts
- **Principal**: `account_id`, platform roles (`operator`), club memberships (club_id → role) — loaded per request (cached ≤ 60 s, invalidated on `Member*`/`Admin*` events).
- **Space**: `private:<account_id>` or `club:<club_id>`. Every item (portfolio, plan, tool, watchlist, analysis, document, alert rule) has exactly one `space_id`.
- **Share**: `(item_ref, audience = community | user:<id>, permission ⊆ {read, run})`.
- **Actions**: `read`, `comment`, `edit`, `delete`, `share`, `run` (tools), `copy` (tools/plans/analyses), `manage_members`, `manage_admins`.

### 2.2 Decision function (implemented once in `iip.shared.authz`, used by every context)

```
allowed(principal, action, item):
  space = item.space
  if space.kind == private:
      if principal.id == space.owner: return True                        # BR-06
      share = active share for item matching (community or principal)    # BR-07, FR-19
      if share is None: return False
      if action == read: return True
      if action == run and item.type == tool and 'run' in share.perms: return True
      if action == copy and item.type in (tool, plan, analysis): return True   # A-R9
      if action == comment and item.type in (portfolio, analysis, tool): return True  # comments on shared items (A-16)
      return False
  if space.kind == club:
      role = principal.role_in(space.club)                               # None if not a member
      if role is None: return community/user share rules as above (club may share items too)
      if action in (read, comment): return True                          # BR-09
      if action == run and item.type == tool: return True                # members run club tools (A-17)
      if action == copy: return True                                     # BR-11 copies, BR-10 imports
      if action in (edit, delete, share): return role in (admin, super_admin)   # BR-08 admins edit documents
      return False
  operator: may hide posts/comments and read content reports only (FR-21); no access to private items
```

Club membership management (domain invariants in `Club`, checked again by policy):

| Action | super_admin | admin | member |
|---|---|---|---|
| invite / accept request / eject **member** | ✓ | ✓ | ✗ |
| eject **admin** / revoke admin | ✓ | ✗ | ✗ |
| grant admin | ✓ | ✗ | ✗ |
| eject super admin | ✗ (nobody) | ✗ | ✗ |
| edit club watchlist, plan, analyses, documents, tools; import tool (FR-25) | ✓ | ✓ | ✗ |
| read all club documents; comment | ✓ | ✓ | ✓ |
| mark as bought into own portfolio (FR-26) | ✓ | ✓ | ✓ |
| leave club | ✗ (Q-10) | ✓ | ✓ |

### 2.3 Sharing (BR-07, FR-19)
- Owner (or club admin for club items) creates/revokes shares. Revoke is immediate (next request).
- Shared portfolio = read-only view of positions and changes **with reasons** (the reason trail is part of the portfolio). Amount display: ASSUMPTION A-18 — owner may choose "weights only" vs "full amounts" when sharing (default weights only, privacy by default; confirm with owner).
- Copies carry `copied_from` provenance and are independent (no back-propagation).

### 2.4 Enforcement points
1. Application use case: `policy.check(principal, action, item)` before any load-for-write or return (G-06).
2. Repository list queries take `viewer` and join `space` / `share` / `membership` filters — a missing check cannot leak lists.
3. Templates receive only already-authorized DTOs.
4. Tests: a generated **authorization matrix test** (roles × actions × item types × space kinds) asserting the table above, plus IDOR tests on every route (04 §4).

## 3. User-built tools and the conversational builder (T4, ADR-009, ADR-010)

### 3.1 Decision
User tools are **declarative ToolSpecs** (a JSON DSL) — not arbitrary Python. The LLM turns conversation into a ToolSpec; a deterministic interpreter runs it in a sandboxed process. Rationale: NFR-09 (isolation) on a single cheap server cannot be met safely with arbitrary code without heavyweight isolation (gVisor/Firecracker/WASM) that one developer must operate; the screener's analyses (filters, scores, ratios, ranks, time-series conditions) are expressible declaratively. An escape hatch to sandboxed Python (Pyodide/WASM) is a possible V2 ADR if users hit DSL limits.

### 3.2 ToolSpec DSL (v1)

```json
{
  "dsl_version": 1,
  "name": "Cheap compounders",
  "universe": {"source": "all_us" | "watchlist:<id>" | "sector:<name>" | "companies:[...]"},
  "params": {"max_pe": {"type": "number", "default": 15, "min": 1, "max": 100}},
  "metrics": {
     "pe": {"expr": "price / avg(eps_diluted, years=5)"},
     "roic": {"builtin_metric": "roic"},
     "rev_cagr_5y": {"expr": "cagr(revenue, years=5)"}
  },
  "filters": [ {"expr": "pe < params.max_pe"}, {"expr": "consecutive_positive(net_income) >= 5"} ],
  "uses_builtin": [ {"tool": "guardrail", "require_verdict": ["PASS"]} ],
  "score": {"weights": {"roic": 0.5, "rev_cagr_5y": 0.5}, "normalize": "rank_pct"},
  "sort": [{"by": "score", "desc": true}],
  "limit": 50,
  "output": {"table": ["name", "pe", "roic", "score"], "watchlist": {"target": "watchlist:<id>", "mode": "replace_tool_entries"}}
}
```

- Expressions are parsed by a small grammar (Lark) into an AST: arithmetic, comparisons, boolean ops, whitelisted functions (`avg`, `median`, `cagr`, `growth`, `lag`, `ttm`, `consecutive_positive`, `percentile_in_industry`, `sma`, `min`, `max`, `abs`, `coalesce`), identifiers = standardized fields (`AnnualFinancials`), market fields (`price`, `market_cap`), screener metrics (`roic`, `owner_earnings`, `fcf`, …), and params. Optionally `fact("us-gaap:Tag")` to read an as-reported tag for one company (bounded).
- No loops, no user-defined functions, no I/O: evaluation is bounded by universe size × expression size (cost estimated before running).
- JSON Schema + semantic validator (unknown identifiers, type checks, cost estimate) — errors are fed back to the LLM and shown to the user.

### 3.3 Runner isolation (NFR-09)
- `iip-tool-runner.service`: separate Unix user `iip-runner`, systemd hardening: `PrivateNetwork=yes` (no network), `ProtectSystem=strict`, `ProtectHome=yes`, `PrivateTmp=yes`, `NoNewPrivileges=yes`, `ReadOnlyPaths=` market.db + store (read-only), **no access to `app.db`** (file permissions), `MemoryMax=384M`, `CPUQuota=50%`, `TasksMax=16`, `SystemCallFilter=@system-service`, `RestrictAddressFamilies=AF_UNIX`.
- Each run: wall-time limit (default 20 s), row/output limits, cancellation; one worker process per run slot (default 2 concurrent).
- Inputs needed from private space (e.g. a user's watchlist as universe) are resolved **by the worker** after authorization and passed as a list of company ids — the runner never sees other users' data.
- Results returned via the socket and stored by the worker as `ToolRun` in the caller's space.

### 3.4 Run flow

```mermaid
sequenceDiagram
  actor U as User
  participant W as iip-web
  participant Q as job queue (app.db)
  participant K as iip-worker
  participant R as iip-tool-runner (sandbox)
  U->>W: Run tool (version v3, params)
  W->>W: policy.check(run) and quota check
  W->>Q: enqueue ToolRun(pending)
  K->>Q: claim
  K->>K: resolve universe ids (authorized)
  K->>R: {spec, inputs, limits} over Unix socket
  R->>R: evaluate against market.db std_financials + prices (read-only)
  R-->>K: rows, metrics, usage
  K->>Q: ToolRun(done) + outbox ToolRunCompleted
  K-->>Watchlists: apply result if output.watchlist (FR-12)
  W-->>U: HTMX poll shows results
```

### 3.5 Conversational builder (FR-11, ADR-010)
- `BuilderSession` in Tools context; worker calls `LlmClient.complete` with: system prompt (DSL grammar, identifiers catalogue, examples, current spec), the conversation, and **structured output** = ToolSpec JSON Schema.
- Loop: LLM proposes spec → validator → (errors back to LLM, max 3 repairs) → **dry run** on a 50-company sample in the runner → summary shown to user ("this tool keeps companies with P/E < 15 and …"); user saves as new `ToolVersion`.
- The LLM never receives other users' data; only the user's own conversation and the public identifier catalogue. Prompt-injection impact is bounded to producing a (validated, sandboxed) spec.
- Data processing agreement with the LLM vendor; prefer EU data residency/zero-retention options where available (Q-05).

### 3.6 Quotas and cost control (NFR-10, A-09)
Config values (defaults proposed, owner to set): tool runs per user/day (50), concurrent runs per user (1), builder messages per user/day (30), monthly LLM token budget per user and global hard cap (`llm_kill_switch` flag when reached), max universe size per run (all ~8k companies allowed but cost-estimated). Usage recorded per user; operator view.

## 4. Privacy and GDPR (NFR-08, FR-27)

### 4.1 Principles
- EU hosting for all personal data (VPS + backups in EU, ADR-015); sub-processors listed in privacy notice (hosting, Cloudflare, email, LLM, error tracking). Cloudflare proxies traffic (DPA, standard contractual clauses).
- Data minimisation: email, display name, optional bio; portfolios and plans are user content. No trackers.
- Consent: terms + privacy notice acceptance at sign-up (versioned `Consent`); email notifications opt-out per kind.
- Private by default (BR-06).

### 4.2 Export and deletion
- `ExportMyData` → worker builds a ZIP (JSON per context + Markdown of posts/plans) and emails a time-limited download link (or in-app download), within minutes.
- `DeleteAccount` → account `pending_deletion`, sessions revoked → **deletion saga** job steps, each idempotent and resumable: Portfolio & Plans (delete), Tools (delete private tools; tools copied by others remain theirs), Watchlists & Alerts (delete), Social (delete shares; posts and comments: delete or anonymise per **Q-10**), Clubs (leave all clubs; if super admin: **Q-10** — transfer or delete club), Notifications (delete), Identity (delete account row, keep a salted hash of the email for 30 days to prevent abuse re-signup — ASSUMPTION A-19). Backups age out within the retention window (35 days), stated in the privacy notice.
- Operator tools: data subject request log.

### 4.3 Other controls
- TLS everywhere (Cloudflare edge + origin cert / tunnel); HSTS; secure cookies.
- Secrets in `/etc/iip/iip.env` 0640; least-privilege Unix users per process (`iip`, `iip-runner`, `fdc`).
- Dependency scanning (pip-audit), SAST (ruff security rules / bandit), container-free but `systemd-analyze security` score target ≤ 3.0 for runner.
- Audit log (`id_audit`) for security-relevant events: sign-in, password change, share/revoke, role changes, exports, deletions, operator moderation.
