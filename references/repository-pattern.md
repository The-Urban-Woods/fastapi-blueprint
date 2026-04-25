# Repository Pattern

## Abstract Base Class

The `Repository` ABC in `app/shared/repository.py` defines the contract for all repositories. It is framework-agnostic — no SQLAlchemy, no Pydantic, no FastAPI imports.

```python
from abc import ABC, abstractmethod
from typing import Generic, TypeVar

ModelType = TypeVar("ModelType")
IdType = TypeVar("IdType")
CreateSchemaType = TypeVar("CreateSchemaType")
UpdateSchemaType = TypeVar("UpdateSchemaType")


class Repository(ABC, Generic[ModelType, IdType, CreateSchemaType, UpdateSchemaType]):

    @abstractmethod
    async def get(self, id: IdType) -> ModelType | None: ...

    @abstractmethod
    async def find_all(
        self, *, skip: int = 0, limit: int = 100, sort: str | None = None
    ) -> list[ModelType]: ...

    @abstractmethod
    async def create(self, obj_in: CreateSchemaType) -> ModelType: ...

    @abstractmethod
    async def update(self, entity: ModelType, obj_in: UpdateSchemaType) -> ModelType: ...

    @abstractmethod
    async def delete(self, entity: ModelType) -> None: ...
```

**Generic type parameters:**
- `ModelType` — the domain entity type
- `IdType` — the type of the entity's identifier (e.g., `int`, `str`, `UUID`)
- `CreateSchemaType` — the input type for creating entities
- `UpdateSchemaType` — the input type for updating entities

## SQLAlchemy Implementation

`SqlAlchemyRepository` in `app/shared/sqlalchemy_repository.py` implements the ABC for SQLAlchemy with async sessions. It binds `IdType` to `int` and bounds the schema types to Pydantic `BaseModel`.

```python
from typing import TypeVar

from pydantic import BaseModel
from sqlalchemy import asc, desc, select
from sqlalchemy.ext.asyncio import AsyncSession

from app.shared.database import Base
from app.shared.repository import Repository

ModelType = TypeVar("ModelType", bound=Base)
CreateSchemaType = TypeVar("CreateSchemaType", bound=BaseModel)
UpdateSchemaType = TypeVar("UpdateSchemaType", bound=BaseModel)


class SqlAlchemyRepository(Repository[ModelType, int, CreateSchemaType, UpdateSchemaType]):
    SORTABLE_FIELDS: set[str] = set()

    def __init__(self, model: type[ModelType], db: AsyncSession):
        self.model = model
        self.db = db

    def _apply_sort(self, stmt, sort: str | None):
        if not sort:
            return stmt
        for field in sort.split(","):
            field = field.strip()
            descending = field.startswith("-")
            field_name = field.lstrip("-")
            if field_name not in self.SORTABLE_FIELDS:
                continue
            column = getattr(self.model, field_name)
            stmt = stmt.order_by(desc(column) if descending else asc(column))
        return stmt

    async def get(self, id: int) -> ModelType | None:
        result = await self.db.execute(select(self.model).where(self.model.id == id))
        return result.scalars().first()

    async def find_all(
        self, *, skip: int = 0, limit: int = 100, sort: str | None = None
    ) -> list[ModelType]:
        stmt = select(self.model)
        stmt = self._apply_sort(stmt, sort)
        result = await self.db.execute(stmt.offset(skip).limit(limit))
        return list(result.scalars().all())

    async def create(self, obj_in: CreateSchemaType) -> ModelType:
        db_obj = self.model(**obj_in.model_dump())
        self.db.add(db_obj)
        await self.db.flush()
        await self.db.refresh(db_obj)
        return db_obj

    async def update(self, entity: ModelType, obj_in: UpdateSchemaType) -> ModelType:
        update_data = obj_in.model_dump(exclude_unset=True)
        for field, value in update_data.items():
            setattr(entity, field, value)
        await self.db.flush()
        await self.db.refresh(entity)
        return entity

    async def delete(self, entity: ModelType) -> None:
        await self.db.delete(entity)
```

