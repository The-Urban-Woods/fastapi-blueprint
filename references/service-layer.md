# Service Layer

## Core Concepts

### 1. KISS (Keep It Simple)

Choose the simplest solution that works. Complexity must be justified by concrete requirements.

### 2. Single Responsibility (SRP)

Each unit should have one reason to change. Separate concerns into focused components.

### 3. Composition Over Inheritance

Build behavior by combining objects, not extending classes.

### 4. Rule of Three

Wait until you have three instances before abstracting. Duplication is often better than premature abstraction.

## Fundamental Patterns

### Apply Single Responsibility Principle

Each class or function should have one reason to change.

```python
# BAD: Endpoint does everything
@router.post("/", status_code=status.HTTP_201_CREATED)
async def create_user(user_in: UserCreate, db: AsyncSession = Depends(get_db)):
    # Validation + database access + business logic all in one
    existing = await db.execute(select(User).where(User.email == user_in.email))
    if existing.scalars().first():
        raise HTTPException(status_code=400, detail="Email already registered")

    user = User(**user_in.model_dump())
    db.add(user)
    await db.flush()
    await db.refresh(user)
    return user

# GOOD: Separated concerns
# service.py — business logic only
class UserService(CrudService[User, int, UserCreate, UserUpdate]):
    def __init__(self, repo: UserRepository) -> None:
        self.repo = repo

    async def create(self, obj_in: UserCreate) -> User:
        existing = await self.repo.get_by_email(obj_in.email)
        if existing:
            raise ValueError("Email already registered")
        return await self.repo.create(obj_in)

# api.py — HTTP concerns only
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
```

### Use Composition Over Inheritance

Build behavior by combining objects rather than inheriting. All dependencies are injected via constructor, resolved by dishka.

```python
# Inheritance: Rigid and hard to test
class EmailNotificationService(NotificationService):
    def __init__(self):
        super().__init__()
        self._smtp = SmtpClient()  # Hard to mock, not injectable

    def notify(self, user: User, message: str) -> None:
        self._smtp.send(user.email, message)

# Composition: Flexible, testable, dishka-compatible
class NotificationService(Service):
    """Send notifications via multiple channels."""

    def __init__(
        self,
        email_sender: EmailSender,
        sms_sender: SmsSender,
    ) -> None:
        self._email = email_sender
        self._sms = sms_sender

    async def notify(self, user: User, message: str, channels: set[str] | None = None) -> None:
        channels = channels or {"email"}

        if "email" in channels:
            await self._email.send(user.email, message)

        if "sms" in channels and user.phone:
            await self._sms.send(user.phone, message)
```

Register in `app/providers.py` — dishka resolves `EmailSender` and `SmsSender` automatically:

```python
class RequestProvider(Provider):
    scope = Scope.REQUEST

    email_sender = provide(EmailSender)
    sms_sender = provide(SmsSender)
    notification_service = provide(NotificationService)
```

Now HTTP changes don't affect business logic, and vice versa.

### Avoiding Common Anti-Patterns

These patterns are commonly found in existing codebases. When applying the blueprint to an existing project, refactor these toward the correct patterns.

**Don't expose internal types:**

```python
# BAD: Leaking ORM model to API
@app.get("/users/{id}")
def get_user(id: str) -> UserModel:  # SQLAlchemy model
    return db.query(UserModel).get(id)

# GOOD: Use response schemas and service layer
@router.get("/{user_id}", response_model=UserRead)
async def get_user(user_id: int, user_service: FromDishka[UserService]):
    user = await user_service.get(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

**Don't mix I/O with business logic:**

```python
# BAD: SQL embedded in business logic
def calculate_discount(user_id: str) -> float:
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    orders = db.query("SELECT * FROM orders WHERE user_id = ?", user_id)
    # Business logic mixed with data access

# GOOD: Repository handles I/O, service has pure logic
def calculate_discount(user: User, order_history: list[Order]) -> float:
    # Pure business logic, easily testable
    if len(order_history) > 10:
        return 0.15
    return 0.0
