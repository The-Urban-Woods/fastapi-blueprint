# Dependency Injection with Dishka

## Why Dishka

Dishka provides framework-agnostic DI with proper scoped lifetimes. Unlike FastAPI's built-in `Depends()`, dishka providers work across FastAPI, Celery, CLI tools, and plain scripts — keeping services and repositories portable.

## Provider Structure

Define providers in `app/providers.py`. Split into two providers by scope:

```python
from collections.abc import AsyncIterable

from dishka import Provider, Scope, provide
from sqlalchemy.ext.asyncio import AsyncEngine, AsyncSession, async_sessionmaker, create_async_engine

from app.shared.config import Settings, get_settings
from app.users.repository import UserRepository
from app.users.service import UserService


class AppProvider(Provider):
    scope = Scope.APP

    @provide
    def get_settings(self) -> Settings:
        return get_settings()

    @provide
    def get_engine(self, settings: Settings) -> AsyncEngine:
        return create_async_engine(settings.DATABASE_URL, echo=True)

    @provide
    def get_session_factory(self, engine: AsyncEngine) -> async_sessionmaker[AsyncSession]:
        return async_sessionmaker(engine, expire_on_commit=False)


class RequestProvider(Provider):
    scope = Scope.REQUEST

    @provide
    async def get_session(
        self, session_factory: async_sessionmaker[AsyncSession]
    ) -> AsyncIterable[AsyncSession]:
        async with session_factory() as session:
            try:
                yield session
                await session.commit()
            except Exception:
                await session.rollback()
                raise

    user_repository = provide(UserRepository)
    user_service = provide(UserService)
```

## Scopes

- **`Scope.APP`** — Created once, lives for the entire application. Use for: `Settings`, `AsyncEngine`, `async_sessionmaker`, HTTP clients, connection pools.
- **`Scope.REQUEST`** — Created per request, disposed after. Use for: `AsyncSession`, repositories, services.

All components at the same scope share instances within that scope. A `UserRepository` and `UserService` at `Scope.REQUEST` receive the same `AsyncSession`.

## Wiring to FastAPI

```python
from dishka import make_async_container
from dishka.integrations.fastapi import setup_dishka

container = make_async_container(AppProvider(), RequestProvider())
setup_dishka(container, app)
```

Close the container in the app lifespan:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    engine = await app.state.dishka_container.get(AsyncEngine)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    await app.state.dishka_container.close()
```

## Adding New Dependencies

To register a new domain's repository and service:

```python
class RequestProvider(Provider):
    scope = Scope.REQUEST

    # ... existing providers ...

    # Add new domain
    order_repository = provide(OrderRepository)
    order_service = provide(OrderService)
```

Dishka auto-resolves constructor parameters. If `OrderService.__init__` takes `repo: OrderRepository`, dishka wires it automatically.

## Class-Based Registration

For classes with simple constructors, use inline `provide()`:

```python
user_repository = provide(UserRepository)
user_service = provide(UserService)
```

For factories needing custom logic, use `@provide` methods:

```python
@provide
def get_engine(self, settings: Settings) -> AsyncEngine:
    return create_async_engine(settings.DATABASE_URL)
```

## Using Outside FastAPI (Celery, CLI)

The same providers work with dishka's Celery or standalone integrations:

```python
# Celery
from dishka.integrations.celery import inject, FromDishka, setup_dishka

container = make_container(AppProvider(), RequestProvider())
setup_dishka(container, celery_app)

@celery_app.task
@inject
def process_user(user_id: int, service: FromDishka[UserService]):
    service.do_something(user_id)
```

```python
# Standalone script
async def main():
    container = make_async_container(AppProvider(), RequestProvider())
    async with container() as request_scope:
        service = await request_scope.get(UserService)
        await service.do_something()
    await container.close()
```
