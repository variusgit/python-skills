---
name: python-ml-engineering
description: ML engineering practices for feature jobs, training pipelines, model serving, batch inference, experiment tracking, and model monitoring — built on PySpark, Airflow, FastAPI, and S3. Use when implementing ML workloads within the production data platform.
---

# ML Engineering (Production Practices)

Practices for implementing ML workloads that are reproducible, operable, and integrated with the production data platform.

Read `./python-best-practices.md` first, then apply this document for ML-specific implementation patterns.

## When to use

- Building feature engineering jobs (PySpark).
- Implementing training pipelines (Airflow + Python).
- Building model serving endpoints (FastAPI) or batch inference jobs (PySpark).
- Integrating experiment tracking (MLflow).
- Implementing model monitoring and drift detection.

## Fixed stack integration

ML workloads are built on the same stack as data jobs:

| Component | Stack tool | Reference |
|-----------|-----------|-----------|
| Feature computation | PySpark jobs | `./python-data-jobs.md` |
| Orchestration | Airflow DAGs | `./python-airflow-patterns.md` |
| Feature/artifact storage | S3 (Parquet) | `./python-data-jobs.md` |
| Online feature store | PostgreSQL (or Redis as add-on with ADR) | `./python-postgresql.md` |
| Model serving (online) | FastAPI | `./python-services-api.md` |
| Batch inference | PySpark | `./python-data-jobs.md` |

ML libraries (scikit-learn, XGBoost, PyTorch, etc.) are add-ons — choose per use case, pin versions, document in ADR if non-trivial.

## Feature engineering

### Feature job structure

Feature jobs are PySpark jobs with ML-specific constraints. Follow the canonical job skeleton from `./python-data-jobs.md` and add:

```python
from pyspark.sql import SparkSession, DataFrame
import pyspark.sql.functions as F

def compute_features(
    events: DataFrame,
    feature_date: str,
) -> DataFrame:
    return (
        events
        .filter(F.col("event_time") < feature_date)
        .groupBy("user_id")
        .agg(
            F.count("*").alias("event_count_lifetime"),
            F.sum("amount").alias("total_amount"),
            F.max("event_time").alias("last_event_time"),
            F.countDistinct("category").alias("distinct_categories"),
        )
        .withColumn("feature_date", F.lit(feature_date))
    )

def main(spark: SparkSession, feature_date: str):
    events = spark.read.schema(EVENT_SCHEMA).parquet("s3://lake/raw/events/")
    features = compute_features(events, feature_date)
    validate_features(features)
    (
        features.write
        .mode("overwrite")
        .partitionBy("feature_date")
        .parquet("s3://lake/features/user_features/")
    )
```

### Point-in-time correctness (critical)

Features must reflect state **as of the label timestamp**, not the current state. Using future data in training causes data leakage — model performs well offline but fails in production.

```python
def build_training_set(
    labels: DataFrame,
    features: DataFrame,
) -> DataFrame:
    return (
        labels
        .join(
            features,
            on=(
                (labels.user_id == features.user_id)
                & (features.feature_date <= labels.label_date)
            ),
        )
        .withColumn(
            "rn",
            F.row_number().over(
                Window.partitionBy("user_id", "label_date")
                .orderBy(F.col("feature_date").desc())
            ),
        )
        .filter(F.col("rn") == 1)
        .drop("rn")
    )
```

Rules:
- Filter features where `feature_date <= label_date`.
- Take the most recent feature snapshot before the label date.
- Never join on current/latest features — always point-in-time.

### Feature versioning

- When computation logic changes, create a new version: `s3://lake/features/user_features_v2/`.
- Do not overwrite existing feature versions — old models may still reference them.
- Track feature version in experiment metadata.

### Feature schema

Define feature schemas explicitly:

```python
from pydantic import BaseModel
from datetime import date

class UserFeatures(BaseModel):
    user_id: int
    event_count_lifetime: int
    total_amount: float
    distinct_categories: int
    last_event_time: str
    feature_date: date
```

Validate schema after computation, before write.

## Training pipeline

### Reproducibility requirements

Training must be deterministic: same data + same config = same model metrics (within floating-point tolerance).

```python
import random
import numpy as np

def set_reproducibility(seed: int = 42) -> None:
    random.seed(seed)
    np.random.seed(seed)
    # Framework-specific seeding
    # torch.manual_seed(seed)
    # tf.random.set_seed(seed)
```

Rules:
- Set random seeds at the start of every training run.
- Pin all ML library versions in dependency manifest.
- Version training data (S3 path includes date or snapshot ID).
- Log the exact data path, feature version, and code version used.

### Training job structure

