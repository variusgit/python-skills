---
name: python-domain-testing-patterns
description: Domain-specific testing recipes — what to test and how for DB/PostgreSQL, Airflow DAGs, data pipelines, ML workloads, messaging (serialization, handlers, producers, DLQ), and service lifecycle (health endpoints, startup/shutdown, graceful drain, config, DI). Use when writing tests for domain-specific code.
---

# Domain-Specific Testing Patterns

What to test and how to test it for each domain: DB, Airflow, data pipelines, ML, messaging, and service lifecycle.

## When to use

Load this reference when the code under test touches a specific domain (database, Airflow, data pipeline, ML, messaging, or service lifecycle). The main SKILL.md covers testing philosophy, strategy, pytest essentials, and CI standards — this document provides **domain-specific recipes**.

---

## Unit tests — what to test

- Invariants (success + failure cases).
- State transitions and business rules.
- Idempotency behavior (same input twice → same persisted effect).
- Boundary validation and normalization (types, time zones, precision).
- Edge cases: empty inputs, None values, boundary conditions.

## Unit tests — what NOT to test

- Third-party libraries (trust them to work).
- Framework wiring with no custom logic.
- One-off glue code with no branching (prefer integration tests).

## Integration tests — when to use

- SQL queries are non-trivial.
- Transaction boundaries matter.
- Uniqueness/locking/concurrency behavior matters.
- Migrations or backfills are risky.

### DB integration (PostgreSQL)

- Use ephemeral Postgres (container) in CI.
- Apply migrations in test setup.
- Validate constraints and indexes for critical queries.
- Test both "happy path" and expected constraint violations.

```python
@pytest.fixture
def db_session():
    session = Session(bind=engine)
    session.begin_nested()
    yield session
    session.rollback()
    session.close()

def test_unique_constraint_prevents_duplicate_orders(db_session):
    order = Order(idempotency_key="abc-123", amount=100)
    db_session.add(order)
    db_session.flush()

    duplicate = Order(idempotency_key="abc-123", amount=200)
    db_session.add(duplicate)
    with pytest.raises(IntegrityError):
        db_session.flush()
```

## Contract tests — when to add

- API response shape changes.
- Error codes or semantics change.
- Event schema changes (producer/consumer compatibility).

Contracts should be machine-checkable: JSON schema / OpenAPI snapshots, schema versioning rules, compatibility test suites.

## Airflow testing

### 1) DAG import/parse tests (must-have)

DAG modules must be importable with **no I/O**.

```python
from airflow.models import DagBag

@pytest.fixture
def dagbag():
    return DagBag(dag_folder="dags/", include_examples=False)

def test_no_import_errors(dagbag):
    assert len(dagbag.import_errors) == 0, f"DAG import errors: {dagbag.import_errors}"

def test_expected_dags_present(dagbag):
    expected = {"etl_customers", "etl_orders", "ml_training"}
    assert expected.issubset(dagbag.dag_ids)
```

### 2) DAG structure tests

Validate: dependencies, pools/queues for expensive tasks, and graph integrity.

```python
def test_task_dependencies(dagbag):
    dag = dagbag.get_dag("etl_customers")
    extract = dag.get_task("extract")
    assert "transform" in [t.task_id for t in extract.downstream_list]

def test_no_dag_cycles(dagbag):
    for dag_id, dag in dagbag.dags.items():
        assert dag.test_cycle() is None, f"Cycle detected in {dag_id}"
```

### 3) Task/business logic tests

Keep business logic **out of DAG files** — put it in importable modules under `dags/libs/`, unit test it separately, and in DAG tests only validate wiring and config.

```python
from dags.libs.customers import transform_customers

def test_transform_customers_drops_nulls():
    raw = [{"id": 1, "name": "Alice"}, {"id": 2, "name": None}]
    result = transform_customers(raw)
    assert len(result) == 1
    assert result[0]["name"] == "Alice"
```

### 4) Operator/Sensor tests (as needed)

