---
name: python-testing
description: Production-grade Python testing — TDD workflow, test pyramid, determinism rules, pytest patterns (fixtures, mocking, parametrize, async), domain-specific testing (DB, Airflow, data pipelines, ML, messaging, service lifecycle), contract tests, and CI quality gates. Use when writing, reviewing, stabilizing, or designing tests for any Python backend/data code.
compatibility: opencode
metadata:
  audience: "senior"
  language: "python"
  domains: "testing,backend,data-engineering,airflow,messaging,ml"
  primary_tools: "pytest,ruff,basedpyright"
---

# Python Testing (Production-Grade)

This is a **task skill** for a dedicated testing agent: it defines *how to decide what to test*, *how to write production-grade tests*, and *how to keep test suites healthy* for Python backend and data systems.

## When to use

- Writing new Python code (follow TDD for non-trivial logic).
- Adding or changing API, DB, Airflow, pipeline, messaging, or service lifecycle contracts.
- Reviewing or stabilizing existing test suites (flaky, slow, brittle).
- Designing test strategy for a new module or service.
- Defining or updating CI quality gates.

Do not use this skill for front-end-only testing or non-Python test frameworks.

## Agent workflow

1. **Classify the code under test** — domain logic / API endpoint / DB operation / Airflow DAG / data pipeline / ML workload / messaging handler / service lifecycle.
2. **Choose test type(s)** using the test pyramid and the decision framework (load domain-specific reference for domain code).
3. **Apply determinism rules** (non-negotiable).
4. **Write tests** using the appropriate pytest patterns from this skill or domain reference.
5. **Validate** — run lint + type-check + pytest; confirm determinism and isolation.
6. **Close the loop** — update checklist, flag any uncovered invariants or risks.

## Testing philosophy

### TDD cycle (default for non-trivial logic)

1. **RED**: Write a failing test that describes the desired behavior.
2. **GREEN**: Write minimal code to make the test pass.
3. **REFACTOR**: Improve code while keeping tests green.

```python
# RED — test describes behavior before implementation exists
def test_calculate_discount_applies_tier_rate():
    result = calculate_discount(order_total=500, tier="gold")
    assert result == 50.0

# GREEN — minimal implementation
def calculate_discount(order_total: float, tier: str) -> float:
    rates = {"gold": 0.10, "silver": 0.05}
    return order_total * rates.get(tier, 0.0)
```

TDD applies to business logic, domain rules, and data transformations. For thin orchestration/wiring (e.g., Airflow DAG definitions), prefer structural and integration tests instead.

### Test pyramid

1. **Unit tests (majority)** — Pure functions and domain logic; no external systems.
2. **Integration tests (some)** — DB queries, transactions, migrations, operator integration; run in CI with containers.
3. **Contract tests (as needed)** — API payloads, event schemas, backward compatibility.
4. **End-to-end tests (few, high value)** — Only for critical flows where integration risk is high.

### Determinism rules (non-negotiable)

- No real network calls in unit tests.
- Control time (freeze or inject a clock); never depend on wall-clock.
- Control randomness (seed or avoid).
- No shared mutable state between tests.
- Tests must pass when run in any order.

### Coverage strategy

- **Target**: 80%+ for core logic modules.
- **Critical paths**: 100% where failures have high impact (money, safety, data loss, security).
- Coverage targets do **not** justify meaningless tests for thin wiring, framework glue, or generated code.
- Optimize for defect prevention, not coverage numbers.

## pytest patterns (essential)

### Fixtures

```python
@pytest.fixture
def sample_user():
    return User(id=1, name="Alice", role="member")

# Setup + teardown with yield
@pytest.fixture
def database():
    db = Database(":memory:")
    db.create_tables()
    yield db
    db.close()

# Scoped fixture (once per module)
@pytest.fixture(scope="module")
def expensive_resource():
    resource = build_expensive_resource()
    yield resource
    resource.cleanup()
```

Keep fixtures **small and explicit** — prefer local fixtures over "god fixtures". Avoid fixtures with hidden side effects (disk/network/database).

#### conftest.py for shared fixtures

```python
# tests/conftest.py
@pytest.fixture
def client():
    app = create_app(testing=True)
    with app.test_client() as client:
        yield client

@pytest.fixture
def auth_headers(client):
    response = client.post("/api/login", json={"username": "test", "password": "test"})
    token = response.json["token"]
    return {"Authorization": f"Bearer {token}"}
```

### Parametrization

```python
@pytest.mark.parametrize("input_email,expected", [
    ("valid@email.com", True),
    ("no-at-sign", False),
    ("@missing-local.com", False),
], ids=["valid", "missing-at", "missing-local"])
def test_email_validation(input_email, expected):
    assert is_valid_email(input_email) is expected
```

Use `ids` for readable test output. Parametrize boundary conditions, error cases, and equivalence classes.

### Mocking strategy

Mock at **system boundaries** — HTTP clients, message brokers, S3/storage, external APIs. Avoid mocking pure domain functions; test them directly.

