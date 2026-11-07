# Architect config

Values marked ASSUMPTION are provisional until the owner confirms (assumption A-12). Task source confirmed by the owner (Q-12).

- task_source: github-issues   # owner, 2026-11-06 (Q-12)
- task_source_details: dsastriques/intelligent_investor_social issues, label `milestone:M0`..`milestone:M7`; collector work is tracked here too (body names the target repo)
- task_status_update: comment the PR link on the issue and add label "in-review"
- developer_skill: developer   # ASSUMPTION A-12 (owner's agentic-team repo); none = architect implements
- devops_skill: devops         # ASSUMPTION A-12
- qa_skill: qa                 # ASSUMPTION A-12
- repo: dsastriques/intelligent_investor_social   # code for collector milestones (M0-M2) lands in dsastriques/financial-data-collector; issues stay here
- base_branch: main
- branch_pattern: feature/<task-id>-<slug>
- commands:
  - build: uv sync --frozen --all-groups
  - unit: uv run pytest tests/unit
  - integration: uv run pytest tests/integration tests/contract   # needs Postgres via DATABASE_URL or testcontainers (ADR-007)
  - e2e: uv run pytest tests/e2e --browser chromium
  - regression: uv run pytest tests/unit tests/integration tests/contract tests/e2e -m "not slow" && uv run pytest -m regression
  - lint: uv run ruff check . && uv run ruff format --check . && uv run mypy && uv run lint-imports
- max_review_iterations: 3
