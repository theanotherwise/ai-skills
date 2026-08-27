---
name: opsolving-python-api-app
description: Create and refactor FastAPI applications using the Opsolving Python API layout with a minimal main entrypoint, composed routers, services, repositories, adapters, configuration, middleware, common helpers, and Alembic migrations. Use for application structure and feature implementation; do not use for Django or Flask projects, deployment configuration, or running infrastructure.
---

# Build layered Python API applications

## Goal

Build a FastAPI application whose entrypoint is only a composition root and whose code is separated by responsibility under `src/app`. Keep HTTP concerns in `api`, use cases in `services`, SQL and persistence mapping in `repositories`, external-resource construction and lifecycle in `adapters`, cross-cutting HTTP/runtime behavior in `middleware`, configuration mechanics in `config`, reusable pure helpers in `common`, and schema history in `migrations`.

The canonical request path is:

```text
main -> api route -> service -> repository -> injected database/client
```

Runtime resources follow a separate path:

```text
main -> lifespan middleware -> adapter -> config
```

Do not collapse these paths into one module. In particular, `main.py` must not construct repositories, execute SQL, open connections, read business configuration, implement application use cases, or contain feature routes beyond an optional root status endpoint.

## Required workflow

1. Read the governing project instructions and inspect the current source root, Python import root, framework version, runtime command, configuration source, external dependencies, migration setup, and tests.
2. Identify the API groups, feature operations, request and response contracts, application use cases, persistence operations, external clients, startup and shutdown resources, and schema changes required by the task.
3. Read [references/architecture.md](references/architecture.md) before creating a new application layout, moving modules, or deciding which layer owns existing code.
4. Read [references/routing-and-features.md](references/routing-and-features.md) before adding or reorganizing routers, endpoint modules, request/response models, dependency providers, services, or repositories.
5. Read [references/runtime-and-data.md](references/runtime-and-data.md) before changing configuration loading, logging, middleware, lifespan behavior, adapters, database pools, or Alembic migrations.
6. Preserve existing public paths, methods, response shapes, configuration keys, migration revision history, and runtime behavior unless the user explicitly requests a behavioral change.
7. Validate the narrowest relevant source and test surface without starting a server, database, container, or external service unless runtime verification is explicitly authorized.

## Layer contract

Use the following ownership rules:

- `src/main.py` configures logging, creates `FastAPI`, attaches lifespan and middleware, exposes an optional root status route, and includes the top-level API router.
- `src/app/api/` owns FastAPI routers, HTTP paths and methods, Pydantic request/response models, FastAPI dependencies, status codes, and translation between HTTP models and service inputs/results.
- `src/app/services/` owns application use cases and business coordination without importing FastAPI, request/response models, database drivers, or global adapter instances.
- `src/app/repositories/` owns queries, persistence commands, transaction-scoped database work, row mapping, and persistence records. It must not own HTTP behavior or business workflow.
- `src/app/adapters/` owns external technology setup: validated technology-specific settings, client or pool construction, inert shared handles, open/close functions, and getters used during dependency composition.
- `src/app/config/` owns generic configuration and logging infrastructure. It does not own a feature's business settings or SQL.
- `src/app/middleware/` owns application lifespan and cross-cutting HTTP middleware. It coordinates adapters but does not implement feature use cases.
- `src/app/common/` owns small, dependency-light helpers that are genuinely shared. It is not a miscellaneous location for code with unclear ownership.
- `src/app/migrations/` owns Alembic configuration, environment setup, revision templates, and ordered schema revisions. Migrations are executed separately, never as an import or API startup side effect.
- `src/etc/` contains runtime YAML such as application and logging configuration. Python modules validate and translate these values before technical components use them.

Keep framework coupling at the edge. FastAPI and HTTP-specific Pydantic models belong in `api` or `middleware`; services and repositories must remain callable outside an HTTP request. A route may import an adapter getter, repository, and service to compose a request dependency, but it delegates the operation immediately to the service.

## Minimal application entrypoint

Keep `src/main.py` visually small enough to audit as the complete application composition root:

```python
from fastapi import FastAPI

from app.api import router as api_router
from app.config.logger import configure_logging
from app.middleware.lifespan import lifespan
from app.middleware.logging import RequestLoggingMiddleware


configure_logging()

app = FastAPI(title="Example API", lifespan=lifespan)
app.add_middleware(RequestLoggingMiddleware)


@app.get("/")
def read_root() -> dict[str, str]:
    return {"status": "ok"}


app.include_router(api_router)
```

Register additional feature routers inside `app.api`, not in `main.py`. Put startup and shutdown implementation in `middleware/lifespan.py`. Construct external resources in adapters. Do not add a second application factory unless the existing runtime or tests require one.

## Feature implementation

Implement a database-backed feature as one vertical path through the existing layers:

```text
api/<group>/<endpoint>.py
        |
        v
services/<resource>.py
        |
        v
repositories/<resource>.py
        |
        v
adapter-provided client or pool
```

The leaf route owns HTTP validation and response serialization. Its dependency provider composes the adapter handle, repository, and service. The service owns the use case. The repository owns the query and maps raw rows to typed immutable records. Do not skip the service for a business operation merely because its first implementation is a one-line pass-through; keep the boundary when the operation represents an application use case expected to grow.

Use explicit constructor injection. Do not introduce a global service locator, automatic router discovery, repository registry, or hidden dependency container. Name the exported router `router` in every routing module and alias it at the importing aggregator, for example `from app.api.users.list import router as list_router`.

## Import and side-effect rules

Imports may construct inert Python objects, parse local configuration when the established adapter contract requires it, and register router metadata. Imports must not connect to a database, call a remote API, run migrations, start background work, or inspect mutable external state.

Open long-lived resources in lifespan startup and close them in `finally`. Create clients or pools in adapters with lazy or closed initialization when the library supports it. Migrations remain a separate operational command and must never be triggered by `main.py`, a route import, or lifespan startup.

Avoid cycles by preserving the intended direction. `common` and generic `config` must not import API, services, repositories, adapters, or middleware. Repositories must not import services or API. Services must not import API or middleware. Adapters must not import repositories or services.

## Existing applications

Do not reorganize an existing application solely because this skill was selected. Apply the layout when the user asks to create an application, add a feature within this architecture, or refactor toward it. During a refactor, move one complete behavior path at a time and update imports, router registration, dependency providers, tests, runtime paths, logging namespaces, configuration file lookup, and migration commands together.

Do not create empty packages or speculative abstractions. Add a module or subpackage only when it owns real behavior. Preserve a small single-file adapter, repository, or service until its responsibilities genuinely require a package split.

## Validation

Run the project's existing focused tests, static analysis, formatting checks, type checks, and Python compilation that cover changed modules. At minimum, inspect every changed file, compile or syntax-check the changed Python source when dependencies are not required, check router registration and import direction by inspection, run `git diff --check`, and review repository status.

Do not install dependencies, start Uvicorn or Gunicorn, launch Docker Compose, open ports, connect to a database, apply migrations, or call external services without explicit authorization. If imports or tests require unavailable dependencies, report the exact blocked validation instead of treating a textual structure review as runtime proof.
