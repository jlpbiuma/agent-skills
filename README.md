# Agent Skills Collection

Reusable agent skills for AI-assisted development. Each skill provides structured guidance, code examples, good/bad practices, and checklists for a specific domain.

## Skills Overview

| Skill | Category | Description |
|-------|----------|-------------|
| [architecture-patterns](architecture-patterns/) | Backend | Layered architecture with Repository Pattern, Pure Entities, Thin Controllers |
| [code-review-excellence](code-review-excellence/) | Process | Systematic code review with severity labels and architectural guardrails |
| [python-fastapi-development](python-fastapi-development/) | Backend | Full FastAPI development workflow: setup, DB, auth, testing, deployment |
| [pydantic](pydantic/) | Backend | Pydantic v2 validation, serialization, settings, and ORM integration |
| [nextjs-app-router-patterns](nextjs-app-router-patterns/) | Frontend | Next.js 14+ App Router: Server/Client Components, streaming, service layer |
| [fullstack-monorepo-architecture](fullstack-monorepo-architecture/) | Architecture | Monorepo structure for backend + frontend + shared library + DB tooling |
| [docker-multi-environment](docker-multi-environment/) | DevOps | Docker Compose patterns for dev, prod, and CI environments |
| [fastapi-layered-architecture](fastapi-layered-architecture/) | Backend | Routes, Controllers, Repositories, Utils, Validation layer separation |
| [database-migration-tooling](database-migration-tooling/) | Database | Alembic migrations, idempotent seeders, sync, backup, schema parity tests |
| [vscode-debug-fullstack](vscode-debug-fullstack/) | Tooling | VS Code launch.json for debugging Python and Node.js multi-service stacks |
| [cicd-pipeline-design](cicd-pipeline-design/) | DevOps | CI/CD pipeline stages, Jenkinsfile patterns, credential management |

## Categories

### Backend Architecture

- **architecture-patterns** — Core architectural rules: Repository Pattern isolates all DB access, Entities contain pure business logic, Controllers orchestrate without touching the database, Routes only declare endpoints. Includes dependency flow tables and violation flags.

- **fastapi-layered-architecture** — Detailed implementation of a layered FastAPI backend: folder structure, example code for each layer (routes, controllers, repositories, utils, validation, models), static data loading, and FastAPI dependencies.

- **python-fastapi-development** — End-to-end workflow for building a FastAPI API: project setup with `uv`, SQLAlchemy 2.0 models, Alembic migrations, JWT authentication, custom error handling, testing with pytest, and Pydantic settings management.

- **pydantic** — Pydantic v2 essentials: BaseModel, Field constraints, validators (`field_validator`, `model_validator`), serialization, settings management with `BaseSettings`, FastAPI/SQLAlchemy integration, CRUD schema variants, and v1-to-v2 migration reference. An extended `reference.md` covers advanced topics like custom types, generics, strict mode, and property-based testing.

### Frontend

- **nextjs-app-router-patterns** — Next.js 14+ App Router patterns enforcing a centralized service layer for all data fetching. Covers Server Components, Client Components, Server Actions, Parallel Routes, Streaming with Suspense, Metadata/SEO, and caching strategies with TanStack Query.

### Full-Stack & Infrastructure

- **fullstack-monorepo-architecture** — How to structure a monorepo with backend, frontend, shared library (`common/`), database tooling, and root-level orchestration. Defines component responsibilities, visibility rules (what each component can and cannot see), and shared data patterns.

- **docker-multi-environment** — Docker Compose architecture for development, production, and CI. Multi-stage Dockerfiles, entrypoint routers, named volumes, bridge networks, healthchecks, and service dependency ordering.

- **cicd-pipeline-design** — Generic CI/CD pipeline design: stage ordering (validate, DB prep, data pipeline, build, deploy), Jenkinsfile patterns, credential management, image digest verification, and environment promotion.

### Database & Migrations

- **database-migration-tooling** — Database lifecycle management: Alembic migration setup and `env.py` configuration, idempotent seeders with `INSERT ... ON CONFLICT`, data synchronization with `COPY CSV`, `pg_dump` backups, schema parity tests, and a container entrypoint router for running DB operations in Docker.

### Development Tooling

- **vscode-debug-fullstack** — VS Code `launch.json` configurations for debugging multi-service applications: Python debugger for FastAPI/Uvicorn, Node.js debugger for Next.js, compound launchers for full-stack debugging, and `settings.json` for interpreter paths and test runners.

### Process & Quality

- **code-review-excellence** — Structured code review methodology with three severity labels (`[blocking]`, `[important]`, `[nit]`). Backend guardrails (controller DB access, impure entities, fat routers), frontend guardrails (direct HTTP in components, untyped responses), and collaborative feedback templates.

## Installation

Skills from the GitHub repository can be installed with:

```bash
npx skills add jlpbiuma/agent-skills@architecture-patterns -y
npx skills add jlpbiuma/agent-skills@code-review-excellence -y
npx skills add jlpbiuma/agent-skills@python-fastapi-development -y
npx skills add jlpbiuma/agent-skills@pydantic -y
npx skills add jlpbiuma/agent-skills@nextjs-app-router-patterns -y
npx skills add jlpbiuma/agent-skills@fullstack-monorepo-architecture -y
npx skills add jlpbiuma/agent-skills@docker-multi-environment -y
npx skills add jlpbiuma/agent-skills@fastapi-layered-architecture -y
npx skills add jlpbiuma/agent-skills@database-migration-tooling -y
npx skills add jlpbiuma/agent-skills@vscode-debug-fullstack -y
npx skills add jlpbiuma/agent-skills@cicd-pipeline-design -y
```

## Design Principles

All skills follow these guidelines:

1. **Generalist** — No project-specific references. Examples use generic domains (users, products, orders).
2. **Actionable** — Every skill includes code examples for good and bad practices.
3. **Concise** — SKILL.md stays under 500 lines. Extended reference goes in `reference.md`.
4. **Checklistable** — Each skill ends with a verification checklist.
5. **Composable** — Skills complement each other without duplication. Use `architecture-patterns` for rules, `fastapi-layered-architecture` for implementation, and `code-review-excellence` for enforcement.
