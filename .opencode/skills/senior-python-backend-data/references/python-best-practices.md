---
name: python-best-practices
description: Pythonic idioms, PEP 8 standards, type hints, and best practices for building robust, efficient, and maintainable Python applications.
---


# Python 3.10+ Best Practices (Production Code)
## Scope & Clarifications

This guide is a **production Python** best-practices reference.

Applies to:
- Backend services, libraries, CLIs, data processing modules, and reusable components.

Idiomatic Python patterns and best practices for building robust, efficient, and maintainable applications.

## When to use

- Writing new Python code
- Reviewing Python code
- Refactoring existing Python code
- Designing Python packages/modules

## 1. Python Version and Language Features
- Default target is Python 3.10+. If the runtime/platform dictates a different supported version (e.g., an orchestrator runtime), target the highest version supported by that runtime and avoid unsupported language features.
- Prefer modern syntax:
  - Use type hints wherever they add clarity.
  - Use `|` for union types instead of `typing.Union`.
  - Use built-in generics: `list[str]`, `dict[str, Any]`.
- Avoid deprecated features or syntax from Python <3.9.
- Favor standard library solutions before introducing third-party dependencies.

## 2. Type Safety and Typing
- All public functions, methods, and classes must have type annotations.
- Typing is preferred throughout the codebase, including:
  - internal functions,
  - intermediate transformation steps,
  - complex local variables.
- Avoid `Any` unless technically unavoidable; document the reason.
- Use:
  - `pydantic` for structured data,
  - `Protocol` for interface-like behavior.
- Configure and use `basedpyright` as the primary static type checker. It provides stricter validation than `pyright` and helps maintain high code quality.
- Configuration must be defined in `pyproject.toml`.

## 3. Code Style and Formatting
- Follow PEP 8.
- Use Ruff for linting, and use the repo-standard formatter setup (Ruff formatter if enabled, otherwise the project’s formatter).
- For import ordering, use the repo-standard approach (isort or Ruff import-sorting rules). Choose one per repo; do not mix.
- Do not leave commented-out or dead code in production.
- Use descriptive names that explain the purpose.
- Avoid single-letter names except for loop counters (e.g., `i`, `j` in simple loops).

### Example: Descriptive Naming
```python
# Good
user_count = 10
def calculate_total_price(items):
    pass

# Bad
uc = 10
def calc(i):
    pass
```

## 4. Project Structure and Architecture Boundaries
- Separate concerns clearly:
  - business logic,
  - I/O,
  - configuration,
  - orchestration.
- Use a `src/{pythonPackageName}` layout.
- Keep transformation logic independent from storage and orchestration.

Prefer a **functional core / imperative shell**:
- **Domain**: rules, invariants, state transitions (framework-free)
- **Application**: use cases, orchestration, transaction boundaries
- **Infrastructure**: DB/HTTP/broker/S3/Spark adapters
- **Interface**: HTTP handlers, Airflow tasks/operators, message consumers

Rules:
- Keep handlers thin; push complexity into domain/application code.
- Avoid leaking framework objects into domain logic.
- Make I/O dependencies injectable so core logic is unit-testable.

### Imports Must Be Side-Effect Free
- No network, DB, or S3 calls at module import time.
- For Airflow, DAG modules must be importable with **zero I/O**.
- Lazy-initialize heavy clients inside functions or factories.

## 5. Functions and Design
- Functions should do one thing and do it clearly.
- Prefer pure, deterministic functions.
- Avoid deeply nested logic.
- Keep functions reasonably small and readable.
- Prefer clarity over cleverness.
- Python code should be readable, explicit, and follow the principle of least surprise.

## 6. Error Handling
- Never use bare `except`.
- Catch specific exceptions only.
- Fail fast on invalid input.
- Wrap external system interactions (databases, APIs, filesystems) with explicit error handling.
- Never silently ignore errors.
- Include helpful error messages.

### Example: Helpful Error Messages
```python
import logging # root logger should be setup before program runs

# Good
try:
    result = process_data(data)
except ValueError as e:
    logging.error("Invalid data format: %s", e)
    raise
except ConnectionError as e:
    logging.error("Failed to connect: %s", e)
    return None

# Bad
try:
    result = process_data(data)
except:
    pass
```

