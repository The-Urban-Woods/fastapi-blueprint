# Alembic Migrations

Database schema migrations using Alembic with async SQLAlchemy. Covers setup, CLI commands, autogenerate, and deployment patterns.

## Initial Setup

Install Alembic:

```bash
uv add alembic
```

Initialize the migrations directory:

```bash
alembic init alembic
```

This creates:

```
alembic/
├── env.py          # Migration environment configuration
├── script.py.mako  # Template for new migration files
└── versions/       # Generated migration scripts
alembic.ini         # Alembic configuration
```

## Configuration

### alembic.ini

Set the file template to use timestamps for chronological sorting:

```ini
[alembic]
script_location = alembic
file_template = %%(epoch)d_%%(rev)s_%%(slug)s
```

Remove or leave blank the `sqlalchemy.url` line — the URL comes from `Settings` via `env.py`.

Configure Ruff as a post-write hook to format generated migrations:

```ini
[post_write_hooks]
hooks = ruff_format

ruff_format.type = console_scripts
ruff_format.entrypoint = ruff
ruff_format.options = format REVISION_SCRIPT_FILENAME
```

### env.py (Async)

Configure `env.py` to use the async engine and read the database URL from `Settings`:

```python
import asyncio
from logging.config import fileConfig

from alembic import context
from sqlalchemy.ext.asyncio import create_async_engine

from app.shared.config import get_settings
from app.shared.database import Base

# Alembic Config object
config = context.config

# Set up loggers from alembic.ini
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# MetaData for autogenerate support
target_metadata = Base.metadata

# Get database URL from application settings
settings = get_settings()


def run_migrations_offline() -> None:
    """Run migrations without an engine connection."""
    context.configure(
        url=settings.DATABASE_URL,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_migrations_online() -> None:
    """Run migrations with an async engine connection."""
    engine = create_async_engine(settings.DATABASE_URL)

    async with engine.connect() as connection:
        await connection.run_sync(do_run_migrations)

    await engine.dispose()


if context.is_offline_mode():
    run_migrations_offline()
else:
    asyncio.run(run_migrations_online())
```

The `DATABASE_URL` can also be overridden via the environment variable `DATABASE_URL` — `Settings` picks it up automatically via pydantic-settings.

### Import All Models

Alembic autogenerate compares `Base.metadata` against the database. Models must be imported before `env.py` references `Base.metadata`, otherwise their tables won't be detected.

Add model imports in `env.py` after the `Base` import:

```python
from app.shared.database import Base
from app.users.models import User  # noqa: F401
# Import all domain models here so Base.metadata is complete
```

Alternatively, create an `app/shared/models.py` that re-exports all models:

```python
# app/shared/models.py
from app.users.models import User  # noqa: F401
from app.orders.models import Order  # noqa: F401
```

Then import once in `env.py`:

```python
import app.shared.models  # noqa: F401
```

## CLI Commands

### Creating a Migration

Manual (empty upgrade/downgrade to fill in):

```bash
alembic revision -m "add orders table"
```

Autogenerate (compares models to database):

```bash
alembic revision --autogenerate -m "add orders table"
```

### Running Migrations

```bash
alembic upgrade head          # Apply all pending migrations
alembic upgrade +1            # Apply next single migration
alembic upgrade ae1027        # Apply up to specific revision (prefix match)
```

### Downgrading

```bash
alembic downgrade -1          # Revert last migration
alembic downgrade base        # Revert all migrations
alembic downgrade ae1027      # Revert to specific revision
```

### Viewing State

```bash
alembic current               # Show current database revision
alembic history               # Show migration chain
alembic history --verbose     # Show full revision metadata
alembic history -r-3:current  # Show last 3 migrations
```

## Autogenerate

### What It Detects

- Table additions and removals
- Column additions and removals
- Nullable status changes
- Index and unique constraint changes
- Foreign key constraint changes
- Column type changes (enabled by default since Alembic 1.12)

### What It Cannot Detect

- **Table or column renames** — appears as a drop + add. Edit the migration manually to use `op.rename_table()` or `op.alter_column(new_column_name=...)`.
- **Anonymous constraints** — constraints without explicit names cannot be tracked. This is why the naming convention on `Base.metadata` is required (see [sqlalchemy-usage.md](sqlalchemy-usage.md)).
- **`Enum` type changes** on databases that don't support them natively
- **`CHECK` constraint changes**

Always review autogenerated migrations before applying them. They are candidates, not final scripts.

## Data Migrations

Never import ORM models in migration scripts. Models change over time, breaking older migrations that reference them.

### Raw SQL (Preferred for Simple Cases)

```python
def upgrade():
    op.execute(
        "UPDATE users SET email = LOWER(email) WHERE email != LOWER(email);"
    )

def downgrade():
    pass  # Data transformation is not reversible
```

### Inline Table Definition (For Complex Logic)

```python
from sqlalchemy import table, column, String, select

def upgrade():
    users = table("users", column("id"), column("username", String), column("email", String))

    conn = op.get_bind()
    result = conn.execute(select(users).where(users.c.email == None))

    for row in result:
        conn.execute(
            users.update()
            .where(users.c.id == row.id)
            .values(email=f"{row.username}@example.com")
        )
```

The inline `table()` definition is a snapshot — it won't break when the actual model changes later.

## Docker Orchestration

Run migrations as a separate container that completes before the app starts:

```yaml
services:
  db:
    image: postgres:17
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      start_period: 10s

  migrations:
    build: .
    command: ["alembic", "upgrade", "head"]
    depends_on:
      db:
        condition: service_healthy
    environment:
      - DATABASE_URL=postgresql+asyncpg://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db/${POSTGRES_DB}

  app:
    build: .
    command: ["uvicorn", "app.main:app", "--host", "0.0.0.0"]
    depends_on:
      migrations:
        condition: service_completed_successfully
    environment:
      - DATABASE_URL=postgresql+asyncpg://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db/${POSTGRES_DB}
```

The `migrations` service runs once and exits. The `app` service only starts after migrations complete successfully. Both read `DATABASE_URL` from the same environment variables.

## CI Tips

### Detect Missing Migrations with `alembic check`

Run `alembic check` in CI to catch forgotten migrations. It compares your models against the latest revision without touching the database:

```bash
alembic check
```

If models have changed but no migration exists, it exits with a non-zero code:

```
FAILED: New upgrade operations detected:
  add_column('users', Column('phone', String(50)))
```

Add it as a step in your CI pipeline alongside linting and tests.

### Stairway Test

If you experience frequent migration failures in CI, consider implementing a "stairway test" — a test that applies each migration step-by-step (`upgrade +1`, `downgrade -1`) through the entire chain. This catches broken downgrade scripts and ordering issues. See [alembic-quickstart](https://github.com/alvassin/alembic-quickstart) for a reference implementation.
