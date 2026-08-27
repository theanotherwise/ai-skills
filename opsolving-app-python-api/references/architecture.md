# Opsolving Python API application architecture

Read this reference before creating the application structure, relocating code, or deciding which layer owns a responsibility.

## Canonical source tree

```text
src/
|-- main.py
|-- etc/
|   |-- config.yaml
|   `-- logger.yaml
`-- app/
    |-- adapters/
    |   |-- __init__.py
    |   `-- postgresql.py
    |-- api/
    |   |-- __init__.py
    |   `-- <group>/
    |       |-- __init__.py
    |       |-- <endpoint>.py
    |       `-- <endpoint>.py
    |-- common/
    |   |-- __init__.py
    |   `-- timestamps.py
    |-- config/
    |   |-- config.py
    |   `-- logger.py
    |-- middleware/
    |   |-- __init__.py
    |   |-- lifespan.py
    |   `-- logging.py
    |-- migrations/
    |   |-- __init__.py
    |   |-- alembic.ini
    |   |-- env.py
    |   |-- script.py.mako
    |   `-- versions/
    |       |-- 0001_initial_schema.py
    |       `-- <revision>_<intent>.py
    |-- models/
    |   |-- __init__.py
    |   |-- base.py
    |   `-- <resource>.py
    |-- repositories/
    |   |-- __init__.py
    |   `-- <resource>.py
    `-- services/
        |-- __init__.py
        `-- <resource>.py
```

This is the complete architectural vocabulary, not a requirement to create unused placeholders. A service without persistence does not need a repository or mapped model. A database-backed application keeps the shared SQLAlchemy base even before its first concrete model. An application without a database does not need models, Alembic, or a PostgreSQL adapter. A simple non-database integration stays in one adapter module until it has enough independent behavior to justify a subpackage.

The import root is `src`: runtime packaging copies or exposes `src/main.py` as `main` and `src/app` as `app`. Application imports therefore use `from app...`, not `from src.app...`. Preserve the target repository's established packaging mechanism if it differs.

## Dependency map

```text
                         +-------------------+
                         |      main.py      |
                         +---------+---------+
                                   |
                   +---------------+---------------+
                   v                               v
             +-----+------+                  +-----+------+
             |    api     |                  | middleware |
             +-----+------+                  +-----+------+
                   |                               |
          request  |                               | lifecycle
       composition |                               v
                   |                         +-----+------+
                   +------------------------>|  adapters  |---> config
                   |   session factory       +-----+------+
                   v                               |
             +-----+------+                        v
             |  services  |                  AsyncEngine +
             +-----+------+                  sessionmaker
                   |
                   v
           +-------+--------+
           |  repositories  |<--- request AsyncSession
           +-------+--------+
                   |
                   v
                 models
```

`api` may import the adapter's session-factory getter, `AsyncSession` for dependency typing, and repository/service classes only in a dependency provider that assembles the request-scoped use case. This is composition, not use-case execution. The handler itself calls the service and converts the result to an HTTP response.

Repositories receive one `AsyncSession` explicitly. They import SQLAlchemy statement APIs and mapped persistence models, but they do not fetch the global session factory from adapters. The adapter owns the engine and factory lifecycle; the API dependency owns session scope; the repository owns ORM persistence operations.

`common` and generic `config` sit below the feature layers. They may be imported where appropriate, but they do not import upward. `migrations/env.py` reuses validated database settings and `Base.metadata`; it still builds a migration-local synchronous engine with `NullPool` and remains outside the async API request flow.

## Placement decision table

| Responsibility | Location | Boundary |
|---|---|---|
| Create the `FastAPI` object | `src/main.py` | Composition only |
| Define `/api` prefix and include group routers | `app/api/__init__.py` | Explicit router composition |
| Define a group prefix and include leaf routers | `app/api/<group>/__init__.py` | One HTTP resource or functional group |
| Define a route, HTTP models, and service dependency | `app/api/<group>/<endpoint>.py` | HTTP translation and composition |
| Scope a database session shared by several routes | `app/api/dependencies.py` | Request-scoped `AsyncSession` only |
| Coordinate a use case | `app/services/<resource>.py` | No FastAPI or database driver |
| Define mapped tables and relationships | `app/models/<resource>.py` | SQLAlchemy persistence schema only |
| Define the declarative base and naming convention | `app/models/base.py` | Shared ORM metadata |
| Execute ORM statements with an injected session | `app/repositories/<resource>.py` | No HTTP or business workflow |
| Build/dispose an engine and session factory | `app/adapters/postgresql.py` | No ORM queries or use cases |
| Read YAML files safely | `app/config/config.py` | File mechanics, caching, root validation |
| Configure and retrieve loggers | `app/config/logger.py` | Logging infrastructure |
| Start and stop resources | `app/middleware/lifespan.py` | Runtime lifecycle coordination |
| Process every HTTP request cross-cuttingly | `app/middleware/<concern>.py` | Middleware only |
| Format UTC timestamps or another truly shared primitive | `app/common/<concern>.py` | Small and dependency-light |
| Define schema history | `app/migrations/versions/*.py` | Alembic operations only |
| Store runtime YAML | `src/etc/*.yaml` | Data, not Python logic |