## 7. Logging and Observability
- Never use `print` in production code (default).
- Use the `logging` module exclusively.
- Configure logging centrally.
- Use appropriate log levels:
  - `DEBUG` – internal details,
  - `INFO` – normal operation,
  - `WARNING` – recoverable issues,
  - `ERROR` – failed operations,
  - `CRITICAL` – system-wide failure.
- Never log secrets, credentials, or PII.

## 8. Configuration Management
- Do not hardcode:
  - credentials,
  - environment-specific values,
  - machine-specific paths.
- Externalize configuration using environment variables or config files (`.yaml`, `.toml`).
- Provide `.env.example` when environment variables are required.
- Keep secrets out of source control, code, and logs; use secret injection mechanisms.
- Load configuration once into a typed config object.
- Validate required fields at startup (fail fast).

## 9. Performance and Resource Usage
- Be conscious of time and memory complexity.
- Avoid loading unbounded datasets into memory.
- Prefer streaming or chunked processing for large data.
- Optimize only after correctness is ensured and bottlenecks are identified.
- Make batch transforms deterministic; document dedup keys and tie-breakers.
- Prefer replayable storage layouts (partitioned, append-only where feasible).

## 10. Time Semantics
- Prefer timezone-aware datetimes.
- Standardize on UTC internally; convert only at boundaries.
- Never mix naive and aware datetimes.
- For pipelines, document **event time vs processing time** and watermark logic.

## 11. Idempotency Patterns
When retries/replays are possible:
- Define an idempotency key (request_id/event_id/natural key).
- Persist idempotency state when correctness requires it.
- Prefer DB-enforced uniqueness + upserts for safety.
- Treat "retry + non-idempotent write" as a correctness bug.

## 12. Documentation
- Every module should have a docstring.
- Public APIs should document:
  - purpose,
  - parameters,
  - return values,
  - raised exceptions.
- Follow Google or NumPy docstring style.
- Keep `README.md` accurate and up to date.
- Important decisions or complex behaviors (e.g., Nginx proxying, API routing, environment variables) are documented directly in the code or config file using plain-language comments. Avoid redundant or obvious comments. Do not include agent reasoning or markdown syntax.
- Always include clear, concise, and relevant comments for non-trivial configurations or code logic, especially for proxy rules, routing, security settings, and environment-specific behavior.

## 13. Security
- Never commit secrets.
- Validate all external inputs.
- Apply least-privilege access.
- Treat data from external systems as untrusted by default.

### Never Hardcode Secrets
- Use environment variables or secret managers
- Never commit credentials to version control
```python
# Good
import os
api_key = os.environ.get("API_KEY")
# Bad
api_key = "sk-1234567890a134234"
```

## 14. Dependency Management

### Universal Rule

**NEVER edit dependency manifests manually** (pyproject.toml, requirements.txt, package.json, Cargo.toml, go.mod, etc.)
when the project has an established CLI workflow for them.

**ALWAYS use the dependency toolchain adopted by the repo** to add/remove/update deps.
Do not introduce a new package manager unless the project explicitly migrates.

Examples of acceptable repo-standard approaches:
- Poetry (pyproject + lock)
- pip-tools (requirements.in -> requirements.txt)
- requirements.txt + constraints.txt (common in Airflow/dockerized stacks)
- npm/pnpm/yarn for JS, etc.

### After Adding/Updating Dependencies (Generic)

1. Verify installation in a clean environment (container/venv)
2. Run verification: type checking, linting, tests 

> Note: In orchestrator runtimes (e.g., Airflow), dependency constraints may be dictated by the platform.
Follow the runtime’s constraints and the repo’s established process.

### Rationale

- Ensures consistent dependency resolution
- Maintains lock file integrity
- Prevents version conflicts
- Package managers handle transitive dependencies correctly

## 15. Configuration and Infrastructure Analysis

When working with Python projects that involve infrastructure:

