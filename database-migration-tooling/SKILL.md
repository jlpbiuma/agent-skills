---
name: database-migration-tooling
description: >-
  Design database migration workflows with Alembic, idempotent seeders, 
  environment sync scripts, and backup tooling. Use when setting up Alembic, 
  writing migrations, creating seed scripts, syncing data between environments, 
  building database backup strategies, or designing DB tooling containers.
---

# Database Migration & Tooling

## Directory Structure

```
database/
├── alembic.ini              # Alembic config (script_location, logging)
├── alembic/
│   ├── env.py               # Loads ORM metadata + DB config
│   ├── script.py.mako       # Migration template
│   └── versions/            # Linear migration chain
│       ├── 001_initial_schema.py
│       ├── 002_data_tables.py
│       └── ...
├── schemas/                 # SQL DDL files (used by early migrations)
├── seeders/
│   ├── base_seeder.py       # Abstract seeder with idempotent tracking
│   ├── seed_admin.py        # Create default admin user
│   └── seed_user.py         # Create default regular user
├── run_seeders.py           # Execute all seeders
├── sync.py                  # Bulk data sync between environments
├── backup.py                # pg_dump wrapper
└── entrypoint.sh            # Container command router
```

## Alembic Configuration

### env.py — Load ORM Metadata

```python
# GOOD: Import shared models so Alembic sees all tables
from common.base import Base
import common.models  # side-effect: registers tables on Base.metadata

from common.config import POSTGRES_CONFIG

def get_url():
    return f"postgresql+psycopg2://{POSTGRES_CONFIG['user']}:{POSTGRES_CONFIG['password']}@{POSTGRES_CONFIG['host']}:{POSTGRES_CONFIG['port']}/{POSTGRES_CONFIG['database']}"

# Optionally filter to managed tables only
MANAGED_TABLES = {"users", "occupations", "offers", ...}

def include_object(object, name, type_, reflected, compare_to):
    if type_ == "table":
        return name in MANAGED_TABLES
    return True

target_metadata = Base.metadata
```

### Migration Patterns

**SQL-based early migrations** (for complex DDL):

```python
# 001_initial_schema.py
def upgrade():
    schema_dir = Path(__file__).parent.parent.parent / "schemas"
    sql = (schema_dir / "pipeline_config.sql").read_text()
    op.execute(sql)

def downgrade():
    op.drop_table("pipeline_config")
```

**Python-based app migrations** (for ORM-driven tables):

```python
# 003_app_users.py
def upgrade():
    op.create_table(
        "app_users",
        sa.Column("id", sa.Integer, primary_key=True),
        sa.Column("email", sa.String(255), unique=True, nullable=False),
        sa.Column("role", sa.String(50), nullable=False, server_default="user"),
        sa.Column("created_at", sa.DateTime, server_default=sa.func.now()),
    )

def downgrade():
    op.drop_table("app_users")
```

## Idempotent Seeders

```python
# seeders/base_seeder.py
class BaseSeeder:
    name: str  # Unique seeder identifier

    def execute(self, db, batch_id: str, force: bool = False):
        existing = db.query(SeederExecution).filter_by(seeder_name=self.name).first()
        if existing and not force:
            print(f"Seeder '{self.name}' already executed. Use --force to re-run.")
            return
        self.run(db)
        execution = SeederExecution(seeder_name=self.name, batch_id=batch_id)
        db.merge(execution)
        db.commit()

    def run(self, db):
        raise NotImplementedError
```

```python
# seeders/seed_admin.py
class AdminSeeder(BaseSeeder):
    name = "seed_admin"

    def run(self, db):
        if not db.query(AppUser).filter_by(role="admin").first():
            admin = AppUser(
                email=os.environ["ADMIN_EMAIL"],
                role="admin",
                cod_personal=os.environ["ADMIN_COD_PERSONAL"],
            )
            db.add(admin)
            db.commit()
```

## Environment Sync (sync.py)

Pattern for copying data tables from one PostgreSQL instance to another:

