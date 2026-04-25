# Database Setup

## Declarative Base

`app/shared/database.py` contains the SQLAlchemy `Base` class with a constraint naming convention, and shared mixins. Engine creation, session management, and lifecycle are handled by dishka providers — not module-level globals.

```python
from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase

convention = {
    "all_column_names": lambda constraint, table: "_".join(
        [column.name for column in constraint.columns.values()]
    ),
    "ix": "ix__%(table_name)s__%(all_column_names)s",
    "uq": "uq__%(table_name)s__%(all_column_names)s",
    "ck": "ck__%(table_name)s__%(constraint_name)s",
    "fk": "fk__%(table_name)s__%(all_column_names)s__%(referred_table_name)s",
    "pk": "pk__%(table_name)s",
}


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=convention)
```

The naming convention ensures all constraints get predictable names, which is required for Alembic autogenerate. For shared mixins like `TimestampMixin`, see [sqlalchemy-usage.md](sqlalchemy-usage.md).

## Pydantic Schemas

Define request/response schemas in each domain's `schemas.py`:

```python
from datetime import datetime

from pydantic import BaseModel, ConfigDict


class UserBase(BaseModel):
    email: str
    name: str


class UserCreate(UserBase):
    pass


class UserUpdate(BaseModel):
    email: str | None = None
    name: str | None = None
    is_active: bool | None = None


class UserRead(UserBase):
    model_config = ConfigDict(from_attributes=True)

    id: int
    is_active: bool
    created_at: datetime
    updated_at: datetime
```

## Engine and Session via DI

Engine and session factory are created by dishka's `AppProvider`. The session lifecycle (commit/rollback) is managed by the `RequestProvider`. See [dependency-injection.md](dependency-injection.md) for the full provider setup.

Do NOT create module-level `engine` or `async_session` objects. All database infrastructure flows through the DI container.

## Table Creation

Tables are created in the app lifespan using the DI-managed engine:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    engine = await app.state.dishka_container.get(AsyncEngine)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    await app.state.dishka_container.close()
```

Once Alembic is configured, remove `create_all` from the lifespan — having both leads to schema drift where Alembic won't detect changes that `create_all` already applied. The lifespan simplifies to:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    await app.state.dishka_container.close()
```

Schema changes then go through Alembic migrations exclusively. See [alembic-migrations.md](alembic-migrations.md) for setup.

## Transaction Management

- One transaction per request, managed by the dishka session provider
- Repositories call `flush()` to push changes, but never `commit()`
- `commit()` happens automatically when the request scope exits successfully
- `rollback()` happens automatically on exception
- This ensures all repository operations within a single request are atomic

## Further Reading

- **Model definitions, relationships, query patterns, mixins**: See [sqlalchemy-usage.md](sqlalchemy-usage.md)
- **Database migrations with Alembic**: See [alembic-migrations.md](alembic-migrations.md)
- **Repository ABC and SQLAlchemy implementation**: See [repository-pattern.md](repository-pattern.md)
