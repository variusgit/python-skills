---
name: python-pytest-patterns
description: Practical pytest cookbook — assertions, advanced fixtures, parametrization, mocking techniques, async patterns, exception testing, side-effect testing, test classes, and CLI reference. Use when writing or reviewing tests and need specific pytest mechanics.
---

# pytest Patterns (Cookbook)

Practical pytest recipes organized by category. This document covers **how** to use pytest mechanics. For **what** to test, **when**, and testing strategy, see the parent `SKILL.md`.

## When to use

- Writing tests and need a specific pytest pattern (fixture scopes, parametrized fixtures, mocking properties, etc.).
- Reviewing tests for correctness of pytest usage.
- Onboarding to pytest in a project.

## Assertions

### Equality and comparison

```python
assert result == expected
assert result != unexpected
assert result > 0
assert 0 <= result <= 100
```

### Truthiness and identity

```python
assert result              # truthy
assert not result          # falsy
assert result is True      # exactly True
assert result is None      # exactly None
```

### Membership and type

```python
assert item in collection
assert item not in collection
assert isinstance(result, str)
```

### Approximate comparison (floats)

```python
assert result == pytest.approx(3.14, abs=1e-2)
assert result == pytest.approx(expected, rel=1e-3)
```

### Exception testing

```python
with pytest.raises(ValueError):
    do_something_invalid()

# Match exception message with regex
with pytest.raises(ValueError, match=r"invalid input.*"):
    validate_input("bad")

# Inspect exception attributes
with pytest.raises(CustomError) as exc_info:
    raise CustomError("msg", code=400)

assert exc_info.value.code == 400
assert "msg" in str(exc_info.value)
```

### Warnings

```python
with pytest.warns(DeprecationWarning, match="old_func"):
    old_func()
```

## Fixtures (advanced)

For basic fixture usage and conftest.py, see `../SKILL.md`. This section covers advanced patterns.

### Scopes

```python
# function (default) — runs for each test
@pytest.fixture
def fresh_data():
    return {"count": 0}

# module — runs once per test module
@pytest.fixture(scope="module")
def db_connection():
    conn = Database.connect()
    yield conn
    conn.close()

# session — runs once per entire test session
@pytest.fixture(scope="session")
def expensive_resource():
    resource = build_expensive_resource()
    yield resource
    resource.cleanup()
```

Choose the narrowest scope that avoids unnecessary re-creation. Wider scopes (module/session) are for genuinely expensive setup — never for convenience when tests share mutable state.

### Parametrized fixtures

```python
@pytest.fixture(params=["sqlite", "postgresql"], ids=["sqlite", "pg"])
def db(request):
    if request.param == "sqlite":
        db = Database(":memory:")
    else:
        db = Database("postgresql://localhost/test")
    yield db
    db.close()

def test_query_returns_results(db):
    """Runs twice — once per database backend."""
    result = db.query("SELECT 1")
    assert result is not None
```

### Autouse fixtures

```python
@pytest.fixture(autouse=True)
def reset_config():
    """Runs before every test in this module/class automatically."""
    Config.reset()
    yield
    Config.cleanup()

def test_something():
    # reset_config already ran — no need to request it
    assert Config.get("debug") is False
```

Use sparingly. Autouse is appropriate for mandatory setup/teardown (reset state, seed RNG, freeze time). Avoid when the fixture's effect is non-obvious — explicit is better.

### Composing multiple fixtures

```python
@pytest.fixture
def user():
    return User(id=1, name="Alice", role="member")

@pytest.fixture
def admin():
    return User(id=2, name="Bob", role="admin")

@pytest.fixture
def team(user, admin):
    return Team(members=[user, admin])

def test_team_has_admin(team):
    assert any(m.role == "admin" for m in team.members)
```

### Lazy / factory fixtures