```python
from pathlib import Path
import mlflow
import joblib

def train_model(
    training_data_path: str,
    model_output_path: str,
    params: dict,
    experiment_name: str,
) -> dict:
    set_reproducibility(params.get("seed", 42))

    X_train, X_test, y_train, y_test = load_and_split(training_data_path)

    mlflow.set_experiment(experiment_name)
    with mlflow.start_run():
        mlflow.log_params(params)

        model = build_model(params)
        model.fit(X_train, y_train)

        metrics = evaluate_model(model, X_test, y_test)
        mlflow.log_metrics(metrics)

        model_path = Path(model_output_path) / "model.joblib"
        joblib.dump(model, model_path)
        mlflow.log_artifact(str(model_path))

    return metrics
```

### Evaluation and promotion

```python
def evaluate_model(model, X_test, y_test) -> dict:
    predictions = model.predict(X_test)
    return {
        "accuracy": accuracy_score(y_test, predictions),
        "precision": precision_score(y_test, predictions, average="weighted"),
        "recall": recall_score(y_test, predictions, average="weighted"),
        "f1": f1_score(y_test, predictions, average="weighted"),
    }

def should_promote(new_metrics: dict, baseline_metrics: dict, threshold: float = 0.01) -> bool:
    return new_metrics["f1"] >= baseline_metrics["f1"] - threshold
```

Rules:
- Always compare new model against current production baseline.
- Define promotion criteria explicitly (metric threshold, minimum improvement).
- Log promotion decision with justification.
- Support rollback — keep previous model artifact accessible.

### Training DAG pattern

```python
from airflow.decorators import dag, task
from datetime import datetime

@dag(
    dag_id="ml_training_pipeline",
    schedule="@weekly",
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=["ml", "training"],
    default_args={"retries": 1, "execution_timeout": timedelta(hours=4)},
)
def ml_training_pipeline():

    @task()
    def prepare_training_data(feature_date: str) -> str:
        from ml.data_prep import build_training_set
        output_path = f"s3://lake/ml/training_data/dt={feature_date}/"
        build_training_set(feature_date, output_path)
        return output_path

    @task()
    def train(data_path: str) -> dict:
        from ml.training import train_model
        return train_model(
            training_data_path=data_path,
            model_output_path="s3://lake/ml/models/",
            params={"n_estimators": 100, "seed": 42},
            experiment_name="user_churn",
        )

    @task()
    def evaluate_and_promote(metrics: dict):
        from ml.promotion import evaluate_and_promote
        evaluate_and_promote(metrics, model_name="user_churn")

    data_path = prepare_training_data(feature_date="{{ ds }}")
    metrics = train(data_path)
    evaluate_and_promote(metrics)

ml_training_pipeline()
```

Keep training logic in importable modules (`ml/`), not in DAG files.

## Model serving

### Online serving (FastAPI)

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.model = joblib.load("models/current/model.joblib")
    yield

app = FastAPI(title="Prediction Service", lifespan=lifespan)

class PredictionRequest(BaseModel):
    user_id: int
    features: dict[str, float]

class PredictionResponse(BaseModel):
    user_id: int
    prediction: float
    model_version: str

@app.post("/predict", response_model=PredictionResponse)
async def predict(request: PredictionRequest):
    features_array = prepare_features(request.features)
    prediction = app.state.model.predict(features_array)[0]
    return PredictionResponse(
        user_id=request.user_id,
        prediction=float(prediction),
        model_version=app.state.model_version,
    )
```

Rules:
- Load model once at startup (lifespan), not per request.
- Define explicit request/response schemas (Pydantic).
- Include model version in response for traceability.
- Validate input features before prediction (type, range, null checks).
- Add healthcheck that verifies model is loaded.
- Follow all patterns from `./python-services-api.md` (timeouts, error handling, observability).

### Batch inference

```python
def batch_predict(
    spark: SparkSession,
    model_path: str,
    input_path: str,
    output_path: str,
    prediction_date: str,
) -> None:
    model = load_model(model_path)
    model_broadcast = spark.sparkContext.broadcast(model)

    features = spark.read.schema(FEATURE_SCHEMA).parquet(input_path)
    validate_input(features)

    @F.pandas_udf("double")
    def predict_udf(features_series: pd.Series) -> pd.Series:
        local_model = model_broadcast.value
        return pd.Series(local_model.predict(features_series.values.reshape(-1, 1)))

    predictions = features.withColumn("prediction", predict_udf(F.col("feature_vector")))
    validate_predictions(predictions)

    (
        predictions.write
        .mode("overwrite")
        .partitionBy("dt")
        .parquet(output_path)
    )
