# User Management

Registration, profile management, password changes, and admin user operations. Builds on auth dependencies from [authentication.md](authentication.md) and permission patterns from [authorization.md](authorization.md).

User management lives in the `users` domain — not in `auth`. The auth domain handles identity and tokens; the users domain handles user lifecycle.

## User Model

The User model extends the base with authentication and authorization fields:

```python
from datetime import datetime

from sqlalchemy import String, func
from sqlalchemy.orm import Mapped, mapped_column

from app.shared.database import Base, SoftDeleteMixin, TimestampMixin


class User(SoftDeleteMixin, TimestampMixin, Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, index=True)
    name: Mapped[str] = mapped_column(String(255))
    hashed_password: Mapped[str] = mapped_column(String(255))
    is_active: Mapped[bool] = mapped_column(default=True)
    is_superuser: Mapped[bool] = mapped_column(default=False)
```

If using RBAC with an enum, add a `role` column (see [authorization.md](authorization.md) for both enum and table approaches).

`hashed_password` is never exposed in any Read schema. `SoftDeleteMixin` provides `is_deleted` and `deleted_at` (see [sqlalchemy-usage.md](sqlalchemy-usage.md)).

## User Schemas

```python
# app/users/schemas.py
from typing import Annotated

from pydantic import BaseModel, ConfigDict, EmailStr, Field


class UserBase(BaseModel):
    email: Annotated[EmailStr, Field(max_length=255)]
    username: Annotated[str, Field(min_length=3, max_length=50, pattern=r"^[a-z0-9_]+$")]
    name: Annotated[str, Field(min_length=1, max_length=255)]


class UserCreate(UserBase):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    password: Annotated[str, Field(min_length=8, max_length=128)]


class UserUpdate(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    email: Annotated[EmailStr | None, Field(max_length=255, default=None)]
    name: Annotated[str | None, Field(min_length=1, max_length=255, default=None)]


class UserRead(UserBase):
    model_config = ConfigDict(from_attributes=True)

    id: int
    is_active: bool
    created_at: datetime
    updated_at: datetime
```

`UserRead` excludes `hashed_password`, `is_deleted`, `deleted_at`, and `is_superuser`. Admin-facing endpoints can use a separate `UserAdminRead` that includes these fields if needed.

### Internal Schema

The `UserCreateInternal` carries the hashed password from the service to the repository. Uses a `dataclass` since it's an internal-only data carrier with no validation needs (see [schemas.md](schemas.md)):

```python
# app/users/internal_schemas.py
from dataclasses import dataclass


@dataclass
class UserCreateInternal:
    email: str
    username: str
    name: str
    hashed_password: str
```

## Registration

The service validates uniqueness, hashes the password, and converts to the internal schema:

```python
# app/users/service.py
from app.auth.utils import get_password_hash
from app.users.internal_schemas import UserCreateInternal


class UserService(CrudService[User, int, UserCreateInternal, UserUpdate]):
    def __init__(self, repo: UserRepository):
        self.repo = repo

    async def register(self, obj_in: UserCreate) -> User:
        existing_email = await self.repo.get_by_email(obj_in.email)
        if existing_email:
            raise ValueError("Email already registered")

        existing_username = await self.repo.get_by_username(obj_in.username)
        if existing_username:
            raise ValueError("Username already taken")

        hashed = get_password_hash(obj_in.password)
        internal = UserCreateInternal(
            email=obj_in.email,
            username=obj_in.username,
            name=obj_in.name,
            hashed_password=hashed,
        )
        return await self.repo.create(internal)
```

The endpoint:

```python
# app/users/api.py
@router.post("/register", response_model=UserRead, status_code=status.HTTP_201_CREATED)
async def register(
    user_in: UserCreate,
    user_service: FromDishka[UserService],
):
    try:
        return await user_service.register(user_in)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
```

The API receives `UserCreate` (with plain password), the service hashes and converts to `UserCreateInternal`, the repository stores it. The plain password never reaches the data layer.

## Profile Endpoints

Profile endpoints use `get_current_user` from [authentication.md](authentication.md) to identify the caller:

```python
# app/users/api.py
from app.auth.dependencies import get_current_user


@router.get("/me", response_model=UserRead)
async def get_profile(
    current_user: User = Depends(get_current_user),
):
    return current_user


@router.patch("/me", response_model=UserRead)
async def update_profile(
    user_in: UserUpdate,
    current_user: User = Depends(get_current_user),
    user_service: FromDishka[UserService],
):
    return await user_service.update(current_user.id, user_in)
```

