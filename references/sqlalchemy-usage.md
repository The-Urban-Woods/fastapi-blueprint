# SQLAlchemy Usage

Model definitions, relationships, query patterns, and advanced features for async SQLAlchemy 2.0+.

For Base class setup, engine/session via DI, and transaction management see [database-setup.md](database-setup.md). For repository CRUD operations see [repository-pattern.md](repository-pattern.md).

## Constraint Naming Conventions

Define a naming convention on `Base.metadata` so all constraints get predictable, explicit names. This is required for Alembic autogenerate to detect constraint changes reliably.

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

Place this in `app/shared/database.py`, replacing the plain `Base`.

## Mixins

Use mixins to share common columns across models. Define them in `app/shared/database.py` alongside `Base`.

### Timestamp Mixin

```python
from datetime import datetime

from sqlalchemy import func
from sqlalchemy.orm import Mapped, mapped_column


class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(
        server_default=func.now(), onupdate=func.now()
    )
```

Apply to models:

```python
from app.shared.database import Base, TimestampMixin


class User(TimestampMixin, Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    name: Mapped[str] = mapped_column(String(255))
    is_active: Mapped[bool] = mapped_column(default=True)
```

Mixin must appear before `Base` in the class hierarchy.

### Soft Delete Mixin

For models that should be deactivated rather than permanently removed:

```python
from datetime import datetime

from sqlalchemy.orm import Mapped, mapped_column


class SoftDeleteMixin:
    is_deleted: Mapped[bool] = mapped_column(default=False)
    deleted_at: Mapped[datetime | None] = mapped_column(default=None)
```

Apply alongside `TimestampMixin`:

```python
from app.shared.database import Base, TimestampMixin, SoftDeleteMixin


class User(SoftDeleteMixin, TimestampMixin, Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    name: Mapped[str] = mapped_column(String(255))
```

The service layer handles the soft delete logic — setting `is_deleted=True` and `deleted_at=datetime.now()` — and filters out soft-deleted records by default. See [repository-pattern.md](repository-pattern.md) for how `find_all` and `find_paginated` exclude soft-deleted records.

## Model Definitions

Use the SQLAlchemy 2.0 `Mapped` type-hint style for all models. This provides IDE support, mypy compatibility, and readable definitions.

### Column Types

```python
from decimal import Decimal

from sqlalchemy import String, Text, Numeric
from sqlalchemy.orm import Mapped, mapped_column

class Product(Base):
    __tablename__ = "products"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(255))
    description: Mapped[str | None] = mapped_column(Text, default=None)
    price: Mapped[Decimal] = mapped_column(Numeric(10, 2))
    is_available: Mapped[bool] = mapped_column(default=True)
```

- `Mapped[str | None]` makes the column nullable
- `Mapped[str]` makes the column `NOT NULL`
- Use `String(n)` for bounded text, `Text` for unbounded
- Use `Numeric(precision, scale)` for monetary values, never `Float`
- Use `Mapped[Decimal]` (not `Mapped[float]`) with `Numeric` columns — this preserves precision across the ORM boundary and matches the Pydantic `Decimal` type in schemas

### Date and Time Columns

Use Python's `datetime` module types with their matching SQLAlchemy column types. The `Mapped` type hint must match the Python type, not `str`.

```python
from datetime import date, datetime, time

from sqlalchemy import Date, DateTime, Time
from sqlalchemy.orm import Mapped, mapped_column


class Event(Base):
    __tablename__ = "events"

    id: Mapped[int] = mapped_column(primary_key=True)
    event_date: Mapped[date] = mapped_column(Date)                          # date only (no time)
    start_time: Mapped[time | None] = mapped_column(Time, nullable=True)    # time only (no date)
    created_at: Mapped[datetime] = mapped_column(DateTime)                  # date + time
    deadline: Mapped[date | None] = mapped_column(Date, nullable=True)      # nullable date
```

| Python type | SQLAlchemy column | Stores | Example |
|---|---|---|---|
| `date` | `Date` | Date only | `2026-04-25` |
| `time` | `Time` | Time only | `14:30:00` |
| `datetime` | `DateTime` | Date + time | `2026-04-25T14:30:00` |