1. **Check configuration sources safely:**
   - Dockerfile ENV instructions
   - docker-compose environment section
   - Kubernetes Secrets / ConfigMaps
   - CI/CD pipeline variable names (not secret values)
   - `.env.example` or documented env schema (do not read `.env` secret values)
   - Configuration files (config.yaml, etc.)
   - Application code (hardcoded fallbacks)

2. **Never assume variable sources:**
   - Don't assume where variables come from
   - Don't assume file paths
   - Don't assume deployment methods

3. **Verify environment coverage:**
   - Check if variables exist in all environments
   - Verify no hardcoded values that won't work in production

4. **Handle unset variables:**
   - Check if there's a fallback in code
   - Verify default values are appropriate

Security rule:
   - Never read or expose secret values from `.env` or secret stores.
   - Validate presence, naming, and wiring of required variables without inspecting secret contents.

**If any answer is "unsure" — ASK the user**

## 16. Prohibited Practices
- Global mutable state.
- Silent exception handling.
- Magic numbers without explanation.
- Mixing business logic with I/O.
- Mixing dependency management systems in one repo (ad-hoc installs or parallel lock/manifest workflows) instead of the repo-standard dependency workflow.

## 17. Package Organization

### Standard Project Layout

A repository may contain one or more projects of different types. For example, an Airflow project orchestrating a Python FastAPI service deployed on Kubernetes.

Choose the layout matching the project type. Common elements across all layouts:

- `ci_configs/` — CI/CD engine configuration (environment-specific YAML files).
- `pyproject.toml` — single source for dependencies, build config, and tool settings.
- `.python-version` — pyenv pin for reproducible Python version.
- `.gitlab-ci.yml` — pipeline definition.
- `.gitignore`, `README.md` — standard repo hygiene.

#### Airflow

```
myproject/
├── ci_configs/
│   ├── prod.yaml
│   └── dev.yaml
├── dags/
│   ├── dag1.py
│   ├── dag2.py
│   ├── resources/
│   │   ├── __init__.py
│   │   ├── random_template.html
│   │   └── random_file.sql
│   └── libs/
│       ├── __init__.py
│       ├── lib.py
│       └── vars.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   └── test_dags.py
├── pyproject.toml
├── README.md
├── .gitlab-ci.yml
├── .python-version
└── .gitignore
```

- `dags/` — DAG definitions at top level; Airflow scheduler scans this directory.
- `dags/libs/` — shared helpers and variables imported by DAGs.
- `dags/resources/` — templates, SQL files, and other static assets used by tasks.

#### Python CLI + K8s REST

```
myproject/
├── ci_configs/
│   ├── prod.yaml
│   └── dev.yaml
├── docker/
│   ├── volumes/
│   │   └── ...
│   ├── app1.Dockerfile
│   ├── app2.Dockerfile
│   ├── Dockerfile
│   └── docker-compose.yaml
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── __main__.py
│       ├── api/
│       │   ├── __init__.py
│       │   └── routes.py
│       ├── core/
│       │   ├── __init__.py
│       │   ├── logging.py
│       │   ├── logging.yaml
│       │   └── utils.py
│       ├── models/
│       │   ├── __init__.py
│       │   └── ...
│       └── storage/
│           ├── __init__.py
│           └── queries/
│               ├── __init__.py
│               ├── descriptive_sql_name1.sql
│               ├── descriptive_sql_name2.sql
│               └── descriptive_sql_name3.sql
├── tests/                 # structure follows test type (unit/, integration/, contract/)
│   ├── __init__.py
│   ├── conftest.py
│   └── ...
├── pyproject.toml
├── README.md
├── .env.example
├── .gitlab-ci.yml
├── .python-version
└── .gitignore
```

- `docker/` — Dockerfiles and compose config. Use multiple named Dockerfiles (`app1.Dockerfile`, `app2.Dockerfile`) when the repo produces more than one image; single `Dockerfile` otherwise.
- `src/mypackage/core/logging.yaml` — logging dict-config loaded via `importlib.resources.files` + `yaml.safe_load` + `logging.config.dictConfig`.
- `src/mypackage/models/` — domain models (Pydantic, dataclasses, domain entities). Zero framework imports.
- `src/mypackage/storage/queries/` — raw SQL files with descriptive names, loaded at runtime.
- `.env.example` — documents required environment variables (no secret values).

