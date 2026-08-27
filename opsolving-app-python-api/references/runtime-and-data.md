# Python API runtime, configuration, adapters, and migrations

Read this reference before changing YAML configuration, structured logging, middleware, lifespan behavior, external-resource adapters, SQLAlchemy engine/session-factory settings, or Alembic execution.

## Configuration layout

Keep deployable runtime data outside Python modules:

```text
src/etc/
|-- config.yaml
`-- logger.yaml

src/app/config/
|-- config.py
`-- logger.py
```

`etc/config.yaml` contains application and integration values. `etc/logger.yaml` contains the Python logging dictionary. Do not mix logger handler definitions into application config or embed application settings in `logger.yaml`.

The generic loader resolves the repository/runtime-owned `etc` directory from its module location, not the process's current directory. Accept a bare YAML filename only, reject directory traversal and non-YAML suffixes, parse with `yaml.safe_load`, require a mapping at the root, and return an isolated copy.

```python
from collections.abc import Mapping
from copy import deepcopy
from functools import lru_cache
from pathlib import Path
from typing import Any

import yaml


CONFIG_DIRECTORY = Path(__file__).resolve().parents[2] / "etc"


@lru_cache(maxsize=None)
def _read_config(config_name: str) -> Mapping[str, Any]:
    config_path = Path(config_name)
    if config_path.name != config_name or config_path.suffix not in {".yaml", ".yml"}:
        raise ValueError("config_name must be a YAML file name without a directory")

    with (CONFIG_DIRECTORY / config_path).open(encoding="utf-8") as config_file:
        config = yaml.safe_load(config_file)

    if not isinstance(config, Mapping):
        raise ValueError(f"{config_name} must contain a YAML mapping at its root")
    return config


def read_config(config_name: str) -> Mapping[str, Any]:
    return deepcopy(_read_config(config_name))
```

Expose a cache-clear function when tests or an explicitly supported reload need it. Do not clear configuration implicitly per request. Preserve the application's configured secret source; do not log secret values or duplicate them into derived settings output.

## Technology-specific settings

The generic loader validates file mechanics and the root type. Each adapter validates its own subtree and translates external naming into typed Python fields. For a PostgreSQL adapter this includes:

- required host, port, database, user, and password values;
- a boolean persistence declaration when the deployment contract uses it;
- connection timeout and SSL mode;
- SQLAlchemy pool size, non-negative maximum overflow, acquisition timeout, and recycle age.

Use frozen dataclasses or another explicit typed structure. Reject booleans where an integer is expected. Parse human-readable duration values into seconds once at the adapter boundary. Validate enumerations such as supported SSL modes before constructing the external client.

YAML may use deployment-oriented camel-case names such as `connectTimeout`, `maxOverflow`, and `acquireTimeout`; Python settings use snake-case names such as `connect_timeout`, `max_overflow`, and `acquire_timeout`. Perform that translation in the adapter. Do not leak raw nested mappings throughout models, repositories, or services.

## Adapter lifecycle

The PostgreSQL adapter has five responsibilities:

1. load and validate technology-specific settings;
2. build a safe SQLAlchemy `URL` with the `postgresql+psycopg` driver;
3. construct one lazy `AsyncEngine` and one `async_sessionmaker` without external I/O;
4. provide a bounded startup connection check and explicit engine disposal;
5. expose a narrow session-factory getter for request dependency composition.

Build the URL with `sqlalchemy.engine.URL.create`, including rounded positive `connect_timeout` and validated `sslmode` query values; do not concatenate credentials into a string. Configure `pool_size`, `max_overflow`, `pool_timeout`, `pool_recycle`, `pool_pre_ping=True`, and `pool_use_lifo=True` on `create_async_engine`.

```python
database_settings = load_database_settings()
database_engine = create_database_engine(database_settings)
database_session_factory = async_sessionmaker(
    database_engine,
    class_=AsyncSession,
    expire_on_commit=False,
)


async def check_database_connection() -> None:
    async with database_engine.connect():
        pass


async def close_database_engine() -> None:
    await database_engine.dispose()


def get_database_session_factory() -> async_sessionmaker[AsyncSession]:
    return database_session_factory
```

Creating the SQLAlchemy engine and session factory at module scope is acceptable because both are inert. Importing the adapter must not connect, create an `AsyncSession`, authenticate remotely, create schema, or migrate data. A session is a mutable unit of work and must be created inside a request dependency or another explicit short-lived scope, never at module level.

Do not put ORM statements, mapped models, or feature transaction rules in the PostgreSQL adapter. The adapter represents engine/session infrastructure; models represent schema; repositories represent persistence behavior; services represent application behavior.

## Lifespan

Keep application startup and shutdown in `app/middleware/lifespan.py`:

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI

from app.adapters.postgresql import (
    check_database_connection,
    close_database_engine,
    database_session_factory,
)
from app.config.logger import get_logger


logger = get_logger(__name__)


@asynccontextmanager
async def lifespan(application: FastAPI):
    logger.info("Starting API", extra={"event": "application_starting"})
    await check_database_connection()
    application.state.database_session_factory = database_session_factory
    try:
        logger.info("API started", extra={"event": "application_started"})
        yield
    finally:
        await close_database_engine()
        logger.info("API stopped", extra={"event": "application_stopped"})
