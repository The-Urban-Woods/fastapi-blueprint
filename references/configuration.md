# Configuration

## Settings Class

Use `pydantic-settings` for environment-based configuration in `app/shared/config.py`. Field names match environment variable names directly (UPPER_CASE):

```python
from functools import lru_cache

from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    DATABASE_URL: str = "sqlite+aiosqlite:///./app.db"
    API_V1_STR: str = "/api/v1"

    model_config = {"env_file": ".env"}


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

## Integration with DI

Settings are provided at `Scope.APP` via the dishka `AppProvider`:

```python
class AppProvider(Provider):
    scope = Scope.APP

    @provide
    def get_settings(self) -> Settings:
        return get_settings()

    @provide
    def get_engine(self, settings: Settings) -> AsyncEngine:
        return create_async_engine(settings.DATABASE_URL, echo=True)
```

The `@lru_cache` on `get_settings()` ensures a single `Settings` instance. The dishka provider wraps it so other DI-managed components can declare `Settings` as a dependency. Validation happens on first call — missing required settings will raise `ValidationError` immediately.

## Environment Variables

Settings fields map to environment variables by name. Override any setting via env var or `.env` file:

```bash
DATABASE_URL=postgresql+asyncpg://user:pass@localhost/mydb
API_V1_STR=/api/v2
```

## Required vs Optional Settings

Provide sensible defaults for local development. Omit defaults for settings that must always be explicitly set (secrets, production URLs):

```python
class Settings(BaseSettings):
    # Has local default — overridden in production
    DATABASE_URL: str = "sqlite+aiosqlite:///./app.db"
    DEBUG: bool = False

    # Always required — no default for secrets
    SECRET_KEY: str
    API_KEY: str

    model_config = {"env_file": ".env"}
```

Missing required fields crash the app at startup with a clear `ValidationError`, not a cryptic `None` failure mid-request.

## Local Development with .env

Create a `.env` file for local development. Never commit this file:

```bash
# .env (add to .gitignore)
SECRET_KEY=local-dev-secret
API_KEY=dev-api-key
DEBUG=true
```

## Type Coercion

Pydantic handles common conversions automatically:

```python
from pydantic import field_validator

class Settings(BaseSettings):
    # "true", "1", "yes" → True
    DEBUG: bool = False

    # String → int
    MAX_CONNECTIONS: int = 100

    # Comma-separated string → list
    ALLOWED_HOSTS: list[str] = []

    @field_validator("ALLOWED_HOSTS", mode="before")
    @classmethod
    def parse_allowed_hosts(cls, v: str | list[str]) -> list[str]:
        if isinstance(v, str):
            return [host.strip() for host in v.split(",") if host.strip()]
        return v

    model_config = {"env_file": ".env"}
```

```bash
ALLOWED_HOSTS=example.com,api.example.com,localhost
MAX_CONNECTIONS=50
```

## Environment-Specific Configuration

Use an environment enum to switch behavior between local, staging, and production:

```python
from enum import Enum
from pydantic import computed_field

class Environment(str, Enum):
    LOCAL = "local"
    STAGING = "staging"
    PRODUCTION = "production"

class Settings(BaseSettings):
    ENVIRONMENT: Environment = Environment.LOCAL

    @computed_field
    @property
    def is_production(self) -> bool:
        return self.ENVIRONMENT == Environment.PRODUCTION

    model_config = {"env_file": ".env"}
```

## Configuration Validation

Add custom validation for settings that depend on each other:

```python
from pydantic import model_validator

class Settings(BaseSettings):
    DB_HOST: str = "localhost"
    DB_PORT: int = 5432
    READ_REPLICA_HOST: str | None = None
    READ_REPLICA_PORT: int = 5432

    @model_validator(mode="after")
    def validate_replica_settings(self):
        if self.READ_REPLICA_HOST == self.DB_HOST and self.READ_REPLICA_PORT == self.DB_PORT:
            raise ValueError("Read replica cannot be the same as primary database")
        return self
```

## Secrets from Files

For container environments (Docker, Kubernetes), read secrets from mounted files:

```python
class Settings(BaseSettings):
    DB_PASSWORD: str
    API_SECRET_KEY: str

    model_config = {
        "env_file": ".env",
        "secrets_dir": "/run/secrets",
    }
```

Pydantic looks for `/run/secrets/db_password` if the env var isn't set.

## Nested Configuration (Larger Projects)

For projects with many settings, group related config into nested models using `env_nested_delimiter`:

```python
from pydantic import BaseModel

class DatabaseSettings(BaseModel):
    HOST: str = "localhost"
    PORT: int = 5432
    NAME: str = "myapp"
    USER: str = "admin"
    PASSWORD: str

class RedisSettings(BaseModel):
    URL: str = "redis://localhost:6379"
    MAX_CONNECTIONS: int = 10

class Settings(BaseSettings):
    DATABASE: DatabaseSettings
    REDIS: RedisSettings
    DEBUG: bool = False

    model_config = {
        "env_nested_delimiter": "__",
        "env_file": ".env",
    }
```

Environment variables use double underscore for nesting:

```bash
DATABASE__HOST=db.example.com
DATABASE__PORT=5432
DATABASE__NAME=myapp
DATABASE__USER=admin
DATABASE__PASSWORD=secret
REDIS__URL=redis://redis.example.com:6379
```

Accessed as `settings.DATABASE.HOST`. Start with a flat settings class — only introduce nesting when the number of settings makes a flat structure hard to navigate.

## Best Practices

1. **Never hardcode config** — all environment-specific values from env vars
2. **Provide dev defaults** — make local development easy, require secrets explicitly
3. **Never commit secrets** — use `.env` files (gitignored) or secret managers
4. **Namespace variables** — `DB_HOST`, `REDIS_URL` for clarity and easy debugging (`env | grep DB_`)
5. **Validate early** — missing config crashes at startup, not mid-request
6. **Use `secrets_dir`** — support mounted secrets in container environments
