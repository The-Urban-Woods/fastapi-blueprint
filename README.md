# FastAPI Blueprint

The FastAPI Blueprint skill helps coding agents generate production-ready FastAPI applications following domain-driven design with the repository-service pattern and Dishka-based dependency injection. It provides architectural guidance, generates idiomatic code, and helps scaffold new domain modules using modern best practices.

## Available Skills

- **`fastapi-blueprint`**: Generates structured FastAPI code with domain-driven architecture. Useful for creating domain modules, CRUD endpoints, repositories, services, dependency injection setup, database models, Pydantic schemas, background jobs, and tests.

## Using Agent Skills

Agent Skills are designed to be used with agentic coding tools like [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Gemini CLI](https://geminicli.com/docs/cli/skills/), [Codex](https://github.com/openai/codex) and more. Activating a skill loads the specific instructions and resources needed for that task.

To use this skill in your own environment you may follow the instructions for your specific tool or use a community tool like [skills.sh](https://skills.sh/).

```bash
npx skills add https://github.com/The-Urban-Woods/fastapi-blueprint
```

## Reference Guides

The skill includes detailed reference guides covering every layer of the architecture:

| Guide | Topics |
| --- | --- |
| [project-structure.md](references/project-structure.md) | Project layout, naming conventions, module init |
| [app-entrypoint.md](references/app-entrypoint.md) | App factory, lifespan, middleware, router registration |
| [configuration.md](references/configuration.md) | Settings, environment variables, pydantic-settings |
| [database-setup.md](references/database-setup.md) | SQLAlchemy Base, ORM models, Pydantic schemas, transactions |
| [repository-pattern.md](references/repository-pattern.md) | Repository ABC, SqlAlchemyRepository, domain repositories |
| [service-layer.md](references/service-layer.md) | Service design, CrudService/Service ABCs, cross-domain services |
| [api-endpoints.md](references/api-endpoints.md) | DishkaRoute, FromDishka, endpoint patterns |
| [dependency-injection.md](references/dependency-injection.md) | Dishka providers, scopes, wiring, usage outside FastAPI |
| [background-jobs.md](references/background-jobs.md) | Celery setup, using services in tasks, retries, workflows |
| [testing.md](references/testing.md) | Test fixtures, TestAppProvider, integration and unit testing |
| [code-style.md](references/code-style.md) | Ruff linting/formatting, ty type checking, naming conventions |
| [uv-package-manager.md](references/uv-package-manager.md) | uv commands, dependencies, lockfile, Docker, CI/CD |
| [task-runner.md](references/task-runner.md) | Poe the Poet task runner, composing tasks, CLI arguments |

## Contributions

We welcome contributions to the FastAPI Blueprint skill. If you would like to contribute, please submit a pull request or open an issue.

### Feedback & Issues

If you encounter a bug, have feedback, or want to suggest an improvement, please file an issue in the [The-Urban-Woods/fastapi-blueprint](https://github.com/The-Urban-Woods/fastapi-blueprint/issues) issue tracker.

### Pull Requests

We accept pull requests for new features, updates, or bug fixes:

1. Make your changes to `SKILL.md` or the files in the `references/` directory.
2. Submit a Pull Request to the `The-Urban-Woods/fastapi-blueprint` repository.

## License

[MIT](LICENSE)
