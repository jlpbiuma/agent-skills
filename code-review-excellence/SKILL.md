---
name: code-review-excellence
description: >-
  Systematic code review with severity labels and architectural guardrails for
  layered backends and frontend service patterns. Use when reviewing pull
  requests, validating generated code, or checking dependency boundaries.
---

# Code Review Excellence

Structured code review that enforces clean architecture boundaries with clear severity labels.

## When to Use This Skill

- Reviewing pull requests
- Validating AI-generated code against architectural rules
- Checking dependency flow between layers
- Providing constructive, actionable feedback

## Severity Labels

| Label | Meaning | Action |
|-------|---------|--------|
| **[blocking]** | Architectural violation or bug | Must fix before merge |
| **[important]** | Missing tests, edge cases, performance | Should fix, discuss if disagree |
| **[nit]** | Naming, formatting, minor cleanup | Nice to have, not blocking |

## Backend Guardrails

### [blocking] Controller DB Access

Does a controller or router import `Session`, `select`, or execute queries directly?

**Action**: Reject. All DB interaction must go through `repositories/`.

```python
# VIOLATION
def get_users(db: Session = Depends(get_session)):
    return db.query(User).all()

# CORRECT
def get_users(db: Session = Depends(get_session)):
    repo = UserRepository(db)
    return repo.get_all()
```

### [blocking] Impure Entities

Does a file in `entities/` import `Session`, make HTTP requests, or access external services?

**Action**: Reject. Entities must be pure Python business logic operating in memory.

### [blocking] Fat Routers

Does a file in `routes/` contain `if/else` business logic, data transformations, or validation beyond schema declaration?

**Action**: Reject. Move logic to `controllers/`. Routes exist only for mapping endpoints to handlers.

### [blocking] Shared Model Contamination

Does the shared models package (`common/models/`) import framework code (FastAPI, Django) or contain business methods?

**Action**: Reject. Shared models define database schema only (columns, relationships, constraints).

### [blocking] Raw SQL in Business Logic

Does a controller or util execute raw SQL via `text()` or string formatting?

**Action**: Reject. Use ORM queries in repositories. Raw SQL is acceptable only in migrations or dedicated query builders.

### [blocking] Missing Downgrade in Migrations

Does an Alembic migration have `upgrade()` but an empty `downgrade()`?

**Action**: Reject. Every migration must be reversible.

## Frontend Guardrails

### [blocking] Direct HTTP in Components

Does a React/Vue component import `axios` directly or make `fetch` calls to raw URIs?

**Action**: Reject. All network calls must go through a centralized service layer (`services/` or `lib/services/`).

```typescript
// VIOLATION
const res = await fetch('/api/users')
const { data } = await axios.get('/api/users')

// CORRECT
import { userService } from '@/services/user'
const users = await userService.getAll()
```

### [blocking] Untyped API Responses

Does a frontend service return `any` or lack TypeScript interfaces?

**Action**: Reject. Define proper TS interfaces that mirror backend response schemas.

### [important] Missing Loading/Error States

Does a page fetch data without `loading.tsx`, Suspense boundaries, or error handling?

**Action**: Should fix. Every data-fetching component needs loading and error states.

## General Guardrails

### [important] Missing Tests

Does a PR add new business logic without corresponding tests?

**Action**: Should fix. New logic needs unit tests; new endpoints need integration tests.

### [important] Hardcoded Configuration

Are database URLs, API keys, or environment-specific values hardcoded in source code?

**Action**: Should fix. Use environment variables or settings files.

### [nit] Inconsistent Naming

Do new functions/variables follow the existing project conventions?

**Action**: Nice to have. Suggest alignment with project style.

### [nit] Redundant Comments

Are there comments that just narrate what the code does (`# increment counter`, `# return result`)?

**Action**: Nice to have. Comments should explain *why*, not *what*.

## Giving Constructive Feedback

Use collaborative language while firmly enforcing architecture:

```markdown
# BAD
"You broke the architecture. Move this SQL to a repository."

# GOOD
"[blocking] I see database queries inside the controller. Per our layered
architecture, controllers should delegate DB access to repositories. Could
we extract this `query()` logic into `UserRepository.get_active()`? This
keeps the controller thin and the query reusable."
```

## Review Checklist

When reviewing any PR, check in this order:

1. **Architecture**: Do imports respect layer boundaries?
2. **Correctness**: Does the logic handle edge cases?
3. **Security**: SQL injection, XSS, exposed secrets?
4. **Tests**: Are changes covered by tests?
5. **Types**: Are request/response schemas properly typed?
6. **Performance**: N+1 queries, missing indexes, unbounded lists?
7. **Documentation**: Are public APIs and complex logic documented?

## Checklist for Reviewers

- [ ] No `Session` imports outside `repositories/`
- [ ] No business logic in routes
- [ ] Entities are pure (no DB, no HTTP)
- [ ] Frontend uses service layer for all API calls
- [ ] All API responses have TypeScript types
- [ ] New logic has corresponding tests
- [ ] No hardcoded secrets or credentials
- [ ] Migrations have both upgrade and downgrade
- [ ] Feedback uses severity labels and collaborative tone
