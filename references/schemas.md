# Pydantic Schemas

Schema design patterns, validation, and serialization for FastAPI request/response models using Pydantic v2.

Schemas live in each domain's `schemas.py`. They handle I/O boundaries only — never pass Pydantic models into repositories or use them as ORM models.

## Naming Convention

Each domain uses a consistent set of schema classes:

| Schema | Purpose | Inherits |
| --- | --- | --- |
| `{Model}Base` | Shared fields across create and read | `BaseModel` |
| `{Model}Create` | Request body for creation | `{Model}Base` |
| `{Model}Update` | Request body for partial updates | `BaseModel` |
| `{Model}Read` | Response body | `{Model}Base` |
| `{Model}Internal` | Internal use when stored fields differ from input (e.g., hashed password) | `BaseModel` |

`Update` inherits from `BaseModel` directly (not `Base`) because all its fields are optional — it has a different shape than `Create` or `Read`.

## ConfigDict

Apply `ConfigDict` based on schema role:

```python
from pydantic import BaseModel, ConfigDict


class UserCreate(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)


class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)
```

| Option | Use on | Why |
| --- | --- | --- |
| `extra="forbid"` | Create, Update | Rejects unexpected fields — prevents clients from injecting data |
| `from_attributes=True` | Read | Enables `model_validate(orm_instance)` to convert SQLAlchemy models |
| `str_strip_whitespace=True` | Create, Update | Strips leading/trailing whitespace from string inputs |

## Field Constraints

Use `Annotated` with `Field` for explicit constraints. These generate OpenAPI validation rules and Swagger documentation automatically.

```python
from typing import Annotated
from pydantic import BaseModel, ConfigDict, EmailStr, Field


class UserBase(BaseModel):
    email: Annotated[EmailStr, Field(max_length=255)]
    name: Annotated[str, Field(min_length=1, max_length=255)]


class UserCreate(UserBase):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    password: Annotated[str, Field(min_length=8, max_length=128)]


class UserUpdate(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    email: Annotated[EmailStr | None, Field(max_length=255, default=None)]
    name: Annotated[str | None, Field(min_length=1, max_length=255, default=None)]
    is_active: bool | None = None


class UserRead(UserBase):
    model_config = ConfigDict(from_attributes=True)

    id: int
    is_active: bool
    created_at: datetime
    updated_at: datetime
```

- `min_length` / `max_length` for strings
- `gt` / `ge` / `lt` / `le` for numbers
- `pattern` for regex validation (e.g., `pattern=r"^[a-z0-9_]+$"` for slugs)

### UUID Fields

When models use UUID primary keys (via `uuid_pk()`), use `uuid.UUID` in schema fields for all ID and foreign key references. Pydantic v2 serializes `uuid.UUID` to strings in JSON responses automatically and validates UUID format on incoming requests.

```python
import uuid

from pydantic import BaseModel, ConfigDict


class BuildingRead(BuildingBase):
    model_config = ConfigDict(from_attributes=True)

    id: uuid.UUID
    created_at: datetime
    updated_at: datetime


class BuildingSpaceCreate(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    building_id: uuid.UUID  # required FK
    unit_number: str


class RentalSpaceBase(BaseModel):
    rental_space_type_id: uuid.UUID | None = None  # optional FK
```

Use `uuid.UUID` for:
- `id` fields on all Read schemas
- Required foreign key fields on Create/Base schemas (e.g., `building_id: uuid.UUID`)
- Optional foreign key fields (e.g., `rental_space_type_id: uuid.UUID | None = None`)
- Nested summary schemas (e.g., `TenantSummary.id: uuid.UUID`)
- Filter schemas when filtering by a UUID FK (e.g., `rental_space_type_id: uuid.UUID | None = None`)

Do NOT use `uuid.UUID` for:
- Fields that are not UUIDs (e.g., Celery task IDs which are arbitrary strings)
- `int` primary keys (use `int` as before)