```python
# 1. Validate schema parity
source_hash = get_schema_md5(source_conn, exclude=["alembic_version", "seeder_executions"])
dest_hash = get_schema_md5(dest_conn, exclude=["alembic_version", "seeder_executions"])
if source_hash != dest_hash:
    raise SystemExit("Schema mismatch. Run migrations on destination first.")

# 2. Disable FK checks during load
dest_conn.execute("SET session_replication_role = replica")

# 3. Truncate target tables (FK-safe: single TRUNCATE ... CASCADE)
tables = ["users", "occupations", "offers", ...]
dest_conn.execute(f"TRUNCATE {', '.join(tables)} CASCADE")

# 4. COPY CSV per table via spooled buffer
for table in tables:
    buffer = SpooledTemporaryFile(max_size=256 * 1024 * 1024)
    source_conn.copy_to(buffer, table, format="csv", header=True)
    buffer.seek(0)
    dest_conn.copy_from(buffer, table, format="csv", header=True)

# 5. Re-enable FK checks
dest_conn.execute("SET session_replication_role = origin")

# 6. Verify row counts
for table in tables:
    assert source_count(table) == dest_count(table)
```

## Backup Script

```python
# backup.py
import subprocess, datetime

def backup(host, port, user, password, db, output_dir="/backups", label=""):
    timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
    filename = f"{db}_{timestamp}{'_' + label if label else ''}.dump"
    filepath = Path(output_dir) / filename

    env = {**os.environ, "PGPASSWORD": password}
    subprocess.run([
        "pg_dump", "-Fc", "-h", host, "-p", str(port),
        "-U", user, "-d", db, "-f", str(filepath)
    ], env=env, check=True)
```

## Container Entrypoint

```bash
#!/bin/sh
set -e
case "$1" in
  migrate)          cd /app/database && exec alembic upgrade head ;;
  seed)             shift; cd /app && exec python database/run_seeders.py "$@" ;;
  test)             cd /app && exec pytest common/test/test_schema_parity.py -v ;;
  migrate-and-seed) cd /app/database && alembic upgrade head && \
                    cd /app && exec python database/run_seeders.py "$@" ;;
  sync)             shift; cd /app && exec python database/sync.py "$@" ;;
  backup)           shift; cd /app && exec python database/backup.py "$@" ;;
  *)                exec "$@" ;;
esac
```

## Schema Parity Test

Verify ORM models match the live database:

```python
# common/test/test_schema_parity.py
def test_schema_parity(db_engine):
    inspector = inspect(db_engine)
    for table in Base.metadata.tables.values():
        db_columns = {c["name"]: c for c in inspector.get_columns(table.name)}
        for col in table.columns:
            assert col.name in db_columns, f"Missing column: {table.name}.{col.name}"
            assert compatible_types(col.type, db_columns[col.name]["type"])
```

## Good Practices

```python
# GOOD: Linear migration chain (no branches)
# 001 → 002 → 003 → ... → 016

# GOOD: Migrations are both upgrade AND downgrade
def upgrade():
    op.add_column("users", sa.Column("role", sa.String(50)))
def downgrade():
    op.drop_column("users", "role")

# GOOD: Schema parity test in CI
# Catches ORM drift before deployment

# GOOD: Backup before sync
backup(label="pre-sync")
sync(dest_host=prod_host)

# GOOD: Seeder idempotency via tracking table
# seeder_executions tracks what ran and when

# GOOD: Separate Dockerfile for DB tooling (minimal deps)
# Only alembic + sqlalchemy + psycopg2 — not the full app stack
```

## Bad Practices

```python
# BAD: Branching migrations (parallel versions)
# Creates merge conflicts and ordering ambiguity

# BAD: Data manipulation in migrations
def upgrade():
    op.execute("UPDATE users SET role = 'admin' WHERE email = 'john@example.com'")
    # Use seeders for data, migrations for schema

# BAD: No downgrade function
def downgrade():
    pass  # Can't rollback!

# BAD: Hardcoded DB URL in alembic.ini
sqlalchemy.url = postgresql://root:password@localhost/mydb
# Use env.py to build URL from environment variables

# BAD: Running migrations as part of the app startup
# Migrations should run via dbtools container, not embedded in the API

# BAD: sync without schema validation
# Could corrupt data if schemas differ

# BAD: Full app image for migrations
# FROM myapp:latest  ← pulls spaCy, torch, etc. just to run alembic
# Use a minimal dbtools image instead
```

## Checklist

- [ ] `env.py` loads ORM metadata from shared models (imports `common.models`)
- [ ] DB URL built from environment variables, not hardcoded
- [ ] Every migration has both `upgrade()` and `downgrade()`
- [ ] Linear chain: no branching versions
- [ ] Seeders are idempotent (tracking table or upsert logic)
- [ ] Sync validates schema parity (MD5 hash) before copying data
- [ ] Backup runs before destructive operations (sync, ETL)
- [ ] Schema parity test runs in CI after migrations
- [ ] Minimal Dockerfile for DB tooling (not the full app stack)
- [ ] Entrypoint routes commands: migrate, seed, test, sync, backup
