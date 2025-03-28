# Contributing to Intric

Thank you for your interest in contributing to Intric! This document provides guidelines and instructions for contributing to the project.

## Table of Contents
- [Code of Conduct](#code-of-conduct)
- [Getting Started](#getting-started)
- [Development Workflow](#development-workflow)
- [Commit Messages](#commit-messages)
- [Coding Standards](#coding-standards)
- [Testing](#testing)
- [Documentation](#documentation)
- [Pull Request Process](#pull-request-process)
- [Dependency Management](#dependency-management)
- [Community](#community)

## Code of Conduct

Our community is dedicated to providing a harassment-free experience for everyone. We do not tolerate harassment of participants in any form. By participating in this project, you agree to abide by these principles. Please ensure your interactions are respectful and constructive.

## Getting Started

### 1. Fork the Repository
- Fork the main Intric repository to your personal GitHub account
- Clone your fork locally:
  ```bash
  git clone https://github.com/YOUR_USERNAME/intric.git
  ```
- Configure the upstream remote:
  ```bash
  git remote add upstream <main_intric_repo_url>
  ```

### 2. Set Up Development Environment
- Follow the instructions in the [Development Guide](./development-guide.md) to set up your local environment
- Install prerequisites (Python, Node.js, Docker, Poetry, pnpm)
- Configure the required environment variables

### 3. Find an Issue
- Look for open issues labeled `good first issue` or `help wanted` in our GitHub issue tracker
- Comment on issues you'd like to work on to express interest
- For new ideas or bug fixes not already listed, open a new issue first to discuss the proposed changes

## Development Workflow

We follow a standard Git workflow using feature branches:

### 1. Stay Up-to-Date
Before starting work, ensure your local `main` branch is up-to-date:
```bash
git checkout main
git fetch upstream
git pull upstream main
```

### 2. Create a Feature/Fix Branch
Create a new branch from the up-to-date `main` branch:
```bash
# For new features
git checkout -b feature/your-descriptive-feature-name

# For bug fixes
git checkout -b fix/short-issue-description
```

### 3. Develop & Commit
- Make your code changes
- Commit frequently with clear, descriptive messages following the Conventional Commits format
- Run tests and linters locally to validate your changes

### 4. Keep Branch Updated (Recommended)
Periodically update your feature branch with the latest changes:
```bash
git fetch upstream
git rebase upstream/main
# Resolve any conflicts that arise during rebase
```

### 5. Push Your Branch
Push your branch to your fork on GitHub:
```bash
git push origin feature/your-descriptive-feature-name
```

## Commit Messages

We follow the [Conventional Commits specification](https://www.conventionalcommits.org/) to make the commit history easier to understand and to automate changelog generation.

### Format

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### Common Types

| Type | Purpose |
|------|---------|
| `feat` | A new feature |
| `fix` | A bug fix |
| `docs` | Documentation only changes |
| `style` | Code style changes (formatting, etc.) |
| `refactor` | Code changes (no new features/fixes) |
| `perf` | Performance improvements |
| `test` | Adding or correcting tests |
| `build` | Build system or dependency changes |
| `ci` | CI configuration changes |
| `chore` | Other changes not modifying src/test |

### Example

```
feat(assistants): add support for streaming responses via SSE

Implement Server-Sent Events endpoint for assistant sessions to allow
for real-time streaming of generated tokens to the frontend.

Ref: #123
```

## Coding Standards

### General
- Follow Domain-Driven Design principles (see [Domain Driven Design](./domain-driven-design.md))
- Ensure new features align with the existing architecture
- Organize code by business domain

### Python (Backend)
- Follow **PEP 8** style guide
- Use **Type Hints** for all function/method parameters and return values
- Write clear **Docstrings** for all public functions, classes, and modules
- Use **Ruff** for linting and formatting:
  ```bash
  cd backend
  # Check for issues
  poetry run ruff check .
  # Automatically format code
  poetry run ruff format .
  ```

### TypeScript/JavaScript (Frontend)
- Follow the project's **ESLint** configuration
- Use **Prettier** for code formatting
- Follow **SvelteKit** conventions for components, routing, and load functions
- Maintain responsive and accessible design principles
- Document component props and events using JSDoc comments
- Run checks before committing:
  ```bash
  cd frontend
  # Check for linting issues across workspaces
  pnpm lint
  # Automatically format code across workspaces
  pnpm format
  ```

## Testing

### Write Tests
All new features and bug fixes should include corresponding tests:
- Unit tests for isolated logic
- Integration tests for component interactions
- API tests for endpoints
- E2E tests for critical user flows

### Maintain Coverage
- Aim for high test coverage (80%+ for backend code)
- Ensure changes don't significantly decrease coverage

### Run Tests Locally
Before submitting a PR, run all relevant tests:

```bash
# Run backend tests
cd backend
poetry run pytest

# Run frontend tests
cd frontend
pnpm test  # Runs tests across all frontend workspaces
# Or target specific workspace: pnpm --filter @intric/web test
```

## Documentation

- **Update Guides**: If your changes affect setup, configuration, deployment, architecture, or user workflows, update the relevant Markdown guides in the `/docs` directory.

- **API Documentation**: For backend changes, ensure FastAPI routes have clear docstrings, as these are used to generate the OpenAPI/Swagger documentation.

- **Code Documentation**: Add or update Python docstrings and TS/JS JSDoc comments to explain complex logic, component usage, or function parameters/return values.

- **READMEs**: Update `README.md` files if setup or usage instructions change.

## Pull Request Process

### 1. Create Pull Request
Once your branch is ready and pushed to your fork, create a Pull Request (PR) from your branch to the upstream repository's `main` branch.

### 2. Title and Description
- Use a clear, descriptive title summarizing the changes (similar to a conventional commit message)
- Provide a detailed description explaining:
  - What changes were made
  - Why these changes are needed
  - How the implementation works
  - Any testing considerations
- Link to any related GitHub issues using keywords like `Fixes #123`, `Closes #456`, or `Ref #789`

### 3. Review
- The maintainers will review your PR
- Address any feedback by pushing new commits to your feature branch
- Engage in discussion on the PR comments if clarification is needed

### 4. CI Checks
- Automated checks (tests, linting, security scans) must pass
- Monitor the status checks on your PR and fix any failures

### 5. Merging
- Once approved and all checks pass, a maintainer will merge your PR
- We typically use **squash merging** to combine all commits from the feature branch into a single commit on the `main` branch
- Ensure your PR title and description are well-written as they form the basis of the squashed commit message

## Dependency Management

### Backend (Python)
Use Poetry for managing dependencies:

```bash
cd backend
# Add a new dependency
poetry add <package-name>
# Add a new development dependency
poetry add --group dev <package-name>
# Update dependencies
poetry update
```

Commit both the updated `pyproject.toml` and `poetry.lock` files.

### Frontend (TS/JS)
Use pnpm workspaces:

```bash
cd frontend
# Add dependency to a specific workspace (e.g., web app)
pnpm add <package-name> --filter @intric/web
# Add dev dependency to a specific workspace
pnpm add <package-name> --filter @intric/web --save-dev
# Add dependency to the root (shared across workspaces)
pnpm add <package-name> -w
```

Commit both the updated relevant `package.json` and the root `pnpm-lock.yaml` file.

## Community

Join our community to discuss development, ask questions, and get help:

- **Community Forum**: Join by emailing [digitalisering@sundsvall.se](mailto:digitalisering@sundsvall.se)
- **GitHub Issues**: Use for bug reports, feature requests, and specific technical questions related to the codebase

Thank you for contributing to Intric!