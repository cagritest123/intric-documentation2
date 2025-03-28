# Intric Deployment Guide

## TLDR

> **Essential Requirements & Steps**
> 
> **Requirements**: Docker Engine 20.10+, Docker Compose 2.x+, Git. Recommended 4GB+ RAM, 10GB+ Disk.
> 
> **Build Method**: Use `build_and_push.sh` script to build & push images to a Docker registry. Requires Docker login and registry details.
> 
> **Configuration**: Use a `.env` file based on `.env.example` for all required variables (like `POSTGRES_PASSWORD`, `JWT_SECRET`, LLM keys).
> 
> **Deployment Steps**:
> 1. Configure `.env` (all variables)
> 2. `docker login <your-registry>`
> 3. `docker compose pull`
> 4. `docker compose --profile init up db-init --remove-orphans` (First time only)
> 5. `docker compose up -d`
> 
> **Updates**: Update `.env` (esp. `IMAGE_TAG`), pull images, run migrations, restart services.
> 
> **Production**: Implement reverse proxy for SSL/TLS, persistent volumes, backups, monitoring, and resource limits.

## Table of Contents
- [Prerequisites](#prerequisites)
- [System Requirements](#system-requirements)
- [Configuration](#configuration)
  - [Runtime Variables](#runtime-variables)
  - [Deployment Process Variables](#deployment-process-variables)
  - [Setting Up `.env`](#setting-up-env)
  - [Environment Variable Priority & Inconsistencies](#environment-variable-priority--inconsistencies)
- [Building Docker Images (Nexus Registry)](#building-docker-images-nexus-registry)
  - [Dockerfile Overview](#dockerfile-overview)
  - [Authentication](#authentication)
  - [Using `build_and_push.sh` (Recommended)](#using-build_and_pushsh-recommended)
  - [CI/CD Integration Notes](#cicd-integration-notes)
- [Deployment Steps (Docker Compose)](#deployment-steps-docker-compose)
  - [Initial Deployment](#initial-deployment)
  - [Verification](#verification)
- [Updating the Deployment](#updating-the-deployment)
- [Production Considerations](#production-considerations)
  - [Reverse Proxy & SSL/TLS](#reverse-proxy--ssltls)
  - [Persistent Volumes](#persistent-volumes)
  - [Database Backups](#database-backups)
  - [Monitoring & Health Checks](#monitoring--health-checks)
  - [Resource Limits](#resource-limits)
  - [Secrets Management](#secrets-management)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- Server (Linux recommended) with Docker Engine 20.10.x or later
- Docker Compose 2.x or later
- Git (for versioning used by build script)
- Access to a Docker registry (like Nexus) where Intric images are stored/pushed
- Credentials for the Docker registry
- Outbound internet connectivity (for pulling base images, connecting to LLM APIs)

## System Requirements

| Resource | Minimum | Recommended | Notes |
|----------|---------|-------------|-------|
| **RAM** | 2GB | 4GB+ | Requirements increase with usage and data size |
| **CPU** | 1 vCPU | 2+ vCPU | More CPU cores improve performance under load |
| **Disk Space** | 10GB | 50GB+ | Vector embeddings can require significantly more space |

- **Docker Images**: ~1-2GB for Backend, Frontend, Worker
- **Database Storage**: Text data is relatively small, but `pgvector` embeddings can require substantially more space (10x-30x original text)
- **Additional Space**: Needed for logs and temporary files

## Configuration

Intric deployment relies heavily on environment variables, managed via a `.env` file in the project root directory where `docker-compose.yml` resides.

### Runtime Variables

These variables are used by the application containers while running. **Refer to `configuration.md` for the complete list of runtime variables required by the application.**

**Key Required Runtime Variables:**

| Variable | Purpose | Required | Notes |
|----------|---------|----------|-------|
| `POSTGRES_PASSWORD` | Database authentication | **Yes** | Set a strong password |
| `JWT_SECRET` | Authentication token signing | **Yes** | Generate securely (e.g., `openssl rand -hex 32`) |
| `JWT_EXPIRY_TIME` | Token lifetime (minutes) | **Yes** | Balance security vs. user experience |
| `API_KEY_LENGTH` | API key generation | **Yes** | Determines security of generated API keys |
| **Database Connection** | Connection details | **Yes** | `POSTGRES_HOST=db`, `POSTGRES_PORT=5432`, etc. |
| **Redis Connection** | Queue broker connection | **Yes** | `REDIS_HOST=redis`, `REDIS_PORT=6379` |
| **File Upload Limits** | Control upload sizes | **Yes** | `UPLOAD_MAX_FILE_SIZE`, etc. |
| **LLM Provider API Key** | At least one LLM service | **Yes** | `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc. |

### Deployment Process Variables

These variables are needed during the build or deployment process itself.

| Variable | Description | Required | Example Value |
|----------|-------------|----------|--------------|
| `NEXUS_REGISTRY` | URL of your Docker registry | **Yes** | `your.nexus.registry.com` |
| `IMAGE_TAG` | Version tag for Docker images | **Yes** | `1.2.3` or `latest` |
| `NEXUS_USERNAME` | Registry authentication username | No* | `your_nexus_user` |
| `NEXUS_PASSWORD` | Registry authentication password | No* | `your_nexus_password` |
| `FRONTEND_PORT` | Host port for frontend container | No | `80` or `443` (default: `3000`) |
| `BACKEND_PORT` | Host port for backend container | No | Usually not exposed directly (default: `8123`) |
| `SERVICE_FQDN_FRONTEND` | Public domain name | No** | `intric.yourcompany.com` |

\* *Required if not authenticated through `docker login`*  
\** *Needed for public access/CSRF protection*

### Setting Up `.env`

1. Copy the template file to `.env`:
   ```bash
   cp .env.example .env  # Use the correct template name
   ```

2. Edit the `.env` file with all required variables:
   ```bash
   nano .env
   ```

3. **Security**: Ensure the `.env` file has restricted permissions and is **never** committed to version control.

### Environment Variable Priority & Inconsistencies

- Docker Compose loads variables from the `.env` file and substitutes them in `docker-compose.yml`.
- The pattern `${VARIABLE_NAME:-default_value}` in `docker-compose.yml` means environment variables will fall back to defaults if not set.

> **Important:** The values set in your `.env` file must align with the application's expectations (as documented in `configuration.md`). These will override any potentially incorrect defaults in the `docker-compose.yml`.

## Building Docker Images (Nexus Registry)

Images need to be built from the source code and pushed to your specified registry.

### Dockerfile Overview

- **Backend/Worker:** Multi-stage build using `python:3.10-slim-bullseye` base
  - Installs build tools and Poetry
  - Installs dependencies using `poetry install --no-root`
  - Copies only necessary code and dependencies to final image
  - Runs as non-root user (`intric`)

- **Frontend:** Multi-stage build using `node:22-slim` base
  - Installs `pnpm`, copies package files, installs dependencies
  - Builds UI packages and the SvelteKit app
  - Final stage uses `nginx:stable-alpine-perl`
  - Copies built assets and custom `nginx.conf`
  - Runs as non-root user (`nginx`)

### Authentication

Before pushing images, log in to your Docker registry:
```bash
# Interactive login
docker login ${NEXUS_REGISTRY}

# Or for CI/CD (using pre-defined env vars)
echo $NEXUS_PASSWORD | docker login ${NEXUS_REGISTRY} -u $NEXUS_USERNAME --password-stdin
```

### Using `build_and_push.sh` (Recommended)

The included `build_and_push.sh` script automates building and pushing the images.

1. **Ensure the script is executable:**
   ```bash
   chmod +x build_and_push.sh
   ```

2. **Set Environment Variables:**
   ```bash
   export NEXUS_REGISTRY="your.nexus.registry.com"
   # IMAGE_TAG is optional; script detects from git tag or generates one
   # export IMAGE_TAG="1.2.3"
   # NEXUS_USERNAME/PASSWORD are optional if already logged in
   # export NEXUS_USERNAME="your_username"
   # export NEXUS_PASSWORD="your_password"
   ```

3. **Run the script:**
   ```bash
   ./build_and_push.sh
   ```

The script uses Docker BuildKit, handles multi-stage builds efficiently, tags images based on your CI/CD strategy, and pushes them to the specified registry.

### CI/CD Integration Notes

Align your CI/CD pipeline with your release strategy:

| Stage | Actions | Tagging Strategy |
|-------|---------|------------------|
| **On PR** | Run tests, linters, security scans | No image pushing |
| **On Merge to `main`** | Run tests, build and push images | Commit hash or `dev-latest` |
| **On Release (Git Tag)** | Build and push versioned images | Git tag (e.g., `v1.2.3`) and possibly `latest` |

## Deployment Steps (Docker Compose)

These steps assume you have configured your `.env` file and the required images are available in your registry.

### Initial Deployment

1. **Pull Images:** Fetch the specified images from your registry:
   ```bash
   docker compose pull
   ```

2. **Initialize Database:** Run the database initialization service (first deployment only):
   ```bash
   docker compose --profile init up db-init --remove-orphans
   ```
   Wait for this command to complete successfully. Check logs with `docker compose logs db-init` if needed.

3. **Start Services:** Start the main application services:
   ```bash
   docker compose up -d
   ```

### Verification

1. **Check Container Status:** Ensure all services are running:
   ```bash
   docker compose ps
   ```
   You should see `backend`, `frontend`, `worker`, `db`, `redis` with status `running` or `healthy`.

2. **Check Logs:** Monitor logs for any errors:
   ```bash
   docker compose logs -f backend
   docker compose logs -f worker
   # Check other services as needed
   ```

3. **Check Health Check Status (Detailed):**
   ```bash
   docker inspect --format '{{json .State.Health}}' $(docker compose ps -q backend)
   # Repeat for other services (frontend, db, redis)
   ```

4. **Check Connectivity:** Access the frontend via browser at `http://<server_ip>:<FRONTEND_PORT>` or `https://${SERVICE_FQDN_FRONTEND}` if using a reverse proxy.

## Updating the Deployment

1. **Update `.env`:** Change `IMAGE_TAG` to the new version tag and modify any other variables if needed.

2. **Pull New Images:**
   ```bash
   docker compose pull
   ```

3. **Apply Database Migrations:** Before restarting the application, apply schema changes:
   ```bash
   docker compose exec backend alembic upgrade head
   ```
   Check the output for successful migration. If this fails, investigate before proceeding.

4. **Restart Services:** Recreate the services to use the new images and configuration:
   ```bash
   docker compose up -d --remove-orphans
   ```
   Monitor logs and health status after restart.

## Production Considerations

### Reverse Proxy & SSL/TLS

> **⚠️ Security Warning:** Do not expose backend (`8123`) or frontend (`3000`) ports directly.

Use a **reverse proxy** (e.g., Nginx, Traefik, Caddy) to:
- Handle **SSL/TLS termination** (HTTPS)
- Route traffic for `${SERVICE_FQDN_FRONTEND}` to the `frontend` service (port `3000`)
- Route API traffic (e.g., `/api/v1/`) to the `backend` service (port `8123`)

Ensure the `ORIGIN` env var matches your public HTTPS domain for CSRF protection.

### Persistent Volumes

The `docker-compose.yml` defines named volumes (`postgres_data`, `redis_data`, `backend_data`) using the default `local` driver.

For easier backups or use with network storage, you can map these to specific host directories:

```yaml
volumes:
  postgres_data:
    driver: local
    driver_opts:
      type: none
      device: /srv/intric/postgres_data  # Choose a secure path on host
      o: bind
  # Similar configuration for redis_data and backend_data
```

Ensure chosen host paths exist and have correct permissions.

### Database Backups

Implement **regular, automated backups** of the PostgreSQL database:

```bash
# Example backup command for cron
docker compose exec -T db pg_dump -U ${POSTGRES_USER:-postgres} -d ${POSTGRES_DB:-postgres} | gzip > /path/to/backups/intric_backup_$(date +%Y%m%d_%H%M%S).sql.gz
```

Store backups securely and test restoration periodically.

### Monitoring & Health Checks

The `docker-compose.yml` includes `healthcheck` directives for key services. Docker uses these to determine service health.

Consider integrating with external monitoring tools:
- Prometheus/Grafana for metrics
- ELK/Loki for centralized logging
- Container resource monitoring (CPU, RAM, Disk I/O)

### Resource Limits

Define CPU and memory limits in `docker-compose.yml` using `deploy.resources`:

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

Tune based on observed performance.

### Secrets Management

> **⚠️ Critical:** Do not commit `.env` files with production secrets.

Use secure methods:
- Inject secrets as environment variables from a secure CI/CD pipeline
- Use Docker secrets (`secrets:` block in compose)
- Use tools like HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault

## Troubleshooting

| Problem | Check | Solution |
|---------|-------|----------|
| **Database Connection Errors** | Password/host/port in `.env` | Ensure `db` container is running (`docker compose ps`) |
| **Authentication Errors** | `JWT_SECRET` consistency | Verify `JWT_EXPIRY_TIME`, `JWT_AUDIENCE`, `JWT_ISSUER` |
| **Worker Task Issues** | Worker & Redis logs | Check Redis connectivity and task queue |
| **Image Pull Errors** | Registry login status | Verify `NEXUS_REGISTRY`/`IMAGE_TAG` in `.env` |
| **Migration Failures** | Logs of `db-init` or migration command | May indicate DB connection issues |
| **Health Check Failures** | `docker inspect` output | Check service logs |

For detailed troubleshooting steps, refer to the [Troubleshooting Guide](./troubleshooting.md).