# Intric Development Guide

## TLDR

> **Quick Setup**
> 
> **Prerequisites**: Python 3.10+, Node.js 20+, Docker & Docker Compose, Poetry, pnpm 8.9.0+, Git, libmagic, ffmpeg.
> 
> **Start Services**:
> 1. Clone repo & install OS dependencies (`libmagic1`, `ffmpeg`)
> 2. Start DB/Redis: `cd backend && docker compose up -d`
> 3. Setup Backend:
>    ```bash
>    cd backend
>    poetry install
>    cp .env.template .env  # Edit essential vars
>    poetry run python init_db.py  # First time DB setup
>    poetry run uvicorn intric.server.main:app --reload --host 0.0.0.0 --port 8123
>    ```
> 4. Setup Frontend:
>    ```bash
>    cd frontend
>    pnpm install
>    cd apps/web
>    cp .env.example .env  # Edit if needed
>    cd ../../
>    pnpm --filter @intric/web dev
>    ```
> 5. (Optional) Start Worker: `cd backend && poetry run arq intric.worker.arq.WorkerSettings`
> 6. Access: `http://localhost:3000` (Login: `user@example.com` / `Password1!`)

This guide provides detailed instructions for setting up a development environment for the Intric platform and contributing to the project.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Initial Setup](#initial-setup)
- [Running the Application Locally](#running-the-application-locally)
- [Using Devcontainer (Recommended)](#using-devcontainer-recommended)
- [Project Structure](#project-structure)
- [Backend Development](#backend-development)
- [Frontend Development](#frontend-development)
- [Architectural Patterns](#architectural-patterns)
- [Testing](#testing)
- [Linting](#linting)
- [Documentation](#documentation)
- [Contributing Guidelines](#contributing-guidelines)

## Prerequisites

Ensure the following tools are installed on your system:

| Tool | Version | Notes |
|------|---------|-------|
| **Python** | 3.10+ | Required for backend development |
| **Poetry** | Latest | Python dependency management |
| **Node.js** | 20+ | Required for frontend development |
| **pnpm** | 8.9.0+ | Node.js package manager |
| **Docker & Docker Compose** | Latest | For infrastructure services |
| **Git** | Latest | Version control |
| **System Libraries** | - | `libmagic` and `ffmpeg` for file processing |

**Installing system dependencies:**
```bash
# Example for Debian/Ubuntu
sudo apt-get update && sudo apt-get install -y libmagic1 ffmpeg
```

## Initial Setup

### 1. Clone Repository
```bash
git clone <repository_url> intric
cd intric
```

### 2. Install Backend Dependencies
```bash
cd backend
poetry install
cd ..
```

### 3. Install Frontend Dependencies
```bash
cd frontend
pnpm install  # Installs dependencies for all frontend workspaces
cd ..
```

### 4. Start Infrastructure Services
The backend includes a `docker-compose.yml` specifically for starting development dependencies:
```bash
cd backend
docker compose up -d  # Starts db and redis containers
cd ..
```

### 5. Configure Environment Variables

#### Backend
```bash
cd backend
cp .env.template .env
# Edit .env with your preferred editor
nano .env
```

Essential variables to configure:
- `POSTGRES_HOST=localhost` (or `db` if running backend in Docker)
- `POSTGRES_PORT=5432`
- `POSTGRES_USER=postgres`
- `POSTGRES_PASSWORD=your_chosen_password` (must match backend `docker-compose.yml`)
- `POSTGRES_DB=postgres`
- `REDIS_HOST=localhost` (or `redis` if running backend in Docker)
- `REDIS_PORT=6379`
- `JWT_SECRET=generate_a_strong_random_secret` (e.g., using `openssl rand -hex 32`)

#### Frontend
```bash
cd frontend/apps/web
cp .env.example .env
# Edit .env if necessary
nano .env
```

Essential variables to configure:
```env
VITE_INTRIC_BACKEND_URL=http://localhost:8123
```

### 6. Initialize Database (First Time Only)
```bash
cd backend
poetry run python init_db.py
cd ..
```
This creates the database schema and a default user (`user@example.com` / `Password1!`).

## Running the Application Locally

Run each service in a separate terminal:

### 1. Backend API
```bash
cd backend
poetry run uvicorn intric.server.main:app --reload --host 0.0.0.0 --port 8123
```
The `--reload` flag automatically restarts the server on code changes.

### 2. Frontend Dev Server
```bash
cd frontend
pnpm --filter @intric/web dev
```
This specifically targets the `dev` script within the `apps/web` workspace.

### 3. Background Worker (Optional)
Needed for features like document processing or crawling:
```bash
cd backend
poetry run arq intric.worker.arq.WorkerSettings
```

### Access the Application
- Frontend UI: `http://localhost:3000`
- Backend API: `http://localhost:8123`
- Backend API Docs (Swagger): `http://localhost:8123/docs`
- **Default Login**: `user@example.com` / `Password1!`

## Using Devcontainer (Recommended)

This project supports development inside a Docker container using the VS Code Remote - Containers extension, providing a consistent environment.

### 1. Prerequisites
- Docker
- VS Code
- Remote - Containers extension

### 2. Configure Environment Files
Before launching, ensure `.env` files exist:
```bash
# In the ./backend directory
cp .env.template .env
# Edit backend/.env with secrets (at least JWT_SECRET)

# In the ./frontend/apps/web directory
cp .env.example .env
# Edit frontend/apps/web/.env if needed (VITE_INTRIC_BACKEND_URL default is likely fine)
```

### 3. Open Project in Container
- Open the project folder in VS Code
- Use the command palette (`Ctrl+Shift+P` or `Cmd+Shift+P`) and select "Remote-Containers: Reopen in Container"
- VS Code will build/start the containers defined in `.devcontainer/docker-compose.yml`

### 4. Post-Create Setup
The `postCreateCommand` installs Python and Node dependencies automatically.

### 5. Initialize Database (First Time Only)
Open a terminal *inside the devcontainer* (`Terminal > New Terminal`):
```bash
cd backend
poetry run python init_db.py
```

### 6. Run Services
Open separate terminals *inside the devcontainer* for each service:
- **Backend**: `cd backend && poetry run uvicorn intric.server.main:app --reload --host 0.0.0.0 --port 8123`
- **Frontend**: `cd frontend && pnpm --filter @intric/web dev`
- **Worker** (Optional): `cd backend && poetry run arq intric.worker.arq.WorkerSettings`

### 7. Access
Services are automatically port-forwarded. Access via `http://localhost:3000`.

## Project Structure

```
intric/
├── backend/                # Backend Python (FastAPI) application
│   ├── src/intric/         # Source code, organized by domain (DDD)
│   │   ├── server/         # FastAPI app setup, routers, middleware
│   │   ├── database/       # SQLAlchemy models, SessionManager
│   │   ├── worker/         # ARQ Worker settings and tasks
│   │   ├── <domain>/       # Feature domains (e.g., assistants, spaces)
│   │   │   ├── api/        # FastAPI router, models, assembler
│   │   │   ├── <domain>.py # Core domain entity/logic
│   │   │   ├── <domain>_repo.py # Repository (data access)
│   │   │   └── <domain>_service.py # Application service (use cases)
│   │   └── ...
│   ├── alembic/            # Database migrations
│   ├── tests/              # Pytest tests (unit, integration)
│   ├── .env.template       # Backend environment variable template
│   ├── docker-compose.yml  # Infrastructure (DB, Redis) for local dev
│   └── pyproject.toml      # Poetry config and dependencies
├── frontend/               # Frontend monorepo (pnpm workspace)
│   ├── apps/
│   │   └── web/            # Main SvelteKit application
│   │       ├── src/        # SvelteKit source
│   │       │   ├── lib/    # Components, stores, features, core logic
│   │       │   └── routes/ # Application pages/routes
│   │       ├── .env.example # Frontend environment variable template
│   │       └── package.json
│   ├── packages/
│   │   ├── intric-js/      # JS client library for Intric API
│   │   └── ui/             # Reusable Svelte UI component library
│   ├── package.json        # Root frontend dependencies
│   └── pnpm-workspace.yaml # pnpm workspace config
├── .devcontainer/          # VS Code Devcontainer configuration
├── docker-compose.yml      # Docker Compose for PRODUCTION deployment
├── build_and_push.sh       # Script for building/pushing Docker images
└── .env.example            # Example environment variables for PRODUCTION
```

## Backend Development

### Technology Stack
- **Framework**: FastAPI
- **ORM**: SQLAlchemy with AsyncPG
- **Migrations**: Alembic
- **Database**: PostgreSQL + pgvector
- **Task Queue**: ARQ (Redis)
- **Dependencies**: Poetry
- **Testing**: Pytest

### Key Points
- Follows Domain-Driven Design (see Architectural Patterns).
- Uses FastAPI's built-in Dependency Injection extensively.
- Asynchronous operations (`async`/`await`) are used throughout.
- Database interactions via SQLAlchemy async sessions and repositories.
- Background tasks offloaded to ARQ workers.

### Database Migrations
To modify database schema:

1. Modify SQLAlchemy models in `backend/src/intric/database/tables/`

2. Generate a new migration script:
   ```bash
   cd backend
   poetry run alembic revision --autogenerate -m "Your migration description"
   ```

3. Review the generated script in `backend/alembic/versions/`

4. Apply migrations:
   ```bash
   cd backend
   poetry run alembic upgrade head
   ```

## Frontend Development

### Technology Stack
- **Framework**: SvelteKit
- **Package Manager**: pnpm (using workspaces)
- **UI Library**: Custom components in `packages/ui` + Tailwind CSS
- **API Client**: Custom client in `packages/intric-js`
- **State Management**: Svelte stores, often encapsulated in "Manager" classes
- **Styling**: Tailwind CSS + standard CSS/SCSS

### Key Points
- Monorepo structure managed by pnpm workspaces (`frontend/`)
- Reusable UI components live in `packages/ui`
- API client library lives in `packages/intric-js`
- Main web application lives in `apps/web`
- Uses SvelteKit's load functions for server-side data fetching

## Architectural Patterns

### Domain-Driven Design (Backend)

The backend heavily utilizes DDD principles:

#### Bounded Contexts
Code organized by feature domain (e.g., `assistants`, `spaces`, `users`).

#### Layers within Contexts
- **API Layer** (`api/`): Handles HTTP requests/responses, Pydantic models, assembling DTOs
- **Application Layer** (`*_service.py`): Orchestrates use cases, interacts with repositories and other services
- **Domain Layer** (`<domain>.py`): Core domain entities, encapsulating state and domain logic
- **Infrastructure Layer** (`*_repo.py`): Abstracts data persistence logic (SQLAlchemy)

#### Dependency Injection
FastAPI's dependency injection system is used to provide repositories, services, etc., to routers and other services, promoting loose coupling.

See `domain-driven-design.md` for more conceptual details.

## Testing

### Backend Testing
Uses `pytest`. Run tests via:
```bash
cd backend
poetry run pytest
```

Tests are organized into `unit`, `integration`, etc., under `backend/tests/`. Fixtures often use dependency injection overrides for mocking.

### Frontend Testing
Uses Vitest (likely for unit/component tests) and Playwright (likely for end-to-end tests).

Run tests via pnpm:
```bash
cd frontend
# Example commands (check package.json for actual script names)
pnpm test  # Runs all tests across workspaces
pnpm --filter @intric/web test  # Runs tests specifically for the web app
pnpm --filter @intric/ui test  # Runs tests specifically for the UI package
```

## Linting

Code style and quality are maintained using linters.

### Backend Linting
Uses Ruff (check `pyproject.toml` for config):
```bash
cd backend
poetry run ruff check .
poetry run ruff format .
```

### Frontend Linting
Uses ESLint and Prettier (check configs like `eslint.config.js`, `.prettierrc`):
```bash
cd frontend
pnpm lint  # Check linting issues across workspaces
pnpm format  # Format code across workspaces
```

Run these linters before committing code. CI checks likely enforce these.

## Documentation

- **API Docs**: Auto-generated by FastAPI/Swagger UI at `/docs` endpoint of running backend.
- **Code Docs**: Use clear docstrings (Python) and comments (JS/TS) for non-obvious logic. Type hints are mandatory in Python.
- **Project Docs**: Written in Markdown (`.md` files) and intended to be built using Sphinx (see Sphinx setup guide if available).

## Contributing Guidelines

Refer to `contributing.md` for detailed guidelines on:
- Branching strategy (feature branches from `develop`)
- Committing code (conventional commits)
- Writing tests
- Submitting Pull Requests
- Code review process
- Coding standards (PEP 8, ESLint/Prettier)