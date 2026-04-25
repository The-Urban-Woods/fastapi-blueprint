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