For custom operators/sensors: unit test input validation and result interpretation; integration test with a stubbed backend where meaningful.

## Data pipeline testing (batch jobs)

- Unit tests for transformations (small in-memory dataframes/samples).
- Validation tests for **schema + constraints** (nullability, uniqueness, ranges).
- For incremental loads, test watermark logic and partition selection.
- For deduplication, test keys and tie-breaking rules.

## ML workload testing

### 1) Feature computation tests

Validate point-in-time correctness and feature determinism.

```python
def test_feature_computation_excludes_future_data():
    events = [
        {"user_id": 1, "event_time": "2025-01-10", "amount": 100},
        {"user_id": 1, "event_time": "2025-01-20", "amount": 200},  # future
    ]
    features = compute_features(events, feature_date="2025-01-15")
    assert features[0]["total_amount"] == 100  # only pre-cutoff events

def test_feature_computation_is_deterministic():
    result1 = compute_features(sample_events, feature_date="2025-01-15")
    result2 = compute_features(sample_events, feature_date="2025-01-15")
    assert result1 == result2

def test_feature_schema_matches_expected():
    features = compute_features(sample_events, feature_date="2025-01-15")
    for f in features:
        UserFeatures(**f)  # Pydantic validation
```

### 2) Training reproducibility tests

```python
def test_training_is_reproducible():
    metrics1 = train_model(data_path=FIXTURE_DATA, params={"seed": 42})
    metrics2 = train_model(data_path=FIXTURE_DATA, params={"seed": 42})
    assert metrics1["f1"] == pytest.approx(metrics2["f1"], abs=1e-6)

def test_model_evaluation_against_baseline():
    metrics = train_model(data_path=FIXTURE_DATA, params=DEFAULT_PARAMS)
    assert metrics["f1"] >= 0.7, "Model performance below acceptable threshold"
```

### 3) Model serving tests

```python
def test_prediction_endpoint_returns_valid_response(client):
    response = client.post("/predict", json={
        "user_id": 1,
        "features": {"event_count": 10, "total_amount": 500.0},
    })
    assert response.status_code == 200
    data = response.json()
    assert "prediction" in data
    assert "model_version" in data
    assert isinstance(data["prediction"], float)

def test_prediction_rejects_invalid_features(client):
    response = client.post("/predict", json={
        "user_id": 1,
        "features": {},  # missing required features
    })
    assert response.status_code == 422
```

### 4) Batch inference tests

```python
def test_batch_predictions_have_no_nulls(spark):
    predictions = run_batch_inference(spark, model_path=MODEL_FIXTURE, input_path=INPUT_FIXTURE)
    null_count = predictions.filter(F.col("prediction").isNull()).count()
    assert null_count == 0

def test_batch_prediction_count_matches_input(spark):
    input_df = spark.read.parquet(INPUT_FIXTURE)
    predictions = run_batch_inference(spark, model_path=MODEL_FIXTURE, input_path=INPUT_FIXTURE)
    assert predictions.count() == input_df.count()
```

### 5) Monitoring and drift tests

```python
def test_drift_detection_flags_shifted_distribution():
    reference = np.random.normal(0, 1, 1000)
    shifted = np.random.normal(2, 1, 1000)  # clear drift
    results = compute_drift_metrics(reference, shifted, ["feature_a"])
    assert results["feature_a"]["drifted"] is True

def test_drift_detection_passes_stable_distribution():
    reference = np.random.normal(0, 1, 1000)
    stable = np.random.normal(0, 1, 1000)
    results = compute_drift_metrics(reference, stable, ["feature_a"])
    assert results["feature_a"]["drifted"] is False
```

## Messaging testing

### Principles

- Test message handlers as pure functions: serialized message in → side effects out.
- Separate transport (broker connection, ack/nack) from business logic.
- Never depend on a running broker in unit tests; use in-memory fakes or mocks.
- Integration tests with a real broker (containerized) belong in CI only.

