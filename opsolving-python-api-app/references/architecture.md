# Python API application architecture

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
    |-- repositories/
    |   |-- __init__.py
    |   `-- <resource>.py
    `-- services/
        |-- __init__.py
        `-- <resource>.py
```

This is the complete architectural vocabulary, not a requirement to create unused placeholders. A service without persistence does not need a repository. An application without a database does not need Alembic or a PostgreSQL adapter. A simple integration stays in one adapter module until it has enough independent behavior to justify a subpackage.

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
                   v                         +-----+------+
             +-----+------+                  |  adapters  |
             |  services  |                  +-----+------+
             +-----+------+                        |
                   |                               v
                   v                            config
           +-------+--------+
           |  repositories  |
           +-------+--------+
                   |
                   v
        injected pool/client from adapter
```

`api` may import an adapter getter and repository class only in a dependency provider that assembles a service. This is composition, not use-case execution. The handler itself calls the service and converts the result to an HTTP response.

Repositories receive technical handles explicitly. They may import the external driver's types for annotations and operations, but they do not fetch global resources from adapters. This keeps SQL ownership in repositories and resource lifecycle ownership in adapters.

`common` and generic `config` sit below the feature layers. They may be imported where appropriate, but they do not import upward. `migrations/env.py` may reuse validated database settings from the database adapter so migration and runtime connection parameters agree; it still builds its own migration engine and remains outside the API request flow.

## Placement decision table

| Responsibility | Location | Boundary |
|---|---|---|
| Create the `FastAPI` object | `src/main.py` | Composition only |
| Define `/api` prefix and include group routers | `app/api/__init__.py` | Explicit router composition |
| Define a group prefix and include leaf routers | `app/api/<group>/__init__.py` | One HTTP resource or functional group |
| Define a route, HTTP models, and FastAPI dependency | `app/api/<group>/<endpoint>.py` | HTTP translation only |
| Coordinate a use case | `app/services/<resource>.py` | No FastAPI or database driver |
| Execute SQL and map rows | `app/repositories/<resource>.py` | No HTTP or business workflow |
| Build/open/close a database pool or external client | `app/adapters/<technology>.py` | No queries or use cases |
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

Leaf modules own cohesive HTTP operations. Colocate small Pydantic request and response models with their endpoint. Add a group-local `models.py` only when the same HTTP schema is genuinely shared by several leaf modules. Do not move persistence records or service models into `api` merely because they appear in a response.

### `app/services`

A service names and executes an application use case. It accepts explicit dependencies through its constructor, applies authorization or business rules supplied to the application layer, coordinates one or more repositories or service-safe clients, and returns a typed application result.

Services never receive `Request`, return `Response`, raise `HTTPException`, declare FastAPI dependencies, or execute raw SQL. They must not import `main`, `api`, or middleware. A simple service may return an immutable repository record while the persistence and application representation are identical; create a service-owned model when the business representation diverges.

Use a class such as `UserService` when several related operations share dependencies. A focused pure function is acceptable when no shared dependency state exists. Do not create abstract base classes, registries, command buses, or protocol layers unless multiple implementations or the existing project actually need them.

### `app/repositories`

A repository groups persistence operations for one resource or aggregate. Its module may define frozen dataclasses such as `UserRecord`, a repository class, SQL statements, row-to-record mapping, and transaction-scoped database operations.

Inject the pool, connection factory, or client in the constructor. Do not obtain it from an adapter global inside the repository. Keep external-driver-specific cursor and row handling here. Use parameterized queries for values; never interpolate untrusted values into SQL.

Repositories do not know routes, Pydantic response models, cookies, headers, or HTTP errors. They also do not decide cross-feature business completion. Translate raw database results into typed records before returning them to a service.

### `app/adapters`

An adapter binds a concrete technology to the application. For PostgreSQL this includes settings types, YAML-to-settings validation, connection-string construction, an inert `AsyncConnectionPool`, `open_database_pool`, `close_database_pool`, and `get_database_pool`.

The adapter owns connection behavior, timeouts, TLS mode, pool sizing, health checks supported by the driver, and technical logging. It does not own SQL or user-facing error responses. Create the handle with `open=False` or the library's lazy equivalent so importing the module performs no network I/O.

Use one module per technology while it is cohesive, such as `postgresql.py`, `redis.py`, or `payments.py`. If it grows into independent client, auth, settings, and error responsibilities, replace the module with a same-named package and expose a small intentional facade from its `__init__.py`.

### `app/config`

`config.py` owns the location and safe reading of YAML files. Restrict callers to a filename rather than accepting arbitrary paths, accept only `.yaml` or `.yml`, validate that the root is a mapping, cache the parsed source, and return a deep copy so one consumer cannot mutate another consumer's configuration.

Technology-specific validation belongs next to the technology. For example, the PostgreSQL adapter translates camel-case YAML keys such as `connectTimeout` into snake-case Python fields and validates duration formats, pool bounds, and SSL modes. Do not turn the generic YAML reader into a catalog of every application's settings.

`logger.py` owns structured logging configuration, formatters, and `get_logger`. Configure logging once at application composition. Other modules use `get_logger(__name__)` and stable event names in `extra`; they do not call `dictConfig` themselves.

### `app/middleware`

`lifespan.py` owns the FastAPI async lifespan context. It opens adapters in a deterministic order, stores handles in `application.state` only when request-time consumers need that access pattern, yields after startup succeeds, and closes resources in reverse order from a `finally` block.

Other middleware modules own cross-cutting HTTP behavior such as structured request logging, correlation IDs, or response headers. Middleware may inspect request and resolved endpoint metadata, but it must not execute feature use cases or persistence operations.

### `app/common`

Put only stable, broadly reusable primitives here: timestamp formatting, neutral identifiers, small value conversions, or similarly dependency-light helpers. A helper used by only one feature stays with that feature. Do not use `common`, `utils`, or `helpers` to avoid choosing between API, service, repository, adapter, and config ownership.

### `app/migrations`

Keep Alembic's `alembic.ini`, `env.py`, `script.py.mako`, and `versions` together. Revision files use an ordered identifier plus intent, declare exact `revision` and `down_revision`, and implement both `upgrade` and `downgrade` whenever reversal is technically supported.

Schema changes required by a repository change belong in a new revision. Do not edit an already released revision to represent a later schema state. Do not run migrations during module import, FastAPI startup, or request handling. Generating or applying a migration may require separate authorization under the target project's policies.

## Naming and file boundaries

Use lowercase snake-case Python filenames and plural resource modules when the public API/resource is plural, for example `users.py`, `repositories/users.py`, and `services/users.py`. Use singular class names such as `UserRepository`, `UserService`, `UserRecord`, and `UserResponse`.

Every route module exports `router`. Import it with a role-qualified alias in the parent aggregator. Every logger uses the module namespace. Give structured events stable snake-case names such as `application_started` or `postgresql_pool_opened`.

Use `__init__.py` for explicit router composition, deliberate package exports, or a short package purpose. Do not fill package initializers with client construction or implicit discovery. The absence of an `app/__init__.py` is valid when the runtime deliberately uses namespace packages; preserve the target project's existing convention rather than changing it incidentally.

## Structural review checklist

- `main.py` remains a small composition root.
- The top-level API router is included once.
- Every group router explicitly includes each leaf router.
- Route modules contain HTTP contracts and dependency composition, not SQL.
- Services contain use cases and do not import FastAPI.
- Repositories contain queries and do not obtain global clients themselves.
- Adapters create and manage technical resources without performing queries.
- Importing modules performs no network I/O or migration.
- Lifespan opens and closes each long-lived resource exactly once.
- Generic config loading stays separate from technology-specific settings validation.
- Common code is genuinely shared and dependency-light.
- Schema changes have new ordered migration revisions.