Architectural layers inside `src/mypackage/`:

| Directory | Layer | Rules |
|-----------|-------|-------|
| `api/` | Interface | Thin handlers, input validation, error shaping. No business logic. |
| `core/` | Cross-cutting | Config, logging, shared utilities. |
| `models/` | Domain | Domain models, business rules, invariants. Zero framework imports. |
| `storage/` | Infrastructure | DB access, SQL queries, external API clients, repositories. |

Domain logic has zero framework imports. Infrastructure adapters are injectable.

#### Pure dbt

```
myproject/
├── analyses/
│   └── .gitkeep
├── ci_configs/
│   ├── prod.yaml
│   └── dev.yaml
├── docker/
│   └── Dockerfile
├── macros/
│   └── .gitkeep
├── models/
│   ├── intermediate/
│   │   ├── my_first_dbt_model.sql
│   │   ├── my_second_dbt_model.sql
│   │   └── schema.yml
│   ├── marts/
│   │   ├── my_first_dbt_model.sql
│   │   ├── my_second_dbt_model.sql
│   │   └── schema.yml
│   └── staging/
│       ├── my_first_dbt_model.sql
│       ├── my_second_dbt_model.sql
│       └── schema.yml
├── seeds/
│   └── .gitkeep
├── snapshots/
│   └── .gitkeep
├── tests/
│   └── .gitkeep
├── .env.example
├── .gitignore
├── .gitlab-ci.yml
├── .python-version
├── README.md
├── dbt_project.yml
├── pyproject.toml
└── selectors.yml
```

- `models/staging/` — light view wrappers over source tables.
- `models/intermediate/` — joined/filtered temp tables for performance.
- `models/marts/` — final output tables consumed by downstream.
- Each model layer has its own `schema.yml` for column docs, tests, and contracts.

#### Spark

```
myproject/
├── ci_configs/
│   ├── prod.yaml
│   └── dev.yaml
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── __main__.py
│       ├── core/
│       │   ├── __init__.py
│       │   ├── logging.py
│       │   ├── logging.yaml
│       │   └── utils.py
│       ├── models/
│       │   ├── __init__.py
│       │   └── ...
│       └── storage/
│           ├── __init__.py
│           └── queries/
│               ├── __init__.py
│               └── descriptive_sql_name.sql
├── tests/                 # structure follows test type (unit/, integration/, contract/)
│   ├── __init__.py
│   ├── conftest.py
│   └── ...
├── pyproject.toml
├── README.md
├── .env.example
├── .gitlab-ci.yml
├── .python-version
└── .gitignore
```

- Same `src/` layout as Python CLI + K8s REST, but without `docker/` and without `api/` — Spark jobs are submitted to cluster via `spark-submit`, not served as HTTP endpoints.
- `src/mypackage/__main__.py` — entry point for `spark-submit`.

### Import Conventions

```python
# Good: Import order - stdlib, third-party, local
import os
import sys
from pathlib import Path

import requests
from fastapi import FastAPI

from mypackage.models import User
from mypackage.utils import format_name

# Good: Use isort or Ruff for automatic import sorting
```

## 18. Final Principle
Readable, typed, testable, and explicit code is more valuable than clever code.
Production Python should be predictable, observable, and easy to maintain.

## Checklist

- Typing is explicit for public APIs and non-trivial internals.
- Lint/format/type-check/test commands follow repo standards.
- Business logic is separated from I/O and orchestration.
- External calls are timeout-bounded, retry-safe, and observable.
- Secrets/PII are not present in code, logs, or committed artifacts.

## Failure modes

- Hidden import-time side effects causing startup/runtime instability.
- Bare `except` or swallowed errors masking data loss/corruption.
- Unbounded memory/query patterns causing production saturation.
- Repo toolchain drift (manual manifest edits, mixed format/lint workflows).
- Weak typing in boundary code causing contract drift at runtime.