### 1) Message serialization / schema tests

Validate that messages survive round-trip serialization and reject malformed payloads.

```python
from app.events.schemas import OrderCreatedEvent

def test_event_round_trip_serialization():
    event = OrderCreatedEvent(order_id=1, total=99.99, currency="USD")
    raw = event.model_dump_json()
    restored = OrderCreatedEvent.model_validate_json(raw)
    assert restored == event

def test_event_rejects_missing_required_fields():
    with pytest.raises(ValidationError):
        OrderCreatedEvent.model_validate({"order_id": 1})  # missing total, currency

def test_event_schema_backward_compatible():
    """Old payload (without new optional field) still deserializes."""
    old_payload = '{"order_id": 1, "total": 99.99, "currency": "USD"}'
    event = OrderCreatedEvent.model_validate_json(old_payload)
    assert event.order_id == 1
```

### 2) Handler / consumer logic tests

Test handlers as plain functions with injected dependencies.

```python
@pytest.fixture
def mock_repo():
    return Mock(spec=OrderRepository)

def test_order_created_handler_persists_order(mock_repo):
    event = OrderCreatedEvent(order_id=42, total=100.0, currency="USD")
    handle_order_created(event, repo=mock_repo)
    mock_repo.save.assert_called_once()
    saved = mock_repo.save.call_args[0][0]
    assert saved.id == 42
    assert saved.total == 100.0

def test_handler_is_idempotent(mock_repo):
    event = OrderCreatedEvent(order_id=42, total=100.0, currency="USD")
    mock_repo.exists.return_value = True
    handle_order_created(event, repo=mock_repo)
    mock_repo.save.assert_not_called()

def test_handler_rejects_negative_total(mock_repo):
    event = OrderCreatedEvent(order_id=42, total=-10.0, currency="USD")
    with pytest.raises(ValueError, match="negative total"):
        handle_order_created(event, repo=mock_repo)
```

### 3) Producer tests

Verify that produced messages have correct routing, schema, and headers.

```python
def test_producer_publishes_correct_event(mock_publisher):
    publish_order_event(order_id=1, total=50.0, publisher=mock_publisher)
    mock_publisher.publish.assert_called_once()
    msg = mock_publisher.publish.call_args[0][0]
    assert msg["type"] == "order.created"
    assert msg["payload"]["order_id"] == 1

def test_producer_sets_idempotency_key(mock_publisher):
    publish_order_event(order_id=1, total=50.0, publisher=mock_publisher)
    msg = mock_publisher.publish.call_args[0][0]
    assert "idempotency_key" in msg["metadata"]
```

### 4) Dead letter / retry behavior tests

```python
def test_failed_message_routes_to_dlq(mock_publisher, mock_repo):
    mock_repo.save.side_effect = DatabaseError("connection lost")
    event = OrderCreatedEvent(order_id=42, total=100.0, currency="USD")

    with pytest.raises(DatabaseError):
        handle_order_created(event, repo=mock_repo, publisher=mock_publisher)

    mock_publisher.send_to_dlq.assert_called_once()

def test_transient_failure_is_retryable():
    handler = RetryableHandler(max_retries=3)
    handler.process = Mock(side_effect=[TransientError, TransientError, "ok"])
    result = handler.execute(sample_message)
    assert result == "ok"
    assert handler.process.call_count == 3
```

### 5) Integration tests (broker in container)

```python
@pytest.fixture(scope="session")
def broker():
    """Spin up a containerized message broker for integration tests."""
    # Use testcontainers or docker-compose
    yield broker_connection
    broker_connection.close()

@pytest.mark.integration
def test_end_to_end_publish_consume(broker):
    published = OrderCreatedEvent(order_id=99, total=200.0, currency="EUR")
    producer = Producer(broker)
    consumer = Consumer(broker, group_id="test-group")

    producer.publish(published)
    received = consumer.poll(timeout=5.0)

    assert received is not None
    assert received.order_id == published.order_id
```