## Layer details

### `src/main.py`

Treat `main.py` as the module the process manager imports, normally as `main:app`. It may configure logging, construct the FastAPI application, attach lifespan and middleware, define a trivial root health/status response, and include the single top-level API router.

Do not put feature prefixes, repositories, service construction, database settings, client creation, SQL, authentication calls, or startup implementation here. If `main.py` grows when a feature is added, the feature is probably registered at the wrong level.

### `app/api`

This is the only feature layer that knows FastAPI request objects, `APIRouter`, `Depends`, HTTP status codes, headers, cookies, and HTTP request/response models. Keep top-level and group-level `__init__.py` files active as explicit router aggregators rather than passive package markers.

Leaf modules own cohesive HTTP operations. Colocate small Pydantic request and response models with their endpoint. Add a group-local `models.py` only when the same HTTP schema is genuinely shared by several leaf modules; this is an HTTP-schema module and is distinct from `app/models`, which contains SQLAlchemy mappings. Do not expose ORM models as response models.

An API dependency creates and closes one `AsyncSession` per request or use-case invocation. Keep a generic session dependency in `app/api/dependencies.py` when several route groups share it; keep feature-only repository/service composition beside that feature. Never store a live session globally or reuse one across requests.

### `app/services`

A service names and executes an application use case. It accepts explicit dependencies through its constructor, applies authorization or business rules supplied to the application layer, coordinates one or more repositories or service-safe clients, and returns a typed application result.

Services never receive `Request`, return `Response`, raise `HTTPException`, declare FastAPI dependencies, receive `AsyncSession`, import SQLAlchemy statement APIs, or execute persistence statements. They must not import `main`, `api`, adapters, or middleware. A simple service may consume a mapped entity type returned by a repository while the persistence and application representation are identical; create a service-owned dataclass or value object when business representation or lifetime must be independent of ORM state.

Use a class such as `UserService` when several related operations share dependencies. A focused pure function is acceptable when no shared dependency state exists. Do not create abstract base classes, registries, command buses, or protocol layers unless multiple implementations or the existing project actually need them.

### `app/models`

`base.py` owns `NAMING_CONVENTION` and one `Base(AsyncAttrs, DeclarativeBase)` whose metadata uses that convention. Each resource module defines mapped classes with `Mapped[...]`, `mapped_column`, explicit table names, constraints, indexes, foreign keys, and relationships that reflect the persistence schema.

`models/__init__.py` exports `Base` and imports every mapped model module required by Alembic. Importing `Base` alone does not register models that Python has never imported; missing model imports produce incomplete `Base.metadata` and broken autogenerate diffs.

Models do not contain Pydantic request/response schemas, FastAPI dependencies, repository queries, commits, or business orchestration. Avoid lazy-loading surprises outside the request session; repositories choose eager-loading strategies required by a use case.

### `app/repositories`

A repository groups ORM persistence operations for one resource or aggregate. Its module defines a repository class and SQLAlchemy `select`, `insert`, `update`, or `delete` statements as needed. It may return mapped entities or convert them to service-safe dataclasses when ORM lifetime or persistence fields must remain hidden.

Inject an `AsyncSession` in the constructor. Do not obtain a session or factory from an adapter global inside the repository. Use SQLAlchemy expression APIs and bound parameters; do not build SQL with untrusted string interpolation. Use `scalars`, `scalar_one_or_none`, or explicit row handling according to the query shape.

Repositories do not know routes, Pydantic response models, cookies, headers, or HTTP errors. They also do not decide cross-feature business completion. A repository may `flush` when generated values are required, but transaction ownership and `commit` must follow one explicit use-case contract rather than being hidden across unrelated repository helpers.

### `app/adapters`

An adapter binds a concrete technology to the application. For PostgreSQL this includes typed settings, YAML-to-settings validation, safe SQLAlchemy `URL` construction with `postgresql+psycopg`, a lazy `AsyncEngine`, one `async_sessionmaker[AsyncSession]`, a startup connection check, engine disposal, and `get_database_session_factory`.