**Anti-pattern — using `str` for date columns:**

```python
# Wrong — loses type safety, breaks Pydantic serialization
start_date: Mapped[str] = mapped_column(Date)
```

Always use `Mapped[date]` with `Date`, `Mapped[time]` with `Time`, and `Mapped[datetime]` with `DateTime`. The ORM handles conversion between Python types and database types automatically.

For timestamps managed by the database (`created_at`, `updated_at`), use the `TimestampMixin` with `server_default=func.now()` — see the Timestamp Mixin section above.

### Indexes and Constraints

```python
from sqlalchemy import Index, UniqueConstraint

class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    status: Mapped[str] = mapped_column(String(50))
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())

    __table_args__ = (
        Index("ix__orders__status_created", "status", "created_at"),
        UniqueConstraint("user_id", "status", name="uq__orders__user_status"),
    )
```

- Add `index=True` on foreign keys and frequently filtered columns
- Use `__table_args__` for composite indexes and constraints
- Follow the naming convention pattern: `ix__table__columns`, `uq__table__columns`

## Relationships

### One-to-Many

```python
from sqlalchemy import ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column, relationship

class User(TimestampMixin, Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(255))

    orders: Mapped[list["Order"]] = relationship(back_populates="user")


class Order(TimestampMixin, Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)

    user: Mapped["User"] = relationship(back_populates="orders")
```

- Always use `back_populates` (explicit) over `backref` (implicit)
- Index foreign key columns — they are used in joins and lookups
- Use string annotations (`"Order"`) to avoid circular imports

### Many-to-Many

```python
from sqlalchemy import Column, ForeignKey, Table

student_course = Table(
    "student_course",
    Base.metadata,
    Column("student_id", ForeignKey("students.id"), primary_key=True),
    Column("course_id", ForeignKey("courses.id"), primary_key=True),
)


class Student(Base):
    __tablename__ = "students"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(255))

    courses: Mapped[list["Course"]] = relationship(
        secondary=student_course, back_populates="students"
    )


class Course(Base):
    __tablename__ = "courses"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(255))

    students: Mapped[list["Student"]] = relationship(
        secondary=student_course, back_populates="courses"
    )
```

Use an association table for pure many-to-many. If the join table needs extra columns (e.g., `enrolled_at`), promote it to a full model with two foreign keys.

### Cross-Domain Relationships

When a relationship references a model from another domain module (e.g., `RentalAgreement.tenant` → `User`), a direct import would create circular dependencies. Use `TYPE_CHECKING` for the type hint and string annotations for the relationship target.

```python
from __future__ import annotations

from typing import TYPE_CHECKING

from sqlalchemy import ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.shared.database import Base, TimestampMixin

if TYPE_CHECKING:
    from app.users.models import User


class RentalAgreement(TimestampMixin, Base):
    __tablename__ = "rental_agreements"

    tenant_id: Mapped[str] = mapped_column(ForeignKey("users.id"), index=True)

    tenant: Mapped["User"] = relationship("User")
```

- `from __future__ import annotations` — ensures all annotations are treated as strings at runtime
- `if TYPE_CHECKING:` — provides IDE and type checker support without introducing runtime imports
- `Mapped["User"]` — resolves via SQLAlchemy’s mapper registry

**Setting up the reverse relationship on the other domain's model:**

If both domains require navigation, define both sides explicitly using back_populates and string references.

```python
# app/rental_spaces/models.py
from __future__ import annotations

from typing import TYPE_CHECKING

from sqlalchemy import ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.shared.database import Base

if TYPE_CHECKING:
    from app.buildings.models import BuildingSpace


class RentalSpace(Base):
    __tablename__ = "rental_spaces"

    id: Mapped[int] = mapped_column(primary_key=True)
    building_space_id: Mapped[int] = mapped_column(
        ForeignKey("building_spaces.id"),
        unique=True,
        index=True,
    )

    building_space: Mapped["BuildingSpace"] = relationship(
        "BuildingSpace",
        back_populates="rental_space",
    )
```

