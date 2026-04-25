# Testing

## Test Configuration

Configure pytest for async testing in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

Required dev dependencies:

```toml
[dependency-groups]
dev = [
    "httpx>=0.28",
    "pytest>=9.0",
    "pytest-asyncio>=1.3",
]
```

## Test Fixtures

Create a `TestAppProvider` that replaces `AppProvider` with an in-memory database. Reuse the production `RequestProvider` as-is — the repositories and services work identically.

```python
# tests/conftest.py
import pytest
from dishka import Provider, Scope, make_async_container, provide
from dishka.integrations.fastapi import setup_dishka
from fastapi import FastAPI
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import AsyncEngine, AsyncSession, async_sessionmaker, create_async_engine

from app.providers import RequestProvider
from app.shared.database import Base

TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"


class TestAppProvider(Provider):
    scope = Scope.APP

    @provide
    def get_engine(self) -> AsyncEngine:
        return create_async_engine(TEST_DATABASE_URL)

    @provide
    def get_session_factory(self, engine: AsyncEngine) -> async_sessionmaker[AsyncSession]:
        return async_sessionmaker(engine, expire_on_commit=False)


@pytest.fixture
async def client():
    from app.users.api import router as users_router

    container = make_async_container(TestAppProvider(), RequestProvider())

    engine = await container.get(AsyncEngine)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    app = FastAPI()
    app.include_router(users_router, prefix="/api/v1/users", tags=["users"])
    setup_dishka(container, app)

    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as c:
        yield c

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

    await container.close()
    await engine.dispose()
```

## Test File Organization

Choose based on domain complexity:

**Option A: File per domain** — for simple or small domains, a single test file is sufficient.

```
tests/
├── conftest.py
├── test_users.py
└── test_orders.py
```

**Option B: Module per domain** — for complex or large domains, split tests into a directory with separate files per concern.

```
tests/
├── conftest.py
├── users/
│   ├── __init__.py
│   ├── test_create.py
│   ├── test_read.py
│   ├── test_update.py
│   └── test_delete.py
└── orders/
    ├── __init__.py
    ├── test_placement.py
    └── test_fulfillment.py
```

Start with Option A. Upgrade to Option B when a test file grows beyond 300-500 lines or when the domain has distinct functional areas that benefit from separate files. Both options can coexist in the same project — use whichever fits each domain.

## Key Testing Patterns

- **Replace `AppProvider`, reuse `RequestProvider`** — only the infrastructure layer changes between production and tests. Services and repositories run the same code.
- **Fresh database per test** — each test gets a new in-memory SQLite instance with clean tables.
- **No `app.dependency_overrides`** — dishka replaces FastAPI's `Depends()` entirely. Override by swapping providers at the container level.
- **Import routers inside fixtures** — avoids circular import issues when the test module loads before the app.

## Writing Tests

```python
import pytest
from httpx import AsyncClient

USER_URL = "/api/v1/users"


async def create_user(client: AsyncClient, email: str = "test@example.com", name: str = "Test User"):
    return await client.post(USER_URL + "/", json={"email": email, "name": name})


async def test_create_user(client: AsyncClient):
    resp = await create_user(client)
    assert resp.status_code == 201
    data = resp.json()
    assert data["email"] == "test@example.com"
    assert data["name"] == "Test User"
    assert data["is_active"] is True


async def test_create_user_duplicate_email(client: AsyncClient):
    await create_user(client)
    resp = await create_user(client)
    assert resp.status_code == 400
    assert resp.json()["detail"] == "Email already registered"


async def test_get_user_not_found(client: AsyncClient):
    resp = await client.get(f"{USER_URL}/999")
    assert resp.status_code == 404
```

## Unit Testing Services Directly

Services are plain Python classes — test them without HTTP:

```python
@pytest.fixture
async def user_service():
    container = make_async_container(TestAppProvider(), RequestProvider())

    engine = await container.get(AsyncEngine)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    async with container() as request_scope:
        yield await request_scope.get(UserService)

    await container.close()
    await engine.dispose()


async def test_create_user_directly(user_service: UserService):
    user = await user_service.create_user(UserCreate(email="test@example.com", name="Test"))
    assert user.email == "test@example.com"
```