The adapter owns connect timeout, TLS mode, pool size, overflow, acquisition timeout, recycling, pre-ping, LIFO behavior, and technical lifecycle logging. It does not own ORM models, feature queries, sessions for individual requests, or user-facing errors. Constructing the engine is inert; the first connection happens only during the explicit lifespan check or later session use.

Use one module per technology while it is cohesive, such as `postgresql.py`, `redis.py`, or `payments.py`. If it grows into independent client, auth, settings, and error responsibilities, replace the module with a same-named package and expose a small intentional facade from its `__init__.py`.

### `app/config`

`config.py` owns the location and safe reading of YAML files. Restrict callers to a filename rather than accepting arbitrary paths, accept only `.yaml` or `.yml`, validate that the root is a mapping, cache the parsed source, and return a deep copy so one consumer cannot mutate another consumer's configuration.

Technology-specific validation belongs next to the technology. The PostgreSQL adapter translates keys such as `connectTimeout`, `maxOverflow`, `acquireTimeout`, and `recycle` into typed snake-case fields, permits zero overflow, and validates duration formats and SSL modes. Do not turn the generic YAML reader into a catalog of every application's settings.

`logger.py` owns structured logging configuration, formatters, and `get_logger`. Configure logging once at application composition. Other modules use `get_logger(__name__)` and stable event names in `extra`; they do not call `dictConfig` themselves.

### `app/middleware`

`lifespan.py` owns the FastAPI async lifespan context. For PostgreSQL it performs one bounded `database_engine.connect()` check, stores the session factory in `application.state` only when request dependencies use state as the authoritative source, yields after startup succeeds, and disposes the engine in `finally`.

Other middleware modules own cross-cutting HTTP behavior such as structured request logging, correlation IDs, or response headers. Middleware may inspect request and resolved endpoint metadata, but it must not execute feature use cases or persistence operations.

### `app/common`

Put only stable, broadly reusable primitives here: timestamp formatting, neutral identifiers, small value conversions, or similarly dependency-light helpers. A helper used by only one feature stays with that feature. Do not use `common`, `utils`, or `helpers` to avoid choosing between API, service, repository, adapter, and config ownership.

### `app/migrations`

Keep Alembic's `alembic.ini`, `env.py`, `script.py.mako`, and `versions` together. `env.py` imports the model package, sets `target_metadata = Base.metadata`, reuses `database_settings.url`, and creates a migration-local synchronous engine with `NullPool`. Revision files use an ordered identifier plus intent, declare exact `revision` and `down_revision`, and implement both `upgrade` and `downgrade` whenever reversal is technically supported.

Every model module must be imported before Alembic reads metadata. Schema changes to a mapped model belong in a new reviewed revision; do not edit an already released revision to represent a later schema state. Do not call `Base.metadata.create_all()` or run migrations during module import, FastAPI startup, or request handling. Generating or applying a migration may require separate authorization under the target project's policies.

## Naming and file boundaries

Use lowercase snake-case Python filenames and plural resource modules when the public API/resource is plural, for example `models/users.py`, `repositories/users.py`, and `services/users.py`. Use singular class names such as `User`, `UserRepository`, `UserService`, and `UserResponse`.

Every route module exports `router`. Import it with a role-qualified alias in the parent aggregator. Every logger uses the module namespace. Give structured events stable snake-case names such as `application_started`, `postgresql_connection_checked`, and `postgresql_engine_disposed`.

Use `__init__.py` for explicit router composition, deliberate package exports, or a short package purpose. Do not fill package initializers with client construction or implicit discovery. The absence of an `app/__init__.py` is valid when the runtime deliberately uses namespace packages; preserve the target project's existing convention rather than changing it incidentally.

## Structural review checklist

- `main.py` remains a small composition root.
- The top-level API router is included once.
- Every group router explicitly includes each leaf router.
- Route modules contain HTTP contracts and dependency composition, not SQL.
- Services contain use cases and import neither FastAPI nor SQLAlchemy session/statement APIs.
- Every mapped class inherits the shared `Base`, and every model module is registered for Alembic metadata.
- Repositories contain ORM statements and receive one `AsyncSession` explicitly.
- API dependencies create and close sessions; no session is global or shared across requests.
- Adapters create the lazy engine and session factory without performing feature queries.
- Importing modules performs no network I/O or migration.
- Lifespan checks connectivity and disposes the engine exactly once.
- Generic config loading stays separate from technology-specific settings validation.
- Common code is genuinely shared and dependency-light.
- Schema changes have new ordered migration revisions based on complete `Base.metadata`.
