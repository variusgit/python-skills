---
name: python-services-api
description: Production Python service and microservice patterns — FastAPI, aiohttp, gRPC, API contracts, project structure, dependency injection, healthchecks, graceful shutdown, inter-service communication. Use when designing, implementing, or reviewing HTTP/RPC services.
---

# Microservices & Web Services

Practical patterns for building production Python services and microservices. Covers framework selection, service structure, API contracts, lifecycle management, and inter-service communication.

Read `./python-best-practices.md` first for architecture principles (functional core / imperative shell, thin handlers, layered boundaries). This document builds on those foundations with service-specific implementation patterns.

## When to use

- Designing or implementing a new HTTP/RPC service.
- Choosing between FastAPI, aiohttp, or gRPC for a workload.
- Structuring a service project (routers, dependencies, models).
- Defining or evolving API contracts (request/response, errors, pagination, compatibility).
- Adding healthchecks, graceful shutdown, or inter-service communication.
- Reviewing service architecture for production readiness.

## Framework selection guide

| Criteria | FastAPI | aiohttp | gRPC |
|----------|---------|---------|------|
| **Best for** | API-first services, CRUD, public APIs | Low-level async, websockets, custom protocols | Inter-service RPC, strict contracts, streaming |
| **Validation** | Pydantic (built-in) | Manual or third-party | Protobuf (schema-enforced) |
| **API docs** | Auto-generated OpenAPI | Manual | Protobuf as contract |
| **Async** | Native (ASGI) | Native | Native |
| **Learning curve** | Low | Medium | Medium-High |
| **When to pick** | Default choice for new services | Need websockets, SSE, or HTTP client+server in one lib | Need strict typed contracts between services, high throughput, or streaming |

**Decision shortcut:**
- New service with REST API? **FastAPI**.
- Need websockets or advanced async HTTP client? **aiohttp**.
- Service-to-service calls where latency and contract strictness matter? **gRPC**.
- Multiple protocols needed? Run FastAPI + gRPC in the same process via separate ports.

## Service structure (project layout)

Follows the layered architecture from `python-best-practices.md`:

```
src/myservice/
├── main.py              # Application factory + entrypoint
├── config.py            # Typed config, env loading, validation
├── domain/              # Business rules, invariants (framework-free)
│   ├── models.py
│   └── services.py
├── application/         # Use cases, orchestration, transaction boundaries
│   └── use_cases.py
├── infrastructure/      # DB, HTTP clients, external adapters
│   ├── database.py
│   ├── repositories.py
│   └── clients.py
├── api/                 # HTTP interface layer
│   ├── dependencies.py  # Depends factories (DB session, auth, config)
│   ├── errors.py        # Exception handlers, error shapes
│   ├── middleware.py     # CORS, request ID, logging
│   └── routes/
│       ├── health.py
│       ├── users.py
│       └── orders.py
└── proto/               # gRPC protobuf definitions (if applicable)
    └── service.proto
```

Rules:
- Routers and handlers are thin — delegate to domain/application layer.
- Domain layer has zero framework imports.
- Infrastructure adapters are injectable.

## FastAPI patterns

### Application factory + lifespan

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from myservice.config import Settings
from myservice.infrastructure.database import Database

@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = Settings()
    app.state.db = Database(settings.database_url)
    await app.state.db.connect()
    yield
    await app.state.db.disconnect()

def create_app() -> FastAPI:
    app = FastAPI(title="My Service", lifespan=lifespan)
    app.include_router(health_router, prefix="/health", tags=["health"])
    app.include_router(users_router, prefix="/api/v1/users", tags=["users"])
    app.add_middleware(RequestIdMiddleware)
    return app
```

### Router organization

```python
from fastapi import APIRouter, Depends, HTTPException, status
from myservice.api.dependencies import get_db_session, get_current_user
from myservice.domain.models import UserCreate, UserResponse

router = APIRouter()

@router.post("/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(
    payload: UserCreate,
    session = Depends(get_db_session),
    current_user = Depends(get_current_user),
):
    user = await user_service.create(session, payload)
    return user
```

### Dependency injection via Depends

```python
from fastapi import Depends, Request
from sqlalchemy.ext.asyncio import AsyncSession

async def get_db_session(request: Request):
    async with request.app.state.db.session() as session:
        yield session

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    session: AsyncSession = Depends(get_db_session),
) -> User:
    user = await auth_service.validate_token(session, token)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid token")
    return user

def get_settings(request: Request) -> Settings:
    return request.app.state.settings
