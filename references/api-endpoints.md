# API Endpoints

## Router Setup

Use `DishkaRoute` as the route class to enable automatic DI injection without `@inject` decorators.

```python
from dishka.integrations.fastapi import DishkaRoute, FromDishka
from fastapi import APIRouter, HTTPException, status

from app.users.schemas import UserCreate, UserRead, UserUpdate
from app.users.service import UserService

router = APIRouter(route_class=DishkaRoute)
```

## Endpoint Pattern

Inject services via `FromDishka[ServiceType]`. Endpoints handle HTTP concerns only — status codes, error mapping, response models.

```python
@router.post("/", response_model=UserRead, status_code=status.HTTP_201_CREATED)
async def create_user(
    user_in: UserCreate,
    user_service: FromDishka[UserService],
):
    try:
        user = await user_service.create(user_in)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    return user


@router.get("/", response_model=list[UserRead])
async def list_users(
    user_service: FromDishka[UserService],
    skip: int = 0,
    limit: int = 100,
):
    return await user_service.find_all(skip=skip, limit=limit)


@router.get("/{user_id}", response_model=UserRead)
async def get_user(
    user_id: int,
    user_service: FromDishka[UserService],
):
    user = await user_service.get(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user


@router.patch("/{user_id}", response_model=UserRead)
async def update_user(
    user_id: int,
    user_in: UserUpdate,
    user_service: FromDishka[UserService],
):
    user = await user_service.update(user_id, user_in)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user


@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(
    user_id: int,
    user_service: FromDishka[UserService],
):
    deleted = await user_service.delete(user_id)
    if not deleted:
        raise HTTPException(status_code=404, detail="User not found")
```

## Paginated and Filtered Endpoints

For large collections, use `PaginatedListResponse` with `PaginatedRequestQuery` and a domain filter schema. Both are injected via `Depends()` — this is an accepted use of `Depends()` since it handles FastAPI query parameter grouping, not DI.

```python
from fastapi import Depends

from app.shared.schemas import PaginatedListResponse, PaginatedRequestQuery
from app.users.schemas import UserFilter, UserRead


@router.get("/", response_model=PaginatedListResponse[UserRead])
async def list_users(
    user_service: FromDishka[UserService],
    pagination: PaginatedRequestQuery = Depends(),
    filters: UserFilter = Depends(),
):
    result = await user_service.find_paginated(
        filters=filters,
        page=pagination.page,
        items_per_page=pagination.items_per_page,
        sort=pagination.sort,
    )
    return PaginatedListResponse(
        data=result.items,
        total_count=result.total_count,
        has_more=(pagination.page * pagination.items_per_page) < result.total_count,
        page=pagination.page,
        items_per_page=pagination.items_per_page,
    )
```

- `pagination: PaginatedRequestQuery = Depends()` provides `page`, `items_per_page`, and `sort` with defaults, validation, and OpenAPI descriptions defined in one place (see [schemas.md](schemas.md))
- `filters: UserFilter = Depends()` unpacks the filter schema fields as individual query parameters in OpenAPI docs
- `sort` accepts a comma-separated string: `name,-created_at` (dash prefix for descending). See [repository-pattern.md](repository-pattern.md) for sort validation and `SORTABLE_FIELDS` whitelist.
- The endpoint constructs the `PaginatedListResponse` from the `PaginatedResult` returned by the service

For simple lists that don't need pagination metadata, keep the existing `list[UserRead]` pattern with `skip`/`limit`.

## Nested Resource Endpoints

For child resources under a parent URL (e.g., `/buildings/{building_id}/spaces`), the parent ID comes from the URL path. Inject it into the Create schema using `model_copy` before passing to the service — never construct ORM objects in the endpoint.

```python
@router.post(
    "/{building_id}/spaces",
    response_model=BuildingSpaceRead,
    status_code=status.HTTP_201_CREATED,
)
async def create_building_space(
    building_id: int,
    space_in: BuildingSpaceCreate,
    building_service: FromDishka[BuildingService],
    space_service: FromDishka[BuildingSpaceService],
    _admin: User = Depends(require_role(Role.ADMIN)),
):
    building = await building_service.get(building_id)
    if not building:
        raise HTTPException(status_code=404, detail="Building not found")
    space_in = space_in.model_copy(update={"building_id": building_id})
    return await space_service.create(space_in)
```

Key points:
- The `BuildingSpaceCreate` schema includes `building_id` as a field (inherited from its Base), but the client doesn't need to send it — the URL provides it
- `model_copy(update={"building_id": building_id})` sets the parent ID on the schema before passing to the service
- The service receives a complete schema and passes it to the repository — no ORM construction in the endpoint
- Validate the parent exists before creating the child

**Anti-pattern — constructing ORM objects in the endpoint:**

```python
# Wrong — endpoint reaches into repository layer
@router.post("/{building_id}/spaces", ...)
async def create_building_space(...):
    db = space_service.repo.db  # Violates layering
    space = BuildingSpace(building_id=building_id, **space_in.model_dump())
    db.add(space)
    await db.flush()
    return space
```

The endpoint should never touch `AsyncSession`, repository internals, or ORM models directly. All entity creation flows through `service.create()`.

## Parameter Ordering

Python requires parameters without defaults before parameters with defaults. Place `FromDishka` parameters before query parameters with defaults:

```python
# Correct
async def list_users(
    user_service: FromDishka[UserService],  # no default
    skip: int = 0,                          # has default
    limit: int = 100,                       # has default
): ...

# SyntaxError — default params before non-default
async def list_users(
    skip: int = 0,
    user_service: FromDishka[UserService],
): ...
```

## Router Registration

Register routers in `app/main.py` with versioned prefixes:

```python
from app.users.api import router as users_router
from app.orders.api import router as orders_router

app.include_router(users_router, prefix=f"{settings.API_V1_STR}/users", tags=["users"])
app.include_router(orders_router, prefix=f"{settings.API_V1_STR}/orders", tags=["orders"])
```

## Rules

- Endpoints handle **HTTP concerns only** — status codes, request/response mapping, error translation
- Never put business logic in endpoints — delegate to services
- Use `response_model` for response serialization, Pydantic schemas for request validation
- Map service exceptions to HTTP exceptions at the endpoint level
- Do NOT inject `AsyncSession` into endpoints — services and repositories handle database access internally
