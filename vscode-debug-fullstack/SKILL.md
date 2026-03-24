---
name: vscode-debug-fullstack
description: >-
  Configure VS Code launch.json for debugging full-stack projects with Python
  backends (FastAPI/Django), Node frontends (Next.js/React), database tooling,
  ETL pipelines, and compound launchers. Use when setting up debug
  configurations, creating launch.json, configuring multi-service debugging,
  or adding pytest/jest debug profiles.
---

# VS Code Debug Launch Configurations

## File Location

```
.vscode/
├── launch.json       # Debug configurations
└── settings.json     # Interpreter, PYTHONPATH, linters, test runners
```

## Configuration Categories

A full-stack project needs these debug configuration groups:

| Group              | Debugger  | Purpose                                      |
|--------------------|-----------|----------------------------------------------|
| **Database**       | debugpy   | Run migrations, seeders                      |
| **Backend (Dev)**  | debugpy   | API server with hot reload                   |
| **Backend (Prod)** | debugpy   | API server without reload (prod env)         |
| **Frontend**       | node      | Next.js/React dev server with inspector      |
| **ETL Steps**      | debugpy   | Individual pipeline steps with full args     |
| **Tests**          | debugpy   | Pytest current file / all tests              |
| **Utilities**      | debugpy   | Run any Python file                          |
| **Compounds**      | —         | Launch backend + frontend together           |

## Backend Configurations

### Development (with reload)

```json
{
  "name": "Backend: Development",
  "type": "debugpy",
  "request": "launch",
  "module": "uvicorn",
  "args": ["backend.app:app", "--host", "0.0.0.0", "--port", "9016", "--reload"],
  "jinja": true,
  "justMyCode": false,
  "envFile": "${workspaceFolder}/backend/.env.development",
  "console": "integratedTerminal",
  "cwd": "${workspaceFolder}",
  "python": "${workspaceFolder}/.venv/bin/python"
}
```

### Production (no reload)

```json
{
  "name": "Backend: Production",
  "type": "debugpy",
  "request": "launch",
  "module": "uvicorn",
  "args": ["backend.app:app", "--host", "0.0.0.0", "--port", "9016"],
  "jinja": true,
  "justMyCode": false,
  "envFile": "${workspaceFolder}/backend/.env.production",
  "console": "integratedTerminal",
  "cwd": "${workspaceFolder}",
  "python": "${workspaceFolder}/.venv/bin/python"
}
```

**Key**: Use a different port than Docker (e.g., `9016` locally vs `9015` in containers) to avoid conflicts.

## Frontend Configuration (Next.js)

```json
{
  "name": "Frontend: Next.js",
  "type": "node",
  "request": "launch",
  "program": "${workspaceFolder}/frontend/node_modules/.bin/next",
  "runtimeArgs": ["--inspect"],
  "cwd": "${workspaceFolder}/frontend",
  "envFile": "${workspaceFolder}/frontend/.env.development",
  "console": "integratedTerminal",
  "env": { "NODE_ENV": "development" },
  "skipFiles": ["<node_internals>/**", "${workspaceFolder}/frontend/.next/**"],
  "sourceMapPathOverrides": {
    "webpack://_N_E/*": "${workspaceFolder}/frontend/*"
  },
  "restart": true,
  "pauseForSourceMap": false
}
```

## Database Configurations

```json
{
  "name": "Database: Migrations + Seeders",
  "type": "debugpy",
  "request": "launch",
  "program": "${workspaceFolder}/database/run_migrations_and_seeders.py",
  "console": "integratedTerminal",
  "justMyCode": false,
  "envFile": "${workspaceFolder}/backend/.env.development",
  "cwd": "${workspaceFolder}",
  "python": "${workspaceFolder}/.venv/bin/python"
}
```

Create separate configs for `Migrations Only` and `Seeders Only` using the corresponding script paths.

## ETL Step Configurations

Each step gets its own config with **all CLI arguments explicit**:

```json
{
  "name": "ETL: Step 3 - Build Vectors",
  "type": "debugpy",
  "request": "launch",
  "program": "${workspaceFolder}/etl/03_build_vectors.py",
  "args": [
    "--force", "true",
    "--workers", "4",
    "--chunk_size", "1000",
    "--output_path", "${workspaceFolder}/common/arrays/vectors.npy"
  ],
  "console": "integratedTerminal",
  "justMyCode": false,
  "envFile": "${workspaceFolder}/backend/.env.development",
  "cwd": "${workspaceFolder}",
  "python": "${workspaceFolder}/.venv/bin/python"
}
```

