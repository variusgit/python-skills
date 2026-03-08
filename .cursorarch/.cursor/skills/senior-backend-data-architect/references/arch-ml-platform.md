---
name: arch-ml-platform
description: ML platform architecture — MLOps maturity, system topology, feature stores, training pipelines, model serving patterns, experiment tracking, model monitoring, and integration with the data platform. Use when designing ML infrastructure, feature pipelines, or model serving systems.
---

# ML Platform Architecture

Architectural patterns for designing ML platforms that are reproducible, operable, and integrated with the data platform.

## When to use

- Designing ML infrastructure (training, serving, monitoring).
- Planning feature engineering and feature store architecture.
- Choosing model serving patterns (online, batch, streaming).
- Designing experiment tracking and model lifecycle management.
- Integrating ML workloads with the data platform (Airflow, PySpark, S3).

## MLOps maturity levels

| Level | Description | Characteristics |
|-------|-------------|-----------------|
| **0 — Manual** | Manual training, manual deployment | Jupyter notebooks, no versioning, no monitoring |
| **1 — Automated training** | Training pipeline automated, deployment manual | Airflow-orchestrated training, versioned datasets, manual model promotion |
| **2 — Automated deployment** | Training and deployment automated, monitoring basic | CI/CD for models, automated A/B testing, basic drift detection |
| **3 — Full automation** | Automated retraining triggers, full observability | Auto-retraining on drift, feature store, comprehensive monitoring |

Start at Level 1 (automated training with Airflow). Progress to Level 2 when model count or deployment frequency justifies the investment.

## ML system topology

```
Data Platform (S3, PySpark) → Feature Engineering → Feature Store
                                                         ↓
Training Data → Training Pipeline → Model Registry → Model Serving → Consumers
                     ↑                                    ↓
              Experiment Tracking              Model Monitoring → Retraining Trigger
```

### Core subsystems

| Subsystem | Responsibility | Stack integration |
|-----------|---------------|-------------------|
| **Feature engineering** | Compute and store features from raw data | PySpark jobs, Airflow orchestration, S3 storage |
| **Feature store** | Serve features consistently for training and inference | S3 (offline), PostgreSQL/Redis (online) |
| **Training pipeline** | Data prep → training → evaluation → artifact storage | Airflow DAGs, PySpark for data, S3 for artifacts |
| **Model registry** | Version, tag, and manage model lifecycle | MLflow, S3-backed artifact store |
| **Model serving** | Serve predictions (online, batch, or streaming) | FastAPI (online), PySpark/Airflow (batch) |
| **Model monitoring** | Detect drift, degradation, anomalies | Custom metrics pipeline, Airflow-scheduled checks |

## Feature store architecture

### Offline feature store

Pre-computed features stored for training and batch inference.

- **Storage**: S3 (Parquet), partitioned by entity key and computation date.
- **Computation**: PySpark jobs orchestrated by Airflow.
- **Access pattern**: join features to training data by entity key and point-in-time.
- **Point-in-time correctness**: features must reflect state as of the label timestamp, not the current state. Prevents data leakage.

### Online feature store

Low-latency feature serving for real-time inference.

- **Storage**: PostgreSQL or Redis (add-on, requires ADR).
- **Update**: materialized from offline store or computed from streaming events.
- **Access pattern**: key-value lookup by entity ID, sub-10ms latency.
- **Consistency**: eventual with offline store; bounded staleness acceptable.

### Feature management

- **Feature definition**: documented schema, computation logic, source data, freshness SLA.
- **Feature versioning**: when computation logic changes, create a new version rather than modifying in place.
- **Feature lineage**: track which raw data produces which features, which models consume them.
- **Feature reuse**: same feature definition used in training and serving (training-serving skew prevention).

## Training pipeline architecture

### Pipeline stages

```
Data Preparation → Feature Assembly → Model Training → Evaluation → Artifact Storage → Promotion
```

| Stage | Tool | Output |
|-------|------|--------|
| **Data preparation** | PySpark | Cleaned, filtered training dataset (S3) |
| **Feature assembly** | PySpark / Feature store lookup | Feature matrix with labels (S3) |
| **Training** | Python (scikit-learn, XGBoost, PyTorch, etc.) | Trained model artifact |
| **Evaluation** | Python | Metrics (accuracy, precision, recall, AUC), comparison with baseline |
| **Artifact storage** | S3 + Model registry | Versioned model, metadata, metrics |
| **Promotion** | Manual gate or automated rule | Model tagged for serving (staging → production) |

### Orchestration with Airflow

