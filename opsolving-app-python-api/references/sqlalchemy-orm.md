# SQLAlchemy ORM models, sessions, repositories, and migrations

Read this reference before adding or changing declarative models, relationships, ORM statements, session dependencies, write transactions, model registration, or Alembic metadata integration.

## Persistence structure

```text
app/
|-- adapters/
|   `-- postgresql.py          AsyncEngine and async_sessionmaker
|-- api/
|   |-- dependencies.py       request-scoped AsyncSession when shared
|   `-- <group>/               service/repository composition
|-- models/
|   |-- __init__.py            Base export and concrete model registration
|   |-- base.py                DeclarativeBase and naming convention
|   `-- <resource>.py          mapped tables and relationships
|-- repositories/
|   `-- <resource>.py          ORM statements against injected session
|-- services/
|   `-- <resource>.py          use cases without session management
`-- migrations/
    |-- env.py                 database URL and Base.metadata
    `-- versions/              reviewed schema revisions
```

The adapter owns reusable engine and factory infrastructure. The API dependency owns the lifetime of one session. Models own mapped schema. Repositories own ORM statements. Services own application decisions. Alembic owns schema transitions.

Do not move these responsibilities into a single database module. In particular, do not put mapped classes in repositories, query methods on `Base`, request sessions in application state, or schema creation in lifespan.

## Declarative base

Keep one shared base in `app/models/base.py`:

```python
from sqlalchemy import MetaData
from sqlalchemy.ext.asyncio import AsyncAttrs
from sqlalchemy.orm import DeclarativeBase


NAMING_CONVENTION = {
    'ix': 'ix_%(column_0_label)s',
    'uq': 'uq_%(table_name)s_%(column_0_name)s',
    'ck': 'ck_%(table_name)s_%(constraint_name)s',
    'fk': 'fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s',
    'pk': 'pk_%(table_name)s',
}


class Base(AsyncAttrs, DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)
```

Use the deterministic naming convention for every mapped table. Stable constraint and index names make Alembic diffs reviewable and allow downgrade operations to address the same objects reliably.

All mapped classes inherit this one `Base`. Do not create a base per feature, call `declarative_base()` in model modules, or replace the naming convention for one table.

## Model modules

Define one cohesive mapped resource or tightly related persistence group per module:

```python
from datetime import datetime

from sqlalchemy import DateTime, String, func
from sqlalchemy.orm import Mapped, mapped_column

from app.models.base import Base


class User(Base):
    __tablename__ = 'users'

    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(
        String(255),
        nullable=False,
        unique=True,
    )
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        nullable=False,
        server_default=func.now(),
    )
```

Use SQLAlchemy 2 typed mappings with `Mapped[...]` and `mapped_column`. Make nullability, lengths, uniqueness, indexes, foreign keys, delete behavior, and server defaults explicit when they are part of the schema contract.

ORM models are persistence models. Do not add FastAPI `Depends`, Pydantic request/response inheritance, HTTP status decisions, repository queries, commits, or unrelated business workflows to them. Small persistence-local invariants or relationship helpers are acceptable when they do not perform I/O implicitly.

## Model registration

Alembic sees only mapped classes that Python imported before `target_metadata` is inspected. Register every model from `app/models/__init__.py`:

```python
from app.models.base import Base
from app.models.users import User


__all__ = ['Base', 'User']
```

When adding `models/orders.py`, import `Order` or the module from this package before generating or running an autogenerate comparison. A file on disk does not register its table automatically.

Avoid runtime directory scanning for model discovery. Explicit imports expose the metadata surface, fail predictably on import errors, and keep Alembic behavior deterministic.

## Engine and session factory

The PostgreSQL adapter creates one lazy async engine and one factory:

```python
from math import ceil

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine


database_engine = create_async_engine(
    database_settings.url,
    pool_size=database_settings.pool.size,
    max_overflow=database_settings.pool.max_overflow,
    pool_timeout=database_settings.pool.acquire_timeout,
    pool_recycle=ceil(database_settings.pool.recycle),
    pool_pre_ping=True,
    pool_use_lifo=True,
)

database_session_factory = async_sessionmaker(
    database_engine,
    class_=AsyncSession,
    expire_on_commit=False,
)
```

The engine owns the connection pool; the session factory creates units of work. `expire_on_commit=False` keeps already loaded scalar state usable after a successful commit, but it does not make lazy relationships safe after session closure.

Creating the engine and factory performs no connection. Do not create a module-global `AsyncSession`, do not share one session across concurrent requests, and do not create a second engine inside a repository.

## Request-scoped session

Create and close sessions through a FastAPI dependency:

```python
from collections.abc import AsyncIterator

from sqlalchemy.ext.asyncio import AsyncSession

from app.adapters.postgresql import get_database_session_factory


async def get_database_session() -> AsyncIterator[AsyncSession]:
    session_factory = get_database_session_factory()
    async with session_factory() as session:
        yield session
