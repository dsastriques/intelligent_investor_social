# ADR-010: LLM for the conversational tool builder

- Status: Proposed — owner decision Q-05 / Q-02
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
FR-11 conversational building; NFR-10 budget incl. LLM cost; NFR-08 GDPR (conversation content is personal data).

## Options
1. Anthropic API (small model default, larger model for repairs), structured output to the ToolSpec schema.
2. Other hosted vendor.
3. Self-hosted open model (needs GPU; not affordable on the target VPS).

## Decision
Proposed option 1 behind an `LlmClient` port (swappable): structured output, validator loop (max 3 repairs), dry run, per-user quotas, global monthly cap and kill-switch flag; only the user's own conversation and the public identifier catalogue are sent. Final vendor/model and budget: owner.

## Consequences
+ high-quality spec generation with bounded cost; vendor swappable.
− external sub-processor for personal data (DPA, retention settings); costs scale with usage (quotas).
