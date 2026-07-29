# 001-tdd-fastapi

A FastAPI-based URL summarization service with TDD practices, managed with `uv`.

## Overview

This project provides a REST API for submitting URLs and receiving automatically generated text summaries. It uses:

- **FastAPI** — modern, high-performance web framework
- **Tortoise ORM** — async ORM with Aerich for database migrations
- **PostgreSQL** — relational database
- **Newspaper3k + NLTK** — web scraping and text summarization
- **Pydantic Settings** — environment-based configuration
- **Ruff** — linting and formatting
- **Docker** — containerized development and production environments

## Project Structure

```
.
├── app/
│   ├── api/          # API route handlers
│   │   ├── ping.py   # Health check endpoint
│   │   ├── summaries.py  # CRUD operations for summaries
│   │   └── crud.py   # Database CRUD operations
│   ├── models/
│   │   ├── tortoise.py  # Tortoise ORM models
│   │   └── pydantic.py  # Pydantic request/response schemas
│   ├── config.py      # Application settings
│   ├── db.py          # Database initialization
│   ├── main.py        # FastAPI application factory
│   └── summarizer.py  # Background summary generation
├── test/              # Pytest test suite
├── migrations/        # Aerich database migrations
├── db/                # PostgreSQL Docker setup
├── Dockerfile         # Development container
├── Dockerfile.prod    # Production container
├── docker-compose.yml # Local development stack
└── pyproject.toml     # Project configuration
```

## Getting Started

### Prerequisites

- [Python 3.14+](https://www.python.org/downloads/)
- [uv](https://docs.astral.sh/uv/) — Python package manager
- [Docker](https://www.docker.com/) — for local development with PostgreSQL

### Local Development

```bash
# Install all dependencies (prod + dev)
uv sync --all-extras

# Install production dependencies only
uv sync
```

### Docker Development

Start the full stack (app + PostgreSQL) with docker-compose:

```bash
# Build and start services
docker compose up --build

# Run in detached mode
docker compose up -d --build

# Stop services
docker compose down
```

The API will be available at `http://localhost:8004`.

## API Endpoints

| Method   | Path                        | Description              |
|----------|-----------------------------|--------------------------|
| `GET`    | `/ping`                     | Health check             |
| `POST`   | `/summaries/`               | Create a new summary     |
| `GET`    | `/summaries/`               | List all summaries       |
| `GET`    | `/summaries/{id}/`          | Get a specific summary   |
| `PUT`    | `/summaries/{id}/`          | Update a summary         |
| `DELETE` | `/summaries/{id}/`          | Delete a summary         |

### Example: Create a Summary

```bash
curl -X POST http://localhost:8004/summaries/ \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/article"}'
```

Response:
```json
{
  "id": 1,
  "url": "https://example.com/article"
}
```

The summary is generated asynchronously in the background. Poll the endpoint to check for completion:

```bash
curl http://localhost:8004/summaries/1/
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

# Run with HTML coverage report
uv run pytest --cov=app --cov-report=html

# Run a specific test file
uv run pytest test/test_summaries.py -v

# Run with verbose output
uv run pytest -v
```

## Database Migrations

Aerich handles database migrations:

```bash
# Create a new migration
uv run aerich migrate --name "description"

# Apply migrations
uv run aerich upgrade

# Downgrade to a specific migration
uv run aerich downgrade -n <migration_name>
```

## Configuration

All project configuration lives in `pyproject.toml`:

- **Dependencies**: `[project.dependencies]` (prod) and `[project.optional-dependencies.dev]` (dev)
- **Ruff**: `[tool.ruff]` — lint, format, and line length settings
- **Aerich**: `[tool.aerich]` — database migration configuration

Environment variables:

| Variable          | Description                      | Default        |
|-------------------|----------------------------------|----------------|
| `DATABASE_URL`    | PostgreSQL connection string     | (required)     |
| `DATABASE_TEST_URL`| Test database connection string  | (required)     |
| `ENVIRONMENT`     | App environment (dev/prod)       | `dev`          |
| `TESTING`         | Enable testing mode              | `0`            |
| `PORT`            | Server port (production)         | `8000`         |

## Docker Images

Two Dockerfiles are provided:

- **`Dockerfile`** — Development image with uvicorn, hot-reload enabled
- **`Dockerfile.prod`** — Production image with Gunicorn + Uvicorn workers

## License

Private
