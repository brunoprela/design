# CI/CD Patterns

## Context & Problem

Without continuous integration, boundary violations, type errors, broken tests, and schema drift accumulate silently until someone tries to deploy. Without continuous delivery, deployments are manual, error-prone, and infrequent — which makes them riskier.

CI/CD automates the verification and delivery pipeline. Every commit is tested, every merge to main is deployable, and the time between writing code and running it in production is minimized.

## Design Decisions

### Pipeline Stages

```mermaid
graph LR
    A[Commit] --> B[Lint & Type Check]
    B --> C[Unit Tests]
    C --> D[Integration Tests]
    D --> E[Build]
    E --> F[Deploy to Staging]
    F --> G[Deploy to Production]

    style B fill:#1a3a5c,stroke:#333
    style C fill:#1a3a5c,stroke:#333
    style D fill:#1a3a5c,stroke:#333
```

| Stage | Speed | What It Catches |
|---|---|---|
| Lint & type check | Seconds | Style, type errors, import violations |
| Module boundary check | Seconds | Tach violations |
| Unit tests | Seconds | Business logic bugs |
| Integration tests | Minutes | DB queries, Kafka serialization, API contracts |
| Build | Seconds | Packaging errors |
| Deploy to staging | Minutes | Configuration, environment-specific issues |

### GitHub Actions Workflow

```yaml
# .github/workflows/ci.yml

name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install ruff mypy tach
      - run: ruff check .
      - run: ruff format --check .
      - run: mypy app/ --strict
      - run: tach check

  test-unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -e ".[test]"
      - run: pytest tests/unit -x --timeout=30

  test-integration:
    runs-on: ubuntu-latest
    needs: [lint, test-unit]
    services:
      postgres:
        image: timescale/timescaledb:latest-pg16
        env:
          POSTGRES_DB: test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -e ".[test]"
      - run: pytest tests/integration -x --timeout=120
        env:
          DATABASE_URL: postgresql+asyncpg://test:test@localhost:5432/test

  openapi-check:
    runs-on: ubuntu-latest
    needs: [lint]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -e .
      - run: python scripts/export_openapi.py
      - run: diff openapi.json openapi.committed.json
```

### What Each Check Enforces

| Check | Tool | Enforces |
|---|---|---|
| Code formatting | `ruff format` | Consistent style, no debates |
| Linting | `ruff check` | Code quality, import order, unused vars |
| Type checking | `mypy --strict` | Type safety, Protocol compliance |
| Module boundaries | `tach check` | No cross-module import violations |
| Unit tests | `pytest tests/unit` | Business logic correctness |
| Integration tests | `pytest tests/integration` | DB, Kafka, API correctness |
| OpenAPI spec | `diff openapi.json` | API contract not silently changed |
| Secret detection | `detect-secrets` | No credentials in code |
| Migration check | `alembic check` | No unapplied migrations |

### Branch Strategy

```
main        ← always deployable
  └── feature/add-risk-module     ← short-lived, PR to main
  └── feature/openfga-documents   ← short-lived, PR to main
```

- **main** is always deployable — every merge triggers CI, passing CI means deployable
- **Feature branches** are short-lived (1-3 days), merged via PR with required CI checks
- No long-lived develop/staging branches — they drift and create merge pain

### Deployment Strategy

```mermaid
graph TD
    PR[PR Merged to Main] --> CI[CI Pipeline Passes]
    CI --> BUILD[Build Container Image]
    BUILD --> STAGING[Deploy to Staging]
    STAGING --> SMOKE[Smoke Tests]
    SMOKE --> GATE{Manual Approval?}
    GATE -->|yes| PROD[Deploy to Production]
    GATE -->|auto| PROD
```

**For staging:** automatic deployment on merge to main.
**For production:** either automatic (if you trust the test suite) or with a manual approval gate.

### Makefile for Developer Workflow

```makefile
.PHONY: lint test test-unit test-integration migrate seed

lint:
	ruff check .
	ruff format --check .
	mypy app/ --strict
	tach check

test: test-unit test-integration

test-unit:
	pytest tests/unit -x --timeout=30

test-integration:
	pytest tests/integration -x --timeout=120

migrate:
	alembic -n positions upgrade head
	alembic -n market_data upgrade head
	alembic -n risk upgrade head

seed:
	python scripts/seed_data.py

check: lint test
	@echo "All checks passed"
```

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Flaky tests | Timing dependencies, shared state | Isolate tests, use testcontainers, no sleep |
| Slow CI | Too many integration tests, no caching | Cache pip dependencies, parallelize jobs, run unit tests first |
| CI passes, prod fails | Environment difference | Integration tests use real infrastructure (testcontainers), staging mirrors prod |
| Merge conflicts | Long-lived branches | Short-lived branches, frequent merges to main |
| Skipped checks | Developer force-merges | Branch protection rules, required status checks |

## Related Documents

- [Tach Enforcement](../patterns/modularity/tach-enforcement.md) — module boundary checks in CI
- [Contract Testing](../patterns/testing/contract-testing.md) — API contract verification
- [OpenAPI Contracts](../patterns/api/openapi-contracts.md) — spec drift detection
- [Secret Management](secret-management.md) — CI/CD secret handling