```

The engine is lazy, so startup performs one explicit connection checkout and release. If PostgreSQL is unavailable, fail startup rather than serving requests with a broken persistence dependency. Dispose the engine in `finally`; for multiple resources, check them in dependency order and close them in reverse order.

Storing the session factory in `application.state` is optional when the adapter getter is already authoritative. Use state only from FastAPI dependency composition, not from services or repositories. Never store an `AsyncSession` in application state because a session cannot be shared safely between concurrent requests.

Do not call `Base.metadata.create_all`, run Alembic upgrades, seed data, retry forever, or launch workers from lifespan. Schema management and background processing are separate operational workflows.

## HTTP middleware

Put one cross-cutting concern in each middleware module. `middleware/logging.py` may log method, path, status, resolved handler namespace, file, and line. Other valid concerns include correlation IDs or explicitly required response headers.

Middleware must call the downstream application exactly once unless its documented purpose is to reject the request. It must preserve response and exception behavior. Avoid reading request bodies unless the concern requires it, because doing so changes streaming and memory behavior. Never log authorization headers, cookies, request bodies, query values, or response data indiscriminately.

Middleware is not a feature controller. It does not construct services, query repositories, or encode one endpoint's business rule. Endpoint-specific behavior stays in the route or service.

## Structured logging

`app/config/logger.py` owns the JSON formatter, `dictConfig` setup, and logger lookup. `src/etc/logger.yaml` owns handler, formatter, level, and propagation configuration for application, Uvicorn, and Gunicorn logger namespaces.

Emit one JSON object per record with at least a UTC timestamp, level, logger namespace/method/file/line, and message. Add structured fields with `extra`, using stable keys and event names. Format exceptions and stack information when present.

Each module obtains `logger = get_logger(__name__)`. Only composition configures logging. Do not reconfigure logging in routes, services, repositories, adapters, or middleware. Avoid duplicate handlers by setting propagation deliberately in YAML.

Log lifecycle transitions and bounded technical metadata such as database host, port, database name, and pool sizes only when the target project's security rules permit them. Never log passwords, full connection strings, access tokens, authorization values, or secret configuration mappings.

## Common helpers

Use `app/common` for pure shared behavior with no FastAPI, driver, filesystem, or network coupling. A timestamp helper may return UTC ISO 8601 with millisecond precision and a `Z` suffix:

```python
from datetime import datetime, timezone


def utc_timestamp() -> str:
    return datetime.now(timezone.utc).isoformat(timespec="milliseconds").replace("+00:00", "Z")
```

Route-specific timezone formatting may remain in the route when it is part of the endpoint response. Move it to `common` only if several independent modules share the same semantic conversion. Do not use `common` for database models, API schemas, or adapter settings.

## Alembic layout

Keep migrations colocated under the application package:

```text
app/migrations/
|-- alembic.ini
|-- env.py
|-- script.py.mako
`-- versions/
    |-- 0001_initial_schema.py
    `-- 0002_create_users.py
```

`alembic.ini` points `script_location` at its own directory and makes the runtime application importable. Do not store a plaintext database URL there. `env.py` imports `database_settings` and `Base`, sets `target_metadata = Base.metadata`, and returns `database_settings.url` so runtime and migration connection parameters stay consistent.

The migration environment uses a migration-local synchronous `create_engine(database_url(), poolclass=pool.NullPool)` even though the application uses `AsyncEngine`; the `postgresql+psycopg` driver supports both. Keep that engine local to the migration command, dispose it after use, and configure offline and online modes explicitly.

Import every concrete mapped model module before Alembic reads `Base.metadata`. The normal contract is for `app.models.__init__` to import `Base` and the model classes or modules. If a model is not imported, autogenerate cannot see its table even though its file exists.

## Revision rules

Use a stable ordered revision identifier and a concise intent in the filename. Each revision declares `revision`, `down_revision`, `branch_labels`, and `depends_on`. Keep the chain unambiguous unless the project explicitly uses branches.

Put schema operations in `upgrade` and the reverse operations in `downgrade`. An intentionally empty baseline revision may use `pass`; later revisions should describe real transitions. Use SQLAlchemy/Alembic schema constructs instead of ORM sessions or application repositories.

Seed data inside a schema migration only when it is an intentional, deterministic part of the schema/application baseline. Operational or environment-specific data does not belong in a migration. Avoid importing service or route code from revisions.

When a mapped model adds or changes a table, column, relationship constraint, index, or naming rule, include the corresponding new revision in the same change unless the schema is owned externally. Review autogenerated operations manually, especially constraint names, server defaults, nullability, indexes, data backfills, and destructive changes. Never rewrite an already released migration merely to match current model code.

Creating a revision file is a source-code change. Running `upgrade`, `downgrade`, or any seed operation mutates a database and requires whatever explicit authorization the target project mandates.

## Runtime and data review checklist

- YAML lookup is anchored to `src/etc`, not the current working directory.
- Generic config loading validates file name, suffix, and root mapping.
- Callers receive isolated configuration data.
- Adapter settings are typed and validate all required technical values.
- Importing the adapter creates only a lazy engine and session factory; it performs no external I/O.
- Engine pool size, overflow, timeout, recycle, pre-ping, and LIFO behavior are explicit.
- Lifespan checks one connection before yielding and disposes the engine in `finally`.
- Live `AsyncSession` objects are request-scoped and never global or application-scoped.
- Middleware remains cross-cutting and does not contain use cases.
- Logging is configured once and secrets are excluded.
- Models own schema mappings; repositories own ORM statements; adapters own engine/session infrastructure.
- Alembic receives complete `Base.metadata` after every model module is imported.
- Migrations are separate from API startup and request handling.
- Every mapped-schema change is paired with an appropriate reviewed revision.
