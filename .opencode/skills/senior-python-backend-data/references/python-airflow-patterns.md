---
name: python-airflow-patterns
description: Build production Apache Airflow DAGs with best practices for operators, sensors, and deployment. Use when creating data pipelines, orchestrating workflows, or scheduling batch jobs.
---

# Apache Airflow DAG Patterns
## Scope & Clarifications

This guide applies to **Apache Airflow DAG code**: DAG definitions, task wiring, operators/sensors, scheduling, retries, callbacks, and deployment concerns.

Clarifications to avoid common contradictions:
- **DAG files are orchestration glue**: keep them thin; move heavy logic into importable modules to keep parsing fast and enable unit testing.
- **Dependencies**: follow the repo's established dependency workflow (often requirements/constraints in Airflow stacks). Do not introduce a new package manager unless the project explicitly migrates.
- **Python version**: DAG code must be compatible with the Airflow runtime’s supported Python version. Avoid language features not supported by that runtime.

Production-ready patterns for Apache Airflow including DAG design, operators, sensors, and deployment strategies.

## When to use

- Creating data pipeline orchestration with Airflow
- Designing DAG structures and dependencies
- Implementing custom operators and sensors
- Setting up Airflow in production
- Debugging failed DAG runs

Read `python-best-practices.md` first, then use this document for Airflow-specific constraints.

## Core Concepts

### 1. DAG Design Principles

| Principle       | Description                         |
| --------------- | ----------------------------------- |
| **Idempotent**  | Running twice produces same result  |
| **Atomic**      | Tasks succeed or fail completely    |
| **Incremental** | Process only new/changed data       |
| **Observable**  | Logs, metrics, alerts at every step |

### 2. Task Dependencies

```python
# Linear
task1 >> task2 >> task3

# Fan-out
task1 >> [task2, task3, task4]

# Fan-in
[task1, task2, task3] >> task4

# Complex
task1 >> task2 >> task4
task1 >> task3 >> task4
```

### 3. Scheduling Semantics & Data Intervals
- Be explicit about timezone (prefer UTC unless business requires local time).
- Understand `start_date`, `catchup`, and data intervals to avoid accidental massive backfills.
- Prefer dataset-driven triggers when pipelines depend on upstream availability rather than wall-clock.

### 4. Idempotency & Retry Safety
- Tasks must be safe to retry: same inputs → same durable outputs.
- If external side effects exist, encode idempotency keys/dedup.
- Retries must be bounded and paired with timeouts.

### 5. Resource Controls
- Use pools/queues for scarce dependencies (DB, APIs, Spark clusters).
- Cap concurrency for mapped tasks to avoid task explosion.
- Prefer batching when cardinality is large.

## Patterns

### Pattern 1: TaskFlow API (Airflow 2.0+)

> Note: The following example keeps logic inline for brevity. In production, keep DAG files thin:
> move extract/transform/load logic into importable modules (e.g., `dags/common/` or `src/`),
> then call those functions from tasks to improve testability and DAG parse performance.


```python
# dags/taskflow_example.py
from datetime import datetime
from airflow.decorators import dag, task
from airflow.models import Variable

@dag(
    dag_id='taskflow_etl',
    schedule='@daily',
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=['etl', 'taskflow'],
)
def taskflow_etl():
    """ETL pipeline using TaskFlow API"""

    @task()
    def extract(source: str) -> dict:
        """Extract data from source"""
        import pandas as pd

        df = pd.read_csv(f's3://bucket/{source}/{{ ds }}.csv')
        return {'data': df.to_dict(), 'rows': len(df)}

    @task()
    def transform(extracted: dict) -> dict:
        """Transform extracted data"""
        import pandas as pd

        df = pd.DataFrame(extracted['data'])
        df['processed_at'] = datetime.now()
        df = df.dropna()
        return {'data': df.to_dict(), 'rows': len(df)}

    @task()
    def load(transformed: dict, target: str):
        """Load data to target"""
        import pandas as pd

        df = pd.DataFrame(transformed['data'])
        df.to_parquet(f's3://bucket/{target}/{{ ds }}.parquet')
        return transformed['rows']

    @task()
    def notify(rows_loaded: int):
        """Send notification"""
        import logging
        logger = logging.getLogger(__name__)
        logger.info("Loaded %s rows", rows_loaded)


    # Define dependencies with XCom passing
    extracted = extract(source='raw_data')
    transformed = transform(extracted)
    loaded = load(transformed, target='processed_data')
    notify(loaded)

# Instantiate the DAG
taskflow_etl()
```

