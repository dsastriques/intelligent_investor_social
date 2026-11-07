# ADR-010: LLM for the conversational tool builder (premium, user-paid tokens)

- Status: **Accepted** (owner, 2026-11-06, Q-05: Anthropic Claude, Haiku-class default, Sonnet-class for repairs). Premium and metering follow BR-17 and FR-29, with billing in ADR-020.
- Date: 2026-11-06 (v0.2)
- Deciders: architect (design mode); owner for vendor, price and margin

## Context
- FR-11: conversational building.
- BR-17 / FR-29 (new in v0.2): the builder is a **premium** feature. Users pay for the tokens their sessions consume, so LLM cost is **outside** the platform budget (NFR-10). Users without credit cannot start builder sessions. Running built-in and already-built tools stays free (ASSUMPTION in reqs).
- NFR-08: conversation content is personal data.
- v0.1 capped LLM cost with per-user quotas and a global monthly cap paid by the platform. That cap has no budget role any more. It stays only as a safety limit against bugs and runaway loops.

## Options
1. Anthropic API: a small model by default (Haiku-class, $1 / $5 per Mtok input/output for Haiku 4.5 per [pricepertoken.com](https://pricepertoken.com/pricing-page/model/anthropic-claude-haiku-4.5); the first-party price page could not be fetched, **verify at M5**), a larger model for repairs, and structured output to the ToolSpec schema.
2. Another hosted vendor.
3. Self-hosted open model (needs a GPU; not affordable).

## Decision
Option 1 behind the `LlmClient` port: **Anthropic Claude**. The default model is **Haiku-class** for spec generation. **Sonnet-class** is used for repair turns after the first validator failure (at most 3 repairs). Model ids live in config (`llm.models.default`, `llm.models.repair`) and are pinned per price-table version. Exact ids and prices are verified at M5 against Anthropic's docs; the first-party price page was not reachable on 2026-11-06. A DPA and the zero/short-retention settings are part of M5 set-up. v0.2 changes:
1. **Entitlement check before every builder session and message** (ADR-020): `Entitlements.can_use(user, feature="tool_builder")` requires an active premium entitlement and an available credit balance ≥ `billing.session_min_credit` (config).
2. **Reserve, call, settle** for every LLM call. Before the call, reserve the worst-case cost (input tokens counted locally + `max_tokens` output, at the current price table and margin). Call the vendor. Then settle with the **vendor-reported usage** (input, output, cache-write and cache-read tokens) and release the rest. If the reservation fails, the session pauses with "out of credit". A call is never started that the user's credit cannot cover.
3. **Metering record** per call: user, session, tool, model, token counts, vendor request id, price-table version, cost in USD, charged EUR (ECB rate of the day, margin). Users see totals per session and per month in-app (FR-29).
4. **Cost hygiene that also lowers user prices:** prompt caching of the static system prompt (DSL grammar and identifier catalogue), the small model by default, at most 3 repairs (each one metered), and a dry run in the sandbox (not metered: it is a tool run, not LLM).
5. **Safety limits** (config, not budget): a per-user daily token ceiling, a global hourly token-rate alarm, and the `llm_kill_switch` flag.
6. Only the user's own conversation and the public identifier catalogue go to the vendor. A DPA and zero/short retention settings are part of Q-05.
7. **v0.7 (FR-33):** the same builder serves **scripts** (`BuilderSession.artefact = script`, structured output = ScriptSpec JSON Schema; the cached system prompt adds the concept vocabulary, function list and built-in catalogue). Premium gating and reserve/call/settle metering are identical. `UsageRecord.tool_id` is generalised to `artefact_ref`. Running scripts is free, like running tools.

Indicative cost per builder message (ASSUMPTION A-24, to be measured in M5): ≈ 10k input tokens (≈ 8k cached) + ≈ 2k output, so about $0.01–0.02 per call with Haiku 4.5 prices, ≤ $0.06 with 3 repairs. A 15-message session costs about $0.2–0.9 before margin.

## Consequences
+ The platform never carries LLM cost (BR-17). Prepaid reservation means there are no unpaid balances.
+ Cost is visible per session, so users learn what tools cost to build.
− Metering accuracy depends on vendor usage fields and a correct price table. The table is versioned, changed only by a reviewed config PR, and reconciled monthly against the vendor invoice (RB-09).
− A builder outage or a stuck vendor leaves reservations open. They expire after 15 minutes and are released by a sweeper job.