- Each stage is an Airflow task. Training DAG runs on schedule or triggered by new data.
- Keep business logic (feature computation, training code) in importable Python modules, not in DAG files.
- Use Airflow for orchestration, not for compute — PySpark handles heavy processing.
- Training tasks must be idempotent: re-running with the same data produces the same model (set random seeds, pin dependencies).

### Experiment tracking

Track per training run:
- **Parameters**: hyperparameters, feature set version, data version.
- **Metrics**: evaluation metrics on train/validation/test sets.
- **Artifacts**: model file, feature importance, evaluation plots.
- **Lineage**: which data, which features, which code version produced this model.

MLflow or similar tool, with S3 as artifact backend.

## Model serving patterns

### Online inference (real-time)

- **Architecture**: model loaded in a FastAPI/gRPC service, predictions on request.
- **Latency**: typically p99 < 100ms (model dependent).
- **Scaling**: horizontal via K8s, stateless service instances.
- **Features**: served from online feature store (pre-computed) or computed on request.
- **Versioning**: canary deployment (small traffic percentage to new model), A/B testing.

### Batch inference

- **Architecture**: Airflow-orchestrated PySpark job, reads input, applies model, writes predictions to S3/DB.
- **Latency**: minutes to hours (acceptable for recommendations, risk scoring, periodic classification).
- **Scaling**: PySpark parallelism.
- **When**: predictions are pre-computed for all entities, consumed later. Lower cost than always-on serving.

### Streaming inference

- **Architecture**: model deployed in a stream processor, predictions per event.
- **Latency**: sub-second.
- **When**: real-time fraud detection, live personalization, event-driven triggers.
- **Stack impact**: requires streaming infrastructure (Kafka + consumer), treat as add-on with ADR.

### Selection guide

| Criteria | Online | Batch | Streaming |
|----------|--------|-------|-----------|
| Latency requirement | < 100ms | Hours acceptable | < 1 second |
| Prediction trigger | User request | Schedule/data arrival | Event |
| Infrastructure cost | Always-on service | Ephemeral compute | Always-on consumer |
| Complexity | Medium | Low | High |
| Default choice | For user-facing features | For periodic scoring | Only when event-driven is required |

## Model monitoring

### What to monitor

| Signal | What it detects | How |
|--------|----------------|-----|
| **Input data drift** | Feature distributions shifted from training data | Statistical tests (KS, PSI) on feature distributions |
| **Prediction drift** | Model output distribution changed | Monitor prediction distribution over time |
| **Performance degradation** | Model accuracy dropped | Compare predictions against delayed ground truth labels |
| **Latency** | Serving time increased | p50/p95/p99 of inference latency |
| **Error rate** | Serving failures | HTTP error rate, exception rate |

### Retraining triggers

- **Scheduled**: retrain on a fixed cadence (weekly, monthly) as baseline.
- **Drift-triggered**: retrain when data drift exceeds threshold.
- **Performance-triggered**: retrain when ground truth metrics degrade beyond threshold.
- **Manual**: retrain on business request (new features, new data source).

### Monitoring pipeline

Airflow-scheduled job that:
1. Reads recent predictions and features.
2. Computes drift metrics and performance metrics (if labels available).
3. Stores metrics in PostgreSQL or emits as structured logs.
4. Alerts on threshold violations.

## Checklist

- ML system topology is defined (feature store, training, serving, monitoring).
- Feature store separates offline (training) and online (serving) with point-in-time correctness.
- Training pipeline is automated (Airflow), idempotent, and produces versioned artifacts.
- Experiment tracking captures parameters, metrics, artifacts, and lineage.
- Serving pattern matches latency requirements (online, batch, or streaming).
- Model versioning supports rollback (previous model can be restored).
- Monitoring covers data drift, prediction drift, and serving health.
- Retraining strategy is defined (scheduled, drift-triggered, or performance-triggered).
- ML workloads integrate with data platform (shared S3 storage, Airflow orchestration, PySpark processing).

## Failure modes

- **Training-serving skew** — features computed differently in training vs serving, causing silent accuracy loss.
- **No point-in-time correctness** — training data uses future information (data leakage), inflating offline metrics.
- **No model versioning** — can't rollback to previous model when new one underperforms.
- **No monitoring** — model degrades silently, stale predictions served for weeks.
- **Over-engineering** — building Level 3 MLOps for a team with 2 models. Start at Level 1.
- **Streaming inference without justification** — always-on infrastructure cost for a use case that tolerates batch.
- **Manual deployment** — model promotion through ad-hoc scripts instead of automated, auditable pipeline.
- **Feature store without ownership** — features exist but nobody maintains them, drift goes undetected.
