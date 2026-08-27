# Python API routing and feature implementation

Read this reference before adding routes, composing router groups, defining HTTP models, or implementing a service and repository path.

## Router hierarchy

Compose routers explicitly in three levels:

```text
main.py
`-- app.api.router                         prefix=/api
    `-- app.api.<group>.router             prefix=/<group>
        `-- app.api.<group>.<endpoint>.router
```

`main.py` imports only the top-level router. `app/api/__init__.py` imports group routers. A group `__init__.py` imports its leaf routers. Do not import every leaf into `main.py`, scan packages dynamically, or hide routing in a registry.

### Top-level router

`app/api/__init__.py` owns the stable API prefix and group composition:

```python
from fastapi import APIRouter
from pydantic import BaseModel

from app.api.users import router as users_router
from app.common.timestamps import utc_timestamp


class ApiResponse(BaseModel):
    response: str
    timestamp: str


router = APIRouter(prefix="/api", tags=["api"])


@router.get("", response_model=ApiResponse)
def read_api() -> ApiResponse:
    return ApiResponse(response="/api", timestamp=utc_timestamp())


router.include_router(users_router)
```

The optional group-root response documents or probes the group itself. Use an empty decorator path for the exact prefix rather than creating an accidental trailing-slash variant. Omit the route if the application does not need it, but keep router composition in this file.

### Group router

`app/api/users/__init__.py` owns `/users`, group tags, and its leaf modules:

```python
from fastapi import APIRouter

from app.api.users.list import router as list_router
from app.api.users.read import router as read_router


router = APIRouter(prefix="/users", tags=["users"])
router.include_router(list_router)
router.include_router(read_router)
```

Use one group per coherent HTTP resource or functional area. A group may define its own root operation when that operation naturally represents the resource collection. Avoid redundant prefixes in leaf routers: if the group owns `/users`, a list leaf uses `""` and an item leaf uses `"/{user_id}"`, not `/users` again.

### Leaf route

A leaf module owns a cohesive operation or small related endpoint set. It declares HTTP-only models, one local router, dependency providers, route decorators, and handlers. It must be understandable without reading `main.py`.

```python
from collections.abc import AsyncIterator

from fastapi import APIRouter, Depends
from pydantic import BaseModel
from sqlalchemy.ext.asyncio import AsyncSession

from app.adapters.postgresql import get_database_session_factory
from app.repositories.users import UserRepository
from app.services.users import UserService


class UserResponse(BaseModel):
    username: str


class UsersResponse(BaseModel):
    users: list[UserResponse]


router = APIRouter(tags=["users.list"])


async def get_database_session() -> AsyncIterator[AsyncSession]:
    session_factory = get_database_session_factory()
    async with session_factory() as session:
        yield session


def get_user_service(
    session: AsyncSession = Depends(get_database_session),
) -> UserService:
    repository = UserRepository(session)
    return UserService(repository)


@router.get("", response_model=UsersResponse)
async def list_users(service: UserService = Depends(get_user_service)) -> UsersResponse:
    users = await service.list_users()
    return UsersResponse(
        users=[UserResponse(username=user.username) for user in users],
    )
```

The session dependency obtains the application-scoped `async_sessionmaker`, creates exactly one `AsyncSession`, yields it for the request, and closes it through the context manager. The service dependency injects that session into the repository and the repository into the service. Neither dependency performs the use case, executes ORM statements, commits implicitly, or constructs another engine.

Move `get_database_session` to `app/api/dependencies.py` when several route groups need it. Keep a feature-specific service provider with the feature unless several sibling modules share the exact same composition.

## Choosing route files

Prefer one leaf module per independently understandable endpoint concern, not mechanically one file per HTTP verb. These are reasonable shapes:

```text
api/users/
|-- __init__.py
|-- list.py
|-- detail.py
`-- sessions.py
```

```text
api/help/
|-- __init__.py
|-- date.py
|-- time.py
`-- users.py
```

Keep a small resource in one `users.py` leaf if its operations share contracts and dependencies and the file remains cohesive. Split it when unrelated request models, authorization paths, dependencies, or response behavior make navigation harder. Do not create `routes.py`, `handlers.py`, `controllers.py`, and `schemas.py` for a single tiny endpoint merely to imitate a larger design.

## HTTP model rules

Pydantic models in `api` describe the public HTTP contract. Use separate request and response names when direction matters, such as `CreateUserRequest`, `UserResponse`, and `UsersResponse`. Do not return database rows, cursors, arbitrary mappings, or driver records directly from a route.

Map service results explicitly into response models. This prevents persistence fields from becoming public accidentally and makes response changes visible at the boundary. A timestamp or envelope added to the HTTP response belongs in the API model even when its value comes from `common`.

Keep endpoint-local models in the leaf file. Move models to `api/<group>/models.py` only when several sibling route modules use the same HTTP contract. Never move an HTTP model into a repository or use a repository record as `response_model` simply to avoid mapping.

## Handler rules

Choose `async def` when the handler awaits service or repository-backed work. A pure CPU-light or immediate route may use `def`, allowing FastAPI to execute it appropriately. Do not make a complete call chain async when no operation awaits I/O, and do not block the event loop with synchronous network or database work inside `async def`.

A handler should normally perform only these actions:

1. receive validated path, query, header, body, and dependency values;
2. translate HTTP inputs to explicit service arguments;
3. await or call one application use case;
4. translate the result into the declared response model.

HTTP status codes, headers, cookies, and `HTTPException` mapping remain at this boundary. Business decisions remain in the service. SQLAlchemy statements, session lifecycle, engine management, and YAML access do not belong in handlers.

A pure presentation endpoint such as a root status, current date, or current time response may call the standard library and a small `common` helper directly. Do not manufacture an empty service or repository only for structural symmetry. Introduce the service boundary as soon as the endpoint represents an application use case, applies business rules, coordinates dependencies, performs persistence, or calls an external system.

## Service pattern

`app/services/users.py` owns user use cases without knowing about HTTP or session management:

```python
from app.models.users import User
from app.repositories.users import UserRepository


