---
name: fullstack-monorepo-architecture
description: >-
  Design full-stack monorepo architectures with separate backend, frontend,
  shared library, database tooling, and orchestration layers. Use when
  scaffolding a new project, restructuring a monorepo, designing component
  boundaries, or deciding what each module should see and not see.
---

# Full-Stack Monorepo Architecture

## Core Principle

A monorepo groups related components under one root while keeping each component independently deployable. Each component has a **single responsibility** and **explicit boundaries** for what it can import.

## Recommended Structure

```
project-root/                    # Orchestration layer
├── backend/                     # API service (Git submodule or folder)
├── frontend/                    # Web client (Git submodule or folder)
├── common/                      # Shared library (models, functions, data)
├── database/                    # Schema migrations + tooling
├── etl/                         # Data pipelines (optional)
├── docker-compose.yaml          # Dev stack
├── docker-compose.prod.yaml     # Prod stack (images only)
├── Dockerfile.dbtools           # DB tooling image
├── pyproject.toml / package.json # Root dependency management
└── .gitmodules                  # If using submodules
```

## Component Responsibilities

| Component      | Responsibility                                              | Owns                          |
|----------------|-------------------------------------------------------------|-------------------------------|
| **Root**       | Orchestration: Docker Compose, CI/CD, shared env, deps      | Compose files, Jenkinsfile    |
| **Backend**    | REST/GraphQL API, auth, business logic                      | Routes, controllers, utils    |
| **Frontend**   | UI, user interaction, client state                          | Pages, components, stores     |
| **Common**     | Shared code: ORM models, DB sessions, utility functions     | Models, functions, static data|
| **Database**   | Schema lifecycle: migrations, seeds, sync, backup           | Alembic/migrations, scripts   |
| **ETL**        | Data pipelines: extract, transform, load                    | Step scripts, pipeline config |

## Visibility Rules (Who Sees What)

```
             Root configs
                 │
    ┌────────────┼────────────┐
    ▼            ▼            ▼
 Backend      Database       ETL
    │            │            │
    └────────────┼────────────┘
                 ▼
              Common  ◀── Single source of truth
                 │
                 ▼
            Databases (Postgres, MySQL, Oracle...)

 Frontend ──HTTP──▶ Backend API only
```

**Dependency direction**: Always inward toward `common`. Never import from `backend`, `frontend`, `database`, or `etl` into each other.

### Per-Component Access

| Component  | CAN access                                      | CANNOT access                           |
|------------|--------------------------------------------------|-----------------------------------------|
| Backend    | common.*, own code, databases, external APIs     | Frontend, ETL, database tooling         |
| Frontend   | Backend API (HTTP only), public env vars         | common.*, databases, backend code       |
| Common     | Databases, own code, filesystem (static data)    | Backend, Frontend, ETL, Database        |
| Database   | common.base, common.models, common.config        | Backend, Frontend, ETL                  |
| ETL        | common.*, databases                              | Backend, Frontend, Database tooling     |
| Root       | All build contexts (Dockerfiles)                 | Runtime code or application state       |

## Good Practices

```
# GOOD: Common is the single source of truth for ORM models
# backend/models/__init__.py
from common.models import User, Occupation, Offer  # re-export

# GOOD: Backend re-exports DB sessions from common
# backend/database/postgres.py
from common.database.postgres import get_postgres_session  # re-export

# GOOD: Frontend only talks to backend via HTTP
const response = await apiClient.get("/api/occupations");

# GOOD: Database tooling imports common for metadata
# database/alembic/env.py
from common.base import Base
import common.models  # registers tables on Base.metadata
```

## Bad Practices

```
# BAD: Backend imports from ETL
from etl.step_03 import build_vectors  # cross-component dependency

# BAD: Frontend imports Python modules
import common.models  # frontend is JS/TS, can't import Python

# BAD: Common imports from backend
from backend.controllers.auth import get_current_user  # circular dependency

# BAD: ETL imports from database tooling
from database.seeders import seed_admin  # wrong direction

# BAD: Duplicating ORM models in backend instead of using common
# backend/models/user.py
class User(Base):  # duplicate! should re-export from common
    __tablename__ = "users"
    ...
```

## Shared Data Patterns

### Static files owned by Common

```
common/
├── arrays/          # Pre-computed vectors (.npy) — git-ignored, populated by ETL
├── static/          # Lexicons, CSVs — committed or generated
└── data/            # Fixtures/samples — local only, not in production
```

**Rule**: Backend reads these at startup and caches in memory. ETL writes them during pipeline execution.

### Named volumes for container sharing

```yaml
volumes:
  shared_data:  # e.g., numpy arrays shared between ETL runs and backend

services:
  backend:
    volumes:
      - shared_data:/app/common/arrays
```

## Git Submodules vs Monorepo Folders

| Approach        | Use when                                    | Trade-off                        |
|-----------------|---------------------------------------------|----------------------------------|
| **Submodules**  | Teams own separate repos, independent CI     | Complex git workflow, sync issues|
| **Folders**     | Single team, unified CI, shared deps         | Simpler git, coupled releases    |

If using submodules, the root repo holds `.gitmodules` and integration files (Compose, CI, workspace config). Each submodule has its own git history and can be versioned independently.

## Dependency Management

- **Python monorepo**: Single `pyproject.toml` + lockfile at root; all components share the same venv.
- **Node frontend**: Own `package.json` inside `frontend/`; independent from Python deps.
- **DB tooling**: Minimal deps (only ORM + migration lib); separate Dockerfile with pinned versions.

## Checklist for New Projects

- [ ] Define component boundaries before writing code
- [ ] Create `common/` with shared `Base`, models, config, and DB sessions
- [ ] Backend and ETL import from `common`, never from each other
- [ ] Frontend communicates via HTTP only — no shared code with backend
- [ ] Root owns Docker Compose and CI/CD — no application logic
- [ ] Database tooling imports `common.base` + `common.models` for migration metadata
- [ ] One `.env` template at root; components load what they need
- [ ] Named volumes for shared data between containers
