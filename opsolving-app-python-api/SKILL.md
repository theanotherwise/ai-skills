---
name: opsolving-app-python-api
description: Create and refactor FastAPI applications using the Opsolving Python API layout with a minimal main entrypoint, composed routers, services, SQLAlchemy ORM models and repositories, async session infrastructure, middleware, configuration, and Alembic migrations. Use for application structure and feature implementation; do not use for Django or Flask projects, deployment configuration, or running infrastructure.
---

# Build layered Python API applications

## Goal

Build a FastAPI application whose entrypoint is only a composition root and whose code is separated by responsibility under `src/app`. Keep HTTP concerns in `api`, use cases in `services`, SQLAlchemy declarative mappings in `models`, ORM statements and persistence operations in `repositories`, engine and session-factory infrastructure in `adapters`, cross-cutting HTTP/runtime behavior in `middleware`, configuration mechanics in `config`, reusable pure helpers in `common`, and schema history in `migrations`.

The canonical request path is:

```text
main -> api route/dependency -> service -> repository -> request AsyncSession -> ORM model/database
```

Runtime resources follow a separate path:

```text
main -> lifespan middleware -> adapter -> AsyncEngine/sessionmaker -> config
```

Do not collapse these paths into one module. In particular, `main.py` must not construct repositories, execute SQL, open connections, read business configuration, implement application use cases, or contain feature routes beyond an optional root status endpoint.

## Required workflow

1. Read the governing project instructions and inspect the current source root, Python import root, framework version, runtime command, configuration source, external dependencies, migration setup, and tests.
2. Identify the API groups, feature operations, request and response contracts, application use cases, persistence operations, external clients, startup and shutdown resources, and schema changes required by the task.
3. Read [references/architecture.md](references/architecture.md) before creating a new application layout, moving modules, or deciding which layer owns existing code.
4. Read [references/routing-and-features.md](references/routing-and-features.md) before adding or reorganizing routers, endpoint modules, request/response models, dependency providers, services, or repositories.
5. Read [references/sqlalchemy-orm.md](references/sqlalchemy-orm.md) before adding or changing declarative models, relationships, ORM queries, session scope, write transactions, metadata registration, or ORM-backed migrations.
6. Read [references/runtime-and-data.md](references/runtime-and-data.md) before changing configuration loading, logging, middleware, lifespan behavior, SQLAlchemy engine/session-factory infrastructure, or Alembic execution.
7. Preserve existing public paths, methods, response shapes, configuration keys, migration revision history, and runtime behavior unless the user explicitly requests a behavioral change.
8. Validate the narrowest relevant source and test surface without starting a server, database, container, or external service unless runtime verification is explicitly authorized.

## Layer contract

Use the following ownership rules:

- `src/main.py` configures logging, creates `FastAPI`, attaches lifespan and middleware, exposes an optional root status route, and includes the top-level API router.
- `src/app/api/` owns FastAPI routers, HTTP paths and methods, Pydantic request/response models, FastAPI dependencies, request-scoped session creation, status codes, and translation between HTTP models and service inputs/results.
- `src/app/services/` owns application use cases and business coordination without importing FastAPI, request/response models, database drivers, or global adapter instances.
- `src/app/models/` owns SQLAlchemy declarative mappings, the shared `Base`, deterministic constraint naming, table relationships, and persistence-only model behavior. It does not own HTTP schemas or use cases.
- `src/app/repositories/` owns SQLAlchemy statements, persistence commands, ORM result handling, eager-loading choices, and persistence-specific absence semantics. Each repository receives an `AsyncSession`; it must not fetch a global session or own HTTP behavior.
- `src/app/adapters/` owns external technology setup: validated technology-specific settings, the lazy SQLAlchemy `AsyncEngine`, the shared `async_sessionmaker`, startup connection checks, engine disposal, and getters used during dependency composition.
- `src/app/config/` owns generic configuration and logging infrastructure. It does not own a feature's business settings or SQL.
- `src/app/middleware/` owns application lifespan and cross-cutting HTTP middleware. It coordinates adapters but does not implement feature use cases.
- `src/app/common/` owns small, dependency-light helpers that are genuinely shared. It is not a miscellaneous location for code with unclear ownership.
- `src/app/migrations/` owns Alembic configuration, `Base.metadata` integration, environment setup, revision templates, and ordered schema revisions. Migrations are executed separately, never as an import or API startup side effect.
- `src/etc/` contains runtime YAML such as application and logging configuration. Python modules validate and translate these values before technical components use them.

