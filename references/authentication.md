# Authentication

JWT-based authentication with access/refresh token pattern, password hashing, and user identity resolution. Adapted for Dishka DI.

## Domain Structure

Authentication lives in its own domain:

```
app/auth/
├── __init__.py
├── models.py        # TokenBlacklist model
├── schemas.py       # TokenResponse
├── utils.py         # Password hashing, token creation/verification
├── dependencies.py  # get_current_user, get_optional_user, get_current_superuser
├── service.py       # AuthService — login, logout, refresh logic
└── api.py           # /login, /logout, /refresh endpoints
```

User registration and profile management stay in the `users` domain — auth only handles identity verification and token lifecycle.

## Settings

Extend the `Settings` class in `app/shared/config.py`:

```python
from typing import Literal

from pydantic import SecretStr
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    DATABASE_URL: str = "sqlite+aiosqlite:///./app.db"
    API_V1_STR: str = "/api/v1"

    # Auth
    SECRET_KEY: SecretStr
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7
    COOKIE_SECURE: bool = True
    COOKIE_SAMESITE: Literal["lax", "strict", "none"] = "lax"

    model_config = {"env_file": ".env"}
```

`SECRET_KEY` uses `SecretStr` to prevent accidental logging. Generate with:

```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

Use separate secrets for development, staging, and production.

## Password Hashing

Pure utility functions in `app/auth/utils.py`. No framework imports.

```python
import bcrypt


def get_password_hash(password: str) -> str:
    return bcrypt.hashpw(password.encode(), bcrypt.gensalt()).decode()


def verify_password(plain_password: str, hashed_password: str) -> bool:
    return bcrypt.checkpw(plain_password.encode(), hashed_password.encode())
```

Bcrypt handles salt generation automatically. Each hash includes the salt, so no separate salt column is needed.

## JWT Tokens

### Token Types

The system uses a dual-token approach:

- **Access token** — short-lived (30 min default), sent in `Authorization: Bearer <token>` header. Stored in memory or sessionStorage on the client. Used for API requests.
- **Refresh token** — long-lived (7 days default), stored in an HTTP-only cookie. Cannot be accessed by JavaScript (XSS protection). Used only to obtain new access tokens.

### Token Creation

```python
from datetime import datetime, timedelta, timezone

from jose import jwt

from app.shared.config import Settings


def create_access_token(
    subject: str, settings: Settings, expires_delta: timedelta | None = None
) -> str:
    expire = datetime.now(timezone.utc) + (
        expires_delta or timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
    )
    payload = {
        "sub": subject,
        "exp": expire,
        "iat": datetime.now(timezone.utc),
        "token_type": "access",
    }
    return jwt.encode(payload, settings.SECRET_KEY.get_secret_value(), algorithm=settings.ALGORITHM)


def create_refresh_token(
    subject: str, settings: Settings, expires_delta: timedelta | None = None
) -> str:
    expire = datetime.now(timezone.utc) + (
        expires_delta or timedelta(days=settings.REFRESH_TOKEN_EXPIRE_DAYS)
    )
    payload = {
        "sub": subject,
        "exp": expire,
        "iat": datetime.now(timezone.utc),
        "token_type": "refresh",
    }
    return jwt.encode(payload, settings.SECRET_KEY.get_secret_value(), algorithm=settings.ALGORITHM)
```

The `sub` claim holds the user identifier (username, email, or ID — choose one and be consistent). `token_type` distinguishes access from refresh tokens during verification.

### Token Verification

```python
from dataclasses import dataclass

from jose import JWTError, jwt


@dataclass
class TokenData:
    subject: str


def verify_token(
    token: str, expected_type: str, settings: Settings
) -> TokenData | None:
    try:
        payload = jwt.decode(
            token, settings.SECRET_KEY.get_secret_value(), algorithms=[settings.ALGORITHM]
        )
        subject: str | None = payload.get("sub")
        token_type: str | None = payload.get("token_type")

        if subject is None or token_type != expected_type:
            return None

        return TokenData(subject=subject)
    except JWTError:
        return None
```

Expiration is checked automatically by the JWT library. If the token is expired, `jwt.decode` raises `JWTError`.

## Token Blacklisting

A database model tracks invalidated tokens (logout, refresh rotation):

```python
# app/auth/models.py
from datetime import datetime

from sqlalchemy import String, func
from sqlalchemy.orm import Mapped, mapped_column

from app.shared.database import Base