```

## Abstract Base Classes

Two ABCs in `app/shared/service.py`:

- **`Service`** — marker interface for all services. Use when a service doesn't follow standard CRUD patterns (e.g., `AuthService`, `NotificationService`).
- **`CrudService`** — generic ABC for services that provide standard CRUD operations over a domain entity.

```python
from abc import ABC, abstractmethod
from typing import Generic, TypeVar

ModelType = TypeVar("ModelType")
IdType = TypeVar("IdType")
CreateSchemaType = TypeVar("CreateSchemaType")
UpdateSchemaType = TypeVar("UpdateSchemaType")


class Service(ABC):
    """Marker interface for all services."""
    pass


class CrudService(Service, Generic[ModelType, IdType, CreateSchemaType, UpdateSchemaType]):

    @abstractmethod
    async def get(self, id: IdType) -> ModelType | None: ...

    @abstractmethod
    async def find_all(self, *, skip: int = 0, limit: int = 100, sort: str | None = None) -> list[ModelType]: ...

    @abstractmethod
    async def create(self, obj_in: CreateSchemaType) -> ModelType: ...

    @abstractmethod
    async def update(self, id: IdType, obj_in: UpdateSchemaType) -> ModelType | None: ...

    @abstractmethod
    async def delete(self, id: IdType) -> bool: ...
```

**`CrudService` vs `Repository` differences:**
- `update` takes an `id` (looks up entity internally) — repositories take the entity directly
- `delete` takes an `id` and returns `bool` — repositories take the entity and return `None`
- Services add business logic around the raw data access

## Domain Service (CRUD)

Extend `CrudService` for standard CRUD domains:

```python
from app.shared.service import CrudService
from app.users.models import User
from app.users.repository import UserRepository
from app.users.schemas import UserCreate, UserUpdate


class UserService(CrudService[User, int, UserCreate, UserUpdate]):
    def __init__(self, repo: UserRepository):
        self.repo = repo

    async def create(self, obj_in: UserCreate) -> User:
        existing = await self.repo.get_by_email(obj_in.email)
        if existing:
            raise ValueError("Email already registered")
        return await self.repo.create(obj_in)

    async def get(self, id: int) -> User | None:
        return await self.repo.get(id)

    async def find_all(
        self, *, skip: int = 0, limit: int = 100, sort: str | None = None
    ) -> list[User]:
        return await self.repo.find_all(skip=skip, limit=limit, sort=sort)

    async def update(self, id: int, obj_in: UserUpdate) -> User | None:
        user = await self.repo.get(id)
        if not user:
            return None
        return await self.repo.update(user, obj_in)

    async def delete(self, id: int) -> bool:
        user = await self.repo.get(id)
        if not user:
            return False
        await self.repo.delete(user)
        return True
```

## Paginated and Filtered Queries

For domains whose repository extends `PaginatedSqlAlchemyRepository`, add `find_paginated` to the service. This is not part of the `CrudService` ABC — it's an optional method for domains that need pagination and filtering.

### With Generic Filters

When filters use the suffix convention (exact match, comparisons), the service passes them through to the repository's `_apply_filters`:

```python
from app.shared.paginated_sqlalchemy_repository import PaginatedResult


class OrderService(CrudService[Order, int, OrderCreate, OrderUpdate]):
    def __init__(self, repo: OrderRepository):
        self.repo = repo

    async def find_paginated(
        self,
        *,
        filters: OrderFilter | None = None,
        page: int = 1,
        items_per_page: int = 20,
        sort: str | None = None,
    ) -> PaginatedResult[Order]:
        conditions = (
            self.repo._apply_filters(filters.model_dump(exclude_unset=True))
            if filters
            else None
        )
        return await self.repo.find_paginated(
            page=page, items_per_page=items_per_page, sort=sort, conditions=conditions
        )