```

Rules:
- Broadcast model to executors — do not load per row.
- Use `pandas_udf` for vectorized prediction (not row-level Python UDF).
- Validate predictions: null check, range check, distribution sanity.
- Write pattern: overwrite-by-partition for idempotency.
- Follow all patterns from `./python-data-jobs.md`.

## Experiment tracking (MLflow)

### What to log per run

| Category | What to log | Why |
|----------|------------|-----|
| **Parameters** | Hyperparameters, feature version, data path, seed | Reproducibility |
| **Metrics** | Train/validation/test metrics (accuracy, F1, AUC, etc.) | Model selection |
| **Artifacts** | Model file, feature importance, evaluation plots | Auditability |
| **Tags** | Data version, code commit hash, environment | Lineage |

### MLflow integration pattern

```python
import mlflow

def log_experiment(
    experiment_name: str,
    params: dict,
    metrics: dict,
    model_path: str,
    tags: dict | None = None,
) -> str:
    mlflow.set_experiment(experiment_name)
    with mlflow.start_run() as run:
        mlflow.log_params(params)
        mlflow.log_metrics(metrics)
        mlflow.log_artifact(model_path)
        if tags:
            mlflow.set_tags(tags)
        return run.info.run_id
```

Rules:
- One experiment per model/use case.
- Log everything needed to reproduce the result.
- Never log secrets or PII as parameters or tags.
- Store artifacts in S3, not local filesystem.

## Model monitoring

### Drift detection job

```python
import numpy as np
from scipy.stats import ks_2samp

def compute_drift_metrics(
    reference_features: np.ndarray,
    current_features: np.ndarray,
    feature_names: list[str],
) -> dict[str, dict]:
    results = {}
    for i, name in enumerate(feature_names):
        stat, p_value = ks_2samp(reference_features[:, i], current_features[:, i])
        psi = compute_psi(reference_features[:, i], current_features[:, i])
        results[name] = {
            "ks_statistic": float(stat),
            "ks_p_value": float(p_value),
            "psi": float(psi),
            "drifted": p_value < 0.05 or psi > 0.2,
        }
    return results

def compute_psi(reference: np.ndarray, current: np.ndarray, bins: int = 10) -> float:
    ref_hist, edges = np.histogram(reference, bins=bins)
    cur_hist, _ = np.histogram(current, bins=edges)
    ref_pct = (ref_hist + 1) / (len(reference) + bins)
    cur_pct = (cur_hist + 1) / (len(current) + bins)
    return float(np.sum((cur_pct - ref_pct) * np.log(cur_pct / ref_pct)))
```

### Monitoring DAG pattern

```python
@dag(
    dag_id="ml_model_monitoring",
    schedule="@daily",
    catchup=False,
    default_args={"retries": 1},
)
def ml_model_monitoring():

    @task()
    def compute_drift(model_name: str, monitoring_date: str) -> dict:
        from ml.monitoring import compute_drift_for_model
        return compute_drift_for_model(model_name, monitoring_date)

    @task()
    def check_thresholds_and_alert(drift_results: dict):
        from ml.monitoring import check_and_alert
        check_and_alert(drift_results)

    drift = compute_drift(model_name="user_churn", monitoring_date="{{ ds }}")
    check_thresholds_and_alert(drift)

ml_model_monitoring()
```

### What to monitor

| Signal | Threshold | Action |
|--------|-----------|--------|
| Feature drift (PSI > 0.2 for any feature) | Alert | Investigate data pipeline, consider retraining |
| Prediction distribution shift (KS p < 0.01) | Alert | Compare with ground truth when available |
| Model latency p99 increase > 50% | Alert | Investigate model/infra, consider optimization |
| Prediction error rate > 5% | Alert | Check input validation, model health |
| Ground truth accuracy drop > 5% from baseline | Trigger retraining | Retrain with recent data |

## Checklist

- Feature jobs enforce point-in-time correctness (no future data leakage).
- Feature versions are immutable; new logic → new version path.
- Training is reproducible: seeds set, dependencies pinned, data versioned.
- Experiment tracking logs parameters, metrics, artifacts, and lineage.
- Evaluation compares new model against production baseline before promotion.
- Model serving validates inputs and includes model version in response.
- Batch inference broadcasts model, validates predictions, writes idempotently.
- Monitoring detects feature drift and prediction distribution shift.
- Retraining triggers are defined (scheduled or drift-based).

## Failure modes

- **Data leakage** — features computed with future data; model appears accurate offline, fails in production.
- **Training-serving skew** — features computed differently in training (PySpark) vs serving (Python); predictions diverge.
- **Non-reproducible training** — random seeds not set, dependency versions not pinned; re-training produces different model.
- **Model loading per request** — loading model from S3 on every prediction instead of startup; latency explosion.
- **Row-level UDF for prediction** — Python UDF instead of pandas_udf for batch inference; 100x slower.
- **Missing monitoring** — model degrades silently; stale predictions served for weeks.
- **Overwriting feature versions** — changing computation logic in place; old models trained on different features with same path.
- **No promotion gate** — new model deployed without comparison to baseline; regression goes to production.
