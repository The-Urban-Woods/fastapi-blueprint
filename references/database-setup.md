# Database Setup

## Declarative Base

`app/shared/database.py` contains only the SQLAlchemy `Base` class. Engine creation, session management, and lifecycle are handled by dishka providers — not module-level globals.

```python
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass
```

## SQLAlchemy Models

Define models in each domain's `models.py`, inheriting from `Base`:

```python
from datetime import datetime

from sqlalchemy import String, func
from sqlalchemy.orm import Mapped, mapped_column

from app.shared.database import Base


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    name: Mapped[str] = mapped_column(String(255))
    is_active: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(
        server_default=func.now(), onupdate=func.now()
    )
```

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

## Transaction Management

- One transaction per request, managed by the dishka session provider
- Repositories call `flush()` to push changes, but never `commit()`
- `commit()` happens automatically when the request scope exits successfully
- `rollback()` happens automatically on exception
- This ensures all repository operations within a single request are atomic
