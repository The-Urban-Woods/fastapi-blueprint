---
name: fastapi-blueprint
description: Generates FastAPI code with domain-driven structure, repository+service pattern, and Dishka dependency injection. Trigger when creating FastAPI applications, adding domain modules, implementing CRUD endpoints, setting up repositories or services, configuring dependency injection, writing tests, or modifying any Python backend API code in this project.
license: MIT
metadata:
  author: Ariën Tolner, The Urban Woods
  version: "1.0"
---

# FastAPI Developer

Generate production-ready FastAPI code following domain-driven structure with the repository+service pattern and Dishka for dependency injection.

## Before Writing Code

1. Check the existing project structure — follow established patterns, do not introduce new conventions
2. Identify which domain module the work belongs to (e.g., `users/`, `orders/`)
3. If creating a new domain, scaffold all files: `models.py`, `schemas.py`, `repository.py`, `service.py`, `api.py`, `__init__.py`

## Architecture Rules

- **Domain-driven structure** — organize by business domain, not technical layer. Each domain has its own models, schemas, repository, service, and API. Read [project-structure.md](references/project-structure.md)
- **Abstract base classes** — repositories implement the `Repository` ABC; CRUD services implement the `CrudService` ABC; non-CRUD services extend the `Service` marker. ABCs live in `app/shared/` and are framework-agnostic.
- **SQLAlchemy implementation** — domain repositories extend `SqlAlchemyRepository` (which implements the `Repository` ABC), not the ABC directly.
- **Dishka for all DI** — do NOT use FastAPI's `Depends()` for service or repository injection. Use `FromDishka[T]` and `DishkaRoute`. `Depends()` is only acceptable for non-DI FastAPI features (e.g., path parameters, security schemes). Read [dependency-injection.md](references/dependency-injection.md)
- **No singletons** — do NOT create module-level instances of repositories or services. All instantiation flows through dishka providers.
- **No `db` in method signatures** — `AsyncSession` is injected into repositories via constructor, not passed as a method parameter. Services never see `AsyncSession`.

## Layer Responsibilities

| Layer                            | Knows About                                         | Never Imports                    |
| -------------------------------- | --------------------------------------------------- | -------------------------------- |
| **API** (`api.py`)               | FastAPI, Pydantic schemas, Service classes, dishka  | SQLAlchemy, Repository classes   |
| **Service** (`service.py`)       | Repository classes, domain models, Pydantic schemas | FastAPI, dishka, SQLAlchemy      |
| **Repository** (`repository.py`) | SQLAlchemy, domain models, Pydantic schemas         | FastAPI, dishka, Service classes |
| **Models** (`models.py`)         | SQLAlchemy, `Base`                                  | Everything else                  |
| **Schemas** (`schemas.py`)       | Pydantic                                            | Everything else                  |

## Implementation Guides

### Project Setup & Structure

- **Project layout, naming, module init**: Read [project-structure.md](references/project-structure.md)
- **App factory, lifespan, middleware, router registration**: Read [app-entrypoint.md](references/app-entrypoint.md)
- **Settings, environment variables, pydantic-settings**: Read [configuration.md](references/configuration.md)
- **uv commands, dependencies, lockfile, Docker, CI/CD**: Read [uv-package-manager.md](references/uv-package-manager.md)
- **Poe the Poet task runner, composing tasks, CLI arguments, env vars**: Read [task-runner.md](references/task-runner.md)

### Data Layer

- **SQLAlchemy Base, Pydantic schemas, DI-managed engine/session, transaction management**: Read [database-setup.md](references/database-setup.md)
- **Model definitions, relationships, query patterns, mixins, upserts**: Read [sqlalchemy-usage.md](references/sqlalchemy-usage.md)
- **Repository ABC, SqlAlchemyRepository, domain repositories**: Read [repository-pattern.md](references/repository-pattern.md)
- **Database migrations with Alembic — setup, CLI, autogenerate, deployment**: Read [alembic-migrations.md](references/alembic-migrations.md)

### Business & API Layer

- **Service design principles, CrudService/Service ABCs, anti-patterns, cross-domain services**: Read [service-layer.md](references/service-layer.md)
- **DishkaRoute, FromDishka, endpoint patterns, parameter ordering**: Read [api-endpoints.md](references/api-endpoints.md)

### Code Quality

- **Ruff linting/formatting, ty type checking, naming conventions, docstrings**: Read [code-style.md](references/code-style.md)

### Background Jobs

- **Celery setup, using services in tasks, retries, idempotency, workflows**: Read [background-jobs.md](references/background-jobs.md)

### DI & Testing

- **Dishka providers, scopes, wiring, usage outside FastAPI (Celery, CLI)**: Read [dependency-injection.md](references/dependency-injection.md)
- **Test fixtures, TestAppProvider, integration and unit testing**: Read [testing.md](references/testing.md)

## Quick Reference: Adding a New Domain Module

1. Create `app/{domain}/` with `models.py`, `schemas.py`, `repository.py`, `service.py`, `api.py`, `__init__.py`
2. Model inherits `Base`, repository extends `SqlAlchemyRepository`, service extends `CrudService` (or `Service` for non-CRUD)
3. Add `{domain}_repository = provide({Domain}Repository)` and `{domain}_service = provide({Domain}Service)` to `RequestProvider` in `app/providers.py`
4. Add `app.include_router(router, prefix=..., tags=[...])` in `app/main.py`
5. Add test file in `tests/test_{domain}.py`

## Parameter Naming Convention

When overriding ABC methods in domain implementations, use the **same generic parameter names** as the ABC defines: `id`, `obj_in`, `entity`. Do NOT use domain-specific names like `user_id` or `user_in` — the class name already provides that context.

```python
# Correct — matches ABC signature
async def get(self, id: int) -> User | None: ...
async def create(self, obj_in: UserCreate) -> User: ...
async def update(self, id: int, obj_in: UserUpdate) -> User | None: ...

# Wrong — domain-specific names diverge from ABC
async def get(self, user_id: int) -> User | None: ...
async def create(self, user_in: UserCreate) -> User: ...
```

## Common Mistakes to Avoid

- Using `Depends(get_db)` in endpoints — the session is managed by dishka, injected into repositories
- Importing `AsyncSession` in service or endpoint modules — only repositories touch the session
- Creating module-level `user_service = UserService()` singletons — use dishka providers
- Calling `commit()` in repositories — use `flush()`, let the dishka session provider handle commit/rollback
- Re-exporting `router` from `__init__.py` — this causes circular imports with the DI container
- Using domain-specific parameter names (`user_id`, `user_in`) in ABC method overrides — use the generic names (`id`, `obj_in`) to stay consistent with the contract