### Pattern 2: Branching and Conditional Logic

```python
# dags/branching_example.py
from datetime import datetime
from airflow.decorators import dag, task
from airflow.operators.python import BranchPythonOperator
from airflow.operators.empty import EmptyOperator
from airflow.utils.trigger_rule import TriggerRule

@dag(
    dag_id='branching_pipeline',
    schedule='@daily',
    start_date=datetime(2024, 1, 1),
    catchup=False,
)
def branching_pipeline():

    @task()
    def check_data_quality() -> dict:
        """Check data quality and return metrics"""
        quality_score = 0.95  # Simulated
        return {'score': quality_score, 'rows': 10000}

    def choose_branch(**context) -> str:
        """Determine which branch to execute"""
        ti = context['ti']
        metrics = ti.xcom_pull(task_ids='check_data_quality')

        if metrics['score'] >= 0.9:
            return 'high_quality_path'
        elif metrics['score'] >= 0.7:
            return 'medium_quality_path'
        else:
            return 'low_quality_path'

    quality_check = check_data_quality()

    branch = BranchPythonOperator(
        task_id='branch',
        python_callable=choose_branch,
    )

    high_quality = EmptyOperator(task_id='high_quality_path')
    medium_quality = EmptyOperator(task_id='medium_quality_path')
    low_quality = EmptyOperator(task_id='low_quality_path')

    # Join point - runs after any branch completes
    join = EmptyOperator(
        task_id='join',
        trigger_rule=TriggerRule.NONE_FAILED_MIN_ONE_SUCCESS,
    )

    quality_check >> branch >> [high_quality, medium_quality, low_quality] >> join

branching_pipeline()
```

### Pattern 3: Sensors and External Dependencies

```python
# dags/sensor_patterns.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.decorators import task
from airflow.sensors.filesystem import FileSensor
from airflow.sensors.base import PokeReturnValue
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor
from airflow.sensors.external_task import ExternalTaskSensor
from airflow.operators.python import PythonOperator

with DAG(
    dag_id='sensor_example',
    schedule='@daily',
    start_date=datetime(2024, 1, 1),
    catchup=False,
) as dag:

    # Wait for file on S3
    wait_for_file = S3KeySensor(
        task_id='wait_for_s3_file',
        bucket_name='data-lake',
        bucket_key='raw/{{ ds }}/data.parquet',
        aws_conn_id='aws_default',
        timeout=60 * 60 * 2,  # 2 hours
        poke_interval=60 * 5,  # Check every 5 minutes
        mode='reschedule',  # Free up worker slot while waiting
    )

    # Wait for another DAG to complete
    wait_for_upstream = ExternalTaskSensor(
        task_id='wait_for_upstream_dag',
        external_dag_id='upstream_etl',
        external_task_id='final_task',
        execution_date_fn=lambda dt: dt,  # Same execution date
        timeout=60 * 60 * 3,
        mode='reschedule',
    )

    # Custom sensor using @task.sensor decorator
    @task.sensor(poke_interval=60, timeout=3600, mode='reschedule')
    def wait_for_api() -> PokeReturnValue:
        """Custom sensor for API availability"""
        import requests

        response = requests.get('https://api.example.com/health')
        is_done = response.status_code == 200

        return PokeReturnValue(is_done=is_done, xcom_value=response.json())

    api_ready = wait_for_api()

    def process_data(**context):
        import logging
        logger = logging.getLogger(__name__)
        api_result = context["ti"].xcom_pull(task_ids="wait_for_api")
        logger.info("API returned: %s", api_result)


    process = PythonOperator(
        task_id='process',
        python_callable=process_data,
    )

    [wait_for_file, wait_for_upstream, api_ready] >> process
```

### Pattern 4: Error Handling and Alerts

