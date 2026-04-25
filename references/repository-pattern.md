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
    async def find_all(self, *, skip: int = 0, limit: int = 100) -> list[ModelType]: ...

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
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.shared.database import Base
from app.shared.repository import Repository

ModelType = TypeVar("ModelType", bound=Base)
CreateSchemaType = TypeVar("CreateSchemaType", bound=BaseModel)
UpdateSchemaType = TypeVar("UpdateSchemaType", bound=BaseModel)


class SqlAlchemyRepository(Repository[ModelType, int, CreateSchemaType, UpdateSchemaType]):
    def __init__(self, model: type[ModelType], db: AsyncSession):
        self.model = model
        self.db = db

    async def get(self, id: int) -> ModelType | None:
        result = await self.db.execute(select(self.model).where(self.model.id == id))
        return result.scalars().first()

    async def find_all(self, *, skip: int = 0, limit: int = 100) -> list[ModelType]:
        result = await self.db.execute(
            select(self.model).offset(skip).limit(limit)
        )
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

## Domain Repository

Extend `SqlAlchemyRepository` with entity-specific queries:

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.shared.sqlalchemy_repository import SqlAlchemyRepository
from app.users.models import User
from app.users.schemas import UserCreate, UserUpdate


class UserRepository(SqlAlchemyRepository[User, UserCreate, UserUpdate]):
    def __init__(self, db: AsyncSession):
        super().__init__(User, db)

    async def get_by_email(self, email: str) -> User | None:
        result = await self.db.execute(select(User).where(User.email == email))
        return result.scalars().first()
```

## Rules

- Domain repositories extend `SqlAlchemyRepository`, not the `Repository` ABC directly
- Repositories contain **data access logic only** — no business rules, no HTTP concerns
- Custom queries go in the domain repository, not in services
- Do NOT instantiate repositories manually — let dishka provide them
- To support a different database backend, create a new implementation of the `Repository` ABC (e.g., `MongoRepository`)
