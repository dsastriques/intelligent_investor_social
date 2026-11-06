# Architect config

Values marked ASSUMPTION are provisional until the owner confirms (owner question Q-12, assumption A-12).

- task_source: github-issues   # ASSUMPTION (Q-12); alternatives: trello, manual
- task_source_details: dsastriques/intelligent_investor_social issues (label per milestone M0..M7); collector tasks in dsastriques/financial-data-collector issues
- task_status_update: comment the PR link on the issue and add label "in-review"
- developer_skill: developer   # ASSUMPTION A-12 (owner's agentic-team repo); none = architect implements
- devops_skill: devops         # ASSUMPTION A-12
- qa_skill: qa                 # ASSUMPTION A-12
- repo: dsastriques/intelligent_investor_social   # collector milestones (M0-M2): dsastriques/financial-data-collector
- base_branch: main
- branch_pattern: feature/<task-id>-<slug>
- commands:
  - build: uv sync --frozen --all-groups
  - unit: uv run pytest tests/unit
  - integration: uv run pytest tests/integration tests/contract
  - e2e: uv run pytest tests/e2e --browser chromium
  - regression: uv run pytest tests/unit tests/integration tests/contract tests/e2e -m "not slow" && uv run pytest -m regression
  - lint: uv run ruff check . && uv run ruff format --check . && uv run mypy && uv run lint-imports
- max_review_iterations: 3