```

### With FilterBuilder

When a domain needs complex filter logic (search across columns, conditional expressions), use a `FilterBuilder` instead:

```python
from app.shared.paginated_sqlalchemy_repository import PaginatedResult
from app.users.filters import UserFilterBuilder


class UserService(CrudService[User, int, UserCreate, UserUpdate]):
    def __init__(self, repo: UserRepository):
        self.repo = repo

    async def find_paginated(
        self,
        *,
        filters: UserFilter | None = None,
        page: int = 1,
        items_per_page: int = 20,
        sort: str | None = None,
    ) -> PaginatedResult[User]:
        conditions = UserFilterBuilder.build(filters) if filters else None
        return await self.repo.find_paginated(
            page=page, items_per_page=items_per_page, sort=sort, conditions=conditions
        )
```

The service decides which approach fits the domain. The repository's `find_paginated` is agnostic — it only sees the resulting conditions list.

## Domain Service (Non-CRUD)

For services that don't follow CRUD patterns, extend the `Service` marker:

```python
from app.shared.service import Service


class AuthService(Service):
    def __init__(self, user_repo: UserRepository):
        self.user_repo = user_repo

    async def authenticate(self, email: str, password: str) -> User | None: ...

    async def refresh_token(self, token: str) -> str: ...
```

## Key Design Decisions

- **Constructor injection** — repositories are passed via `__init__`, wired automatically by dishka
- **No `db` parameter** — services do not know about `AsyncSession`. The repository handles all database access internally.
- **No framework imports** — services import nothing from FastAPI or dishka. They are plain Python classes, reusable in Celery tasks, CLI scripts, or tests.
- **Business exceptions** — raise `ValueError` or custom domain exceptions. The API layer maps these to HTTP responses.

## Rules

- Services contain **business logic only** — validation rules, orchestration, domain constraints
- Never import FastAPI, dishka, or SQLAlchemy in service modules
- Return domain models or raise exceptions — never return HTTP responses
- A service may depend on multiple repositories if the business operation spans domains
- Do NOT instantiate services manually — let dishka provide them

## Cross-Domain Services

When business logic spans multiple domains, inject multiple repositories:

```python
class OrderService(CrudService[Order, int, OrderCreate, OrderUpdate]):
    def __init__(self, order_repo: OrderRepository, user_repo: UserRepository):
        self.order_repo = order_repo
        self.user_repo = user_repo

    async def create(self, obj_in: OrderCreate) -> Order:
        user = await self.user_repo.get(obj_in.user_id)
        if not user or not user.is_active:
            raise ValueError("User not found or inactive")
        return await self.order_repo.create(obj_in)
```

Dishka resolves both repositories automatically from the constructor signature.

## Troubleshooting

**A class is growing and seems to have multiple responsibilities, but splitting it feels wrong.**
Apply the "reason to change" test: list every change that could require editing this class. If the list has items from different domains (e.g., HTTP parsing AND business rules AND formatting), split it. If all changes stem from the same domain concern, the class may be appropriately sized.

**Injecting all dependencies through the constructor is producing constructors with 7+ parameters.**
This is a sign of too many responsibilities in one class, not a problem with dependency injection. Split the class into smaller units first, then each constructor naturally becomes smaller.

**Composition is producing deeply nested wrapper objects that are hard to trace.**
Keep the composition shallow (2-3 levels). If wrapping is the only mechanism, consider whether a Protocol-based approach or simple function composition would be cleaner than a chain of decorator objects.

**The rule of three says not to abstract yet, but the duplication is causing bugs when one copy is updated but not the other.**
Duplication that diverges in dangerous ways should be abstracted sooner. The rule of three is a heuristic, not a law. If the copies are already diverging incorrectly, extract immediately and add a test that exercises the shared behavior.

**A service layer is importing from the API layer, breaking the dependency direction.**
This is a layering violation. The service layer must not import from handlers. Introduce a shared types/models layer that both can import from, keeping the dependency arrow pointing downward (API → Service → Repository).