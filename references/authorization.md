# Authorization

Permission patterns for controlling what authenticated users can do. Builds on the authentication dependencies from [authentication.md](authentication.md).

## Overview

Three patterns, from simple to complex:

| Pattern | Use When | Enforced At |
| --- | --- | --- |
| **Superuser check** | Admin-only endpoints | Endpoint (`Depends()`) |
| **Resource ownership** | Users modify their own data | Service layer |
| **Role-based access (RBAC)** | Multiple permission levels | Endpoint (`Depends()`) + service |

Choose the simplest pattern that fits. Most projects start with superuser + ownership and add RBAC only when needed.

## Superuser Check

The simplest authorization pattern. Use `get_current_superuser` from [authentication.md](authentication.md) as an endpoint dependency:

```python
from app.auth.dependencies import get_current_superuser


# As a dependency parameter — when you need the user object
@router.get("/admin/stats")
async def admin_stats(
    current_user: User = Depends(get_current_superuser),
):
    ...

# As endpoint dependency — when you only need the guard
@router.delete(
    "/admin/items/{item_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    dependencies=[Depends(get_current_superuser)],
)
async def delete_item(item_id: int, item_service: FromDishka[ItemService]):
    ...
```

Use `dependencies=[Depends(get_current_superuser)]` when you don't need the user object in the endpoint body. Use the parameter form when you do.

For admin user management endpoints (list users, hard delete, tier assignment), see [user-management.md](user-management.md).

## Resource Ownership

Check ownership in the **service layer**, not in a dependency. This keeps business logic where it belongs and allows superuser bypass as a business rule.

```python
# app/posts/service.py
class PostService(CrudService[Post, int, PostCreate, PostUpdate]):
    def __init__(self, repo: PostRepository):
        self.repo = repo

    async def update(self, id: int, obj_in: PostUpdate, current_user: User) -> Post:
        post = await self.repo.get(id)
        if not post:
            raise ValueError("Post not found")
        if post.created_by_user_id != current_user.id and not current_user.is_superuser:
            raise PermissionError("You can only edit your own posts")
        return await self.repo.update(post, obj_in)

    async def delete(self, id: int, current_user: User) -> bool:
        post = await self.repo.get(id)
        if not post:
            return False
        if post.created_by_user_id != current_user.id and not current_user.is_superuser:
            raise PermissionError("You can only delete your own posts")
        await self.repo.delete(post)
        return True
```

The endpoint passes the current user to the service:

```python
# app/posts/api.py
from app.auth.dependencies import get_current_user


@router.patch("/{post_id}", response_model=PostRead)
async def update_post(
    post_id: int,
    post_in: PostUpdate,
    post_service: FromDishka[PostService],
    current_user: User = Depends(get_current_user),
):
    try:
        return await post_service.update(post_id, post_in, current_user)
    except ValueError as e:
        raise HTTPException(status_code=404, detail=str(e))
    except PermissionError as e:
        raise HTTPException(status_code=403, detail=str(e))
```

The ownership validation pattern is always: **retrieve resource, check owner, allow or deny**. The superuser bypass is a business decision — include it where appropriate.

### Ownership Model Column

Models that support ownership need a `created_by_user_id` foreign key:

```python
class Post(TimestampMixin, Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(255))
    content: Mapped[str] = mapped_column(Text)
    created_by_user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)

    created_by: Mapped["User"] = relationship()
```

## Role-Based Access Control (RBAC)

For applications with multiple permission levels beyond superuser/regular user.

### Role as Enum on User Model

The simpler approach — suitable when roles are fixed and rarely change:

```python
import enum

from sqlalchemy import Enum


class Role(str, enum.Enum):
    USER = "user"
    MODERATOR = "moderator"
    ADMIN = "admin"


class User(TimestampMixin, Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    name: Mapped[str] = mapped_column(String(255))
    hashed_password: Mapped[str] = mapped_column(String(255))
    role: Mapped[Role] = mapped_column(Enum(Role), default=Role.USER)
    is_superuser: Mapped[bool] = mapped_column(default=False)
```

### Role as Separate Table