## Key Design Decisions

- **ABC is framework-agnostic** — `Repository` imports nothing from SQLAlchemy or Pydantic. It can be implemented with any database backend.
- **`SqlAlchemyRepository` is the concrete base** — it binds the ABC to SQLAlchemy + Pydantic. Domain repositories extend this, not the ABC directly.
- **`db: AsyncSession` via constructor** — injected by dishka at `Scope.REQUEST`. The session is shared across all repositories and services within a request.
- **`flush()` instead of `commit()`** — repositories flush changes but do not commit. Commit/rollback is handled by the dishka session provider.
- **`exclude_unset=True` for updates** — only applies fields explicitly provided, enabling PATCH semantics.
- **Sorting via `_apply_sort`** — all `find_all` queries support an optional `sort` parameter. Format is comma-separated fields, `-` prefix for descending: `sort="name,-created_at"`. Guarded by `SORTABLE_FIELDS` whitelist — fields not in the whitelist are silently ignored.

## Paginated SQLAlchemy Repository

For domains with large collections that need pagination, filtering, and sorting, extend `PaginatedSqlAlchemyRepository` instead of `SqlAlchemyRepository`. It adds `find_paginated` with filter conditions support.

Define in `app/shared/paginated_sqlalchemy_repository.py`:

```python
from dataclasses import dataclass
from typing import Generic

from sqlalchemy import func, select

from app.shared.sqlalchemy_repository import SqlAlchemyRepository, ModelType, CreateSchemaType, UpdateSchemaType


@dataclass
class PaginatedResult(Generic[ModelType]):
    items: list[ModelType]
    total_count: int


class PaginatedSqlAlchemyRepository(
    SqlAlchemyRepository[ModelType, CreateSchemaType, UpdateSchemaType]
):
    async def find_paginated(
        self,
        *,
        page: int = 1,
        items_per_page: int = 20,
        sort: str | None = None,
        conditions: list | None = None,
    ) -> PaginatedResult[ModelType]:
        stmt = select(self.model)
        if conditions:
            stmt = stmt.where(*conditions)
        stmt = self._apply_sort(stmt, sort)

        # Total count (before offset/limit)
        count_stmt = select(func.count()).select_from(stmt.subquery())
        total_count = (await self.db.execute(count_stmt)).scalar_one()

        # Items for current page
        offset = (page - 1) * items_per_page
        items_stmt = stmt.offset(offset).limit(items_per_page)
        items = list((await self.db.execute(items_stmt)).scalars().all())

        return PaginatedResult(items=items, total_count=total_count)

    def _apply_filters(self, filters: dict) -> list:
        conditions = []
        for key, value in filters.items():
            if value is None:
                continue
            if key.endswith("_gte"):
                col = getattr(self.model, key.removesuffix("_gte"))
                conditions.append(col >= value)
            elif key.endswith("_gt"):
                col = getattr(self.model, key.removesuffix("_gt"))
                conditions.append(col > value)
            elif key.endswith("_lte"):
                col = getattr(self.model, key.removesuffix("_lte"))
                conditions.append(col <= value)
            elif key.endswith("_lt"):
                col = getattr(self.model, key.removesuffix("_lt"))
                conditions.append(col < value)
            elif key.endswith("_like"):
                col = getattr(self.model, key.removesuffix("_like"))
                conditions.append(col.ilike(f"%{value}%"))
            else:
                col = getattr(self.model, key)
                conditions.append(col == value)
        return conditions
```

The `conditions` parameter accepts a list of SQLAlchemy column expressions — these can come from the generic `_apply_filters` or a domain-specific `FilterBuilder`.

### Filtering

#### Generic Suffix-Based Filters

For simple filtering, `_apply_filters` translates a dict with suffix conventions into WHERE clauses:

