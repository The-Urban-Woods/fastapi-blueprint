# Project Structure

## Domain-Driven Layout

Organize by business domain, not technical layer. Each domain module is self-contained with its own models, schemas, repository, service, and API.

```
app/
├── main.py                 # FastAPI app factory, lifespan, middleware, router includes
├── providers.py            # Dishka DI providers (AppProvider, RequestProvider)
├── shared/                 # Cross-cutting concerns shared across domains
│   ├── __init__.py
│   ├── config.py           # pydantic-settings Settings class
│   ├── database.py         # SQLAlchemy Base declarative class
│   └── repository.py       # Generic BaseRepository
├── users/                  # Domain module
│   ├── __init__.py         # Public API exports via __all__
│   ├── models.py           # SQLAlchemy ORM models
│   ├── schemas.py          # Pydantic request/response schemas
│   ├── repository.py       # UserRepository extends BaseRepository
│   ├── service.py          # UserService business logic
│   └── api.py              # FastAPI router with endpoints
├── orders/                 # Another domain module (same structure)
│   ├── __init__.py
│   ├── models.py
│   ├── schemas.py
│   ├── repository.py
│   ├── service.py
│   └── api.py
tests/
├── conftest.py             # Test fixtures, test DI providers
├── test_users.py           # Simple domain: single file
└── orders/                 # Complex domain: module with multiple files
    ├── __init__.py
    ├── test_placement.py
    └── test_fulfillment.py
main.py                     # Entry point (uvicorn runner)
pyproject.toml
```

## Key Principles

- **One concept per file** — `models.py`, `schemas.py`, `repository.py`, `service.py`, `api.py` each have a single responsibility
- **Split at 300-500 lines** — if a file grows beyond this, extract into sub-modules
- **Flat within domains** — avoid nesting directories inside a domain module
- **`shared/` for cross-cutting code** — database base classes, generic repositories, configuration, common exceptions

## Module Initialization (`__init__.py`)

Every domain module exports its public API via `__all__`. Do NOT export the router from `__init__.py` — this avoids circular imports when the DI container imports from domain modules.

```python
# app/users/__init__.py
from .models import User
from .schemas import UserCreate, UserRead, UserUpdate
from .service import UserService

__all__ = [
    "User",
    "UserCreate",
    "UserRead",
    "UserService",
    "UserUpdate",
]
```

Import routers directly from the `api` module in `main.py`:

```python
# app/main.py
from app.users.api import router as users_router
```

## Naming Conventions

- `snake_case` for all files and modules: `user_repository.py`
- Match class names to file names: `UserService` in `service.py`
- Use absolute imports: `from app.users.models import User`
- Avoid abbreviations: `repository.py` not `repo.py`

## Adding a New Domain Module

1. Create the domain directory with `models.py`, `schemas.py`, `repository.py`, `service.py`, `api.py`, `__init__.py`
2. Register the repository and service in `app/providers.py` under `RequestProvider`
3. Include the router in `app/main.py`
4. Add test fixtures and test file under `tests/`
