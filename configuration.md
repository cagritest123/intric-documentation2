# Intric Configuration Guide

## TLDR

> **Key Configuration Information**
>
> - **Configuration Method**: Primarily via environment variables (`.env` file or system environment)
> - **Key Files**: Backend settings defined in `backend/src/intric/main/config.py`, Frontend settings in `.env` (using `VITE_` prefix)
> - **Required Backend Variables**: `POSTGRES_PASSWORD`, `JWT_SECRET`, database connection details, Redis connection details, and at least one LLM API Key
> - **Required Frontend Variables**: `VITE_INTRIC_BACKEND_URL` for the frontend to connect to the backend
> - **Deployment Variables**: See `deployment-guide.md` for production-specific variables

This guide provides comprehensive information about configuring the Intric platform through environment variables.

## Table of Contents
- [Configuration Overview](#configuration-overview)
- [Backend Environment Variables](#backend-environment-variables)
  - [Database](#database)
  - [Redis](#redis)
  - [LLM / AI Services](#llm--ai-services)
  - [JWT Security](#jwt-security)
  - [API Keys](#api-keys)
  - [MobilityGuard OIDC (Optional)](#mobilityguard-oidc-optional)
  - [File Uploads & Limits](#file-uploads--limits)
  - [Web Crawling](#web-crawling)
  - [Feature Flags](#feature-flags)
  - [Development & Testing](#development--testing)
  - [General Settings](#general-settings)
- [Frontend Environment Variables](#frontend-environment-variables)
- [Docker Configuration Notes](#docker-configuration-notes)
- [Advanced Configuration](#advanced-configuration)

## Configuration Overview

Intric uses environment variables for configuration, following the 12-factor app methodology. Variables can be provided via:

1. **`.env` files**: Placed in `backend/` and `frontend/apps/web/` directories for local development
2. **System Environment Variables**: Set directly in the shell or deployment environment (recommended for production secrets)
3. **Docker Compose**: Using `environment` or `env_file` in `docker-compose.yml`

Settings defined in `backend/src/intric/main/config.py` (using Pydantic's `BaseSettings`) automatically read from environment variables or `.env` files. Frontend variables typically follow Vite conventions with a `VITE_` prefix.

## Backend Environment Variables

These variables configure the FastAPI backend, worker, and related services.

### Database

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `POSTGRES_HOST` | Database server hostname/IP | `None` | **Yes**\* |
| `POSTGRES_PORT` | Database server port | `None` | **Yes**\* |
| `POSTGRES_USER` | Database username | `None` | **Yes**\* |
| `POSTGRES_PASSWORD` | Database password | `None` | **Yes** |
| `POSTGRES_DB` | Database name | `None` | **Yes**\* |
| `DATABASE_URL` | Full database connection URL | `None` | No† |

\* *Required if `DATABASE_URL` is not provided directly*  
† *Alternative to individual connection parameters*

### Redis

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `REDIS_HOST` | Redis server hostname/IP | `None` | **Yes** |
| `REDIS_PORT` | Redis server port | `None` | **Yes** |

### LLM / AI Services

> **Note**: At least one LLM provider key or service URL is typically required for core functionality.

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `OPENAI_API_KEY` | API Key for OpenAI services | `None` | No\* |
| `ANTHROPIC_API_KEY` | API Key for Anthropic services | `None` | No\* |
| `INFINITY_URL` | URL for Infinity embedding service | `None` | No |
| `VLLM_MODEL_URL` | URL for vLLM service | `None` | No |
| `WHISPER_MODEL_URL` | URL for Whisper transcription | `None` | No |
| `WHISPER_MODEL_NAME` | Whisper model name | `whisper-1` | No |
| `USING_AZURE_MODELS` | Enable Azure OpenAI | `False` | No |
| `AZURE_API_KEY` | API Key for Azure OpenAI | `None` | No\* |
| `AZURE_ENDPOINT` | Azure OpenAI endpoint URL | `None` | No† |
| `AZURE_API_VERSION` | Azure API version | `None` | No† |

\* *At least one LLM API key is required for functionality*  
† *Required if `USING_AZURE_MODELS=true`*

### JWT Security

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `JWT_SECRET` | Secret key for signing JWT tokens | `None` | **Yes** |
| `JWT_ALGORITHM` | Algorithm for JWT signing | `HS256` | No |
| `JWT_AUDIENCE` | Expected audience claim | `intric` | No |
| `JWT_ISSUER` | Expected issuer claim | `intric` | No |
| `JWT_EXPIRY_TIME` | Token expiry time (minutes) | `None` | **Yes** |
| `JWT_TOKEN_PREFIX` | Auth header prefix | `Bearer` | No |

### API Keys

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `API_KEY_HEADER_NAME` | HTTP Header name for API keys | `X-API-Key` | No |
| `API_KEY_LENGTH` | Byte length for key entropy | `None` | **Yes** |

### MobilityGuard OIDC (Optional)

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `MOBILITYGUARD_DISCOVERY_ENDPOINT` | OIDC discovery URL | `None` | No |
| `MOBILITYGUARD_CLIENT_ID` | OIDC client ID | `None` | No |
| `MOBILITYGUARD_CLIENT_SECRET` | OIDC client secret | `None` | No |
| `MOBILITYGUARD_TENANT_ID` | Tenant UUID for MG users | `None` | No |

### File Uploads & Limits

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `UPLOAD_MAX_FILE_SIZE` | Max size for file uploads (bytes) | `None` | **Yes** |
| `UPLOAD_FILE_TO_SESSION_MAX_SIZE` | Max size for chat file uploads | `None` | **Yes** |
| `UPLOAD_IMAGE_TO_SESSION_MAX_SIZE` | Max size for chat image uploads | `None` | **Yes** |
| `TRANSCRIPTION_MAX_FILE_SIZE` | Max size for transcription files | `None` | **Yes** |
| `MAX_IN_QUESTION` | Max file references per question | `None` | **Yes** |

### Web Crawling

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `USING_CRAWL` | Enable/disable crawling feature | `True` | No |
| `CRAWL_MAX_LENGTH` | Max crawl duration (seconds) | `14400` | No |
| `CLOSESPIDER_ITEMCOUNT` | Max pages to crawl per run | `20000` | No |
| `OBEY_ROBOTS` | Honor robots.txt rules | `True` | No |
| `AUTOTHROTTLE_ENABLED` | Enable polite crawling | `True` | No |

### Feature Flags

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `USING_ACCESS_MANAGEMENT` | Enable access management | `True` | No |
| `USING_INTRIC_PROPRIETARY` | Enable proprietary features | `False` | No |
| `HOSTING_INTRIC_PROPRIETARY` | Host proprietary components | `False` | No |
| `USING_IAM` | Enable alternative IAM integration | `False` | No |

### Development & Testing

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `DEV` | Development mode | `False` | No |
| `TESTING` | Testing mode | `False` | No |

### General Settings

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `LOGLEVEL` | Logging level | `INFO` | No |
| `API_PREFIX` | Base path for API routes | `/api/v1` | No |

## Frontend Environment Variables

These variables are configured via a `.env` file in `frontend/apps/web/` or passed during build.

| Variable | Description | Default | Required | Client-Exposed |
|----------|-------------|---------|----------|---------------|
| `INTRIC_BACKEND_URL` | Backend URL (build time) | `http://localhost:8123` | No | No |
| `VITE_INTRIC_BACKEND_URL` | Backend URL (runtime) | `None` | **Yes** | Yes |
| `VITE_PUBLIC_URL` | Public base URL | `/` | No | Yes |
| `VITE_FEEDBACK_FORM_URL` | External feedback form URL | `None` | No | Yes |
| `VITE_MOBILITY_GUARD_AUTH` | MobilityGuard Auth URL | `None` | No | Yes |
| `VITE_SHOW_TEMPLATES` | Show templates feature | `true` | No | Yes |

> **Note**: Variables with `VITE_` prefix are exposed to the client-side browser code. The `INTRIC_BACKEND_URL` (without prefix) is used during build time in `vite.config.ts`.

## Docker Configuration Notes

When deploying with Docker (see `deployment-guide.md`), environment variables are passed into containers using:

- An `env_file` directive in `docker-compose.yml` pointing to a central `.env` file
- Variables defined directly under the `environment:` key for each service

> **Security Note**: Ensure secrets (passwords, API keys, JWT tokens) are handled securely in production environments.

## Advanced Configuration

### Resource Limits

Set resource limits via Docker Compose `deploy.resources`:

```yaml
services:
  backend:
    # ... other configuration ...
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
        reservations:
          cpus: '0.25'
          memory: 512M
```

### Custom Settings

The `Settings` class in `config.py` can be extended for custom application configuration needs. Backend settings are accessible via dependency injection throughout the application.