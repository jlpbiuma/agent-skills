---
name: cicd-pipeline-design
description: >-
  Design CI/CD pipelines for full-stack projects with stages for infrastructure
  validation, database migrations, ETL, image building, environment sync, and
  production deployment via SSH. Use when creating Jenkinsfiles, GitHub Actions
  workflows, designing deployment strategies, or automating build-test-deploy
  cycles.
---

# CI/CD Pipeline Design

## Pipeline Philosophy

A production pipeline is a **sequential safety net**: each stage validates a precondition before proceeding. If any stage fails, the pipeline stops — no partial deployments.

## Stage Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    VALIDATE INFRASTRUCTURE                  │
│  1. Test SSH to prod    2. Check source DBs    3. Check     │
│     (network access)       (MySQL, Oracle)        target DB │
│                            (ping/isready)        (Postgres) │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│                    PREPARE DATABASE                          │
│  4. Run migrations      5. Validate schema    6. Seed       │
│     (Alembic)              parity (pytest)       users      │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│                    DATA PIPELINE (optional)                  │
│  7. Backup dev DB       8. Run ETL            9. Sync data  │
│     before ETL             (trigger job)         to prod    │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│                    BUILD & DEPLOY                            │
│  10. Build images       11. Push to registry  12. Pull &    │
│      (backend, front)       (GHCR/ECR)           deploy on  │
│                                                    prod     │
└─────────────────────────────────────────────────────────────┘
```

## Jenkinsfile Pattern

### Parameters (Control Flow)

```groovy
parameters {
    booleanParam(name: 'RUN_MIGRATIONS', defaultValue: true,
                 description: 'Run database migrations (Alembic)')
    booleanParam(name: 'RUN_ETL', defaultValue: true,
                 description: 'Run ETL pipeline (forces migrations + seeds)')
    booleanParam(name: 'RUN_SYNC', defaultValue: true,
                 description: 'Sync ETL data from dev to production')
    booleanParam(name: 'BUILD_BACKEND', defaultValue: false,
                 description: 'Build and push backend image')
    booleanParam(name: 'BUILD_FRONTEND', defaultValue: false,
                 description: 'Build and push frontend image')
    booleanParam(name: 'DEPLOY_BACKEND', defaultValue: false,
                 description: 'Pull and deploy latest backend on prod')
    booleanParam(name: 'DEPLOY_FRONTEND', defaultValue: false,
                 description: 'Pull and deploy latest frontend on prod')
    string(name: 'NUM_WORKERS', defaultValue: '4',
           description: 'Parallel workers for ETL')
}
```

### Environment (Credentials)

```groovy
environment {
    PROJECT_PATH = credentials('project-path')

    // Source databases
    MYSQL_HOST = credentials('mysql-host')
    ORACLE_HOST = credentials('oracle-host')

    // Target database
    PG_HOST = credentials('postgres-host')
    PG_USER = credentials('postgres-user')
    PG_PASS = credentials('postgres-password')

    // Production server
    PROD_IP   = credentials('prod-server-ip')
    PROD_USER = credentials('prod-ssh-user')
    PROD_PATH = credentials('prod-project-path')
}
```

### Stage 1: Validate Infrastructure

```groovy
stage('Validate SSH') {
    steps {
        sshagent(credentials: ['prod-ssh']) {
            sh 'ssh -o StrictHostKeyChecking=no $PROD_USER@$PROD_IP "uname -a"'
        }
    }
}

stage('Validate Source DB') {
    steps {
        sh 'export MYSQL_PWD="$MYSQL_PASS"; mysqladmin -h "$MYSQL_HOST" ping'
    }
}

stage('Validate Target DB') {
    steps {
        script {
            def ready = sh(script: 'pg_isready -h "$PG_HOST" -U "$PG_USER"',
                           returnStatus: true)
            if (ready != 0) {
                dir(env.PROJECT_PATH) {
                    sh 'docker compose up -d postgres'
                }
                // Retry loop (12 attempts, 5s each)
            }
        }
    }
}
```

### Stage 2: Database Operations

```groovy
stage('Migrations') {
    when { expression { params.RUN_MIGRATIONS || params.RUN_ETL } }
    steps {
        dir(env.PROJECT_PATH) {
            sh 'docker compose --profile tools build dbtools'
            sh 'docker compose --profile tools push dbtools'
            sh 'docker compose --profile tools run --rm dbtools migrate'
        }
    }
}

stage('Schema Parity') {
    when { expression { params.RUN_MIGRATIONS || params.RUN_ETL } }
    steps {
        dir(env.PROJECT_PATH) {
            sh 'docker compose --profile tools run --rm dbtools test'
        }
    }
}