class TokenBlacklist(Base):
    __tablename__ = "token_blacklist"

    id: Mapped[int] = mapped_column(primary_key=True)
    token: Mapped[str] = mapped_column(String(500), unique=True, index=True)
    expires_at: Mapped[datetime]
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
```

Store the token's `exp` claim as `expires_at` so expired blacklist entries can be cleaned up periodically.

Check the blacklist during token verification in the auth service:

```python
async def is_token_blacklisted(self, token: str) -> bool:
    result = await self.repo.get_by_token(token)
    return result is not None

async def blacklist_token(self, token: str, settings: Settings) -> None:
    payload = jwt.decode(
        token, settings.SECRET_KEY.get_secret_value(), algorithms=[settings.ALGORITHM]
    )
    exp_timestamp = payload.get("exp")
    expires_at = datetime.fromtimestamp(exp_timestamp, tz=timezone.utc)
    await self.repo.create(TokenBlacklistCreate(token=token, expires_at=expires_at))
```

For high-traffic applications, consider Redis for blacklist storage instead of the database — token lookups happen on every authenticated request.

## Authentication Dependencies

Define in `app/auth/dependencies.py`. These combine `Depends()` for the OAuth2 security scheme (a FastAPI feature) with `FromDishka` for service injection. The `@inject` decorator is required — `DishkaRoute` only auto-injects at the endpoint level, not in sub-dependencies called via `Depends()`.

```python
from dishka.integrations.fastapi import FromDishka, inject
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

from app.auth.service import AuthService
from app.auth.utils import verify_token
from app.shared.config import Settings
from app.users.models import User

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="api/v1/auth/login")
optional_oauth2_scheme = OAuth2PasswordBearer(tokenUrl="api/v1/auth/login", auto_error=False)
```

### get_current_user

Returns the authenticated `User` ORM model. Raises 401 if the token is invalid, blacklisted, or the user doesn't exist.

```python
@inject
async def get_current_user(
    auth_service: FromDishka[AuthService],
    settings: FromDishka[Settings],
    token: str = Depends(oauth2_scheme),
) -> User:
    token_data = verify_token(token, "access", settings)
    if not token_data:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or expired token",
            headers={"WWW-Authenticate": "Bearer"},
        )

    if await auth_service.is_token_blacklisted(token):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token has been revoked",
        )

    user = await auth_service.get_user_by_subject(token_data.subject)
    if user is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User not found",
        )
    return user
```

**Why `@inject` + `FromDishka` instead of `Depends()`:**
- `FromDishka[AuthService]` resolves from the Dishka container — same as endpoint parameters
- `@inject` is required because this function is called via `Depends()`, not as an endpoint. `DishkaRoute` only processes `FromDishka` on the top-level endpoint function, not its sub-dependencies.
- `FromDishka` parameters go before `Depends()` parameters (Python requires non-default params before default params)
- Do NOT use `= None` defaults on `FromDishka` parameters — Dishka's `@inject` strips them from FastAPI's parameter resolution and fills them from the container. Using `= None` silences the type checker but introduces a type lie.

### get_optional_user

For endpoints that work both authenticated and unauthenticated:

```python
@inject
async def get_optional_user(
    auth_service: FromDishka[AuthService],
    settings: FromDishka[Settings],
    token: str | None = Depends(optional_oauth2_scheme),
) -> User | None:
    if not token:
        return None
    try:
        return await get_current_user(
            auth_service=auth_service, settings=settings, token=token
        )
    except HTTPException:
        return None
```

### get_current_superuser

Wraps `get_current_user` with a superuser check:

```python
async def get_current_superuser(
    current_user: User = Depends(get_current_user),
) -> User:
    if not current_user.is_superuser:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Insufficient permissions",
        )
    return current_user
```

## Auth Service

`AuthService` extends the `Service` marker — it's a non-CRUD service. Injected via Dishka.

```python
from app.shared.service import Service
from app.auth.utils import verify_password, create_access_token, create_refresh_token
from app.users.repository import UserRepository
from app.auth.repository import TokenBlacklistRepository


class AuthService(Service):
    def __init__(
        self, user_repo: UserRepository, blacklist_repo: TokenBlacklistRepository
    ):
        self.user_repo = user_repo
        self.blacklist_repo = blacklist_repo

    async def authenticate(self, username_or_email: str, password: str) -> User | None:
        if "@" in username_or_email:
            user = await self.user_repo.get_by_email(username_or_email)
        else:
            user = await self.user_repo.get_by_username(username_or_email)

        if not user or not verify_password(password, user.hashed_password):
            return None
        return user

    async def get_user_by_subject(self, subject: str) -> User | None:
        return await self.user_repo.get_by_username(subject)

    async def is_token_blacklisted(self, token: str) -> bool:
        return await self.blacklist_repo.exists_by_token(token)

    async def blacklist_token(self, token: str, settings: Settings) -> None:
        payload = jwt.decode(
            token, settings.SECRET_KEY.get_secret_value(), algorithms=[settings.ALGORITHM]
        )
        expires_at = datetime.fromtimestamp(payload["exp"], tz=timezone.utc)
        await self.blacklist_repo.create(
            TokenBlacklistCreate(token=token, expires_at=expires_at)
        )
