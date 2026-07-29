# 001-tdd-fastapi

A FastAPI project with TDD practices, managed with `uv`.

## Getting Started

```bash
# Install all dependencies (prod + dev)
uv sync --all-extras

# Install production dependencies only
uv sync
```

## Code Quality

This project uses **Ruff** for linting and formatting, replacing flake8, black, and isort.

```bash
# Check code quality
uv run ruff check .

# Fix auto-fixable issues
uv run ruff check . --fix

# Format code
uv run ruff format .

# Check formatting without changing files
uv run ruff format --check .
```

## Testing

```bash
# Run all tests
uv run pytest

# Run tests with coverage report
uv run pytest --cov=app
```

## Configuration

All project configuration lives in `pyproject.toml`:

- **Dependencies**: `[project.dependencies]` (prod) and `[project.optional-dependencies.dev]` (dev)
- **Ruff**: `[tool.ruff]` — lint, format, and line length settings
- **Aerich**: `[tool.aerich]` — database migration configuration