```python
# app/buildings/models.py
from __future__ import annotations

from typing import TYPE_CHECKING

from sqlalchemy.orm import Mapped, relationship

from app.shared.database import Base

if TYPE_CHECKING:
    from app.rental_spaces.models import RentalSpace


class BuildingSpace(Base):
    __tablename__ = "building_spaces"

    id: Mapped[int] = mapped_column(primary_key=True)

    rental_space: Mapped["RentalSpace | None"] = relationship(
        "RentalSpace",
        back_populates="building_space",
    )
```

**Prefer unidirectional relationships by default** 

Only define reverse relationships when there is a clear use case. Unnecessary bidirectional links increase coupling between domains and make the model graph harder to reason about.

```python
class RentalSpace(Base):
    __tablename__ = "rental_spaces"

    building_space_id: Mapped[int] = mapped_column(
        ForeignKey("building_spaces.id"),
        unique=True,
        index=True,
    )

    building_space: Mapped["BuildingSpace"] = relationship("BuildingSpace")
```

Query from the owning domain when reverse navigation is not required:

```python
stmt = select(RentalSpace).where(
    RentalSpace.building_space_id == building_space_id
)
```


### Filtering

```python
from sqlalchemy import select

stmt = select(User).where(User.is_active == True, User.name.ilike("%alice%"))
result = await self.db.execute(stmt)
users = list(result.scalars().all())
```

### Ordering

```python
stmt = select(Order).order_by(Order.created_at.desc())
```

### Joins

```python
stmt = (
    select(Order)
    .join(User)
    .where(User.email == "alice@example.com")
)
result = await self.db.execute(stmt)
orders = list(result.scalars().all())
```

### Aggregations

```python
from sqlalchemy import func, select

stmt = select(func.count(Order.id)).where(Order.user_id == user_id)
result = await self.db.execute(stmt)
count = result.scalar_one()
```

### Eager Loading (Prevent N+1)

Use `selectinload` for async — it batches related rows via `IN` clauses in a second query. Do not use `joinedload` with async sessions as it can cause issues with implicit IO.

```python
from sqlalchemy.orm import selectinload

stmt = select(User).options(selectinload(User.orders)).limit(100)
result = await self.db.execute(stmt)
users = list(result.scalars().all())
```

For nested relationships, chain the load options:

```python
stmt = select(User).options(
    selectinload(User.orders).selectinload(Order.items)
)
```

### Keyset (Cursor-Based) Pagination

Prefer keyset pagination over `OFFSET` for stable, index-friendly paging at scale.

```python
async def find_paginated(
    self, *, limit: int = 50, after_id: int | None = None
) -> list[Order]:
    stmt = select(Order).order_by(Order.id.asc()).limit(limit)
    if after_id is not None:
        stmt = stmt.where(Order.id > after_id)
    result = await self.db.execute(stmt)
    return list(result.scalars().all())
```

The caller passes the last seen `id` as cursor. No rows are skipped or duplicated under concurrent inserts.

## Upsert (Insert or Update)

Use dialect-specific `ON CONFLICT` for atomic insert-or-update. Both PostgreSQL and SQLite support this via their respective `insert` dialects, so a single helper works for both production and tests.

### Helper

Place this in `app/shared/sqlalchemy_repository.py` or a dedicated `app/shared/upsert.py`:

```python
from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.dialects.sqlite import insert as sqlite_insert


def upsert_stmt(
    model,
    values: dict,
    conflict_fields: list[str],
    update_fields: list[str],
    dialect_name: str,
):
    insert_fn = {
        "postgresql": pg_insert,
        "sqlite": sqlite_insert,
    }.get(dialect_name)

    if insert_fn is None:
        raise NotImplementedError(f"Upsert not supported for dialect: {dialect_name}")

    stmt = insert_fn(model).values(**values)

    return stmt.on_conflict_do_update(
        index_elements=[getattr(model, field) for field in conflict_fields],
        set_={field: getattr(stmt.excluded, field) for field in update_fields},
    )
```

