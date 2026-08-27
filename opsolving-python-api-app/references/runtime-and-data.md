# Runtime, configuration, adapters, and migrations

Read this reference before changing YAML configuration, structured logging, middleware, lifespan behavior, external-resource adapters, database pool settings, or Alembic migrations.

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
- pool minimum, maximum, acquisition timeout, idle timeout, and maximum lifetime;
- relationships such as maximum pool size being greater than or equal to minimum size.

Use frozen dataclasses or another explicit typed structure. Reject booleans where an integer is expected. Parse human-readable duration values into seconds once at the adapter boundary. Validate enumerations such as supported SSL modes before constructing the external client.

YAML may use deployment-oriented camel-case names such as `connectTimeout` and `maxLifetime`; Python settings use snake-case names such as `connect_timeout` and `max_lifetime`. Perform that translation in the adapter. Do not leak raw nested mappings throughout services or repositories.

## Adapter lifecycle

An adapter module for a long-lived resource has four responsibilities:

1. load and validate technology-specific settings;
2. construct a client or pool without external I/O;
3. provide explicit async or sync open/close functions;
4. expose a narrow getter for dependency composition.

For PostgreSQL, build the connection string with the driver's safe helper rather than string concatenation. Configure bounded connect, acquire, idle, and lifetime timeouts. Construct `AsyncConnectionPool` with `open=False`; opening belongs to lifespan startup.

```python
database_settings = load_database_settings()
database_pool = create_database_pool(database_settings)


async def open_database_pool() -> None:
    await database_pool.open(
        wait=True,
        timeout=database_settings.connection.connect_timeout,
    )


async def close_database_pool() -> None:
    await database_pool.close()


def get_database_pool() -> AsyncConnectionPool:
    return database_pool
```

Module-level construction is acceptable only when it is inert. Importing the adapter must not connect, authenticate remotely, run health checks, create schema, or migrate data. If a client library cannot be constructed inertly, construct it during lifespan and expose it through application state or an explicitly initialized holder resolved by an API dependency.

Do not put SQL in the PostgreSQL adapter. Do not put payment, user, or other feature rules in an external-service adapter. The adapter represents the technology contract; repositories and services represent application behavior.

## Lifespan

Keep application startup and shutdown in `app/middleware/lifespan.py`:

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI

from app.adapters.postgresql import close_database_pool, database_pool, open_database_pool
from app.config.logger import get_logger


logger = get_logger(__name__)


@asynccontextmanager
async def lifespan(application: FastAPI):
    logger.info("Starting API", extra={"event": "application_starting"})
    await open_database_pool()
    application.state.database_pool = database_pool
    try:
        logger.info("API started", extra={"event": "application_started"})
        yield
    finally:
        await close_database_pool()
        logger.info("API stopped", extra={"event": "application_stopped"})
```

Open multiple resources in dependency order and close them in reverse order. If startup of a later resource fails, ensure already opened resources are closed. Keep shutdown in `finally`. Let unrecoverable startup failure prevent readiness instead of serving requests with a missing dependency.

Storing a handle in `application.state` is optional when the adapter getter is already authoritative. Use state when FastAPI dependency functions need request-local access to the active application, not as a hidden service locator consumed throughout the application.

Do not run Alembic upgrades, seed data, retry forever, or launch workers from lifespan. Those are separate operational workflows.

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

`alembic.ini` points `script_location` at its own directory and makes the runtime application importable. Do not store a plaintext database URL there. `env.py` builds the migration URL from validated adapter settings so runtime and migration connection parameters stay consistent.

The migration environment may use a synchronous SQLAlchemy engine even when the application repository uses Psycopg's async pool. Keep that engine local to the migration command and dispose it after use. Configure offline and online modes explicitly.

## Revision rules

Use a stable ordered revision identifier and a concise intent in the filename. Each revision declares `revision`, `down_revision`, `branch_labels`, and `depends_on`. Keep the chain unambiguous unless the project explicitly uses branches.

Put schema operations in `upgrade` and the reverse operations in `downgrade`. An intentionally empty baseline revision may use `pass`; later revisions should describe real transitions. Use SQLAlchemy/Alembic schema constructs instead of application repositories.

Seed data inside a schema migration only when it is an intentional, deterministic part of the schema/application baseline. Operational or environment-specific data does not belong in a migration. Avoid importing service or route code from revisions.

When a repository starts reading or writing a new table, column, constraint, or index, include the corresponding new revision in the same change unless the schema is owned externally. Never rewrite an already released migration merely to match current repository code.

Creating a revision file is a source-code change. Running `upgrade`, `downgrade`, or any seed operation mutates a database and requires whatever explicit authorization the target project mandates.

## Runtime and data review checklist

- YAML lookup is anchored to `src/etc`, not the current working directory.
- Generic config loading validates file name, suffix, and root mapping.
- Callers receive isolated configuration data.
- Adapter settings are typed and validate all required technical values.
- Importing adapters performs no external I/O.
- Clients and pools have bounded timeouts and explicit open/close functions.
- Lifespan opens resources before yielding and closes them in `finally`.
- Middleware remains cross-cutting and does not contain use cases.
- Logging is configured once and secrets are excluded.
- Repositories, not adapters, own SQL.
- Migrations are separate from API startup and request handling.
- Every schema-dependent repository change is paired with the appropriate new revision.