```

Register in `app/providers.py`:

```python
class RequestProvider(Provider):
    scope = Scope.REQUEST

    blacklist_repository = provide(TokenBlacklistRepository)
    auth_service = provide(AuthService)
```

## Auth Endpoints

```python
# app/auth/api.py
from typing import Annotated

from dishka.integrations.fastapi import DishkaRoute, FromDishka
from fastapi import APIRouter, Cookie, Depends, HTTPException, Response, status
from fastapi.security import OAuth2PasswordRequestForm

from app.auth.dependencies import get_current_user, oauth2_scheme
from app.auth.schemas import TokenResponse
from app.auth.service import AuthService
from app.shared.config import Settings
from app.users.models import User

router = APIRouter(route_class=DishkaRoute)


@router.post("/login", response_model=TokenResponse)
async def login(
    response: Response,
    form_data: Annotated[OAuth2PasswordRequestForm, Depends()],
    auth_service: FromDishka[AuthService],
    settings: FromDishka[Settings],
):
    user = await auth_service.authenticate(form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
        )

    access_token = create_access_token(user.username, settings)
    refresh_token = create_refresh_token(user.username, settings)

    response.set_cookie(
        key="refresh_token",
        value=refresh_token,
        httponly=True,
        secure=settings.COOKIE_SECURE,
        samesite=settings.COOKIE_SAMESITE,
        max_age=settings.REFRESH_TOKEN_EXPIRE_DAYS * 24 * 60 * 60,
    )
    return TokenResponse(access_token=access_token, token_type="bearer")


@router.post("/refresh", response_model=TokenResponse)
async def refresh(
    response: Response,
    auth_service: FromDishka[AuthService],
    settings: FromDishka[Settings],
    refresh_token: str | None = Cookie(None),
):
    if not refresh_token:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Refresh token missing")

    token_data = verify_token(refresh_token, "refresh", settings)
    if not token_data:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid refresh token")

    if await auth_service.is_token_blacklisted(refresh_token):
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Token has been revoked")

    # Rotate: blacklist old refresh token, issue new pair
    await auth_service.blacklist_token(refresh_token, settings)

    new_access_token = create_access_token(token_data.subject, settings)
    new_refresh_token = create_refresh_token(token_data.subject, settings)

    response.set_cookie(
        key="refresh_token",
        value=new_refresh_token,
        httponly=True,
        secure=settings.COOKIE_SECURE,
        samesite=settings.COOKIE_SAMESITE,
        max_age=settings.REFRESH_TOKEN_EXPIRE_DAYS * 24 * 60 * 60,
    )
    return TokenResponse(access_token=new_access_token, token_type="bearer")


@router.post("/logout")
async def logout(
    response: Response,
    auth_service: FromDishka[AuthService],
    settings: FromDishka[Settings],
    current_user: User = Depends(get_current_user),
    token: str = Depends(oauth2_scheme),
    refresh_token: str | None = Cookie(None),
):
    await auth_service.blacklist_token(token, settings)
    if refresh_token:
        await auth_service.blacklist_token(refresh_token, settings)

    response.delete_cookie(
        key="refresh_token",
        httponly=True,
        secure=settings.COOKIE_SECURE,
        samesite=settings.COOKIE_SAMESITE,
    )
    return {"message": "Successfully logged out"}
```

## Auth Schemas

```python
# app/auth/schemas.py
from pydantic import BaseModel


class TokenResponse(BaseModel):
    access_token: str
    token_type: str
```

## User Registration and Profile

User registration, profile management, password changes, and account deletion live in the `users` domain — not in auth. See [user-management.md](user-management.md) for the full implementation including the `UserCreateInternal` pattern, profile endpoints (`/me`), and soft/hard delete flows.

## Extension Points

- **OAuth2/social login** — add providers (Google, GitHub) as additional auth strategies. The `AuthService` can delegate to provider-specific logic while keeping the same `get_current_user` dependency.
- **API key authentication** — for service-to-service communication, add a `get_api_key_user` dependency that validates an API key from the `X-API-Key` header.
- **Multi-factor authentication** — add a TOTP verification step between credential validation and token issuance in the `AuthService`.
