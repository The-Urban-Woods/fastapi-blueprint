# Code Style & Documentation

Consistent code style and clear documentation make codebases maintainable. This reference covers modern Python tooling, naming conventions, and documentation standards.

## Linting & Formatting with Ruff

Use `ruff` as an all-in-one linter and formatter. It replaces flake8, isort, and black with a single fast tool.

```toml
# pyproject.toml
[tool.ruff]
line-length = 120
target-version = "py312"

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort
    "B",    # flake8-bugbear
    "C4",   # flake8-comprehensions
    "UP",   # pyupgrade
    "SIM",  # flake8-simplify
]
ignore = ["E501"]  # Line length handled by formatter

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

```bash
ruff check --fix .  # Lint and auto-fix
ruff format .       # Format code
```

## Type Checking with ty

Use `ty` (by Astral, makers of ruff and uv) for type checking. It is 10-100x faster than mypy, written in Rust, and uses a rule-based system with configurable severities.

### Installation

```bash
uv add --dev ty    # As dev dependency
uvx ty check       # Or run directly without installing
```

### Configuration

```toml
# pyproject.toml
[tool.ty.environment]
python-version = "3.12"

[tool.ty.rules]
# Set rule severities: "error", "warn", or "ignore"
# All rules default to sensible severities

[tool.ty.terminal]
output-format = "full"

# Relax rules for test files
[[tool.ty.overrides]]
include = ["tests/**"]
[tool.ty.overrides.rules]
possibly-unresolved-reference = "ignore"
```

### Running

```bash
ty check              # Type check the project
ty check src/         # Check specific directory
ty check --watch      # Watch mode with incremental analysis
```

### Suppression Comments

```python
# ty-native format:
result = 10 + "test"  # ty: ignore[unsupported-operator]

# Multiple rules:
func("one", 5)  # ty: ignore[missing-argument, invalid-argument-type]

# PEP 484 compatible format (also works with mypy):
result = 10 + "test"  # type: ignore[ty:unsupported-operator]
```

### Key Rules

Common rules you may want to configure:

| Rule | Default | Description |
|------|---------|-------------|
| `invalid-argument-type` | error | Wrong type passed to function |
| `invalid-return-type` | error | Return type doesn't match annotation |
| `invalid-assignment` | error | Assigning incompatible type |
| `possibly-unresolved-reference` | warn | Name might not be defined |
| `unused-ignore-comment` | warn | Suppression comment not needed |
| `deprecated` | warn | Using deprecated API |
| `unresolved-import` | error | Cannot resolve import |

### Handling Unresolved Imports

For compiled extensions or modules ty can't resolve:

```toml
[tool.ty.analysis]
# Treat specific imports as Any instead of errors
replace-imports-with-any = ["compiled_module"]

# Allow specific unresolved imports
allowed-unresolved-imports = ["some_module.*"]
```

## Naming Conventions

Follow PEP 8 with emphasis on clarity over brevity.

**Files and Modules:**

```python
# Good: Descriptive snake_case
user_repository.py
order_processing.py
http_client.py

# Avoid: Abbreviations
usr_repo.py
ord_proc.py
```

**Classes and Functions:**

```python
# Classes: PascalCase
class UserRepository:
    pass

class HTTPClientFactory:  # Acronyms stay uppercase
    pass

# Functions and variables: snake_case
def get_user_by_email(email: str) -> User | None:
    retry_count = 3
    max_connections = 100

# Module-level constants: SCREAMING_SNAKE_CASE
MAX_RETRY_ATTEMPTS = 3
DEFAULT_TIMEOUT_SECONDS = 30
```

## Import Organization

Group imports in a consistent order: standard library, third-party, local. Use absolute imports exclusively.

```python
# Standard library
import os
from collections.abc import Callable
from typing import Any

# Third-party packages
import httpx
from pydantic import BaseModel
from sqlalchemy import Column

# Local imports
from app.users.models import User
from app.users.service import UserService
```

## Google-Style Docstrings

Write docstrings for all public classes, methods, and functions.

**Simple function:**

```python
def get_user(user_id: str) -> User:
    """Retrieve a user by their unique identifier."""
    ...
```

**Complex function:**

```python
def process_batch(
    items: list[Item],
    max_workers: int = 4,
    on_progress: Callable[[int, int], None] | None = None,
) -> BatchResult:
    """Process items concurrently using a worker pool.

    Processes each item in the batch using the configured number of
    workers. Progress can be monitored via the optional callback.

    Args:
        items: The items to process. Must not be empty.
        max_workers: Maximum concurrent workers. Defaults to 4.
        on_progress: Optional callback receiving (completed, total) counts.

    Returns:
        BatchResult containing succeeded items and any failures with
        their associated exceptions.

    Raises:
        ValueError: If items is empty.
        ProcessingError: If the batch cannot be processed.
    """
    ...
```

**Class docstring:**

```python
class UserService:
    """Service for managing user operations.

    Provides methods for creating, retrieving, updating, and
    deleting users with proper validation and error handling.

    Attributes:
        repo: The data access layer for user persistence.
    """
    ...
```

## Line Length and Formatting

Set line length to 120 characters for modern displays.

```python
# Readable line breaks for function signatures
def create_user(
    email: str,
    name: str,
    role: UserRole = UserRole.MEMBER,
    notify: bool = True,
) -> User:
    ...

# Chain method calls clearly
result = (
    db.query(User)
    .filter(User.active == True)
    .order_by(User.created_at.desc())
    .limit(10)
    .all()
)

# Format long strings
error_message = (
    f"Failed to process user {user_id}: "
    f"received status {response.status_code} "
    f"with body {response.text[:100]}"
)
```

## Best Practices Summary

1. **Use ruff** for linting and formatting — single tool, fast
2. **Use ty for type checking** — fast, rule-based, configurable severities
3. **120 character lines** — modern standard for readability
4. **Descriptive names** — clarity over brevity
5. **Absolute imports** — more maintainable than relative
6. **Google-style docstrings** — consistent, readable documentation
7. **Document public APIs** — every public class and function needs a docstring
8. **Type annotate all public APIs** — use modern syntax (`str | None`, `list[int]`)
9. **Automate in CI** — run `ruff check`, `ruff format --check`, and `ty check` on every commit
10. **Target Python 3.12+** — use modern language features
