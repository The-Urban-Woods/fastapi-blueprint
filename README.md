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