## Service lifecycle testing

### Principles

- Test startup/shutdown as deterministic sequences, not timing-dependent events.
- Verify graceful shutdown: in-flight requests complete, connections drain, health probes reflect state.
- Test health/readiness endpoints as first-class contracts.
- Use dependency injection to control and observe lifecycle hooks.

### 1) Health and readiness endpoint tests

```python
def test_liveness_returns_200(client):
    response = client.get("/health/live")
    assert response.status_code == 200

def test_readiness_returns_200_when_deps_healthy(client):
    response = client.get("/health/ready")
    assert response.status_code == 200
    body = response.json()
    assert body["database"] == "ok"
    assert body["cache"] == "ok"

def test_readiness_returns_503_when_db_unavailable(client, monkeypatch):
    monkeypatch.setattr("app.deps.db_pool.check", Mock(side_effect=ConnectionError))
    response = client.get("/health/ready")
    assert response.status_code == 503
```

### 2) Startup hook tests

```python
def test_startup_initializes_connection_pool():
    app = create_app(config=test_config)
    with app_lifespan(app) as state:
        assert state.db_pool is not None
        assert state.db_pool.size >= 1

def test_startup_runs_migrations_if_configured():
    config = test_config.copy()
    config["run_migrations_on_start"] = True
    app = create_app(config=config)
    with app_lifespan(app) as state:
        assert state.migrations_applied is True

def test_startup_fails_fast_on_missing_config():
    config = {}  # missing required keys
    with pytest.raises(ConfigurationError):
        create_app(config=config)
```

### 3) Graceful shutdown tests

```python
@pytest.mark.asyncio
async def test_shutdown_drains_in_flight_requests():
    app = create_app(config=test_config)
    async with AsyncClient(app=app, base_url="http://test") as client:
        slow_task = asyncio.create_task(client.get("/slow-endpoint"))
        await asyncio.sleep(0.1)

        shutdown_task = asyncio.create_task(app.shutdown())
        response = await slow_task
        assert response.status_code == 200

        await shutdown_task

async def test_shutdown_closes_db_pool():
    app = create_app(config=test_config)
    async with app_lifespan(app) as state:
        pool = state.db_pool
    assert pool.closed is True

async def test_shutdown_unregisters_from_service_discovery(mock_registry):
    app = create_app(config=test_config, registry=mock_registry)
    async with app_lifespan(app):
        pass
    mock_registry.deregister.assert_called_once()
```

### 4) Configuration and environment tests

```python
def test_config_loads_from_environment(monkeypatch):
    monkeypatch.setenv("DATABASE_URL", "postgresql://localhost/test")
    monkeypatch.setenv("LOG_LEVEL", "DEBUG")
    config = load_config()
    assert config.database_url == "postgresql://localhost/test"
    assert config.log_level == "DEBUG"

def test_config_uses_defaults_for_optional_values(monkeypatch):
    monkeypatch.setenv("DATABASE_URL", "postgresql://localhost/test")
    monkeypatch.delenv("LOG_LEVEL", raising=False)
    config = load_config()
    assert config.log_level == "INFO"  # default

def test_config_rejects_invalid_database_url(monkeypatch):
    monkeypatch.setenv("DATABASE_URL", "not-a-url")
    with pytest.raises(ConfigurationError, match="DATABASE_URL"):
        load_config()
```

### 5) Dependency injection / wiring tests

```python
def test_service_receives_injected_dependencies():
    repo = FakeOrderRepository()
    notifier = FakeNotifier()
    service = OrderService(repo=repo, notifier=notifier)
    service.create_order(order_data)
    assert repo.saved_count == 1
    assert notifier.sent_count == 1

def test_service_works_with_real_dependencies_in_integration(db_session):
    repo = SQLOrderRepository(db_session)
    notifier = LogNotifier()
    service = OrderService(repo=repo, notifier=notifier)
    service.create_order(order_data)
    assert db_session.query(Order).count() == 1
```