- `field` — exact match (`==`)
- `field_gt` / `field_gte` — greater than / greater or equal
- `field_lt` / `field_lte` — less than / less or equal
- `field_like` — case-insensitive contains (`ILIKE %value%`)

Usage in a domain repository:

```python
async def find_filtered_paginated(
    self,
    filters: UserFilter,
    *,
    page: int = 1,
    items_per_page: int = 20,
    sort: str | None = None,
) -> PaginatedResult[User]:
    conditions = self._apply_filters(filters.model_dump(exclude_unset=True))
    return await self.find_paginated(
        page=page, items_per_page=items_per_page, sort=sort, conditions=conditions
    )
```

#### Domain FilterBuilder (Complex Filters)

When filters need logic beyond simple suffix mapping — search across multiple columns, conditional joins, or combined expressions — use a domain-specific `FilterBuilder`:

```python
# app/users/filters.py
from app.users.models import User
from app.users.schemas import UserFilter


class UserFilterBuilder:
    @staticmethod
    def build(filters: UserFilter) -> list:
        conditions = []
        if filters.email:
            conditions.append(User.email == filters.email)
        if filters.is_active is not None:
            conditions.append(User.is_active == filters.is_active)
        if filters.created_from:
            conditions.append(User.created_at >= filters.created_from)
        if filters.created_until:
            conditions.append(User.created_at <= filters.created_until)
        if filters.search:
            search = f"%{filters.search}%"
            conditions.append(User.name.ilike(search))
        return conditions
```

The `FilterBuilder` returns a list of SQLAlchemy conditions that feed directly into `find_paginated`:

```python
# In the service layer
conditions = UserFilterBuilder.build(filters)
return await self.repo.find_paginated(
    page=page, items_per_page=items_per_page, sort=sort, conditions=conditions
)
```

Use the generic `_apply_filters` for domains with simple exact-match and comparison filters. Switch to a `FilterBuilder` when a domain needs search, multi-column expressions, or conditional logic. The repository's `find_paginated` accepts both — it only cares about the resulting `conditions` list.

## Domain Repository

Extend `SqlAlchemyRepository` for simple CRUD domains, or `PaginatedSqlAlchemyRepository` for domains that need pagination and filtering.

### Simple CRUD

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.shared.sqlalchemy_repository import SqlAlchemyRepository
from app.users.models import User
from app.users.schemas import UserCreate, UserUpdate


class UserRepository(SqlAlchemyRepository[User, UserCreate, UserUpdate]):
    SORTABLE_FIELDS = {"name", "email", "created_at"}

    def __init__(self, db: AsyncSession):
        super().__init__(User, db)

    async def get_by_email(self, email: str) -> User | None:
        result = await self.db.execute(select(User).where(User.email == email))
        return result.scalars().first()
```

### With Pagination and Filtering

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.shared.paginated_sqlalchemy_repository import PaginatedSqlAlchemyRepository
from app.orders.models import Order
from app.orders.schemas import OrderCreate, OrderUpdate


class OrderRepository(PaginatedSqlAlchemyRepository[Order, OrderCreate, OrderUpdate]):
    SORTABLE_FIELDS = {"status", "created_at", "total"}

    def __init__(self, db: AsyncSession):
        super().__init__(Order, db)
```

All CRUD methods from `SqlAlchemyRepository` are inherited. The domain repository only needs to set `SORTABLE_FIELDS` and add domain-specific queries.

## Rules

- Domain repositories extend `SqlAlchemyRepository` or `PaginatedSqlAlchemyRepository`, not the `Repository` ABC directly
- Use `SqlAlchemyRepository` for simple CRUD domains; use `PaginatedSqlAlchemyRepository` when the domain needs paginated listing with filtering
- Repositories contain **data access logic only** — no business rules, no HTTP concerns
- Custom queries go in the domain repository, not in services
- Do NOT instantiate repositories manually — let dishka provide them
- To support a different database backend, create a new implementation of the `Repository` ABC (e.g., `MongoRepository`)
