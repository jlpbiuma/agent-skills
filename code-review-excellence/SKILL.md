---
name: code-review-excellence
description: Master effective code review practices to provide constructive feedback, catch bugs early, and strictly enforce the AI-Commerce v2.0 Architecture Decision Record. Use when reviewing PRs or validating generated code.
---

# Code Review Excellence (AI-Commerce specific)

Transform code reviews into an automated guardrail that ensures all code merged to the `amazon-ai` monorepo complies with our strict architectural boundaries.

## When to Use This Skill

- Reviewing pull requests against the backend or frontend
- Checking if a newly generated agent script violates architectural boundaries
- Validating the dependency flow

## Core AI-Commerce Review Checklist

When reviewing any code for this project, you must enforce the following rules with a **Blocking (🔴)** severity if broken:

### Backend Architectural Guardrails

🔴 **[blocking] Controller DB Access**
Does a controller or router import `SQLAlchemy`, `select`, or execute queries? 
*Action*: Reject. All DB interaction must be moved to `repositories/`.

🔴 **[blocking] Impure Entities**
Does a file in `backend/entities/` import `Session` or make HTTP requests?
*Action*: Reject. Entities must be pure Python business logic in memory.

🔴 **[blocking] Fat Routers**
Does a file in `routers/` contain `if/else` business logic or data transformations?
*Action*: Reject. Move logic to `controllers/`. Routers exist only for mapping endpoints to controllers.

🔴 **[blocking] Agent Circular HTTP Calls**
Does a LangGraph `@tool` or node make a `requests.get` call to our own Backend API?
*Action*: Reject. Tools must invoke `repositories/` or `domain/` directly. They are a parallel entry point to Controllers.

🔴 **[blocking] Common Package Contamination**
Does a file in `common/models/` import `FastAPI`, `Pydantic`, or contain business methods?
*Action*: Reject. `common/models/` is the single source of truth for the generic SQL schema (columns only). Methods go to `backend/entities/`.

### Frontend Architectural Guardrails

🔴 **[blocking] Direct Axios in Components**
Does a React Component or Next.js Page import `axios` directly or make `fetch` calls to raw URIs?
*Action*: Reject. All network calls must go through the proxy layer in `frontend/services/`.

🔴 **[blocking] Untyped Responses**
Does a frontend service return `any` or fail to mirror a Pydantic schema from the backend?
*Action*: Reject. Define proper TS interfaces in `frontend/types/`.

## Giving Constructive Feedback

Use collaborative language while firmly enforcing the architecture.

```markdown
❌ Bad: "You broke the architecture. Move this SQL to a repository."

✅ Good: "🔴 [blocking] I see we have some database queries directly inside the controller. According to the AI-Commerce v2.0 ADR, controllers should be thin. Could we extract this `select()` logic into a corresponding class in `repositories/` to maintain our pure dependency flow?"
```

## Severity Labels

- 🔴 **[blocking]** - Must fix before merge (Always use this for architectural violations).
- 🟡 **[important]** - Should fix, discuss if disagree (Edge case logic, missing tests).
- 🟢 **[nit]** - Nice to have, not blocking (Variable naming, minor cleanup).