### Resolving the Dialect

Derive `dialect_name` from the database URL in the dishka provider and inject it as a plain string. This works for both `AppProvider` (reads from `Settings`) and `TestAppProvider` (hardcoded test URL), since both know their URL at engine-creation time.

**AppProvider** (in `app/providers.py`):

```python
from sqlalchemy.engine import make_url

class AppProvider(Provider):
    scope = Scope.APP

    @provide
    def get_dialect_name(self, settings: Settings) -> str:
        return make_url(settings.DATABASE_URL).get_backend_name()
```

**TestAppProvider** (in `tests/conftest.py`):

```python
from sqlalchemy.engine import make_url

TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"

class TestAppProvider(Provider):
    scope = Scope.APP

    @provide
    def get_dialect_name(self) -> str:
        return make_url(TEST_DATABASE_URL).get_backend_name()
```

`make_url().get_backend_name()` returns `"postgresql"` or `"sqlite"` — it handles all URL formats including those with driver suffixes like `+asyncpg` or `+aiosqlite`.

### Usage in a Repository

Repositories that need upsert accept `dialect_name: str` alongside the session:

```python
class UserRepository(SqlAlchemyRepository[User, int, UserCreate, UserUpdate]):
    def __init__(self, db: AsyncSession, dialect_name: str):
        super().__init__(User, db)
        self.dialect_name = dialect_name

    async def upsert(self, user_data: dict) -> User:
        stmt = upsert_stmt(
            model=User,
            values=user_data,
            conflict_fields=["email"],
            update_fields=["name", "updated_at"],
            dialect_name=self.dialect_name,
        ).returning(User)

        result = await self.db.execute(
            stmt,
            execution_options={"populate_existing": True},
        )
        return result.scalar_one()
```

Dishka auto-resolves `dialect_name: str` from the provider when constructing the repository — no manual wiring needed.

- `conflict_fields` — the columns that form the unique constraint to match on
- `update_fields` — the columns to overwrite when a conflict is found
- `populate_existing=True` — ensures the session identity map picks up the returned row, avoiding stale instances
- The `returning(Model)` clause returns the full row so no extra query is needed

## Hybrid Properties

Use hybrid properties for computed values that work both as Python attributes and in SQL queries.

```python
from sqlalchemy.ext.hybrid import hybrid_property


class Product(Base):
    __tablename__ = "products"

    id: Mapped[int] = mapped_column(primary_key=True)
    price: Mapped[Decimal] = mapped_column(Numeric(10, 2))
    tax_rate: Mapped[Decimal] = mapped_column(Numeric(5, 4))

    @hybrid_property
    def total_price(self) -> Decimal:
        return self.price * (1 + self.tax_rate)

    @total_price.expression
    def total_price(cls):
        return cls.price * (1 + cls.tax_rate)
```

The `@hybrid_property` works in Python: `product.total_price`. The `@expression` variant works in SQL: `select(Product).where(Product.total_price > 100)`.

## Optimistic Locking (Optional)

For models where concurrent updates are a concern, add a `version_id` column. SQLAlchemy raises `StaleDataError` when a concurrent modification is detected.

```python
class Inventory(Base):
    __tablename__ = "inventory"

    id: Mapped[int] = mapped_column(primary_key=True)
    sku: Mapped[str] = mapped_column(String(100), unique=True)
    quantity: Mapped[int] = mapped_column(default=0)
    version_id: Mapped[int] = mapped_column(default=1)

    __mapper_args__ = {"version_id_col": version_id}
```

Handle the conflict in the service layer:

```python
from sqlalchemy.orm.exc import StaleDataError


async def reserve_stock(self, sku: str, count: int) -> Inventory:
    try:
        item = await self.repository.get_by_sku(sku)
        item.quantity -= count
        await self.repository.flush()
        return item
    except StaleDataError:
        raise ConflictError(f"Concurrent update on {sku}, retry the request")
```

This is not a default for all models — only add it where concurrent writes to the same row are expected (e.g., inventory, counters, shared resources).
