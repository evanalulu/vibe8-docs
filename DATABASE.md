# Vibe8 Database Schema

Complete PostgreSQL database schema documentation for Vibe8.

## Table of Contents
- [Overview](#overview)
- [Schema Diagram](#schema-diagram)
- [Tables](#tables)
  - [users](#users-table)
  - [analysis_runs](#analysis_runs-table)
  - [jobs](#jobs-table)
- [Indexes](#indexes)
- [Relationships](#relationships)
- [Access Control](#access-control)
- [Migrations](#migrations)
- [Backup and Recovery](#backup-and-recovery)

## Overview

Vibe8 uses **PostgreSQL 15+** for persistent data storage with the following characteristics:

- **ORM**: SQLAlchemy 2.0 with declarative models
- **Migrations**: Alembic for schema versioning
- **Connection Pooling**: Configurable via `DB_POOL_SIZE` and `DB_MAX_OVERFLOW`
- **JSONB Storage**: Metrics and metadata stored as JSONB for flexibility
- **Cloud-Agnostic**: Standard PostgreSQL compatible with any provider

### Database Configuration

Configure via environment variables (see `.env.example` in project root for complete list):
- `DATABASE_URL` - PostgreSQL connection string
- `DB_POOL_SIZE` (default: 5) - Connection pool size
- `DB_MAX_OVERFLOW` (default: 10) - Max overflow connections

## Schema Diagram

```
┌─────────────────────┐
│       users         │
│─────────────────────│
│ id (PK)             │◄────┐
│ email (unique)      │     │
│ auth0_sub (unique)  │     │
│ created_at          │     │
│ last_login          │     │
└─────────────────────┘     │
                            │
                            │ user_id (FK)
                            │
┌─────────────────────┐     │
│   analysis_runs     │     │
│─────────────────────│     │
│ run_id (PK)         │◄────┼────┐
│ user_id (FK)        │─────┘    │
│ created_at          │          │
│ upload_filename     │          │
│ submission_type     │          │
│ project_name        │          │
│ author_type         │          │
│ author_model        │          │
│ git_repo_url        │          │
│ git_branch          │          │
│ git_commit_sha      │          │
│ metrics (JSONB)     │          │
│ file_tree (JSONB)   │          │
│ errors (JSONB)      │          │
└─────────────────────┘          │
                                 │ result_run_id (FK)
                                 │
┌─────────────────────┐          │
│        jobs         │          │
│─────────────────────│          │
│ job_id (PK)         │          │
│ user_id (FK)        │─────┐    │
│ status              │     │    │
│ source_kind         │     └────┼─── user_id (FK)
│ source_metadata     │          │
│ result_run_id (FK)  │──────────┘
│ error_message       │
│ created_at          │
│ updated_at          │
└─────────────────────┘
```

## Tables

### users Table

Stores authenticated user profiles.

**Purpose**: Track registered users and their Auth0 identities

**Schema**:
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR NOT NULL UNIQUE,
  auth0_sub VARCHAR UNIQUE,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  last_login TIMESTAMP
);

CREATE UNIQUE INDEX ix_users_email ON users(email);
CREATE UNIQUE INDEX ix_users_auth0_sub ON users(auth0_sub);
```

**Columns**:

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `id` | UUID | No | Primary key (auto-generated) |
| `email` | VARCHAR | No | User's email address (unique, indexed) |
| `auth0_sub` | VARCHAR | Yes | Auth0 subject identifier (unique, indexed, NULL until first OAuth login) |
| `created_at` | TIMESTAMP | No | Account creation timestamp (default: NOW()) |
| `last_login` | TIMESTAMP | Yes | Last login timestamp (updated on login) |

**Constraints**:
- `email`: UNIQUE, NOT NULL
- `auth0_sub`: UNIQUE, NULL allowed

**Indexes**:
- `ix_users_email`: Fast lookup by email (unique)
- `ix_users_auth0_sub`: Fast lookup by Auth0 subject (unique)

**SQLAlchemy Model**:
```python
# backend/src/infrastructure/database/models/user.py
class User(Base):
    __tablename__ = "users"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    email = Column(String, unique=True, nullable=False, index=True)
    auth0_sub = Column(String, unique=True, nullable=True, index=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    last_login = Column(DateTime(timezone=True), nullable=True)

    # Relationships
    analysis_runs = relationship("AnalysisRun", back_populates="user", cascade="all, delete-orphan")
    jobs = relationship("Job", back_populates="user", cascade="all, delete-orphan")
```

**Access Patterns**:
- Lookup by email on user creation
- Lookup by auth0_sub on authenticated requests
- Update last_login on successful login

---

### analysis_runs Table

Stores complete analysis results with metadata.

**Purpose**: Persist code analysis results for user history

**Schema**:
```sql
CREATE TABLE analysis_runs (
  run_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  upload_filename VARCHAR,
  submission_type VARCHAR NOT NULL,
  project_name VARCHAR,
  author_type VARCHAR,
  author_model VARCHAR,
  git_repo_url VARCHAR,
  git_branch VARCHAR,
  git_commit_sha VARCHAR,
  metrics JSONB NOT NULL,
  file_tree JSONB,
  errors JSONB NOT NULL DEFAULT '[]'::jsonb
);

CREATE INDEX ix_analysis_runs_user_id ON analysis_runs(user_id);
CREATE INDEX ix_analysis_runs_created_at ON analysis_runs(created_at);
```

**Columns**:

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `run_id` | UUID | No | Primary key (auto-generated) |
| `user_id` | UUID | No | Foreign key to `users.id` (indexed) |
| `created_at` | TIMESTAMP | No | Analysis creation timestamp (default: NOW(), indexed) |
| `upload_filename` | VARCHAR | Yes | Filename for upload source (NULL for GitHub) |
| `submission_type` | VARCHAR | No | Source type: `upload` or `github` |
| `project_name` | VARCHAR | Yes | User-assigned project name |
| `author_type` | VARCHAR | Yes | Code authorship type: `human` or `ai` |
| `author_model` | VARCHAR | Yes | Code authorship model (e.g., `developer`, `chatgpt`, `gpt-4`) |
| `git_repo_url` | VARCHAR | Yes | GitHub repository URL (NULL for upload) |
| `git_branch` | VARCHAR | Yes | Git branch name (NULL for upload) |
| `git_commit_sha` | VARCHAR | Yes | Git commit SHA (NULL for upload) |
| `metrics` | JSONB | No | Aggregate metrics object from all 10 collectors (no per-file data) |
| `file_tree` | JSONB | Yes | File tree with per-file metrics and severity scores (NULL for old analyses) |
| `errors` | JSONB | No | Array of error strings (default: empty array) |

**Constraints**:
- `user_id`: FOREIGN KEY → `users(id)` ON DELETE CASCADE
- `submission_type`: Must be `upload` or `github`

**Indexes**:
- `ix_analysis_runs_user_id`: Fast lookup by user
- `ix_analysis_runs_created_at`: Fast sorting by time

**SQLAlchemy Model**:
```python
# backend/src/infrastructure/database/models/analysis.py
class AnalysisRun(Base):
    __tablename__ = "analysis_runs"

    # Core fields
    run_id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    user_id = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False, index=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now(), nullable=False, index=True)

    # Upload source metadata
    upload_filename = Column(String, nullable=True)

    # Project metadata
    submission_type = Column(String, nullable=False)
    project_name = Column(String, nullable=True)

    # Code authorship tracking
    author_type = Column(String, nullable=True)  # "human" or "ai"
    author_model = Column(String, nullable=True)  # "developer", "chatgpt", etc.

    # GitHub source metadata
    git_repo_url = Column(String, nullable=True)
    git_branch = Column(String, nullable=True)
    git_commit_sha = Column(String, nullable=True)

    # Analysis results
    metrics = Column(JSONB, nullable=False)
    file_tree = Column(JSONB, nullable=True)
    errors = Column(JSONB, nullable=False, default=list)

    # Relationships
    user = relationship("User", back_populates="analysis_runs")
    job = relationship("Job", back_populates="result_analysis", uselist=False)
```

**Access Patterns**:
- List by user_id ordered by created_at DESC (history)
- Lookup by run_id (specific analysis)
- Delete by run_id (user-initiated deletion)

**Metrics JSONB Structure** (aggregate values only):
```json
{
  "lint": { "total": 0, "linting_score": 100.0 },
  "cyclomatic": { "avg_cc": 0.0, "complexity_score": 100.0 },
  "docs": { "coverage_pct": 0 },
  "dead_code": { "unused_symbols": 0 },
  "deps": { "total_modules": 0, "package_health": [] },
  "mi": { "mi_avg": 0.0 },
  "size": { "files": 0, "sloc": 0, "avg_file_sloc": 0 },
  "dupes": { "duplicate_lines": 0, "duplicate_pct": 0.0, "duplication_score": 100.0 },
  "security": { "high": 0, "medium": 0, "low": 0, "security_score": 100.0 },
  "churn": { "total_commits": null, "totalChurn": null, "churn_score": null }
}
```

**Note:** Per-file metrics are stored separately in the `file_tree` column and accessed via the file tree endpoint (see API.md).

---

### jobs Table

Tracks async analysis job queue (ARQ integration).

**Purpose**: Manage job lifecycle for async analysis requests

**Schema**:
```sql
CREATE TABLE jobs (
  job_id VARCHAR PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  status VARCHAR NOT NULL,
  source_kind VARCHAR NOT NULL,
  source_metadata JSONB,
  result_run_id UUID REFERENCES analysis_runs(run_id) ON DELETE SET NULL,
  error_message VARCHAR,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX ix_jobs_user_id ON jobs(user_id);
CREATE INDEX ix_jobs_status ON jobs(status);
CREATE INDEX ix_jobs_status_updated ON jobs(status, updated_at);
```

**Columns**:

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `job_id` | VARCHAR | No | Primary key (ARQ-generated unique ID) |
| `user_id` | UUID | No | Foreign key to `users.id` (indexed) |
| `status` | VARCHAR | No | Job status: `pending`, `running`, `completed`, `failed` (indexed) |
| `source_kind` | VARCHAR | No | Source type: `upload` or `github` |
| `source_metadata` | JSONB | Yes | Job-specific data (file base64, repo URL, etc.) |
| `result_run_id` | UUID | Yes | Foreign key to `analysis_runs.run_id` (NULL until completed) |
| `error_message` | VARCHAR | Yes | Error description if status is `failed` |
| `created_at` | TIMESTAMP | No | Job creation timestamp (default: NOW()) |
| `updated_at` | TIMESTAMP | No | Last update timestamp (auto-updated) |

**Constraints**:
- `user_id`: FOREIGN KEY → `users(id)` ON DELETE CASCADE
- `result_run_id`: FOREIGN KEY → `analysis_runs(run_id)` ON DELETE SET NULL
- `status`: Must be `pending`, `running`, `completed`, or `failed`

**Indexes**:
- `ix_jobs_user_id`: Fast lookup by user
- `ix_jobs_status`: Fast filtering by status
- `ix_jobs_status_updated`: Composite index for efficient job cleanup queries

**SQLAlchemy Model**:
```python
# backend/src/infrastructure/database/models/job.py
class Job(Base):
    __tablename__ = "jobs"

    job_id = Column(String, primary_key=True)
    user_id = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False, index=True)
    status = Column(String, nullable=False, index=True)
    source_kind = Column(String, nullable=False)
    source_metadata = Column(JSONB, nullable=True)
    result_run_id = Column(UUID(as_uuid=True), ForeignKey("analysis_runs.run_id", ondelete="SET NULL"), nullable=True)
    error_message = Column(String, nullable=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())

    # Relationships
    user = relationship("User", back_populates="jobs")
    result_analysis = relationship("AnalysisRun", back_populates="job", foreign_keys=[result_run_id])
```

**Job Lifecycle**:
1. **Created** (status: `pending`): Job enqueued to Redis
2. **Picked Up** (status: `running`): ARQ worker processing
3. **Completed** (status: `completed`): `result_run_id` links to analysis
4. **Failed** (status: `failed`): `error_message` populated
5. **Deleted**: Auto-deleted after result retrieval

**Source Metadata Examples**:

**Upload**:
```json
{
  "upload_filename": "example.py",
  "file_data": "base64-encoded-content...",
  "project_name": "My Project",
  "author_type": "human",
  "author_model": "developer"
}
```

**GitHub**:
```json
{
  "git_repo_url": "https://github.com/owner/repo",
  "git_branch": "main",
  "project_name": "Project Analysis",
  "author_type": "ai",
  "author_model": "chatgpt"
}
```

**Access Patterns**:
- Lookup by job_id (status polling)
- List by user_id and status (user's pending jobs)
- Cleanup old jobs (status + updated_at composite index)

---

## Indexes

### Purpose of Indexes

| Index | Table | Columns | Purpose |
|-------|-------|---------|---------|
| `ix_users_email` | users | `email` | Fast user lookup during login/signup (unique) |
| `ix_users_auth0_sub` | users | `auth0_sub` | Fast user lookup on authenticated requests (unique) |
| `ix_analysis_runs_user_id` | analysis_runs | `user_id` | Efficient history queries per user |
| `ix_analysis_runs_created_at` | analysis_runs | `created_at` | Fast sorting by analysis time |
| `ix_jobs_user_id` | jobs | `user_id` | User's job queries |
| `ix_jobs_status` | jobs | `status` | Filter jobs by status |
| `ix_jobs_status_updated` | jobs | `status`, `updated_at` | Job cleanup queries (composite) |

**Naming Convention:** Indexes follow SQLAlchemy's default `ix_<table>_<column>` pattern.

### Index Maintenance

Indexes are automatically maintained by PostgreSQL. No manual intervention required.

## Relationships

### One-to-Many Relationships

**users → analysis_runs**:
- One user can have many analysis runs
- Cascade delete: Deleting a user deletes all their analyses

**users → jobs**:
- One user can have many jobs
- Cascade delete: Deleting a user deletes all their jobs

### Optional One-to-One Relationship

**jobs → analysis_runs**:
- One job optionally links to one analysis run (via `result_run_id`)
- Set NULL on delete: Deleting an analysis doesn't delete the job reference

## Access Control

### Application-Level Authorization

**No Row-Level Security (RLS)**:
- Authorization enforced in application layer (repositories)
- Simpler to debug and maintain
- Flexible for future changes

**Owner-Only Access**:
```python
# Example: Get analysis run with ownership check
def get_analysis(run_id: UUID, user_id: UUID) -> AnalysisRun:
    analysis = db.query(AnalysisRun).filter(AnalysisRun.run_id == run_id).first()
    if not analysis:
        raise NotFoundError()
    if analysis.user_id != user_id:
        raise ForbiddenError("Access denied")
    return analysis
```

**Email Allowlist**:
- Enforced by Auth0 Post-Login Action (before database access)
- See [Authentication Guide](AUTH.md#email-allowlist-configuration)

## Migrations

Vibe8 uses **Alembic** for database schema versioning.

### Migration Commands

```bash
# Create a new migration
alembic revision --autogenerate -m "Description of changes"

# Apply all pending migrations
alembic upgrade head

# Rollback one migration
alembic downgrade -1

# View migration history
alembic history

# View current migration version
alembic current
```

### Migration Workflow

1. **Make model changes** in `backend/src/infrastructure/database/models/`
2. **Generate migration**:
   ```bash
   cd backend
   alembic revision --autogenerate -m "Add author tracking columns"
   ```
3. **Review migration** in `backend/alembic/versions/`
4. **Test migration** on development database:
   ```bash
   alembic upgrade head
   ```
5. **Commit migration file** to git
6. **Deploy** to production (migrations run automatically or manually)

### Migration Best Practices

- Always review auto-generated migrations
- Test migrations on development database first
- Never edit applied migrations (create new migration instead)
- Use descriptive migration messages
- Commit migration files to version control

### Alembic Configuration

```ini
# backend/alembic.ini
[alembic]
script_location = alembic
sqlalchemy.url = driver://user:pass@localhost/dbname  # Overridden by env.py
```

Configuration loaded from environment:
```python
# backend/alembic/env.py
from src.core.config import settings
config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)
```

## Backup and Recovery

### Backup Strategy

**Development**:
```bash
# Backup database
pg_dump -U vibe8 vibe8_dev > backup.sql

# Restore database
psql -U vibe8 vibe8_dev < backup.sql
```

**Production (Google Cloud Platform)**:

For production deployments using containerized PostgreSQL, implement automated backups to Cloud Storage:

```bash
# Automated backup script (schedule with Cloud Scheduler or cron)
pg_dump -U vibe8 vibe8_prod | gzip | gsutil cp - gs://YOUR_BUCKET/backups/backup-$(date +%Y%m%d-%H%M%S).sql.gz

# Restore from backup
gsutil cp gs://YOUR_BUCKET/backups/backup-YYYYMMDD-HHMMSS.sql.gz - | gunzip | psql -U vibe8 vibe8_prod
```

**Note**: If using **Google Cloud SQL** (managed PostgreSQL), backups are automated. See [Cloud SQL Backup Documentation](https://cloud.google.com/sql/docs/postgres/backup-recovery).

**Alternative**: Use Google Kubernetes Engine (GKE) with persistent volumes for containerized deployments with StatefulSets.

### Disaster Recovery

1. **Stop application** to prevent new writes
2. **Restore from latest backup**:
   ```bash
   psql -U vibe8 vibe8_prod < backup.sql
   ```
3. **Run migrations** if schema changed:
   ```bash
   alembic upgrade head
   ```
4. **Restart application**
5. **Verify data integrity**

## Related Documentation

- **[API Reference](API.md)**: Endpoints that interact with database
- **[Architecture Guide](ARCHITECTURE.md)**: System overview and data flows
- **[Design Patterns](DESIGN.md)**: Repository pattern implementation
- **[Authentication Guide](AUTH.md)**: User creation and management
- **[Testing Guide](TESTING.md)**: Testing with database fixtures and repositories

---

**Last Updated**: 2025-01-15
**Platform**: Google Cloud Platform