```python
@pytest.fixture
def make_user():
    """Factory fixture — call it to create users with custom attributes."""
    created = []

    def _make(name: str = "Alice", role: str = "member") -> User:
        user = User(name=name, role=role)
        created.append(user)
        return user

    yield _make

    for u in created:
        u.cleanup()

def test_multiple_users(make_user):
    alice = make_user(name="Alice", role="admin")
    bob = make_user(name="Bob")
    assert alice.role != bob.role
```

## Parametrization (advanced)

For basic `@pytest.mark.parametrize` with IDs, see `../SKILL.md`.

### Multiple parameter axes

```python
@pytest.mark.parametrize("a,b,expected", [
    (2, 3, 5),
    (0, 0, 0),
    (-1, 1, 0),
])
def test_add(a, b, expected):
    assert add(a, b) == expected
```

### Stacking parametrize (cartesian product)

```python
@pytest.mark.parametrize("x", [1, 2])
@pytest.mark.parametrize("y", [10, 20])
def test_multiply(x, y):
    """Runs 4 times: (1,10), (1,20), (2,10), (2,20)."""
    assert multiply(x, y) == x * y
```

### Conditional skip inside parametrize

```python
@pytest.mark.parametrize("backend", [
    "fast",
    pytest.param("gpu", marks=pytest.mark.skipif(
        not torch.cuda.is_available(), reason="No GPU"
    )),
])
def test_compute(backend):
    result = compute(backend=backend)
    assert result is not None
```

## Mocking (advanced)

For mocking strategy (where to mock) and `autospec`, see `../SKILL.md`.

### Configuring return values

```python
from unittest.mock import patch, Mock

@patch("mypackage.db.get_connection")
def test_uses_connection(conn_mock):
    conn_mock.return_value = Mock()
    conn_mock.return_value.execute.return_value = [{"id": 1}]

    result = fetch_users()
    assert result == [{"id": 1}]
    conn_mock.return_value.execute.assert_called_once()
```

### Side effects (sequential returns)

```python
@patch("mypackage.api.fetch")
def test_retry_logic(fetch_mock):
    fetch_mock.side_effect = [
        ConnectionError("timeout"),
        ConnectionError("timeout"),
        {"status": "ok"},
    ]

    result = fetch_with_retry(max_retries=3)
    assert result == {"status": "ok"}
    assert fetch_mock.call_count == 3
```

### Mocking context managers

```python
from unittest.mock import patch, mock_open

@patch("builtins.open", mock_open(read_data="file content"))
def test_read_config(mock_file):
    result = read_config("config.yaml")
    mock_file.assert_called_once_with("config.yaml", "r")
    assert result == "file content"
```

### Mocking properties

```python
from unittest.mock import PropertyMock

@pytest.fixture
def mock_config():
    config = Mock()
    type(config).debug = PropertyMock(return_value=True)
    type(config).api_url = PropertyMock(return_value="https://api.test")
    return config

def test_debug_mode(mock_config):
    assert mock_config.debug is True
```

### Mocking with spec (catch API drift)

```python
from unittest.mock import create_autospec

def test_service_with_spec():
    repo = create_autospec(UserRepository, instance=True)
    repo.find_by_id.return_value = User(id=1, name="Alice")

    service = UserService(repo)
    user = service.get_user(1)

    assert user.name == "Alice"
    repo.find_by_id.assert_called_once_with(1)
    # repo.nonexistent_method() would raise AttributeError
```

## Async testing (advanced)

For basic `@pytest.mark.asyncio` usage, see `../SKILL.md`.

### Async fixtures

```python
import pytest

@pytest.fixture
async def async_client():
    app = create_app()
    async with app.test_client() as client:
        yield client

@pytest.mark.asyncio
async def test_health(async_client):
    response = await async_client.get("/health")
    assert response.status_code == 200
```

### Async context manager testing

```python
@pytest.mark.asyncio
async def test_db_session_lifecycle():
    async with get_session() as session:
        result = await session.execute("SELECT 1")
        assert result is not None
    # session is closed here — verify cleanup if needed
```

## Side-effect testing

### tmp_path (preferred)

