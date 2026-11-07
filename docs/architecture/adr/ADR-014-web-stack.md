# ADR-014: Web stack: FastAPI + Jinja/HTMX

- Status: Accepted
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
FR-28 web UI; NFR-03 p95 < 2 s; NFR-12 one developer + agents, Python preferred.

## Options
1. SPA (React/Next.js) + Python API.
2. FastAPI with server-rendered Jinja templates + HTMX (+ Alpine.js), JSON API for charts.
3. Django full-stack.

## Decision
Option 2 (Django was close; FastAPI chosen for explicit hexagonal wiring without an ORM pulling the domain toward Active Record).

## Consequences
+ single language, small JS surface, fast first paint, simple deploy.
− rich client interactions need care (HTMX partials); a mobile app later would use the `/api/v1` JSON API.