```

This dependency owns session lifetime only. It does not execute feature queries and should not silently commit after every handler. Put it in `app/api/dependencies.py` when several route groups use it, or beside one feature while the dependency is feature-local.

If the application resolves the factory from `request.app.state`, do so inside this dependency and still create a fresh session with `async with`. Services, repositories, and models must not reach into application state.

## Repository reads

Inject the session and express queries with SQLAlchemy 2 statements:

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.users import User


class UserRepository:
    def __init__(self, session: AsyncSession):
        self._session = session

    async def list_users(self) -> list[User]:
        statement = select(User).order_by(User.username)
        result = await self._session.scalars(statement)
        return list(result)

    async def get_user(self, user_id: int) -> User | None:
        return await self._session.get(User, user_id)
```

Use `session.get` for a primary-key lookup. Use `scalars(select(...))` for a mapped-entity collection, `scalar_one_or_none` when zero-or-one is the expected cardinality, and explicit row results when selecting several independent expressions.

Keep filtering, ordering, pagination, locks, and eager-loading options visible in the repository. Parameter values passed through expression APIs are bound safely; do not interpolate request values into textual SQL.

## Relationships and async loading

Declare relationships with typed `Mapped` attributes and matching foreign keys. Set `back_populates`, cascade, passive deletes, and database `ondelete` behavior deliberately rather than relying on unrelated defaults.

Async ORM code must not trigger surprise lazy I/O after the session closes. Use `selectinload`, `joinedload`, or an explicit secondary query when the use case needs related data. `AsyncAttrs` provides awaitable attributes, but explicit repository loading is normally easier to review and test.

Do not serialize a mapped object recursively through relationships. Map only the fields promised by the HTTP response, and avoid cycles or accidental disclosure of persistence-only columns.

## Repository writes

Use `session.add` for new mapped entities, SQLAlchemy statements for bulk operations when appropriate, and `await session.flush()` when generated identifiers or constraint checks are required before the transaction ends:

```python
class UserRepository:
    def __init__(self, session: AsyncSession):
        self._session = session

    async def add_user(self, username: str) -> User:
        user = User(username=username)
        self._session.add(user)
        await self._session.flush()
        return user
```

`flush` synchronizes pending changes within the current transaction; it is not a commit. Do not call `commit()` from every repository method. Several repository operations in one use case must be able to succeed or roll back atomically.

For update/delete operations, make missing-row and concurrency semantics explicit. Do not silently turn a zero-row update into success when the application expects not-found or conflict behavior.

## Transaction ownership

Choose one visible transaction boundary around the complete write use case. A single repository method that is itself the entire atomic operation may open `async with session.begin()`. When a service coordinates multiple repositories, provide an application-level unit-of-work abstraction or transaction coordinator backed by their shared session.

The service should not import `AsyncSession`, construct statements, or know engine details. A SQLAlchemy-backed unit of work may live beside repositories and expose application-level `commit`, rollback, and repository access through a framework-light contract.

On integrity, serialization, or application failure, roll back and re-raise or translate at the correct layer. After a flush or commit error, do not reuse the session until rollback has restored it to a usable state.

## ORM results and HTTP models

Never declare an ORM mapped class as the FastAPI request or response model. Pydantic models in `api` are the public HTTP contract; mapped classes in `app/models` are the persistence contract.

A route maps an entity or service result explicitly:

```python
return UserResponse(
    id=user.id,
    username=user.username,
)
```

When a service should remain independent of ORM lifetime or persistence-only fields, let the repository return a frozen dataclass or another service-safe result. Do not copy every entity mechanically; introduce a separate result type when it creates a real boundary.

## Alembic metadata and revisions

Configure Alembic with the same settings and metadata as the application:

```python
from sqlalchemy.engine import URL

from app.adapters.postgresql import database_settings
from app.models import Base


target_metadata = Base.metadata


def database_url() -> URL:
    return database_settings.url
```

The online migration path may use synchronous `create_engine(database_url(), poolclass=pool.NullPool)` while the application uses `create_async_engine`; Psycopg 3 supports both through the same `postgresql+psycopg` URL.

Autogenerate is a draft, not proof. Review table/column types, nullability, constraint and index names, server defaults, foreign-key delete behavior, data migrations, destructive operations, downgrade correctness, and whether every model was registered before accepting a revision.

Do not use `Base.metadata.create_all()` as a migration substitute. Do not run migrations in lifespan. Do not edit a released migration to match a later model change; add the next ordered revision.

## ORM review checklist

- Every mapped class inherits the one shared `Base`.
- Constraint names use the shared deterministic metadata convention.
- Every concrete model module is imported before Alembic reads metadata.
- Mapped columns use explicit SQLAlchemy 2 `Mapped` typing.
- Relationships and foreign-key delete behavior agree.
- The adapter owns one lazy engine and one session factory.
- A fresh `AsyncSession` is created and closed for each request or unit of work.
- Repositories receive sessions explicitly and use expression APIs.
- Required relationships are eagerly loaded before session closure.
- Write operations have one visible transaction boundary.
- Repository helpers do not each commit independently.
- ORM entities are mapped explicitly to HTTP responses.
- Every mapped-schema change has a reviewed Alembic revision.
- Startup checks connectivity but never creates or migrates schema.
