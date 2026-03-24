---
name: docker-multi-environment
description: >-
  Design Docker Compose setups with separate files for development, production,
  and CI environments. Covers Dockerfiles (multi-stage), service wiring,
  healthchecks, volumes, networks, and env file strategies. Use when creating
  Docker infrastructure, writing Dockerfiles, setting up compose for multiple
  environments, or debugging container orchestration.
---

# Docker Compose Multi-Environment

## Architecture: Three Compose Files

```
project-root/
├── docker-compose.yaml          # Dev: builds from source, all services
├── docker-compose.prod.yaml     # Prod: pre-built images only, fixed names
├── docker-compose.ci.yaml       # CI: override merged with dev compose
├── Dockerfile.dbtools           # Lightweight DB tooling image
├── backend/Dockerfile           # Multi-stage backend
├── frontend/Dockerfile          # Multi-stage frontend
├── postgres/Dockerfile          # Custom DB image (extensions)
└── env.docker.example           # Template for all env vars
```

## File Responsibilities

| File                       | Builds images? | Published ports? | Purpose                              |
|----------------------------|:--------------:|:----------------:|---------------------------------------|
| `docker-compose.yaml`      | Yes            | Yes              | Local dev with full build from source |
| `docker-compose.prod.yaml` | No             | Yes              | Production with pre-built images      |
| `docker-compose.ci.yaml`   | Overrides only | No               | Headless CI testing (merge with dev)  |

### Usage Commands

```bash
# Development
docker compose up -d

# Production (on server)
docker compose -f docker-compose.prod.yaml up -d

# CI (merged override)
docker compose -f docker-compose.yaml -f docker-compose.ci.yaml up -d postgres backend

# DB tooling (on-demand via profile)
docker compose --profile tools run --rm dbtools migrate
```

## Dev Compose Pattern

```yaml
services:
  database:
    build: ./postgres
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    ports:
      - "${DB_PORT}:5432"
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  backend:
    build:
      context: .
      dockerfile: backend/Dockerfile
      args:
        ENV_FILE: .env.production
    env_file: .env.production
    environment:
      - DB_HOST=database          # Docker service name, not localhost
      - DB_PORT=5432              # Internal port, not mapped port
    ports:
      - "9015:9015"
    volumes:
      - shared_data:/app/shared
    depends_on:
      database:
        condition: service_healthy  # Wait for healthcheck, not just start
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9015/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  frontend:
    build:
      context: .
      dockerfile: frontend/Dockerfile
      args:
        ENV_FILE: .env.production
    ports:
      - "3001:3000"
    depends_on:
      backend:
        condition: service_healthy

  dbtools:
    build:
      context: .
      dockerfile: Dockerfile.dbtools
    depends_on:
      database:
        condition: service_healthy
    env_file: .env
    environment:
      - DB_HOST=database
    profiles:
      - tools                     # Only starts with --profile tools

volumes:
  db_data:
  shared_data:

networks:
  default:
    driver: bridge
```

## Prod Compose Pattern

Key differences from dev:

```yaml
services:
  database:
    image: registry.example.com/myapp-postgres:latest   # No build
    container_name: myapp-postgres                       # Fixed name

  backend:
    image: registry.example.com/myapp-backend:latest    # No build
    container_name: myapp-backend
    environment:
      - DB_HOST=myapp-postgres    # Use container_name, not service name
      - IS_PRODUCTION=true
    depends_on:
      database:
        condition: service_healthy

  dbtools:
    image: registry.example.com/myapp-dbtools:latest
    container_name: myapp-dbtools
    entrypoint: ["/app/entrypoint.sh"]   # Explicit (no build context)
    profiles:
      - tools
```

## CI Override Pattern

Minimal file, merged with dev compose:

```yaml
services:
  backend:
    build:
      args:
        ENV_FILE: .env.ci           # CI-specific env
    env_file: .env.ci
    ports: []                        # No published ports in CI
    healthcheck:
      start_period: 90s             # More tolerant for slow CI agents
      retries: 6
```

## Multi-Stage Dockerfile Patterns

### Backend (Python)

```dockerfile
# Stage 1: Builder
FROM python:3.11-slim AS builder
WORKDIR /install
COPY pyproject.toml lockfile* ./
RUN pip install --no-cache-dir -r requirements.txt
# Or: uv sync --frozen --no-dev

# Stage 2: Runner
FROM python:3.11-slim AS runner
ARG ENV_FILE=.env.production

WORKDIR /app
RUN useradd --create-home app

COPY --from=builder /install/.venv /app/.venv
COPY backend/ ./backend/
COPY common/ ./common/
COPY backend/${ENV_FILE} ./backend/.env.production

USER app
EXPOSE 9015
CMD ["python", "-m", "backend.app"]
```