stage('Seed Users') {
    when { expression { params.RUN_MIGRATIONS || params.RUN_ETL } }
    steps {
        dir(env.PROJECT_PATH) {
            sh 'docker compose --profile tools run --rm dbtools seed'
        }
    }
}
```

### Stage 3: Data Sync

```groovy
stage('Sync to Production') {
    when { expression { params.RUN_SYNC } }
    steps {
        dir(env.PROJECT_PATH) {
            // Backup prod before sync
            sh '''docker compose --profile tools run --rm dbtools backup \
                --host "$PROD_IP" --user "$PG_USER" --password "$PG_PASS" \
                --db "$PG_DB" --label prod-pre-sync'''

            // Sync ETL tables
            sh '''docker compose --profile tools run --rm dbtools sync \
                --dest-host "$PROD_IP" --dest-user "$PG_USER" \
                --dest-password "$PG_PASS" --dest-db "$PG_DB"'''
        }
    }
}
```

### Stage 4: Build & Deploy

```groovy
stage('Deploy Images') {
    when { expression { params.DEPLOY_BACKEND || params.DEPLOY_FRONTEND } }
    steps {
        script {
            def services = []
            if (params.DEPLOY_BACKEND) services.add('backend')
            if (params.DEPLOY_FRONTEND) services.add('frontend')
            def list = services.join(' ')

            sshagent(credentials: ['prod-ssh']) {
                // Pull images
                sh """ssh $PROD_USER@$PROD_IP "cd $PROD_PATH && \
                    docker compose -f docker-compose.prod.yaml pull ${list}" """

                // Verify digest matches what was built
                // (compare dev digest vs prod digest)

                // Deploy
                sh """ssh $PROD_USER@$PROD_IP "cd $PROD_PATH && \
                    docker compose -f docker-compose.prod.yaml up -d ${list} && \
                    docker image prune -f" """
            }
        }
    }
}
```

## Good Practices

```groovy
// GOOD: Validate infrastructure before doing any work
// Fail fast if SSH, MySQL, Oracle, or Postgres are unreachable

// GOOD: Image digest verification
// After pushing: store digest. After pulling on prod: compare.
env.BACKEND_DIGEST = sh(script: 'docker inspect --format=... image:latest',
                         returnStdout: true).trim()
// On prod after pull:
def prodDigest = sh(script: 'ssh ... "docker inspect --format=... image:latest"',
                     returnStdout: true).trim()
if (prodDigest != env.BACKEND_DIGEST) {
    error "DIGEST MISMATCH"
}

// GOOD: Backup before destructive operations
// Always backup prod DB before sync or ETL

// GOOD: Schema parity test after migrations
// Catches ORM drift before any data operations

// GOOD: Boolean parameters for optional stages
// Users choose exactly what to run per pipeline execution

// GOOD: SCP env files to prod (not committed to git)
sh 'scp .env.production $PROD_USER@$PROD_IP:$PROD_PATH/.env.production'

// GOOD: Trigger child jobs with parameters
build job: 'ETL Pipeline',
    parameters: [string(name: 'SCOPE', value: params.LOCATION)],
    wait: true

// GOOD: Prune old images after deployment
sh 'docker image prune -f'
```

## Bad Practices

```groovy
// BAD: Deploy without validating infrastructure first
stage('Deploy') { /* no SSH check, no DB check */ }

// BAD: No backup before sync
stage('Sync') {
    sh 'dbtools sync --dest-host prod'  // prod data gone if sync fails
}

// BAD: Hardcoded credentials
environment {
    PG_PASS = 'mysecretpassword'  // Use credentials() binding
}

// BAD: No digest verification
// Pulling "latest" tag — could be a different image than tested

// BAD: Force push without schema validation
// Migrations ran on dev but not on prod → data corruption

// BAD: Single monolithic stage
stage('Do Everything') {
    sh 'migrate && seed && etl && sync && build && deploy'
    // Can't skip steps, can't identify which part failed
}

// BAD: No conditional stages
// Always runs everything — wastes time when only deploying frontend

// BAD: Running migrations inside app startup
// App container runs alembic upgrade head on every restart
CMD ["sh", "-c", "alembic upgrade head && python app.py"]

// BAD: No retry logic for infrastructure checks
def ready = sh(script: 'pg_isready', returnStatus: true)
if (ready != 0) error "DB not ready"  // Should retry, not fail immediately
```

## Pipeline Execution Examples

```bash
# Full pipeline: migrations + ETL + sync + deploy everything
# Parameters: RUN_MIGRATIONS=true, RUN_ETL=true, RUN_SYNC=true,
#             BUILD_BACKEND=true, BUILD_FRONTEND=true,
#             DEPLOY_BACKEND=true, DEPLOY_FRONTEND=true

# Quick frontend deploy: build + push + deploy frontend only
# Parameters: BUILD_FRONTEND=true, DEPLOY_FRONTEND=true
# (all others false)

# Database-only: migrations + seeds (no ETL, no deploy)
# Parameters: RUN_MIGRATIONS=true

# ETL refresh + sync (no image rebuild)
# Parameters: RUN_ETL=true, RUN_SYNC=true
```

## Checklist

- [ ] Infrastructure validation stages run first (SSH, DB pings)
- [ ] Retry logic for transient failures (DB not ready yet)
- [ ] All credentials from Jenkins credential store, never hardcoded
- [ ] Backup before every destructive operation (sync, ETL)
- [ ] Schema parity validated after migrations
- [ ] Image digest verified after pull on production
- [ ] Boolean parameters for each optional pipeline segment
- [ ] Child jobs triggered with `wait: true` for sequential execution
- [ ] Env files transferred via SCP, not committed to git
- [ ] `docker image prune -f` after deployment to reclaim space
- [ ] Conditional `when` blocks on every optional stage