**Why explicit args**: ETL steps have many parameters. Putting them in launch.json makes them visible, reproducible, and debuggable without remembering CLI flags.

## Test Configurations

```json
{
  "name": "Pytest: Current File",
  "type": "debugpy",
  "request": "launch",
  "module": "pytest",
  "args": ["${file}", "-v", "-s"],
  "console": "integratedTerminal",
  "justMyCode": false,
  "envFile": "${workspaceFolder}/backend/.env.development",
  "cwd": "${workspaceFolder}",
  "python": "${workspaceFolder}/.venv/bin/python"
},
{
  "name": "Pytest: All Tests",
  "type": "debugpy",
  "request": "launch",
  "module": "pytest",
  "args": ["backend/test/", "-v", "-s", "--tb=short"],
  "console": "integratedTerminal",
  "justMyCode": false,
  "envFile": "${workspaceFolder}/backend/.env.development",
  "cwd": "${workspaceFolder}",
  "python": "${workspaceFolder}/.venv/bin/python"
}
```

## Compound Launchers

Launch backend + frontend simultaneously:

```json
{
  "compounds": [
    {
      "name": "Full Stack: Development",
      "configurations": ["Backend: Development", "Frontend: Next.js"],
      "presentation": { "group": "fullstack", "order": 1 },
      "stopAll": true
    },
    {
      "name": "Full Stack: Production Env",
      "configurations": ["Backend: Production", "Frontend: Next.js"],
      "presentation": { "group": "fullstack", "order": 2 },
      "stopAll": true
    }
  ]
}
```

`stopAll: true` ensures stopping one service stops both.

## Companion: settings.json

```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
  "python.envFile": "${workspaceFolder}/backend/.env.development",
  "python.testing.pytestEnabled": true,
  "python.testing.pytestArgs": ["backend/test", "common/test"],
  "python.testing.cwd": "${workspaceFolder}",
  "terminal.integrated.env.linux": {
    "PYTHONPATH": "${workspaceFolder}"
  },
  "jest.rootPath": "${workspaceFolder}/frontend",
  "jest.disabledWorkspaceFolders": ["backend", "common"]
}
```

## Good Practices

```json
// GOOD: Explicit python path — avoids using system Python
"python": "${workspaceFolder}/.venv/bin/python"

// GOOD: envFile per environment — same vars as containers use
"envFile": "${workspaceFolder}/backend/.env.development"

// GOOD: cwd at project root — imports resolve correctly
"cwd": "${workspaceFolder}"

// GOOD: justMyCode false — step into library code when debugging
"justMyCode": false

// GOOD: Different local port than Docker to allow both simultaneously
"args": ["--port", "9016"]  // Docker uses 9015

// GOOD: skipFiles to avoid stepping into Node internals
"skipFiles": ["<node_internals>/**", "${workspaceFolder}/frontend/.next/**"]

// GOOD: PYTHONPATH in terminal env — CLI scripts resolve imports
"terminal.integrated.env.linux": { "PYTHONPATH": "${workspaceFolder}" }
```

## Bad Practices

```json
// BAD: No envFile — DB credentials missing, app crashes at start
// (missing "envFile" field)

// BAD: Using system python instead of venv
"python": "/usr/bin/python3"

// BAD: cwd inside backend/ — common.* imports fail
"cwd": "${workspaceFolder}/backend"

// BAD: Same port as Docker — conflicts when both run
"args": ["--port", "9015"]  // Container also uses 9015

// BAD: justMyCode true — can't debug into SQLAlchemy/FastAPI internals
"justMyCode": true

// BAD: No compound launcher — must start backend and frontend separately

// BAD: Hardcoded paths instead of ${workspaceFolder}
"program": "/home/user/project/backend/app.py"

// BAD: ETL steps without args — silently uses wrong defaults
{
  "name": "ETL: Step 3",
  "program": "${workspaceFolder}/etl/03_build_vectors.py"
  // No args! Uses unknown defaults
}
```

## Checklist

- [ ] Backend dev config uses `--reload` and `.env.development`
- [ ] Backend prod config omits `--reload` and uses `.env.production`
- [ ] Local ports differ from Docker ports (e.g., 9016 vs 9015)
- [ ] Frontend config has `skipFiles` and source map overrides
- [ ] All Python configs point to `.venv/bin/python`
- [ ] All configs set `cwd` to project root
- [ ] ETL steps have all CLI args explicitly listed
- [ ] Compound launchers combine backend + frontend with `stopAll: true`
- [ ] `settings.json` sets `PYTHONPATH` for terminal and test runner
- [ ] Jest scoped to `frontend/` to avoid running on Python dirs