### Frontend (Node.js)

```dockerfile
# Stage 1: Builder
FROM node:20-alpine AS builder
ARG ENV_FILE=.env.production
WORKDIR /app
COPY frontend/package*.json ./
RUN npm ci
COPY frontend/ ./
COPY ${ENV_FILE} ./.env.production    # NEXT_PUBLIC_* baked at build time
RUN npm run test -- --ci --passWithNoTests
RUN npm run build

# Stage 2: Runner
FROM node:20-alpine AS runner
ENV NODE_ENV=production
WORKDIR /app
RUN adduser --system --uid 1001 appuser
COPY --from=builder /app/package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
USER appuser
EXPOSE 3000
CMD ["npm", "start"]
```

### DB Tooling (Minimal)

```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN pip install alembic sqlalchemy psycopg2-binary python-dotenv pytest
COPY common/ ./common/          # Only what migrations need
COPY database/ ./database/
COPY database/entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh
WORKDIR /app/database
ENTRYPOINT ["/app/entrypoint.sh"]
CMD ["migrate"]
```

### Entrypoint Router

```bash
#!/bin/sh
set -e
case "$1" in
  migrate)       cd /app/database && exec alembic upgrade head ;;
  seed)          shift; cd /app && exec python database/run_seeders.py "$@" ;;
  test)          cd /app && exec pytest common/test/ -v --tb=short ;;
  migrate-and-seed) cd /app/database && alembic upgrade head && cd /app && exec python database/run_seeders.py "$@" ;;
  sync)          shift; cd /app && exec python database/sync.py "$@" ;;
  backup)        shift; cd /app && exec python database/backup.py "$@" ;;
  *)             exec "$@" ;;
esac
```

## Good Practices

```yaml
# GOOD: Use service_healthy, not just service_started
depends_on:
  database:
    condition: service_healthy

# GOOD: ENV_FILE as build arg to select env at build time
build:
  args:
    ENV_FILE: .env.production

# GOOD: Profiles for on-demand services
profiles:
  - tools  # docker compose --profile tools run --rm dbtools migrate

# GOOD: Named volumes for persistent data
volumes:
  db_data:     # Survives container recreation
  shared_data: # Shared between services

# GOOD: Non-root user in Dockerfiles
RUN useradd --create-home app
USER app

# GOOD: Healthchecks with start_period
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:9015/health"]
  start_period: 40s  # Grace period before first check
```

## Bad Practices

```yaml
# BAD: No healthcheck — dependent services may start before DB is ready
depends_on:
  - database  # Only waits for container start, not readiness

# BAD: Hardcoded credentials in compose
environment:
  POSTGRES_PASSWORD: mysecretpassword  # Use ${VAR} from .env

# BAD: Using 'latest' tag without digest verification in prod
image: myapp:latest  # Could be a different image than tested

# BAD: Bind-mounting source code in production
volumes:
  - ./backend:/app/backend  # Only for dev hot-reload, never prod

# BAD: Running as root in containers
# (no USER directive in Dockerfile)

# BAD: Same compose file for dev and prod
# Leads to: build directives in prod, missing healthchecks, wrong env files

# BAD: Publishing DB ports in production
ports:
  - "5432:5432"  # Exposes DB to host network — use internal network only
```

## Startup Order Diagram

```
1. database       healthcheck: pg_isready
        │
2. backend        depends_on: database (healthy)
        │          healthcheck: curl /health
3. frontend       depends_on: backend (healthy)
        │          healthcheck: HTTP GET /
4. dbtools        depends_on: database (healthy)
                   profiles: [tools] — on-demand only
```

## Checklist

- [ ] Three compose files: dev (build), prod (images), CI (override)
- [ ] Every service has a healthcheck
- [ ] `depends_on` uses `condition: service_healthy`
- [ ] `ENV_FILE` build arg controls which env gets baked into images
- [ ] Non-root users in all Dockerfiles
- [ ] Multi-stage builds to minimize image size
- [ ] DB tooling behind `profiles: [tools]`
- [ ] Named volumes for persistent data; no bind mounts in prod
- [ ] Prod compose uses fixed container names for stable hostnames
- [ ] `.env.example` template documents all required variables