```

### Exception handlers and error shapes

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel

class ErrorResponse(BaseModel):
    error_code: str
    message: str
    details: dict | None = None

async def domain_error_handler(request: Request, exc: DomainError):
    return JSONResponse(
        status_code=exc.status_code,
        content=ErrorResponse(
            error_code=exc.code,
            message=str(exc),
            details=exc.details,
        ).model_dump(),
    )

app = FastAPI()
app.add_exception_handler(DomainError, domain_error_handler)
```

### Middleware (request ID, CORS)

```python
import uuid
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.middleware.cors import CORSMiddleware

class RequestIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
        request.state.request_id = request_id
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## aiohttp patterns

### Application factory + cleanup contexts

```python
from aiohttp import web

async def init_db(app: web.Application):
    app["db"] = await Database.connect(app["settings"].database_url)
    yield
    await app["db"].disconnect()

def create_app() -> web.Application:
    app = web.Application(middlewares=[request_id_middleware, error_middleware])
    app["settings"] = Settings()
    app.cleanup_ctx.append(init_db)
    app.router.add_get("/health", health_handler)
    app.router.add_post("/api/v1/users", create_user_handler)
    return app

async def create_user_handler(request: web.Request) -> web.Response:
    payload = await request.json()
    session = request.app["db"]
    user = await user_service.create(session, payload)
    return web.json_response(user, status=201)
```

### When to prefer aiohttp over FastAPI

- Need a combined HTTP client + server in the same application (e.g., proxy, aggregator).
- Need websockets or server-sent events as a core feature.
- Need fine-grained control over the HTTP protocol (custom transports, streaming responses).
- Already have an aiohttp codebase — don't rewrite without a reason.

## gRPC basics

### When to choose gRPC over REST

- Service-to-service calls where **contract strictness** and **backward compatibility** are critical.
- High-throughput, low-latency internal communication.
- Need **streaming** (server-side, client-side, or bidirectional).
- Want **code-generated** clients/servers from a single schema definition.

Do not use gRPC for public-facing APIs consumed by browsers (use REST/OpenAPI instead).

### Protobuf service definition

```protobuf
// proto/user_service.proto
syntax = "proto3";
package myservice;

service UserService {
  rpc GetUser (GetUserRequest) returns (UserResponse);
  rpc CreateUser (CreateUserRequest) returns (UserResponse);
  rpc ListUsers (ListUsersRequest) returns (stream UserResponse);
}

