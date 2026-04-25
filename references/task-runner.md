# Task Runner with Poe the Poet

Poe the Poet is a task runner that integrates with uv and defines tasks in `pyproject.toml`. It replaces Makefiles and shell scripts with a cross-platform, documented task system.

## Setup

### Installation

```bash
# As a project dev dependency (recommended)
uv add --dev poethepoet

# Run tasks via uv
uv run poe <task>
```

For convenience, create a shell alias:

```bash
# Add to ~/.zshrc or ~/.bashrc
alias poe="uv run poe"
```

### Global Configuration

```toml
# pyproject.toml
[tool.poe]
executor = "uv"                    # Required: run all tasks through uv
envfile = ".env"                   # Recommended: load .env for all tasks
```

Optional: suppress Poe's own info messages (task output is unaffected):

```toml
[tool.poe]
verbosity = -1
```

## Defining Tasks

### Command Tasks

The simplest form — a single command run as a subprocess:

```toml
[tool.poe.tasks]
serve = "uvicorn app.main:app --reload"
lint = "ruff check --fix ."
format = "ruff format ."
typecheck = "ty check"
test = "pytest tests/ -v"
```

### Shell Tasks

For pipes, redirection, and full shell syntax:

```toml
[tool.poe.tasks.coverage-report]
help = "Run tests with coverage and open the HTML report"
shell = "pytest --cov=app --cov-report=html tests/ && open htmlcov/index.html"
```

### Script Tasks

Call Python functions directly:

```toml
[tool.poe.tasks.seed-db]
help = "Seed the database with sample data"
script = "scripts.seed:run"
```

Async functions are automatically executed using `asyncio.run`.

## Documenting Tasks

Always add `help` text to tasks. This makes tasks discoverable via `poe --help` and is especially important for AI-generated tasks.

```toml
[tool.poe.tasks.serve]
help = "Start the development server with hot reload"
cmd = "uvicorn app.main:app --reload"

[tool.poe.tasks.test]
help = "Run the test suite with verbose output"
cmd = "pytest tests/ -v"

[tool.poe.tasks.lint]
help = "Run linter and auto-fix issues"
cmd = "ruff check --fix ."
```

View help for a specific task:

```bash
poe --help serve
```

### Hiding Internal Tasks

Prefix task names with `_` to hide them from documentation. They can only be referenced by other tasks:

```toml
[tool.poe.tasks]
_build = "poetry build"
_test = "pytest"
release = ["_test", "_build"]
```

### Grouping Tasks

Organize related tasks under headings in help output:

```toml
[tool.poe.groups.dev]
heading = "Development"

[tool.poe.groups.dev.tasks.serve]
help = "Start the dev server"
cmd = "uvicorn app.main:app --reload"

[tool.poe.groups.dev.tasks.seed]
help = "Seed the database"
script = "scripts.seed:run"

[tool.poe.groups.quality]
heading = "Code Quality"

[tool.poe.groups.quality.tasks.lint]
help = "Run linter"
cmd = "ruff check --fix ."

[tool.poe.groups.quality.tasks.typecheck]
help = "Run type checker"
cmd = "ty check"
```

## CLI Arguments

### Option Arguments

```toml
[tool.poe.tasks.serve]
help = "Start the development server"
cmd = "uvicorn app.main:app --host ${host} --port ${port}"

[[tool.poe.tasks.serve.args]]
name = "host"
options = ["-h", "--host"]
help = "The host to bind to"
default = "127.0.0.1"

[[tool.poe.tasks.serve.args]]
name = "port"
options = ["-p", "--port"]
help = "The port to listen on"
default = "8000"
```

```bash
poe serve --host 0.0.0.0 --port 9000
```

### Positional Arguments

```toml
[tool.poe.tasks.migrate]
help = "Run database migration"
cmd = "alembic ${command}"

[[tool.poe.tasks.migrate.args]]
name = "command"
help = "Migration command (upgrade, downgrade, revision)"
positional = true
choices = ["upgrade", "downgrade", "revision"]
default = "upgrade"
```

### Boolean Flags

```toml
[tool.poe.tasks.test]
help = "Run tests"
cmd = "pytest tests/ ${verbose:+--verbose} ${cov:+--cov=app}"

[[tool.poe.tasks.test.args]]
name = "verbose"
options = ["-v", "--verbose"]
help = "Enable verbose output"
type = "boolean"

[[tool.poe.tasks.test.args]]
name = "cov"
options = ["--cov"]
help = "Enable coverage reporting"
type = "boolean"
```

```bash
poe test --verbose --cov
```

### Multiple Values

```toml
[tool.poe.tasks.lint-files]
help = "Lint specific files"
cmd = "ruff check ${files}"

[[tool.poe.tasks.lint-files.args]]
name = "files"
positional = true
multiple = true
help = "Files to lint"
```