The type must be consistent across all layers: `Mapped[uuid.UUID]` in the model, `uuid.UUID` in the schema, `Uuid` in the column. See [sqlalchemy-usage.md](sqlalchemy-usage.md#uuid-primary-keys) for the model-side pattern.

### Decimal Fields

Use `Decimal` (not `float`) for monetary values, prices, and precise measurements. This matches the `Numeric` column type in the ORM layer and preserves precision across serialization.

```python
from decimal import Decimal
from typing import Annotated
from pydantic import BaseModel, ConfigDict, Field


class ProductBase(BaseModel):
    price: Annotated[Decimal, Field(ge=0, decimal_places=2)]
    tax_rate: Annotated[Decimal, Field(ge=0, le=1, decimal_places=4)]


class ProductRead(ProductBase):
    model_config = ConfigDict(from_attributes=True)

    id: int
```

Pydantic v2 serializes `Decimal` as strings in JSON by default. This is the correct behavior for precision — clients parse the string into their own decimal type. If you need numeric JSON output for a specific use case, configure it per-field with `PlainSerializer`.

**Anti-pattern — using `float` for monetary values:**

```python
# Wrong — float loses precision (0.1 + 0.2 != 0.3)
class ProductBase(BaseModel):
    price: float
```

The `Decimal` type must be consistent across all three layers: `Mapped[Decimal]` in the model, `Decimal` in the schema, `Numeric(p, s)` in the column. Mixing `float` anywhere in the chain defeats the purpose of using `Numeric`.

### Date and Time Fields

Use Python's `datetime` module types in schemas to match the ORM model types. Pydantic handles serialization to ISO 8601 strings automatically.

```python
from datetime import date, datetime, time

from pydantic import BaseModel, ConfigDict


class EventBase(BaseModel):
    event_date: date                        # "2026-04-25"
    start_time: time | None = None          # "14:30:00"


class EventRead(EventBase):
    model_config = ConfigDict(from_attributes=True)

    id: int
    created_at: datetime                    # "2026-04-25T14:30:00"
```

| Python type | JSON format | Example |
|---|---|---|
| `date` | `YYYY-MM-DD` | `"2026-04-25"` |
| `time` | `HH:MM:SS` | `"14:30:00"` |
| `datetime` | `YYYY-MM-DDTHH:MM:SS` | `"2026-04-25T14:30:00"` |

The types must match across layers: `Mapped[date]` in the model, `date` in the schema, `Date` in the column. Never use `str` for date/time fields — it breaks validation, serialization, and comparison operators.

## Swagger Examples

For complex or business-critical models, add `examples` via `Field` or `model_config` to improve API documentation.

### Per-Field Examples

```python
class OrderCreate(BaseModel):
    model_config = ConfigDict(extra="forbid")

    product_sku: Annotated[str, Field(
        min_length=3,
        max_length=50,
        examples=["WD-CHAIR-001"],
    )]
    quantity: Annotated[int, Field(gt=0, examples=[2])]
    shipping_address: Annotated[str, Field(
        min_length=10,
        examples=["Keizersgracht 100, 1015 AA Amsterdam"],
    )]
```

### Model-Level Examples

```python
class OrderCreate(BaseModel):
    model_config = ConfigDict(
        extra="forbid",
        json_schema_extra={
            "examples": [
                {
                    "product_sku": "WD-CHAIR-001",
                    "quantity": 2,
                    "shipping_address": "Keizersgracht 100, 1015 AA Amsterdam",
                }
            ]
        },
    )

    product_sku: Annotated[str, Field(min_length=3, max_length=50)]
    quantity: Annotated[int, Field(gt=0)]
    shipping_address: Annotated[str, Field(min_length=10)]
```

Only add examples where they clarify format or business meaning — not on every schema.

## Validators

### Field Validator

Use `field_validator` for single-field validation. Keep validators small and focused.

```python
from pydantic import field_validator


class UserCreate(UserBase):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    username: Annotated[str, Field(min_length=3, max_length=30)]

    @field_validator("username")
    @classmethod
    def validate_username(cls, v: str) -> str:
        if v.lower() in ("admin", "root", "system"):
            raise ValueError("Reserved username")
        return v.lower()
```

- Always return the (possibly transformed) value
- Use `@classmethod` decorator
- One validator per concern — don't combine unrelated checks

### Model Validator

Use `model_validator` for cross-field validation that depends on multiple fields.

```python
from pydantic import model_validator


class DateRangeFilter(BaseModel):
    model_config = ConfigDict(extra="forbid")

    start_date: date
    end_date: date

    @model_validator(mode="after")
    def validate_date_range(self) -> "DateRangeFilter":
        if self.end_date < self.start_date:
            raise ValueError("end_date must be after start_date")
        return self
```

Never put database queries or external calls in validators — validation is for data shape and rules only. Uniqueness checks belong in the service layer via database constraints.

## Computed Fields

Use `computed_field` for derived values in response schemas. They appear in the API response and OpenAPI docs but are not stored in the database.

```python
from pydantic import computed_field


class ProductRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    price: Decimal
    tax_rate: Decimal

    @computed_field
    @property
    def total_price(self) -> Decimal:
        return self.price * (1 + self.tax_rate)
```

## Internal Schemas

Use an `Internal` schema when the API input differs from what gets stored — most commonly for password hashing or server-generated fields.

```python
class UserCreate(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    email: Annotated[EmailStr, Field(max_length=255)]
    name: Annotated[str, Field(min_length=1, max_length=255)]
    password: Annotated[str, Field(min_length=8, max_length=128)]


class UserCreateInternal(BaseModel):
    email: str
    name: str
    hashed_password: str
```

The service layer converts between them:

```python
class UserService(CrudService[User, int, UserCreateInternal, UserUpdate]):
    async def create_user(self, obj_in: UserCreate) -> User:
        hashed = hash_password(obj_in.password)
        internal = UserCreateInternal(
            email=obj_in.email,
            name=obj_in.name,
            hashed_password=hashed,
        )
        return await self.repository.create(internal)
```

The API endpoint receives `UserCreate` (with plain password), the service produces `UserCreateInternal` (with hashed password), and the repository stores it. The plain password never reaches the repository.

### File Placement

Internal schemas live in the domain's `schemas.py` alongside the API-facing schemas. The API layer never imports them — they are an implementation detail of the service.

When a domain accumulates many internal schemas, split into two files:

```
app/users/
├── schemas.py            # API-facing: UserBase, UserCreate, UserUpdate, UserRead
├── internal_schemas.py   # Internal: UserCreateInternal, UserPasswordReset, ...
├── service.py            # Imports from both schemas.py and internal_schemas.py
├── api.py                # Only imports from schemas.py
```

Internal-only data carriers that don't need validation, type coercion, or `model_dump` must use a plain `dataclass`, not a Pydantic model. Reserve Pydantic for schemas that actually use its features.

```python
from dataclasses import dataclass


@dataclass
class UserCreateInternal:
    email: str
    name: str
    hashed_password: str
```

## Nested Schemas for Relationships

When returning related data, create slim nested schemas instead of reusing full Read schemas. This avoids exposing unnecessary fields and keeps payloads tight.

```python
class OrderAuthor(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    name: str


class OrderRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    status: str
    created_at: datetime
    user: OrderAuthor
```

Requires the relationship to be eagerly loaded in the repository query (see `selectinload` in [sqlalchemy-usage.md](sqlalchemy-usage.md)):

```python
stmt = select(Order).options(selectinload(Order.user)).where(Order.id == id)
```

Without eager loading, accessing `order.user` triggers a lazy load which fails in async sessions.

## Partial Updates

Update schemas have all fields optional. Combined with `model_dump(exclude_unset=True)` in the repository, only fields explicitly sent by the client are modified.

```python
class UserUpdate(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    email: Annotated[EmailStr | None, Field(max_length=255, default=None)]
    name: Annotated[str | None, Field(min_length=1, max_length=255, default=None)]
    is_active: bool | None = None
```

A request with `{"name": "Alice"}` only updates `name` — `email` and `is_active` are untouched because they were not sent (unset), not because they are `None`.

## Paginated Response

Define the paginated response wrapper in `app/shared/schemas.py`:

```python
from typing import Generic, TypeVar

from pydantic import BaseModel

SchemaType = TypeVar("SchemaType")


class PaginatedListResponse(BaseModel, Generic[SchemaType]):
    data: list[SchemaType]
    total_count: int
    has_more: bool
    page: int | None = None
    items_per_page: int | None = None
```

Usage in endpoints:

```python
@router.get("/", response_model=PaginatedListResponse[UserRead])
async def list_users(...):
```

Response shape:

```json
{
    "data": [{"id": 1, "name": "Alice", ...}, ...],
    "total_count": 150,
    "has_more": true,
    "page": 2,
    "items_per_page": 20
}
```

Use `PaginatedListResponse` for paginated list endpoints. Simple lists that don't need pagination metadata use `list[SchemaType]` directly. Single-resource endpoints return the schema directly (`response_model=UserRead`).

## Paginated Request Query

Define a shared query parameter model for paginated endpoints in `app/shared/schemas.py`:

```python
from pydantic import BaseModel, Field


class PaginatedRequestQuery(BaseModel):
    page: int = Field(1, ge=1)
    items_per_page: int = Field(20, ge=1, le=100)
    sort: str | None = Field(
        None,
        description=(
            "Sort by one or more fields. Format: 'field1,-field2' "
            "where '-' prefix indicates descending order."
        ),
    )
```

Used via `Depends()` in endpoints alongside the filter schema:

```python
@router.get("/", response_model=PaginatedListResponse[UserRead])
async def list_users(
    user_service: FromDishka[UserService],
    pagination: PaginatedRequestQuery = Depends(),
    filters: UserFilter = Depends(),
):
```

This consolidates pagination and sort parameters in one place — defaults, validation bounds, and the sort description are defined once. See [api-endpoints.md](api-endpoints.md) for the full endpoint pattern.

## Filter Schemas

Define a `{Model}Filter` schema for each domain that supports filtering. Filter schemas use `Depends()` in the endpoint to unpack query parameters — this is one of the accepted uses of `Depends()` (see [api-endpoints.md](api-endpoints.md)).

### Simple Filters (Suffix Convention)

For domains with straightforward filtering needs, use field naming with suffix conventions:

```python
from datetime import datetime

from pydantic import BaseModel, ConfigDict


class UserFilter(BaseModel):
    model_config = ConfigDict(extra="forbid")

    email: str | None = None
    is_active: bool | None = None
    created_at_gte: datetime | None = None
    created_at_lte: datetime | None = None
    name_like: str | None = None
```

Suffix conventions:
- No suffix — exact match (`==`)
- `_gt` / `_gte` — greater than / greater or equal
- `_lt` / `_lte` — less than / less or equal
- `_like` — case-insensitive contains (`ILIKE %value%`)

The base repository's `_apply_filters` translates these automatically. See [repository-pattern.md](repository-pattern.md) for the implementation.

### Complex Filters (FilterBuilder)

When a domain needs filters that go beyond the suffix convention — search across multiple columns, conditional logic, or combined expressions — define a `FilterBuilder` alongside the filter schema:

```python
# app/users/schemas.py
class UserFilter(BaseModel):
    model_config = ConfigDict(extra="forbid")

    email: str | None = None
    is_active: bool | None = None
    created_from: datetime | None = None
    created_until: datetime | None = None
    search: str | None = None
```

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

The `FilterBuilder` lives in a separate `filters.py` file in the domain directory since it imports the SQLAlchemy model. The filter schema stays in `schemas.py` — it's a pure Pydantic model with no ORM imports.

Use the suffix convention for simple domains. Switch to a `FilterBuilder` when the filtering logic outgrows what suffixes can express. See [repository-pattern.md](repository-pattern.md) and [service-layer.md](service-layer.md) for how these integrate into the query pipeline.

## Polymorphic Schemas

Use discriminated unions when a single endpoint accepts or returns different shapes based on a type field.

### Polymorphic Request Body

```python
from typing import Annotated, Literal, Union
from pydantic import BaseModel, ConfigDict, Field


class EmailNotification(BaseModel):
    model_config = ConfigDict(extra="forbid")

    type: Literal["email"]
    recipient: Annotated[str, Field(max_length=255)]
    subject: Annotated[str, Field(max_length=255)]
    body: str


class SmsNotification(BaseModel):
    model_config = ConfigDict(extra="forbid")

    type: Literal["sms"]
    phone_number: Annotated[str, Field(pattern=r"^\+\d{10,15}$")]
    message: Annotated[str, Field(max_length=160)]


NotificationCreate = Annotated[
    Union[EmailNotification, SmsNotification],
    Field(discriminator="type"),
]
```

Use in an endpoint:

```python
@router.post("/notifications")
async def send_notification(
    notification: NotificationCreate,
    service: FromDishka[NotificationService],
) -> NotificationRead:
    return await service.send(notification)
```

FastAPI validates the `type` field first and selects the matching schema. Invalid types return a clear error. The OpenAPI docs show both variants.

### Polymorphic Response Body

```python
class EmailNotificationRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    type: Literal["email"]
    id: int
    recipient: str
    subject: str
    status: str


class SmsNotificationRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    type: Literal["sms"]
    id: int
    phone_number: str
    message: str
    status: str


NotificationRead = Annotated[
    Union[EmailNotificationRead, SmsNotificationRead],
    Field(discriminator="type"),
]
```

The discriminator field (`type`) must be present on the ORM model so `from_attributes` can resolve the correct schema variant.

## Testing Schemas

Test validators, constraints, and ORM conversion in the domain's test file.

### Validation Rules

```python
import pytest
from pydantic import ValidationError

from app.users.schemas import UserCreate


def test_create_valid():
    user = UserCreate(email="alice@example.com", name="Alice", password="secure123")
    assert user.email == "alice@example.com"


def test_create_rejects_short_password():
    with pytest.raises(ValidationError) as exc_info:
        UserCreate(email="alice@example.com", name="Alice", password="short")
    assert "password" in str(exc_info.value)


def test_create_rejects_extra_fields():
    with pytest.raises(ValidationError):
        UserCreate(
            email="alice@example.com",
            name="Alice",
            password="secure123",
            is_admin=True,  # not allowed
        )


def test_create_strips_whitespace():
    user = UserCreate(email="alice@example.com", name="  Alice  ", password="secure123")
    assert user.name == "Alice"
```

### ORM Conversion

```python
from app.users.schemas import UserRead


def test_read_from_orm(user_factory):
    orm_user = user_factory(id=1, email="alice@example.com", name="Alice")
    schema = UserRead.model_validate(orm_user)
    assert schema.id == 1
    assert schema.email == "alice@example.com"
```

### Partial Update

```python
from app.users.schemas import UserUpdate


def test_update_exclude_unset():
    update = UserUpdate(name="Bob")
    data = update.model_dump(exclude_unset=True)
    assert data == {"name": "Bob"}
    assert "email" not in data
    assert "is_active" not in data
```

## Anti-Patterns

- **ORM as API response** — never return SQLAlchemy model instances directly from endpoints. Always convert to a Read schema.
- **God schema** — one schema used for create, read, update, and response. Split by use-case.
- **DB queries in validators** — uniqueness checks, existence checks, and any I/O belong in the service layer, not in `field_validator` or `model_validator`.
- **Pydantic in repositories** — repositories receive Pydantic schemas for `create`/`update` (via `model_dump`), but should not import or depend on Pydantic types in their internal logic.
- **Missing `from_attributes`** — forgetting `ConfigDict(from_attributes=True)` on Read schemas causes `model_validate(orm_obj)` to fail silently or raise.
- **Deep nesting** — avoid schemas with more than 2 levels of nesting. Flatten or use separate endpoints for deeply nested data.
