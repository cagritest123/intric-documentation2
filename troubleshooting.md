# Intric Troubleshooting Guide

## TLDR

> **Quick Reference**
> - **Check Logs First**: Use `docker compose logs <service_name>` to identify specific error messages
> - **Verify Configuration**: Double-check your `.env` file against `.env.example`. Use `docker compose config` to see the final configuration
> - **Check Container Status**: Use `docker compose ps` to see if all services are running and healthy
> - **Database Issues**: Verify DB container health, check credentials, test connection with `docker compose exec db pg_isready -U ${POSTGRES_USER:-postgres}`
> - **Network Problems**: Ensure containers are on the same network and can resolve service names

This guide provides solutions for common issues encountered when deploying, developing, or using the Intric platform.

## Table of Contents
- [Deployment & Startup Issues](#deployment--startup-issues)
- [Docker Environment Issues](#docker-environment-issues)
- [Database & Migration Issues](#database--migration-issues)
- [Network & Connectivity Issues](#network--connectivity-issues)
- [Authentication & Authorization Issues](#authentication--authorization-issues)
- [LLM Integration Issues](#llm-integration-issues)
- [Frontend (UI) Issues](#frontend-ui-issues)
- [Backend API Issues](#backend-api-issues)
- [Worker Task Issues](#worker-task-issues)
- [Performance Issues](#performance-issues)
- [Common Error Messages](#common-error-messages)
- [Getting Further Help](#getting-further-help)

## Deployment & Startup Issues

### Container(s) Fail to Start or Exit Immediately

**Symptoms:**
- Services show as "Exited" or are in a restart loop in `docker compose ps`
- Deployment hangs or fails

**Diagnosis & Solutions:**

1.  **Check Logs**: This is the most crucial step. Look for specific error messages:
    ```bash
    docker compose logs <failed_service_name>
    # Example: docker compose logs backend
    ```

2.  **Verify `.env` Configuration**: Ensure all required variables are present and correct:
    ```bash
    # See the interpolated configuration Docker Compose is using
    docker compose config
    ```

3.  **Check Database Initialization**:
    ```bash
    # Check db-init logs if it ran
    docker compose logs db-init
    
    # Re-run if necessary (only safe on first setup or if DB is empty)
    docker compose --profile init up db-init --remove-orphans
    ```

4.  **Check Port Conflicts**:
    ```bash
    sudo lsof -i :${FRONTEND_PORT:-3000}
    sudo lsof -i :${BACKEND_PORT:-8123}
    # Change ports in .env or docker-compose.yml if needed
    ```

5.  **Check Disk Space**: Ensure sufficient disk space is available:
    ```bash
    df -h
    ```

6.  **Check Resource Limits**: If you've set resource limits, ensure they are sufficient:
    ```bash
    docker stats
    ```

### Image Pull Failures

**Symptoms:**
- `docker compose pull` or `docker compose up` fails with "Error pulling image", "manifest not found", or authentication errors

**Diagnosis & Solutions:**

1.  **Verify Registry Credentials**:
    ```bash
    docker login ${NEXUS_REGISTRY}
    ```

2.  **Verify Image Tag**: Check that the `IMAGE_TAG` specified in your `.env` file exists in the registry

3.  **Check Registry URL**: Ensure `NEXUS_REGISTRY` in `.env` is correct

4.  **Network Connectivity**: Verify the host machine can reach the registry URL

## Docker Environment Issues

### Volume Permission Issues

**Symptoms:**
- "Permission denied" errors in logs when services try to write data
- Services failing to start due to inability to access volume mounts

**Diagnosis & Solutions:**

1.  **Check Host Mount Permissions (If Used)**:
    ```bash
    # If using host mounts, ensure correct permissions
    sudo chown -R <container_uid>:<container_gid> /path/on/host
    ```

2.  **Check Default Volume Permissions**: Less common with named volumes, but check Docker daemon logs

3.  **Backend Data Volume**: The `backend_data` volume needs to be writable by the `intric` user

### Resource Constraints (CPU/Memory)

**Symptoms:**
- Services run very slowly, become unresponsive, or crash (`killed`)
- "Out of Memory" (OOM) errors in container logs or Docker events

**Diagnosis & Solutions:**

1.  **Monitor Usage**: Use `docker stats` to see real-time CPU/Memory usage per container

2.  **Increase Docker Resources**: If using Docker Desktop, increase the resources in settings

3.  **Adjust Compose Limits**: If you set `deploy.resources.limits`, consider increasing `memory` or `cpus`

4.  **Profile Application**: If a specific service consistently uses high resources, profile the application code

## Database & Migration Issues

### Connection Failures (Backend/Worker to DB)

**Symptoms:**
- "Could not connect to database", "Connection refused", "Authentication failed" errors in logs
- Services depending on `db` fail their health checks or restart

**Diagnosis & Solutions:**

1.  **Check DB Service Status**:
    ```bash
    docker compose ps db
    docker inspect --format '{{json .State.Health}}' $(docker compose ps -q db)
    ```

2.  **Check DB Logs**:
    ```bash
    docker compose logs db
    ```

3.  **Verify Credentials**: Double-check `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` in your `.env` file

4.  **Test Connectivity from Backend**:
    ```bash
    # Test readiness check used by healthcheck
    docker compose exec db pg_isready -U ${POSTGRES_USER:-postgres} -h db -p 5432
    ```

5.  **Check Network**: Ensure `backend` and `db` are on the same `intric-network`

### Migration Errors (Alembic)

**Symptoms:**
- `db-init` service fails during initial deployment
- `docker compose exec backend alembic upgrade head` fails during an update
- Errors like "Relation does not exist", "Column does not exist"

**Diagnosis & Solutions:**

1.  **Check Logs**: Examine the logs of the failing command

2.  **Check Current Revision**:
    ```bash
    docker compose exec backend alembic current
    ```

3.  **Check History**:
    ```bash
    docker compose exec backend alembic history --verbose
    ```

4.  **Manual Intervention** (Use with Caution): If migrations are stuck, you might need to manually inspect the `alembic_version` table

5.  **Reset Database** (Development ONLY - DELETES ALL DATA!):
    ```bash
    docker compose down -v # Removes containers AND volumes
    docker compose up -d db # Bring DB back up
    # Re-run initialization
    docker compose --profile init up db-init --remove-orphans
    docker compose up -d # Start other services
    ```

### pgvector Issues

**Symptoms:**
- Vector similarity searches fail
- Errors mentioning "vector" type or related functions

**Diagnosis & Solutions:**

1.  **Verify Extension**:
    ```bash
    docker compose exec db psql -U ${POSTGRES_USER:-postgres} -d ${POSTGRES_DB:-postgres} -c "\dx vector"
    ```

2.  **Check Postgres/pgvector Version**: Ensure compatibility with operations used

3.  **Check Embedding Dimensions**: Ensure the vector dimension matches the table column definition

## Network & Connectivity Issues

### Internal Service Communication

**Symptoms:**
- "Connection refused", "Could not resolve host" errors in service logs
- Frontend shows errors trying to fetch API data

**Diagnosis & Solutions:**

1.  **Check Network**:
    ```bash
    docker network inspect $(docker network ls -qf name=intric-network)
    # Check the "Containers" section lists all expected services
    ```

2.  **Verify Service Names**: Containers should use service names as hostnames

3.  **Test Connectivity**:
    ```bash
    docker compose exec backend ping redis # Should resolve and ping
    docker compose exec backend curl -I http://frontend:3000
    ```

4.  **Check `depends_on`**: Ensure `depends_on` conditions are met; check health status of dependencies

### External Access to Frontend

**Symptoms:**
- Cannot access the application via browser using the host's IP or domain name
- Connection timed out or refused

**Diagnosis & Solutions:**

1.  **Check Port Mapping**: Verify the `frontend` service's port mapping in `docker compose ps`

2.  **Check Host Firewall**: Ensure the host machine's firewall allows incoming traffic on the mapped port

3.  **Check Reverse Proxy (If Used)**: If using Nginx, Traefik, etc., ensure it's configured correctly

4.  **Check Frontend Logs**: `docker compose logs frontend`

5.  **Check `SERVICE_FQDN_FRONTEND`**: Ensure this is correctly set in `.env` if required

## Authentication & Authorization Issues

### JWT Token Problems

**Symptoms:**
- "Invalid token", "Signature verification failed", "Token expired" errors in logs or API responses
- User is frequently logged out

**Diagnosis & Solutions:**

1.  **Verify `JWT_SECRET`**: Ensure the `JWT_SECRET` in your `.env` file is correct and hasn't changed

2.  **Check Expiry (`JWT_EXPIRY_TIME`)**: Ensure this value (in minutes) in `.env` is appropriate

3.  **Clock Skew**: Ensure the server running the backend has accurate system time (NTP sync recommended)

4.  **Check Audience/Issuer**: If you changed `JWT_AUDIENCE` or `JWT_ISSUER` defaults, ensure tokens match

5.  **Decode Token**: Use a tool like `jwt.io` to decode a sample token and verify its payload

### Login Failures

**Symptoms:**
- Users cannot log in via UI or API
- "AuthenticationException", "Wrong password", "No such user" errors

**Diagnosis & Solutions:**

1.  **Check Credentials**: For local dev, verify use of default `user@example.com` / `Password1!`

2.  **Check Backend Logs**:
    ```bash
    docker compose logs backend | grep -iE 'auth|login|user'
    ```

3.  **Verify User Exists/Active**: Check the database to ensure the user exists and is `ACTIVE`

4.  **Test API Directly**: Use `curl` or Postman to test the login endpoint

5.  **OIDC Issues (MobilityGuard)**: If using OIDC, verify all `MOBILITYGUARD_*` variables

### API Key Failures

**Symptoms:**
- Programmatic access using an API key fails with 401/403 errors

**Diagnosis & Solutions:**

1.  **Check Key Value**: Ensure the correct API key is being used

2.  **Check Header Name**: Verify the client is sending the key in the correct HTTP header

3.  **Check Backend Logs**: Look for API key validation errors

4.  **Verify Key Exists**: Check the `api_keys` table in the database

## LLM Integration Issues

### API Key / Endpoint Problems

**Symptoms:**
- Errors related to OpenAI, Anthropic, Azure, or other LLM providers
- LLM-dependent features (chat, summarization) fail

**Diagnosis & Solutions:**

1.  **Verify `.env` Variables**: Double-check the relevant API keys and endpoints

2.  **Check Key Validity**: Confirm the API keys are active and valid

3.  **Network Connectivity**: Ensure containers can reach the LLM provider's API endpoints:
    ```bash
    docker compose exec backend curl https://api.openai.com/v1/models
    ```

4.  **Provider Status**: Check the status pages for the LLM providers being used

### Timeouts or Slow Responses

**Symptoms:**
- Chat responses are very slow or fail with timeout errors

**Diagnosis & Solutions:**

1.  **Check LLM Provider Status**: Provider APIs might be slow

2.  **Network Latency**: Check network latency from your server to the LLM provider

3.  **Prompt Complexity**: Very long prompts can take longer. Try simplifying

4.  **Model Choice**: Some LLMs are inherently slower than others

5.  **Application Timeouts**: Consider increasing timeout settings if appropriate

## Frontend (UI) Issues

### Components Not Rendering / Visual Glitches

**Symptoms:**
- Pages are blank, components are missing, or styling looks broken

**Diagnosis & Solutions:**

1.  **Browser Console**: Check the browser's developer console (F12) for errors

2.  **Clear Cache**: Force-reload the page (Ctrl+Shift+R / Cmd+Shift+R)

3.  **Network Tab**: Check the browser's Network tab for asset loading success

4.  **Build Issues**: Ensure the frontend build process completed without errors

### UI Not Updating / State Issues

**Symptoms:**
- Actions performed don't reflect in the UI
- Stale data is displayed

**Diagnosis & Solutions:**

1.  **Browser Console**: Look for Javascript errors related to state management

2.  **Network Tab**: Verify that API calls completed successfully

3.  **Backend Logs**: Check for errors corresponding to failed API calls

4.  **WebSockets (If Used)**: Check WebSocket connection status if applicable

### CORS Errors

**Symptoms:**
- Browser console shows errors like "Access blocked by CORS policy..."

**Diagnosis & Solutions:**

1.  **Check Backend Config**: The default CORS middleware setup is permissive (`allow_origins=["*"]`)

2.  **Check Reverse Proxy**: Ensure it's forwarding the `Origin` header correctly

3.  **Check `ORIGIN` Env Var**: Ensure it matches your public URL for CSRF protection

## Backend API Issues

### Specific Endpoint Failures (4xx/5xx Errors)

**Symptoms:**
- API requests return 404, 422, 500 errors

**Diagnosis & Solutions:**

1.  **Check Backend Logs**: `docker compose logs backend`

2.  **Verify Request**: Ensure the API request details (URL, method, headers, body) are correct

3.  **Check Input Validation**: 422 errors usually mean request body failed Pydantic validation

4.  **Check Service Logic**: 500 errors indicate an unhandled exception in backend code

## Worker Task Issues

### Tasks Not Being Processed

**Symptoms:**
- Background jobs (document uploads, crawls) seem stuck or never complete
- Expected results don't appear in the database or UI

**Diagnosis & Solutions:**

1.  **Check Worker Logs**: `docker compose logs worker`

2.  **Verify Worker is Running**: `docker compose ps worker`

3.  **Check Redis Connection**:
    ```bash
    docker compose exec worker redis-cli -h redis -p ${REDIS_PORT:-6379} ping
    # Should return PONG
    ```

4.  **Check ARQ Queue**:
    ```bash
    docker compose exec redis redis-cli -h redis -p ${REDIS_PORT:-6379} LLEN arq:queue
    ```

5.  **Check Task Definitions**: Ensure the `WorkerSettings` in `arq.py` correctly lists all task functions

## Performance Issues

### Slow API Responses / Database Queries

**Symptoms:**
- API requests take a long time to complete
- High load on the `db` container

**Diagnosis & Solutions:**

1.  **Check Backend Logs**: Look for slow query warnings

2.  **Database Indexing**: Ensure appropriate indexes exist, especially for frequently queried columns

3.  **N+1 Queries**: Check for loops that issue database queries inside the loop

4.  **Resource Limits**: Ensure sufficient CPU/Memory for your containers

5.  **Connection Pool**: Monitor database connection pool usage under high load

### High CPU/Memory Usage

**Symptoms:**
- One or more containers consistently use high CPU or Memory
- General sluggishness or unresponsiveness

**Diagnosis & Solutions:**

1.  **Identify Container**: Use `docker stats` to identify the high-usage container

2.  **Check Logs**: Look for errors or operations correlating with high usage

3.  **Code Profiling**: Profile the application code to pinpoint expensive functions

4.  **Optimize Queries/Logic**: Address inefficient database queries or application logic

5.  **Resource Allocation**: Increase resources allocated to the container(s) if necessary

6.  **Scaling**: Consider scaling horizontally if possible (e.g., more `worker` replicas)

## Common Error Messages

*(Refer also to specific sections above)*

- **"Connection refused"**: Target service not running, not accessible on the network, listening on wrong port/interface, or blocked by firewall.
- **"Permission denied"**: Filesystem permissions issue (often with volumes) or insufficient rights for the action being performed.
- **"No such container"**: Typo in service name, or container failed to start/was removed. Check `docker compose ps`.
- **"Out of memory" / "Killed"**: Container exceeded its memory limit. Increase limit or investigate memory leak/usage.
- **"Authentication required" / 401 Unauthorized**: Missing, invalid, or expired JWT token or API key. Check credentials and `JWT_SECRET`.
- **403 Forbidden**: Authentication succeeded, but the user lacks permission for the specific action (RBAC).
- **422 Unprocessable Entity**: API request body failed validation against the expected Pydantic model. Check API response details for specific field errors.
- **500 Internal Server Error**: Unhandled exception in the backend code. Check backend logs for traceback.
- **502 Bad Gateway / 504 Gateway Timeout**: Often from a reverse proxy unable to reach the backend, or the backend taking too long to respond. Check backend status and logs.

## Getting Further Help

If you're stuck after reviewing this guide:

1.  **Check GitHub Issues** (if applicable) for similar reported problems.

2.  **Consult `configuration.md` and `deployment-guide.md`** again carefully.

3.  **Gather Information**: Prepare detailed logs, error messages, and configuration:
    ```bash
    # Example: Save logs
    docker compose logs > intric-logs-$(date +%Y%m%d).txt
    
    # Get Docker/Compose versions
    docker version
    docker compose version
    ```

4.  **Reach Out**: Contact the development team or community support:
    - Email: [digitalisering@sundsvall.se](mailto:digitalisering@sundsvall.se)