# AI-Commerce Agent Skills

This repository contains custom agent skills tailored for the AI-Commerce Platform v2.0 architecture. These skills enforce strict architectural boundaries, repository patterns, and clean code principles.

## Available Skills

### Backend & Architecture
- **`architecture-patterns`**: Enforces the 7 Core Pillars of the AI-Commerce v2.0 Backend Architecture (Strict decoupled layers, Repository Pattern, Pure Entities, Thin Controllers).
- **`code-review-excellence`**: Automated code review checklist for PRs to strictly enforce architectural boundaries (e.g. Reject direct DB access in controllers).
- **`python-fastapi-development`**: Python FastAPI backend development workflow implementing async patterns, SQLAlchemy 2.0, and the Repository Pattern (no Prisma).
- **`pydantic`**: Pydantic v2 validation specifically configured to integrate with the Project's Repository Pattern and domain layers rather than direct database manipulation.

### Frontend
- **`nextjs-app-router-patterns`**: Master Next.js 14+ App Router patterns ensuring all frontend code fetches data exclusively via the `frontend/services/` layer and not through direct database connections or raw fetch logic.

## Installation

Install any of these skills into your local `.agents` directory using the `skills` CLI:

```bash
npx skills add jlpbiuma/agent-skills@<skill-name> -y
```

Example:
```bash
npx skills add jlpbiuma/agent-skills@architecture-patterns -y
```
