# ADR-022: Formula language and safe evaluator for user scripts

- Status: Proposed (architect, design v0.7)
- Date: 2026-11-07
- Deciders: architect (design mode)

## Context
FR-33 and owner default (4): user scripts are formulas over declared parameters and other scripts, written by the conversational builder, with no arbitrary code. NFR-09: user-built scripts run isolated and resource-limited. BR-25: never crash, never return a meaningless number. BR-24: refuse mixed currencies. ADR-009 already chose a declarative DSL with a Lark grammar for tools, and G-16 forbids `eval`/`exec`/`pickle` of user input.

## Options
1. Python `ast` restricted to a whitelist, then evaluated. Familiar syntax, but the boundary depends on a blacklist/whitelist over a general-purpose AST, and escape bugs are a known class.
2. An existing formula library (e.g. a spreadsheet-formula engine). Extra dependency, float semantics, weak typing, functions we do not want.
3. **Own small grammar** (Lark LALR) → typed AST → tree-walking interpreter with an explicit stack, fixed function whitelist, `Decimal` arithmetic and a step counter. Shared with the ToolSpec expression language.

## Decision
Option 3, per 08 §3:
- **Operators:** `+ - * / ^`, comparisons, `and/or/not`. **Control:** `if`, `coalesce`, `undefined("reason")`, `is_missing`. **Series:** `sum avg median min max stdev count count_if streak first last at window` and element-wise arithmetic. No loops, assignment, attributes, indexing syntax, lambdas, strings (except reasons) or I/O.
- **Static checks at save:** schema, parse limits, name resolution (concepts only, tags rejected), dimensional type check (money / share_count → per-share money, money ± money same currency variable), scalar/series discipline, closure checks. Errors feed the builder's repair loop.
- **Runtime:** `Decimal` with precision 34, `ROUND_HALF_EVEN` and exponent ±99. Division by zero, domain errors, overflow, the step limit and mixed currencies become `undefined` with a reason. Undefined dependencies propagate with the alias as prefix.
- **Limits** (config): 4 KB per formula, 512 AST nodes, depth 32, 32 inputs, series ≤ 1,300 values, closure ≤ 32 scripts or 8 levels deep, 100k evaluation steps, runner request ≤ 2 s.
- **Isolation:** closures containing user scripts are evaluated only in `iip-tool-runner` (ADR-009, 06 §3.3), with inputs already bound. Built-in Python scripts are platform code and may run in-process.

## Consequences
+ The attack surface is a few hundred lines of interpreter, fuzzed with hypothesis (no crash, no limit overrun, deterministic).
+ Deterministic results across machines (software decimal), so content-addressed memoisation is exact.
− Expressiveness is bounded by the whitelist. New functions need a grammar or kernel release, and a function's semantics never change within a kernel major (G-29).
− `Decimal` is slower than float. That is irrelevant at the evaluation sizes allowed by the limits.