```python
from unittest.mock import patch, Mock

# Mock external API call
@patch("mypackage.services.external_api_call")
def test_service_handles_api_failure(api_mock):
    api_mock.side_effect = ConnectionError("timeout")

    with pytest.raises(ServiceUnavailableError):
        process_order(order_id="123")

    api_mock.assert_called_once()

# Use autospec to catch API misuse
@patch("mypackage.repos.UserRepository", autospec=True)
def test_user_creation(repo_mock):
    repo_mock.return_value.save.return_value = User(id=1, name="Alice")

    service = UserService(repo_mock.return_value)
    user = service.create_user(name="Alice")

    assert user.name == "Alice"
    repo_mock.return_value.save.assert_called_once()
```

### Async testing

```python
import pytest

@pytest.mark.asyncio
async def test_async_handler():
    result = await process_event({"type": "order_created", "id": "abc"})
    assert result.status == "processed"

@pytest.mark.asyncio
@patch("mypackage.async_api_call")
async def test_async_with_mock(api_mock):
    api_mock.return_value = {"status": "ok"}
    result = await my_async_function()
    api_mock.assert_awaited_once()
    assert result["status"] == "ok"
```

### Markers and test selection

```python
@pytest.mark.slow
def test_heavy_computation():
    ...

@pytest.mark.integration
def test_db_transaction():
    ...
```

```bash
pytest -m "not slow"           # skip slow tests
pytest -m "integration"        # run only integration
pytest -m "unit and not slow"  # fast unit tests only
```

### Testing API endpoints

```python
def test_create_user(client):
    response = client.post("/api/users", json={
        "name": "Alice",
        "email": "alice@example.com"
    })
    assert response.status_code == 201
    assert response.json["name"] == "Alice"

def test_create_user_rejects_invalid_email(client):
    response = client.post("/api/users", json={
        "name": "Alice",
        "email": "not-an-email"
    })
    assert response.status_code == 422
```

## Test organization

### Directory structure

```
tests/
├── conftest.py
├── unit/
│   ├── test_models.py
│   ├── test_services.py
│   └── test_utils.py
├── integration/
│   ├── test_api.py
│   └── test_database.py
└── contract/
    └── test_api_schemas.py
```

### Naming conventions

Test names should describe behavior:
- `test_creating_order_is_idempotent_when_same_key_reused`
- `test_backfill_skips_already_materialized_partitions`
- `test_expired_token_returns_401`

## CI quality gates

Always run:
- `ruff` (lint + format)
- `basedpyright` (or configured type checker)
- `pytest -q` for unit tests

Additionally (depending on repo):
- Integration tests (DB/services) on main branch or per PR if fast enough.
- DAG parse tests (always for DAG repos).
- Contract checks for public APIs/events.

## pytest configuration

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "--strict-markers",
    "--cov=mypackage",
    "--cov-report=term-missing",
]
markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests",
]
```

## Reference routing

This skill covers testing strategy, essential pytest patterns, and CI standards. Domain-specific recipes and advanced pytest mechanics live in dedicated references.

### Domain-specific testing patterns (DB, Airflow, data pipelines, ML, messaging, service lifecycle)
- @.opencode/skills/python-testing/references/python-domain-testing-patterns.md

Load when the code under test touches a specific domain — what to test, fixture patterns, and integration test recipes for each domain.

### Detailed pytest patterns (fixtures, mocking, assertions, async, CLI)
- @.opencode/skills/python-testing/references/python-pytest-patterns.md

Load when writing or reviewing tests and you need specific pytest recipes (advanced fixtures, parametrized fixtures, mocking properties/context managers, side-effect testing, async fixtures, CLI options).

## Best practices

### Do

- Follow TDD for non-trivial logic (red → green → refactor).
- Test one behavior per test function.
- Use descriptive names: `test_user_login_with_expired_token_returns_401`.
- Use fixtures to eliminate duplication; keep them small and local.
- Mock at system boundaries only.
- Test edge cases: empty inputs, None, boundary conditions, concurrent access.
- Use parametrize for equivalence classes and boundary values.
- Use `autospec=True` when mocking to catch API drift.

### Don't

- Don't test implementation details — test behavior and contracts.
- Don't share mutable state between tests.
- Don't mock domain logic — test it directly.
- Don't write tests that depend on execution order.
- Don't use `time.sleep` in tests — use clock injection or freeze.
- Don't ignore flaky tests — fix determinism immediately.
- Don't chase coverage numbers with meaningless tests.

## Checklist

- Unit tests cover core invariants, idempotency paths, and boundary conditions.
- Integration tests cover DB/transaction/migration risks where applicable.
- Contract tests protect API/event compatibility when boundaries change.
- DAG parse/structure tests are present for Airflow repositories.
- Messaging tests cover serialization, handler idempotency, DLQ routing, and schema compatibility.
- Service lifecycle tests cover health endpoints, startup/shutdown hooks, and graceful drain.
- CI gates run lint, type-check, and deterministic pytest suites.
- No flaky tests in the suite; determinism rules are enforced.

## Failure modes

- **Flaky tests**: isolate time/network; remove sleeps; control randomness; fix shared state.
- **Slow tests**: split integration from unit; cache heavy fixtures; shrink datasets; use markers.
- **Brittle tests**: assert behavior and contracts, not internal call order or mock call counts.
- **Low-value tests**: testing wiring/glue instead of business logic; chasing coverage instead of correctness.
- **Missing coverage**: no tests for invariants, idempotency, or error paths in critical flows.