class UserService:
    def __init__(self, repository: UserRepository):
        self._repository = repository

    async def list_users(self) -> list[User]:
        return await self._repository.list_users()
```

Add validation, authorization decisions based on already resolved identity/context, coordination across repositories, or result transformation here when they are application rules. Keep transport parsing and HTTP error codes in `api`. A service may use a mapped entity type returned by its repository when that representation is sufficient, but it does not construct `select()` statements, receive an `AsyncSession`, or call `commit()`.

Inject every required collaborator through the constructor. Avoid constructing repositories inside service methods, reading adapter globals, or importing FastAPI dependencies. If multiple service operations share the same repository set, keep them on the same cohesive service class.

## Repository pattern

`app/repositories/users.py` owns ORM statements and persistence result semantics:

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.users import User


class UserRepository:
    def __init__(self, session: AsyncSession):
        self._session = session

    async def list_users(self) -> list[User]:
        statement = select(User).order_by(User.username)
        users = await self._session.scalars(statement)
        return list(users)
```

Use SQLAlchemy expression APIs and typed mapped attributes. Choose `scalars`, `scalar_one`, `scalar_one_or_none`, or row results to match the query. Use eager loading when returned relationships are required after repository execution; do not rely on implicit lazy I/O after the request session closes.

The repository owns persistence semantics such as whether an absent row returns `None` or raises a repository-specific error. It may map ORM entities into frozen service-safe records when callers should not depend on persistence models. The service decides what absence means to the application, and the route maps the result explicitly into a Pydantic response.

## Dependency composition

Use FastAPI `Depends` in the route layer. The session dependency reuses the application-scoped session factory but creates a fresh `AsyncSession` per request. A small provider constructs a repository and service from that session. This keeps constructors explicit and makes the session or service dependency straightforward to override in HTTP tests.

When several routes in the same group use identical construction, move the provider to `api/<group>/dependencies.py`; do not move it into `common`. Keep it in `api` because it imports FastAPI-aware composition concerns and assembles route dependencies.

Do not use `application.state` from services, repositories, or models. If application state is the authoritative source for `database_session_factory`, read it in a FastAPI dependency and pass the newly created `AsyncSession` downward explicitly. Never put a live session in application state.

## Write transactions

Session lifetime and transaction lifetime are related but distinct. The API dependency owns session creation and closure. The complete application use case owns the write transaction; choose one visible convention and keep it consistent.

A repository operation that is itself the complete atomic persistence use case may use `async with session.begin()`. When a service coordinates several repositories atomically, use an application-level unit-of-work abstraction or another explicit transaction coordinator so the service stays free of SQLAlchemy APIs while all repositories share the same session and transaction. Repository helpers may `flush` when generated values are needed, but they must not each call `commit()` independently.

On failure, roll back the active transaction and re-raise the original application or persistence error. Do not catch an integrity error and continue using a failed session without rollback. Do not commit in a dependency teardown after the handler has already produced a response unless the project's FastAPI dependency scope and error behavior explicitly guarantee that contract.

## Error ownership

Keep errors at the layer that understands them:

- adapters report invalid technical settings and connection lifecycle failures;
- models enforce persistence shape and database-level constraints;
- repositories report ORM/persistence failures or absence semantics;
- services report application rule failures;
- API handlers or registered API exception handlers translate known application failures into HTTP responses;
- unexpected failures continue to structured logging without being converted into a successful response.

Do not raise `HTTPException` from services or repositories. Do not catch every exception in a repository and replace it with a generic message that removes useful cause information. Never expose connection strings, SQL parameters containing secrets, credentials, or internal stack traces in public response bodies.

## Adding a feature

For a persistence-backed API feature:

1. Define the public path, method, input, output, status, and error behavior.
2. Add or extend the group router and explicitly register its leaf router.
3. Define endpoint-local Pydantic request and response models.
4. Define or update the SQLAlchemy model and ensure its module is registered for `Base.metadata`.
5. Define the repository's ORM statements and persistence result semantics around an injected `AsyncSession`.
6. Define the service operation and its explicit repository collaborators.
7. Compose request session, repository, and service in FastAPI dependency providers.
8. Map the service result into the response model in the handler.
9. Add and review a new Alembic revision when the mapped schema changes.
10. Update focused tests and validate router reachability, session scope, service behavior, ORM statements, model registration, and migration structure without starting infrastructure by default.

## Routing review checklist

- `main.py` includes only `app.api.router`, not leaf routers.
- Parent prefixes are declared once and leaf paths do not duplicate them.
- Every router inclusion is explicit and imported with a clear alias.
- Each endpoint declares a response model unless the response intentionally has no body.
- HTTP request and response models remain in `api`.
- Dependency providers assemble collaborators but perform no use case or I/O setup.
- Handlers delegate to services and contain no SQL.
- Services contain no FastAPI types, sessions, ORM statements, or HTTP status decisions.
- Repositories contain no HTTP models and receive one `AsyncSession` explicitly.
- A live session exists only inside its request or explicit unit-of-work scope.
- ORM entities are mapped explicitly to response models and are not serialized implicitly.
- Async handlers do not call blocking I/O directly.