message GetUserRequest {
  string user_id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message UserResponse {
  string user_id = 1;
  string name = 2;
  string email = 3;
}
```

### Python server skeleton

```python
import grpc
from concurrent import futures
from myservice.proto import user_service_pb2_grpc, user_service_pb2

class UserServiceServicer(user_service_pb2_grpc.UserServiceServicer):
    async def GetUser(self, request, context):
        user = await user_repo.get(request.user_id)
        if not user:
            context.set_code(grpc.StatusCode.NOT_FOUND)
            context.set_details(f"User {request.user_id} not found")
            return user_service_pb2.UserResponse()
        return user_service_pb2.UserResponse(
            user_id=user.id, name=user.name, email=user.email,
        )

async def serve():
    server = grpc.aio.server()
    user_service_pb2_grpc.add_UserServiceServicer_to_server(
        UserServiceServicer(), server,
    )
    server.add_insecure_port("[::]:50051")
    await server.start()
    await server.wait_for_termination()
```

### Dual-protocol service (HTTP + gRPC)

Run FastAPI and gRPC server in the same process on different ports. Use `asyncio.gather` to start both:

```python
import asyncio
import uvicorn

async def main():
    grpc_task = asyncio.create_task(grpc_serve())
    config = uvicorn.Config(create_app(), host="0.0.0.0", port=8000)
    http_server = uvicorn.Server(config)
    await asyncio.gather(http_server.serve(), grpc_task)
```

## API contracts

### Request/response contracts

- Validate inputs early; reject invalid data with clear errors.
- Define stable response models; avoid leaking internal structures.
- Use consistent field naming and types; document time zone semantics.
- Include correlation/request IDs if the platform supports it.

### Backward compatibility (default)

- Avoid breaking changes; if unavoidable, version explicitly.
- For additive changes:
  - add optional fields
  - tolerate unknown fields on input when safe
- Deprecate with a policy (announce, monitor usage, remove later).

### Idempotent writes

For endpoints that create/charge/trigger:
- Support idempotency keys.
- Document idempotency semantics:
  - what is considered "same request"
  - how long keys are retained
  - what happens on partial failure

### Error model

- Provide:
  - stable machine-readable error code
  - human-readable message
  - optional details (field errors) without leaking secrets
- Do not return stack traces to clients.
- Map internal exceptions to consistent error shapes.

### Pagination

- Prefer cursor-based pagination for large datasets.
- Enforce maximum page size.
- Avoid unbounded list endpoints in production paths.

### Timeouts and retries

- Servers must be safe under client retries (idempotency).
- Clients should implement bounded retries with backoff for transient errors.
- Avoid retry storms; rate-limit if necessary.

### Observability at the edge

- Log at the edge (request start/end) with:
  - correlation id
  - key identifiers (non-PII)
  - latency and status
- Emit metrics for rate/latency/error classes.
- Observability patterns (structured logs, metrics, correlation IDs) should be part of every service.

## Service lifecycle

### Healthcheck and readiness endpoints

```python
from fastapi import APIRouter

router = APIRouter()

@router.get("/live")
async def liveness():
    return {"status": "ok"}

@router.get("/ready")
async def readiness(request: Request):
    db_ok = await request.app.state.db.ping()
    if not db_ok:
        raise HTTPException(status_code=503, detail="Database unavailable")
    return {"status": "ready"}
```

- `/live` — process is running (for k8s liveness probe).
- `/ready` — dependencies are available (for k8s readiness probe).
- Never include secrets or internal details in health responses.

### Graceful shutdown

```python
import signal
import asyncio

async def graceful_shutdown(app):
    app.state.shutting_down = True
    await asyncio.sleep(5)  # drain in-flight requests
    await app.state.db.disconnect()
```

- Handle `SIGTERM` / `SIGINT` to start drain.
- Stop accepting new connections, finish in-flight requests.
- Close DB pools, HTTP clients, gRPC channels.
- Set a shutdown timeout to avoid hanging indefinitely.

### Configuration

- Load configuration once into a typed config object (Pydantic `BaseSettings` or similar).
- Validate required fields at startup — fail fast on missing or invalid config.
- Never hardcode secrets; use env vars or secret managers.
- See `./python-best-practices.md` section 8 for details.

## Inter-service communication

### Synchronous (HTTP client)

```python
import httpx

class OrderClient:
    def __init__(self, base_url: str, timeout: float = 5.0):
        self._client = httpx.AsyncClient(
            base_url=base_url,
            timeout=timeout,
        )

    async def get_order(self, order_id: str) -> dict:
        response = await self._client.get(f"/api/v1/orders/{order_id}")
        response.raise_for_status()
        return response.json()
```

- Always set timeouts on outgoing calls.
- Implement bounded retries with backoff for transient errors (5xx, timeouts).
- Consider circuit breaker pattern for critical dependencies to avoid cascading failures.
- Pass correlation/request IDs in headers for tracing.

### Asynchronous (messaging)

For event-driven inter-service communication (Kafka, RabbitMQ, SQS), defer to `python-messaging.md`. Key considerations:
- Prefer async messaging when the caller does not need an immediate response.
- Ensure consumer idempotency (at-least-once delivery).
- Use outbox pattern when DB consistency and event publishing must be atomic.

### gRPC (inter-service)

Use gRPC when:
- Latency matters more than REST overhead allows.
- Both sides benefit from strict protobuf contracts and code generation.
- Streaming is needed (real-time feeds, large result sets).

```python
import grpc
from myservice.proto import order_service_pb2_grpc, order_service_pb2

async def get_order(order_id: str):
    async with grpc.aio.insecure_channel("orders-service:50051") as channel:
        stub = order_service_pb2_grpc.OrderServiceStub(channel)
        response = await stub.GetOrder(
            order_service_pb2.GetOrderRequest(order_id=order_id),
        )
        return response
```

## Checklist

- Framework chosen with clear rationale (FastAPI / aiohttp / gRPC).
- Service follows layered architecture (domain / application / infrastructure / interface).
- Handlers are thin; business logic is in domain/application layer.
- API contracts are explicit (input/output/error models).
- Contracts are backward compatible; breaking changes are versioned.
- Write paths are idempotent where retries are possible.
- Pagination and list limits are enforced.
- Healthcheck and readiness endpoints are implemented.
- Graceful shutdown handles drain and resource cleanup.
- Inter-service calls have timeouts, retries, and correlation IDs.
- Logs/metrics are sufficient to debug issues at the edge.

## Failure modes

- Breaking contract changes without explicit versioning.
- Retry-unsafe writes creating duplicate side effects.
- Leaking internal exception details or stack traces to clients.
- Unbounded list endpoints causing latency and resource spikes.
- Inconsistent error shapes that break clients and observability.
- Missing healthchecks causing traffic to unhealthy instances.
- No graceful shutdown — connections dropped, data lost on deploy.
- Cascading failures from missing timeouts/circuit breakers on inter-service calls.
- Tight coupling between services via shared DB or internal models instead of contracts.
