# UV Package Manager

uv is an extremely fast Python package installer, resolver, and project manager written in Rust. It replaces pip, pip-tools, pyenv, and virtualenv with a single tool.

## Project Setup

### Creating a New Project

```bash
uv init my-project
cd my-project
```

This creates `pyproject.toml`, `.python-version`, `README.md`, and `.gitignore`.

To initialize in the current directory:

```bash
uv init .
```

### Python Version Management

```bash
# Install a Python version
uv python install 3.12

# Pin the project to a version (creates .python-version)
uv python pin 3.12

# List installed versions
uv python list
```

## Dependency Management

### Adding Dependencies

```bash
# Add a package (updates pyproject.toml and uv.lock)
uv add fastapi

# Add with version constraint
uv add "sqlalchemy>=2.0,<3.0"

# Add multiple packages
uv add dishka aiosqlite pydantic-settings

# Add dev dependency
uv add --dev pytest pytest-asyncio httpx

# Add from git
uv add git+https://github.com/user/repo.git

# Add from git at a specific tag
uv add git+https://github.com/user/repo.git@v1.0.0

# Add editable local package
uv add -e ./local-package
```

### Removing and Upgrading

```bash
# Remove a package
uv remove requests

# Upgrade a specific package
uv add --upgrade fastapi

# Upgrade all packages
uv sync --upgrade

# Show dependency tree
uv tree
```

### Lockfile

```bash
# Generate/update uv.lock
uv lock

# Upgrade a single package in the lockfile
uv lock --upgrade-package fastapi

# Install from lockfile (reproducible)
uv sync
```

Always commit `uv.lock` to version control for reproducible builds.

## Running Commands

`uv run` executes commands inside the project's virtual environment without manual activation:

```bash
# Run the app
uv run uvicorn app.main:app --reload

# Run tests
uv run pytest tests/ -v

# Run linting and type checking
uv run ruff check --fix .
uv run ty check

# Run any Python script
uv run python scripts/seed_db.py

# Run with a specific Python version
uv run --python 3.11 python script.py
```

No need to `source .venv/bin/activate` — `uv run` handles it.

### Running One-Off Tools

Use `uvx` to run tools without installing them into the project:

```bash
# Run a tool without installing
uvx ruff check .
uvx ty check

# Run a specific version
uvx ruff@0.8.0 check .
```

## pyproject.toml Configuration

```toml
[project]
name = "my-project"
version = "0.1.0"
description = "My project"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "aiosqlite>=0.22",
    "dishka>=1.10",
    "fastapi>=0.136",
    "pydantic-settings>=2.14",
    "sqlalchemy>=2.0",
    "uvicorn[standard]>=0.46",
]

[dependency-groups]
dev = [
    "httpx>=0.28",
    "pytest>=9.0",
    "pytest-asyncio>=1.3",
    "ruff>=0.11",
    "ty>=0.0",
]

[tool.pytest.ini_options]
asyncio_mode = "auto"
```

## Migrating from Other Tools

```bash
# From requirements.txt
uv add -r requirements.txt

# From poetry (already has pyproject.toml)
uv sync

# Export to requirements.txt (for environments that need it)
uv pip freeze > requirements.txt
```

## Docker Integration

Optimize Docker builds with uv for fast, cached dependency installation:

```dockerfile
FROM python:3.12-slim

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /app

# Install dependencies first (cached layer)
COPY pyproject.toml uv.lock ./
RUN uv sync --no-dev --frozen

# Copy application code
COPY app/ app/

CMD ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Key flags for Docker:
- `--frozen` — fail if `uv.lock` is out of date instead of updating it
- `--no-dev` — skip dev dependencies in production
- `--no-editable` — install packages non-editable (avoids `.pth` files)

## CI/CD

### GitHub Actions

```yaml
- uses: astral-sh/setup-uv@v5

- name: Install dependencies
  run: uv sync

- name: Run tests
  run: uv run pytest

- name: Lint and type check
  run: |
    uv run ruff check .
    uv run ty check
```

## Common Commands Reference

| Command | Purpose |
|---------|---------|
| `uv init` | Create new project |
| `uv add <pkg>` | Add dependency |
| `uv add --dev <pkg>` | Add dev dependency |
| `uv remove <pkg>` | Remove dependency |
| `uv sync` | Install all dependencies from lockfile |
| `uv lock` | Update lockfile |
| `uv run <cmd>` | Run command in project environment |
| `uvx <tool>` | Run tool without installing |
| `uv python install <ver>` | Install Python version |
| `uv python pin <ver>` | Pin project Python version |
| `uv tree` | Show dependency tree |
| `uv venv` | Create virtual environment |
