# Routing and feature implementation

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
from fastapi import APIRouter, Depends
from pydantic import BaseModel

from app.adapters.postgresql import get_database_pool
from app.repositories.users import UserRepository
from app.services.users import UserService


class UserResponse(BaseModel):
    username: str


class UsersResponse(BaseModel):
    users: list[UserResponse]


router = APIRouter(tags=["users.list"])


def get_user_service() -> UserService:
    pool = get_database_pool()
    repository = UserRepository(pool)
    return UserService(repository)


@router.get("", response_model=UsersResponse)
async def list_users(service: UserService = Depends(get_user_service)) -> UsersResponse:
    users = await service.list_users()
    return UsersResponse(
        users=[UserResponse(username=user.username) for user in users],
    )
```

The dependency function is the local composition point. It retrieves a ready adapter resource, injects it into the repository, and injects the repository into the service. It must not perform the use case, open a new long-lived pool, or execute SQL.

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

HTTP status codes, headers, cookies, and `HTTPException` mapping remain at this boundary. Business decisions remain in the service. SQL, driver calls, pool management, and YAML access do not belong in handlers.

A pure presentation endpoint such as a root status, current date, or current time response may call the standard library and a small `common` helper directly. Do not manufacture an empty service or repository only for structural symmetry. Introduce the service boundary as soon as the endpoint represents an application use case, applies business rules, coordinates dependencies, performs persistence, or calls an external system.

## Service pattern

`app/services/users.py` owns user use cases without knowing about HTTP:

```python
from app.repositories.users import UserRecord, UserRepository


class UserService:
    def __init__(self, repository: UserRepository):
        self._repository = repository

    async def list_users(self) -> list[UserRecord]:
        return await self._repository.list_users()
```

Add validation, authorization decisions based on already resolved identity/context, coordination across repositories, or result transformation here when they are application rules. Keep transport parsing and HTTP error codes in `api`.

Inject every required collaborator through the constructor. Avoid constructing repositories inside service methods, reading adapter globals, or importing FastAPI dependencies. If multiple service operations share the same repository set, keep them on the same cohesive service class.

## Repository pattern

`app/repositories/users.py` owns persistence records and SQL:

```python
from dataclasses import dataclass

from psycopg_pool import AsyncConnectionPool


@dataclass(frozen=True)
class UserRecord:
    username: str


class UserRepository:
    def __init__(self, pool: AsyncConnectionPool):
        self._pool = pool

    async def list_users(self) -> list[UserRecord]:
        async with self._pool.connection() as connection:
            cursor = await connection.execute(
                "SELECT username FROM users ORDER BY username"
            )
            rows = await cursor.fetchall()

        return [UserRecord(username=row[0]) for row in rows]
```

Use immutable typed records for repository results. Keep row order and mapping explicit. Parameterize user-controlled values through the driver. Keep connections and cursors scoped with context managers so they return to the pool reliably.

The repository owns persistence semantics such as whether an absent row returns `None` or raises a repository-specific error. The service decides what absence means to the application. The route decides how the application result becomes an HTTP status and body.

## Dependency composition

Use FastAPI `Depends` in the route layer. A small dependency provider may construct a repository and service per request while reusing the application-scoped pool. This keeps service and repository constructors explicit and makes dependency override straightforward in HTTP tests.

When several routes in the same group use identical construction, move the provider to `api/<group>/dependencies.py`; do not move it into `common`. Keep it in `api` because it imports FastAPI-aware composition concerns and assembles route dependencies.

Do not use `application.state` from services or repositories. If the project uses state as the authoritative request-time source, read it in a FastAPI dependency and pass the resolved client/pool downward explicitly.

## Error ownership

Keep errors at the layer that understands them:

- adapters report invalid technical settings and connection lifecycle failures;
- repositories report persistence-specific failures or absence semantics;
- services report application rule failures;
- API handlers or registered API exception handlers translate known application failures into HTTP responses;
- unexpected failures continue to structured logging without being converted into a successful response.

Do not raise `HTTPException` from services or repositories. Do not catch every exception in a repository and replace it with a generic message that removes useful cause information. Never expose connection strings, SQL parameters containing secrets, credentials, or internal stack traces in public response bodies.

## Adding a feature

For a persistence-backed API feature:

1. Define the public path, method, input, output, status, and error behavior.
2. Add or extend the group router and explicitly register its leaf router.
3. Define endpoint-local Pydantic request and response models.
4. Define the service operation and its explicit collaborator inputs.
5. Define repository records and parameterized persistence operations.
6. Compose adapter handle, repository, and service in a FastAPI dependency provider.
7. Map the service result into the response model in the handler.
8. Add a new Alembic revision if the persistence contract requires a schema change.
9. Update focused tests and validate router reachability, service behavior, repository mapping, and migration structure without starting infrastructure by default.

## Routing review checklist

- `main.py` includes only `app.api.router`, not leaf routers.
- Parent prefixes are declared once and leaf paths do not duplicate them.
- Every router inclusion is explicit and imported with a clear alias.
- Each endpoint declares a response model unless the response intentionally has no body.
- HTTP request and response models remain in `api`.
- Dependency providers assemble collaborators but perform no use case or I/O setup.
- Handlers delegate to services and contain no SQL.
- Services contain no FastAPI types or HTTP status decisions.
- Repositories contain no HTTP models and receive clients/pools explicitly.
- Async handlers do not call blocking I/O directly.