```bash
poe lint-files app/main.py app/providers.py
```

### Argument Options Reference

| Option | Type | Description |
|--------|------|-------------|
| `name` | `str` | Argument name (required in array form) |
| `options` | `list[str]` | CLI flags (e.g., `["-v", "--verbose"]`) |
| `help` | `str` | Description for help output |
| `default` | `str/int/float/bool` | Default value, supports `${ENV_VAR}` templating |
| `positional` | `bool` | Treat as positional argument |
| `required` | `bool` | Error if not provided |
| `type` | `str` | `"string"`, `"integer"`, `"float"`, or `"boolean"` |
| `choices` | `list` | Constrain to fixed values |
| `multiple` | `bool/int` | Accept multiple values |

## Composing Tasks

### Sequences

Run tasks in order. Stops on first failure by default:

```toml
[tool.poe.tasks]
_lint = "ruff check --fix ."
_format = "ruff format ."
_typecheck = "ty check"
_test = "pytest tests/ -v"

[tool.poe.tasks.check]
help = "Run all quality checks and tests"
sequence = ["_lint", "_format", "_typecheck", "_test"]
```

### Parallel Execution

Run tasks concurrently:

```toml
[tool.poe.tasks.check-fast]
help = "Run linter and type checker in parallel"
parallel = ["_lint", "_typecheck"]
```

### Combining Sequence and Parallel

```toml
[tool.poe.tasks.ci]
help = "Run full CI pipeline"
sequence = [
    ["_lint", "_typecheck"],   # These run in parallel first
    "_test",                    # Then tests run after both complete
]
```

### Dependencies and Output Capture

Tasks can declare dependencies and capture their output:

```toml
[tool.poe.tasks._get_version]
cmd = "python -c \"import tomllib; print(tomllib.load(open('pyproject.toml','rb'))['project']['version'])\""

[tool.poe.tasks.tag-release]
help = "Create a git tag for the current version"
cmd = "git tag v${_version}"
deps = ["_test"]
uses = { _version = "_get_version" }
```

`deps` run before the task. `uses` captures stdout from a task into a variable.

### Handling Failures in Sequences

```toml
# Continue despite failures, but return non-zero if any failed
[tool.poe.tasks.check-all]
sequence = ["_lint", "_typecheck", "_test"]
ignore_fail = "return_non_zero"
```

## Environment Variables

### Task-Level Variables

```toml
[tool.poe.tasks.serve-prod]
help = "Start the server in production mode"
cmd = "uvicorn app.main:app"
env = { ENVIRONMENT = "production", DEBUG = "false" }
```

### Default Values

Set only if not already defined in the environment:

```toml
[tool.poe.tasks.serve]
cmd = "uvicorn app.main:app"
env.PORT.default = "8000"
```

### Task-Level Env Files

```toml
[tool.poe.tasks.serve]
cmd = "uvicorn app.main:app"
envfile = ["base.env", "local.env"]
```

With optional files:

```toml
[tool.poe.tasks.serve.envfile]
expected = ["base.env"]
optional = ["local.env"]
```

### Variable Precedence (lowest to highest)

1. Host environment (`os.environ`)
2. Project-level env files
3. Project-level env config
4. Task-level env files
5. Task-level env config
6. Output from dependency tasks (`uses`)
7. Task `args`

### Internal Variables

| Variable | Description |
|----------|-------------|
| `POE_ROOT` | Directory containing the pyproject.toml |
| `POE_PWD` | Current working directory when poe was invoked |
| `POE_GIT_DIR` | Git repository root (useful in monorepos) |

## Example: Project Task Configuration

A complete task setup for a FastAPI project:

```toml
[tool.poe]
executor = "uv"
envfile = ".env"

[tool.poe.tasks]
# Development
serve = { help = "Start dev server with hot reload", cmd = "uvicorn app.main:app --reload" }
worker = { help = "Start Celery worker", cmd = "celery -A app.worker worker --loglevel=info" }

# Code Quality
lint = { help = "Lint and auto-fix", cmd = "ruff check --fix ." }
format = { help = "Format code", cmd = "ruff format ." }
typecheck = { help = "Run type checker", cmd = "ty check" }

# Testing
test = { help = "Run test suite", cmd = "pytest tests/ -v" }

# Composed tasks
[tool.poe.tasks.check]
help = "Run all quality checks and tests"
sequence = [["lint", "typecheck"], "test"]

[tool.poe.tasks.ci]
help = "Full CI pipeline: format check, lint, typecheck, test"
sequence = [
    { cmd = "ruff format --check ." },
    ["lint", "typecheck"],
    "test",
]
```