Keep framework coupling at the edge. FastAPI and HTTP-specific Pydantic models belong in `api` or `middleware`. SQLAlchemy engine, session, statement, and transaction APIs belong in `models`, `repositories`, the PostgreSQL adapter, Alembic, and the explicit API dependency that scopes an `AsyncSession`; services do not use those APIs. A service may consume a mapped entity type returned by a repository when that representation is sufficient, but it must not manage ORM lifetime. A route dependency may obtain the session factory, open one request-scoped session, construct a repository and service, and yield or return the composed dependency, but the handler delegates the operation immediately to the service.

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
        +--> request-scoped AsyncSession from adapter session factory
        |
        v
services/<resource>.py
        |
        v
repositories/<resource>.py
        |
        v
models/<resource>.py + AsyncSession
```

The leaf route owns HTTP validation and response serialization. Its dependency provider creates one `AsyncSession` from the adapter-owned `async_sessionmaker`, injects that session into the repository, and injects the repository into the service. The service owns the use case. The repository owns ORM statements and returns mapped entities or service-safe results. The route maps those results explicitly to Pydantic response models; it never serializes an ORM instance implicitly.

Do not share an `AsyncSession` between requests or store one as a module global. Make write-transaction ownership explicit for the complete use case; do not scatter hidden `commit()` calls across helpers. Do not skip the service for a business operation merely because its first implementation is a one-line pass-through.

Use explicit constructor injection. Do not introduce a global service locator, automatic router discovery, repository registry, or hidden dependency container. Name the exported router `router` in every routing module and alias it at the importing aggregator, for example `from app.api.users.list import router as list_router`.

## Import and side-effect rules

Imports may construct inert Python objects, build SQLAlchemy metadata, create a lazy `AsyncEngine`, create an `async_sessionmaker`, parse local configuration when the established adapter contract requires it, and register router metadata. Imports must not open a database connection, create an `AsyncSession`, call a remote API, run migrations, create tables, start background work, or inspect mutable external state.

Use lifespan startup for one bounded database connection check so readiness fails when PostgreSQL is unavailable, then dispose the SQLAlchemy engine in `finally`. Create and close `AsyncSession` objects in request dependencies or an explicit unit-of-work boundary. Migrations remain a separate operational command and must never be triggered by `main.py`, a route import, lifespan startup, or `Base.metadata.create_all()`.

Avoid cycles by preserving the intended direction. `common` and generic `config` must not import API, services, repositories, adapters, or middleware. Repositories must not import services or API. Services must not import API or middleware. Adapters must not import repositories or services.

## Existing applications

Do not reorganize an existing application solely because this skill was selected. Apply the layout when the user asks to create an application, add a feature within this architecture, or refactor toward it. During a refactor, move one complete behavior path at a time and update imports, router registration, dependency providers, tests, runtime paths, logging namespaces, configuration file lookup, and migration commands together.

Do not create empty packages or speculative abstractions. Add a module or subpackage only when it owns real behavior. Preserve a small single-file adapter, repository, or service until its responsibilities genuinely require a package split.

## Validation

Run the project's existing focused tests, static analysis, formatting checks, type checks, and Python compilation that cover changed modules. At minimum, inspect every changed file, compile or syntax-check the changed Python source when dependencies are not required, check router registration and import direction, confirm every mapped model is imported before Alembic reads `Base.metadata`, inspect the paired migration for schema changes, run `git diff --check`, and review repository status.

Do not install dependencies, start Uvicorn or Gunicorn, launch Docker Compose, open ports, connect to a database, apply migrations, or call external services without explicit authorization. If imports or tests require unavailable dependencies, report the exact blocked validation instead of treating a textual structure review as runtime proof.
