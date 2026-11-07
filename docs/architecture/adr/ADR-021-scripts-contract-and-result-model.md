# ADR-021: Scripts as a separate versioned artefact in BC-4 Tools, with one contract and one result model

- Status: Proposed (architect, design v0.7; owner review of Q-16..Q-21 pending)
- Date: 2026-11-07
- Deciders: architect (design mode); owner for items marked Proposed

## Context
Requirements v0.6 add **scripts**: modular, reusable, deterministic calculations over report figures and price information (BR-20). Inputs name concepts, never storage tags (BR-21). Scripts are versioned and composable with pinned versions (BR-22), take scalars or series (BR-23), have typed inputs and outputs with units and currency (BR-24), and always return a status: `ok`, `undefined` with a reason, or `missing input` (BR-25). FR-31 fixes the contract fields, FR-32 the built-in catalogue, FR-33 user scripts built by conversation, and FR-34 running a script for a company and date with a trace. Tools, analysis monitors and dashboards will be built on scripts later. ADR-009 already defines a ToolSpec DSL with inline `metrics.expr`.

## Options
1. **A DSL construct only**: scripts are named `metrics` inside ToolSpecs. No standalone versioning, sharing or run per company and date. Fails BR-22 and FR-34.
2. **A new bounded context "Calculations"** with its own module, tables and facade. Clean separation, but scripts share the Tools lifecycle (builder session, versions, sharing, copy, runner, quotas). A new context duplicates all of it and adds a Tools → Calculations dependency for every tool.
3. **A separate artefact inside BC-4 Tools** (`iip.tools.scripts`): a `Script` aggregate with immutable `ScriptVersion`s and its own `ScriptSpec` JSON schema. It shares the **expression core** (`iip.tools.expr`) with the ToolSpec DSL, and the builder, sharing, runner and metering with tools. ToolSpec `dsl_version 2` may reference scripts by pinned version.

## Decision
Option 3 ("both": a separate artefact, referenced from the DSL).
- Contract `ScriptSpec v1` and types per 08 §2. Result model `ScriptResult = {status, outputs (each with its own status), reason, missing[], trace}` per 08 §2.3. Values are decimal strings; there is no NaN, infinity, or null-with-ok.
- The kernel (`iip.tools.scripts.kernel`) is pure: no I/O and no clock. It never raises across its boundary; internal errors become `undefined("internal evaluation error")` and are alerted (G-30).
- **Where evaluation runs:** closures of built-ins only run in-process (worker, and web for synchronous panels). Any closure that contains a user script runs in `iip-tool-runner` (runner protocol 2, `kind: script`, bound inputs passed in, so the runner needs no data access).
- **ScriptRun** (BC-4): an immutable record with the full trace. It is citable in posts (`script_run:<id>` embed) and retained 90 days, or while cited (A-51).
- **Memoisation:** binding memo keyed by data versions, evaluation memo keyed by closure content hash + canonical inputs hash (08 §6.2).

## Consequences
+ One contract for built-ins and user scripts. Tools, plan reviews (FR-15) and later monitors and dashboards reuse scripts by reference.
+ The ToolSpec DSL and scripts share one grammar, so the LLM learns one expression language.
+ Exact memoisation (pure functions, content-addressed keys) keeps pages within NFR-03.
− BC-4 grows. Scripts sit in their own subpackage with an import-linter contract (only `tools.expr` is shared), so the module can be split later.
− Two artefacts in one builder session type: the session gains an `artefact` discriminator.