```python
# dags/error_handling.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.utils.trigger_rule import TriggerRule
from airflow.models import Variable

def task_failure_callback(context):
    """Callback on task failure"""
    task_instance = context['task_instance']
    exception = context.get('exception')

    # Send to Slack/PagerDuty/etc
    message = f"""
    Task Failed!
    DAG: {task_instance.dag_id}
    Task: {task_instance.task_id}
    Execution Date: {context['ds']}
    Error: {exception}
    Log URL: {task_instance.log_url}
    """
    # send_slack_alert(message)
    import logging
    logger = logging.getLogger(__name__)
    logger.error(message)


def dag_failure_callback(context):
    """Callback on DAG failure"""
    # Aggregate failures, send summary
    pass

with DAG(
    dag_id='error_handling_example',
    schedule='@daily',
    start_date=datetime(2024, 1, 1),
    catchup=False,
    on_failure_callback=dag_failure_callback,
    default_args={
        'on_failure_callback': task_failure_callback,
        'retries': 3,
        'retry_delay': timedelta(minutes=5),
    },
) as dag:

    def might_fail(**context):
        import random
        if random.random() < 0.3:
            raise ValueError("Random failure!")
        return "Success"

    risky_task = PythonOperator(
        task_id='risky_task',
        python_callable=might_fail,
    )

    def cleanup(**context):
        """Cleanup runs regardless of upstream failures"""
        import logging
        logging.getLogger(__name__).info("Cleaning up...")


    cleanup_task = PythonOperator(
        task_id='cleanup',
        python_callable=cleanup,
        trigger_rule=TriggerRule.ALL_DONE,  # Run even if upstream fails
    )

    def notify_success(**context):
        """Only runs if all upstream succeeded"""
        import logging
        logging.getLogger(__name__).info("All tasks succeeded!")


    success_notification = PythonOperator(
        task_id='notify_success',
        python_callable=notify_success,
        trigger_rule=TriggerRule.ALL_SUCCESS,
    )

    risky_task >> [cleanup_task, success_notification]
```

## Project Structure

```
airflow/
├── dags/
│   ├── __init__.py
│   ├── common/
│   │   ├── __init__.py
│   │   ├── operators.py    # Custom operators
│   │   ├── sensors.py      # Custom sensors
│   │   └── callbacks.py    # Alert callbacks
│   ├── etl/
│   │   ├── customers.py
│   │   └── orders.py
│   └── ml/
│       └── training.py
├── plugins/
│   └── custom_plugin.py
├── tests/
│   ├── __init__.py
│   ├── test_dags.py
│   └── test_operators.py
├── docker-compose.yml
└── requirements.txt
```

## Backfill Safety

Backfills must be:
- bounded (explicit range)
- throttled (pools/queues/concurrency caps)
- isolated from production SLAs when necessary
- observable (clear run naming, metrics/alerts)
- validated (sanity checks before promoting outputs)

## Best Practices

### Do's

- **Use TaskFlow API** - Cleaner code, automatic XCom
- **Set timeouts** - Prevent zombie tasks
- **Use `mode='reschedule'`** - For sensors, free up workers
- **Idempotent tasks** - Safe to retry
- **Use deferrable sensors** - Prefer when available; free up worker slots

### Don'ts

- **Don't use `depends_on_past=True`** - Creates bottlenecks
- **Don't hardcode dates** - Use `{{ ds }}` macros
- **Don't use global state** - Tasks should be stateless
- **Don't skip catchup blindly** - Understand implications
- **Don't put heavy logic in DAG file** - Import from modules

## Checklist

- DAG import/parse passes with zero import errors.
- DAG files are declarative and avoid import-time I/O.
- Retries/timeouts are bounded and idempotency-safe.
- Pools/queues/concurrency limits are set for expensive tasks.
- Backfill strategy is bounded, throttled, and observable.
- Alerts route to the correct owners/on-call.

## Failure modes

- Import-time network/DB/S3 calls causing broken DAG parsing.
- Accidental catchup/backfill creating runaway load.
- Non-idempotent tasks paired with retries creating duplicates.
- Sensors without resource controls starving workers.
- Unbounded mapped tasks or fan-out causing scheduler pressure.

## Resources

- [Airflow Documentation](https://airflow.apache.org/docs/)
- [Astronomer Guides](https://docs.astronomer.io/learn)
- [TaskFlow API](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html)
