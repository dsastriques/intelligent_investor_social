# ADR-009: User-built tools: declarative DSL + sandboxed runner

- Status: Accepted
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
FR-11: users create and modify tools by conversation and run them on platform data. NFR-09: isolated from other users' data and the platform, resource-limited. Single small server, one developer (NFR-10, NFR-12).

## Options
1. LLM generates Python executed in containers/gVisor/Firecracker.
2. LLM generates Python executed in WASM (Pyodide) sandbox.
3. LLM generates a declarative ToolSpec (JSON DSL with a small expression grammar); deterministic interpreter in a separate systemd-sandboxed process (no network, read-only public data, no app DB, memory/CPU/time limits).

## Decision
Option 3 for V1. Option 2 is the documented V2 escape hatch if users need more expressiveness.

v0.2 note (ADR-015): the runner runs as a private managed container without DB role or secrets, reading a public analysis snapshot (ADR-019); the systemd-sandbox variant remains the fallback (hosting option C). The decision itself is unchanged.

## Consequences
+ strong isolation with simple ops; deterministic, explainable, versionable tools; easy to share/copy (BR-07, BR-10).
+ prompt-injection blast radius limited to a validated spec.
− expressiveness limited to the DSL's functions; grammar must evolve with user needs (versioned `dsl_version`).
