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

## UUID Primary Key Helper

`app/shared/database.py` also defines a `uuid_pk()` helper and supporting `GenerateUUID` function for models that use UUID primary keys. This uses database-side generation on PostgreSQL (`gen_random_uuid()`) and falls back to Python `uuid.uuid4()` on other dialects (e.g., SQLite for tests).

```python
import uuid
from typing import Any

from sqlalchemy import Uuid
from sqlalchemy.ext.compiler import compiles
from sqlalchemy.orm import mapped_column
from sqlalchemy.sql import functions
from sqlalchemy.sql.compiler import SQLCompiler
from sqlalchemy.sql.elements import ClauseElement


class GenerateUUID(functions.GenericFunction):
    """Database-side UUID4 generation.

    PostgreSQL: gen_random_uuid() (built-in since PG 13)
    Other dialects: falls back to Python uuid.uuid4() via the `default` param.
    """

    type = Uuid()
    inherit_cache = True


@compiles(GenerateUUID, "postgresql")
def _pg_generate_uuid(element: ClauseElement, compiler: SQLCompiler, **kwargs: Any) -> str:
    return "gen_random_uuid()"


@compiles(GenerateUUID, "sqlite")
def _sqlite_generate_uuid(element: ClauseElement, compiler: SQLCompiler, **kwargs: Any) -> str:
    # SQLite has no native UUID function.
    # Return NULL — Python `default=uuid.uuid4` handles all ORM inserts.
    return "NULL"


def uuid_pk(**kwargs: Any) -> Any:
    """UUID primary key with database-side generation on PostgreSQL."""
    return mapped_column(
        Uuid,
        primary_key=True,
        default=uuid.uuid4,
        server_default=GenerateUUID(),
        **kwargs,
    )
```

Usage in models:

```python
import uuid

from app.shared.database import Base, TimestampMixin, uuid_pk


class User(TimestampMixin, Base):
    __tablename__ = "users"

    id: Mapped[uuid.UUID] = uuid_pk()
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
```

For UUID foreign key columns, use the `Uuid` type directly:

```python
from sqlalchemy import Uuid

building_id: Mapped[uuid.UUID] = mapped_column(Uuid, ForeignKey("buildings.id"), index=True)
```

The `Uuid` column type (SQLAlchemy 2.0+) is cross-dialect -- native `UUID` on PostgreSQL, `CHAR(32)` on SQLite. Pydantic v2 serializes `uuid.UUID` to strings in JSON automatically.

For how UUID types flow through schemas, services, and repositories, see [sqlalchemy-usage.md](sqlalchemy-usage.md#uuid-primary-keys).

## Pydantic Schemas

Define request/response schemas in each domain's `schemas.py`. For naming conventions, field constraints, validators, ConfigDict options, and all schema patterns see [schemas.md](schemas.md).

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

## Domain-Prefixed Table Names (Optional)

For larger projects with many domains, auto-generate table names as `{domain}__{entity}` to prevent collisions and make domain ownership visible in the database schema.

```python
import re

from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase, declared_attr

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


def _camel_to_snake(name: str) -> str:
    """CamelCase to snake_case. Handles acronyms: HTTPClient → http_client."""
    s = re.sub(r"([A-Z]+)([A-Z][a-z])", r"\1_\2", name)
    return re.sub(r"([a-z0-9])([A-Z])", r"\1_\2", s).lower()


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=convention)

    @declared_attr.directive
    @classmethod
    def __tablename__(cls) -> str:
        """Auto-generate table name as {domain}__{entity_snake_case}.

        Derives domain from module path: app.{domain}.models.MyModel → domain__my_model

        Override on any model by setting __tablename__ explicitly:
            __tablename__ = "custom_name"
        """
        parts = cls.__module__.split(".")
        # Domain is the first package after the app root
        # app.rental_spaces.models → rental_spaces
        # app.shared.job → shared
        domain = parts[1] if len(parts) >= 3 else ""
        entity = _camel_to_snake(cls.__name__)
        return f"{domain}__{entity}" if domain else entity
```

This produces table names like:

| Module | Class | Table Name |
|---|---|---|
| `app.users.models` | `User` | `users__user` |
| `app.buildings.models` | `BuildingSpace` | `buildings__building_space` |
| `app.rental_spaces.models` | `RentalAgreement` | `rental_spaces__rental_agreement` |
| `app.shared.job` | `BackgroundJob` | `shared__background_job` |

**When to use this pattern:**
- Multiple domains with potential table name collisions (e.g., both `orders` and `returns` domains might have a `Document` model)
- You want the database schema to reflect domain boundaries
- Team convention — everyone agrees on the auto-naming approach

**When to stick with explicit `__tablename__`:**
- Small projects with few domains
- You prefer searchable, explicit table names
- You want full control over naming without convention knowledge

**ForeignKey references use the full prefixed name:**

```python
# With domain-prefixed tables
tenant_id: Mapped[uuid.UUID] = mapped_column(Uuid, ForeignKey("users__user.id"))
building_id: Mapped[uuid.UUID] = mapped_column(Uuid, ForeignKey("buildings__building.id"))
```

**Override for any model by setting `__tablename__` explicitly:**

```python
class TokenBlacklist(Base):
    __tablename__ = "token_blacklist"  # Explicit — skips auto-generation
```

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