```python
def test_write_report(tmp_path):
    output = tmp_path / "report.csv"
    generate_report(output)

    assert output.exists()
    content = output.read_text()
    assert "header" in content
    # tmp_path is automatically cleaned up
```

### tmp_path with subdirectories

```python
def test_nested_output(tmp_path):
    data_dir = tmp_path / "data" / "output"
    data_dir.mkdir(parents=True)

    output_file = data_dir / "result.json"
    output_file.write_text('{"status": "ok"}')

    assert output_file.exists()
```

### tempfile (when tmp_path doesn't fit)

```python
import tempfile
import os

def test_process_file():
    with tempfile.NamedTemporaryFile(mode="w", suffix=".csv", delete=False) as f:
        f.write("id,name\n1,Alice\n")
        path = f.name

    try:
        result = process_csv(path)
        assert result["row_count"] == 1
    finally:
        os.unlink(path)
```

### Environment variables

```python
import os
from unittest.mock import patch

@patch.dict(os.environ, {"API_KEY": "test-key", "DEBUG": "true"})
def test_config_from_env():
    config = load_config()
    assert config.api_key == "test-key"
    assert config.debug is True

@patch.dict(os.environ, {}, clear=True)
def test_missing_env_raises():
    with pytest.raises(ValueError, match="API_KEY"):
        load_config()
```

## Test classes

```python
class TestUserService:
    @pytest.fixture(autouse=True)
    def setup(self):
        self.service = UserService()
        self.repo = Mock(spec=UserRepository)
        self.service.repo = self.repo

    def test_create_user(self):
        self.repo.save.return_value = User(id=1, name="Alice")
        user = self.service.create_user("Alice")
        assert user.name == "Alice"
        self.repo.save.assert_called_once()

    def test_get_nonexistent_user_raises(self):
        self.repo.find_by_id.return_value = None
        with pytest.raises(UserNotFoundError):
            self.service.get_user(999)
```

Use test classes to group related tests that share setup. Prefer module-level functions when tests don't need shared setup.

## pytest CLI reference

```bash
# Run all tests
pytest

# Verbose output
pytest -v

# Stop on first failure
pytest -x

# Stop after N failures
pytest --maxfail=3

# Run last failed tests only
pytest --lf

# Run tests matching keyword expression
pytest -k "user and not slow"

# Run specific file / test
pytest tests/test_users.py
pytest tests/test_users.py::test_create_user
pytest tests/test_users.py::TestUserService::test_create_user

# Run by marker
pytest -m "not slow"
pytest -m "integration"

# Show local variables in tracebacks
pytest -l

# Drop into debugger on failure
pytest --pdb

# Coverage
pytest --cov=mypackage --cov-report=term-missing

# Parallel execution (requires pytest-xdist)
pytest -n auto
```

## Quick reference

| Pattern | Syntax |
|---------|--------|
| Assert exception | `with pytest.raises(Error, match=...)` |
| Assert warning | `with pytest.warns(Warning)` |
| Fixture with teardown | `yield` inside `@pytest.fixture` |
| Scoped fixture | `@pytest.fixture(scope="module")` |
| Parametrized fixture | `@pytest.fixture(params=[...])` |
| Autouse fixture | `@pytest.fixture(autouse=True)` |
| Factory fixture | fixture returns a callable |
| Parametrize test | `@pytest.mark.parametrize("a,b", [...])` |
| Parametrize + skip | `pytest.param(..., marks=pytest.mark.skip)` |
| Mock function | `@patch("module.func")` |
| Mock property | `PropertyMock` on `type(mock)` |
| Mock file open | `@patch("builtins.open", mock_open(...))` |
| Mock env vars | `@patch.dict(os.environ, {...})` |
| Sequential side effects | `mock.side_effect = [val1, val2, ...]` |
| Temp directory | `tmp_path` fixture (pathlib.Path) |
| Async test | `@pytest.mark.asyncio` |
| Parallel run | `pytest -n auto` (xdist) |