More flexible — suitable when roles need to be managed at runtime or when users can have multiple roles:

```python
from sqlalchemy import Column, ForeignKey, Table

user_roles = Table(
    "user_roles",
    Base.metadata,
    Column("user_id", ForeignKey("users.id"), primary_key=True),
    Column("role_id", ForeignKey("roles.id"), primary_key=True),
)


class Role(Base):
    __tablename__ = "roles"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)
    description: Mapped[str | None] = mapped_column(String(255), default=None)


class User(TimestampMixin, Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    name: Mapped[str] = mapped_column(String(255))
    hashed_password: Mapped[str] = mapped_column(String(255))
    is_superuser: Mapped[bool] = mapped_column(default=False)

    roles: Mapped[list["Role"]] = relationship(secondary=user_roles)
```

Choose the approach that fits the project. Start with enum, migrate to table if runtime role management becomes a requirement.

### require_role Dependency Factory

A reusable dependency that checks the user's role against a minimum required level:

```python
# app/auth/dependencies.py
from app.auth.dependencies import get_current_user


ROLE_HIERARCHY = {
    Role.USER: 0,
    Role.MODERATOR: 1,
    Role.ADMIN: 2,
}


def require_role(minimum_role: Role):
    async def check_role(
        current_user: User = Depends(get_current_user),
    ) -> User:
        if current_user.is_superuser:
            return current_user

        user_level = ROLE_HIERARCHY.get(current_user.role, 0)
        required_level = ROLE_HIERARCHY[minimum_role]

        if user_level < required_level:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Insufficient permissions",
            )
        return current_user

    return check_role
```

Usage in endpoints:

```python
@router.delete("/{post_id}", dependencies=[Depends(require_role(Role.MODERATOR))])
async def delete_post(post_id: int, post_service: FromDishka[PostService]):
    deleted = await post_service.delete(post_id)
    if not deleted:
        raise HTTPException(status_code=404, detail="Post not found")
```

For the separate-table approach, replace `current_user.role` with a check against the user's roles list:

```python
def require_role(minimum_role: str):
    async def check_role(
        current_user: User = Depends(get_current_user),
    ) -> User:
        if current_user.is_superuser:
            return current_user

        user_role_names = {r.name for r in current_user.roles}
        if minimum_role not in user_role_names:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Insufficient permissions",
            )
        return current_user

    return check_role
```

When using the table approach, ensure roles are eagerly loaded on the user query to avoid lazy loading in async sessions. See `selectinload` in [sqlalchemy-usage.md](sqlalchemy-usage.md).

## Where Permission Checks Happen

| Check Type | Where | Why |
| --- | --- | --- |
| Authentication (is user logged in?) | `Depends(get_current_user)` | Framework concern — before endpoint runs |
| Role/superuser guard | `Depends(require_role(...))` | Access policy — before endpoint runs |
| Ownership validation | Service method | Business rule — needs entity data |
| Domain-specific rules | Service method | Business logic — e.g., "can't edit published posts" |

Never put permission checks in repositories. Repositories handle data access only.

## Error Responses

Always return `403 Forbidden` for authorization failures, not `404 Not Found`. Returning 404 to hide resource existence is a security-through-obscurity pattern — only use it when resource existence itself is sensitive information (e.g., private user profiles).

Map service exceptions to HTTP responses at the endpoint level:

```python
except PermissionError as e:
    raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail=str(e))
```

Use generic error messages in production. Avoid leaking details like "You are not the owner of this resource" — prefer "Insufficient permissions."

## Anti-Patterns

- **Permission logic in repositories** — repositories handle data, not authorization. Check permissions in the service layer.
- **Hardcoded role strings** — use an enum or constants, not `if user.role == "admin"` scattered across endpoints.
- **Checking permissions after the action** — always validate before performing the operation.
- **Different error codes leaking information** — don't return "resource exists but you can't access it" vs "resource doesn't exist." Use consistent responses.
- **Skipping superuser bypass** — superusers should bypass ownership checks. Forgetting this creates support headaches.
- **Authorization in middleware** — middleware runs on every request including public endpoints. Use endpoint-level dependencies for authorization.