Users can only update their own profile via `/me`. The service validates unique constraints before persisting:

```python
async def update(self, id: int, obj_in: UserUpdate) -> User | None:
    user = await self.repo.get(id)
    if not user:
        return None

    if obj_in.email and obj_in.email != user.email:
        existing = await self.repo.get_by_email(obj_in.email)
        if existing:
            raise ValueError("Email already registered")

    return await self.repo.update(user, obj_in)
```

## Password Change

A dedicated endpoint and schema for changing passwords:

```python
# app/users/schemas.py
class PasswordChange(BaseModel):
    model_config = ConfigDict(extra="forbid")

    current_password: Annotated[str, Field(min_length=1)]
    new_password: Annotated[str, Field(min_length=8, max_length=128)]
```

```python
# app/users/service.py
from app.auth.utils import get_password_hash, verify_password


async def change_password(self, user: User, obj_in: PasswordChange) -> None:
    if not verify_password(obj_in.current_password, user.hashed_password):
        raise ValueError("Current password is incorrect")
    user.hashed_password = get_password_hash(obj_in.new_password)
    await self.repo.db.flush()
```

```python
# app/users/api.py
@router.post("/me/password", status_code=status.HTTP_204_NO_CONTENT)
async def change_password(
    password_in: PasswordChange,
    current_user: User = Depends(get_current_user),
    user_service: FromDishka[UserService],
):
    try:
        await user_service.change_password(current_user, password_in)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
```

## Account Deletion

### Soft Delete (Self-Service)

Users can deactivate their own account. Data is preserved for recovery and audit:

```python
# app/users/service.py
from datetime import datetime, timezone


async def deactivate(self, id: int, current_user: User) -> bool:
    user = await self.repo.get(id)
    if not user:
        return False
    if user.id != current_user.id:
        raise PermissionError("You can only deactivate your own account")

    user.is_deleted = True
    user.deleted_at = datetime.now(timezone.utc)
    user.is_active = False
    await self.repo.db.flush()
    return True
```

```python
# app/users/api.py
@router.delete("/me", status_code=status.HTTP_204_NO_CONTENT)
async def deactivate_account(
    current_user: User = Depends(get_current_user),
    user_service: FromDishka[UserService],
    auth_service: FromDishka[AuthService],
    settings: FromDishka[Settings],
    token: str = Depends(oauth2_scheme),
):
    await user_service.deactivate(current_user.id, current_user)
    await auth_service.blacklist_token(token, settings)
```

After deactivation, the user's tokens are blacklisted so they're immediately logged out.

### Hard Delete (Admin Only)

Permanent removal for legal requirements, data breaches, or abuse:

```python
# app/users/service.py
async def hard_delete(self, id: int) -> bool:
    user = await self.repo.get(id)
    if not user:
        return False
    await self.repo.delete(user)
    return True
```

```python
# app/users/api.py
from app.auth.dependencies import get_current_superuser


@router.delete(
    "/{user_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    dependencies=[Depends(get_current_superuser)],
)
async def hard_delete_user(
    user_id: int,
    user_service: FromDishka[UserService],
):
    deleted = await user_service.hard_delete(user_id)
    if not deleted:
        raise HTTPException(status_code=404, detail="User not found")
```

## Admin User Operations

Admin endpoints live in a separate router or use `get_current_superuser`:

```python
# app/users/api.py
from app.auth.dependencies import get_current_superuser


@router.get(
    "/",
    response_model=PaginatedListResponse[UserRead],
    dependencies=[Depends(get_current_superuser)],
)
async def list_all_users(
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

Consider creating a `UserAdminRead` schema that includes fields like `is_superuser`, `is_active`, `is_deleted`, and `deleted_at` for admin views.

## Repository Queries

The user repository needs lookup methods beyond the standard CRUD:

```python
# app/users/repository.py
class UserRepository(SqlAlchemyRepository[User, int, UserCreateInternal, UserUpdate]):
    SORTABLE_FIELDS = {"name", "email", "username", "created_at"}

    def __init__(self, db: AsyncSession):
        super().__init__(User, db)

    async def get_by_email(self, email: str) -> User | None:
        result = await self.db.execute(select(User).where(User.email == email))
        return result.scalars().first()

    async def get_by_username(self, username: str) -> User | None:
        result = await self.db.execute(select(User).where(User.username == username))
        return result.scalars().first()
```
