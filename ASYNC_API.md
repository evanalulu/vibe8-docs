# Async Job Queue API

Complete guide to Vibe8's asynchronous job queue architecture using ARQ (Async Result Queue) with Redis.

## Overview

The API uses async architecture to handle long-running code analysis tasks (5-30 seconds) without blocking HTTP requests or risking gateway timeouts.

**Flow**: Submit job → Get job_id → Poll status → Fetch result

## Architecture

### Components

| Component | Role | Technology |
|-----------|------|------------|
| **Frontend** | Submit jobs, poll status | React + TypeScript |
| **Backend API** | Enqueue jobs, serve results | FastAPI (port 8000) |
| **Redis** | Message queue | Redis 7 (port 6379) |
| **ARQ Worker** | Process analysis jobs | Python async worker |
| **PostgreSQL** | Store job metadata & results | PostgreSQL 15 |

### Request Flow

```
1. POST /api/metrics/analyze/upload/async
   → Returns: {job_id, status: "pending"} (HTTP 202)

2. GET /api/metrics/jobs/{job_id}  (poll every 1s)
   → Returns: {status: "pending|running|completed|failed"}

3. GET /api/metrics/jobs/{job_id}/result
   → Returns: {result: {...metrics...}}
   → Job deleted after retrieval
```

## Implementation Details

### 1. Job Submission

**Endpoint**: `POST /api/metrics/analyze/upload/async`

**Process**:
1. Validate file (size, type)
2. Check user limits (5 concurrent, 100 daily)
3. Encode file to base64
4. Enqueue to Redis
5. Create job record in PostgreSQL
6. Return job_id immediately

### 2. Worker Processing

**Command**: `arq src.application.workers.analysis_worker.WorkerSettings`

**Configuration**: Worker settings are loaded from environment variables (see [config.py](../backend/src/core/config.py)):
- `ARQ_MAX_JOBS` (default: 20) - Concurrent job capacity per worker
- `ARQ_JOB_TIMEOUT` (default: 600) - Max seconds per job
- `ARQ_POLL_DELAY` (default: 0.1) - Queue polling interval in seconds
- `ARQ_KEEP_RESULT` (default: 3600) - Result retention in seconds

**Process**:
1. Dequeue job from Redis
2. Update status to "running"
3. Run 10 metric collectors (Ruff, Radon, Vulture, Bandit, GitPython, etc.)
4. Store results in `analysis_runs` table
5. Update job status to "completed"
6. Clean up temp files

### 3. Status Polling

**Frontend Hook**: `useJobPolling` polls every 1 second

**Race Condition Protection**:
- `isFetchingResultRef` prevents concurrent fetches
- Polling stops before fetching result
- 404 errors handled gracefully

### 4. Result Retrieval

**Important**: Job is **deleted** after first successful retrieval. Client must cache the result.

## DoS Protection

Per-user limits prevent queue exhaustion:

| Limit Type | Default | Config Variable |
|------------|---------|-----------------|
| Concurrent jobs | 5 | `MAX_CONCURRENT_JOBS_PER_USER` |
| Daily jobs | 100 | `DAILY_JOB_LIMIT_PER_USER` |
| IP rate limit | 60/min | `RATE_LIMIT_PER_MINUTE` |

**Enforcement**: Dependency in [jobs.py](../backend/src/api/routes/jobs.py) checks limits before enqueuing.

**Response** (HTTP 429):
```json
{
  "detail": "Too many concurrent jobs. Maximum 5 jobs allowed at once."
}
```

## Automated Cleanup

**Cron Schedule**: Every 6 hours (midnight, 6am, noon, 6pm)

**Cleanup Rules**:
- Completed/failed jobs: Deleted after 6 hours (`COMPLETED_JOB_RETENTION`)
- Stale pending/running: Deleted after 2 hours (`STALE_JOB_THRESHOLD`)

**Why**: Prevents database bloat from abandoned jobs and crashed workers.

**Manual Cleanup**:
```bash
cd backend
source .venv/bin/activate
python -c "from src.infrastructure.database.repositories.job import JobRepository; \
from src.infrastructure.database.connection import SessionLocal; \
db = SessionLocal(); \
deleted = JobRepository.cleanup_stale_jobs(21600, 7200, db=db); \
print(f'Deleted {deleted} jobs'); \
db.close()"
```

## Redis Persistence

**Risk**: If Redis crashes, all queued jobs are lost unless persistence is configured.

**Production Configuration** (redis.conf):
```bash
# Hybrid mode (AOF + RDB)
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes

# RDB snapshots (backup strategy)
save 900 1    # After 15 min if 1+ key changed
save 300 10   # After 5 min if 10+ keys changed
save 60 10000 # After 1 min if 10000+ keys changed
```

**Production**: Redis runs as a Docker container with AOF persistence. See [DEPLOYMENT.md](DEPLOYMENT.md) for setup instructions.

## Worker Scaling

**Current**: Single worker, configurable concurrent jobs (default: 20 via `ARQ_MAX_JOBS`)

**Scaling Formula**:
```
Total capacity = workers × ARQ_MAX_JOBS
1 worker  = 20 jobs (default)
3 workers = 60 jobs (default)
10 workers = 200 jobs (default)
```

**Deployment Options**:
- Multiple terminals: `arq src.application.workers.analysis_worker.WorkerSettings` (x N)
- Process manager: systemd, supervisord, Docker Compose
- Container orchestration: Kubernetes, ECS

**Note**: 20 concurrent jobs are async tasks in a single process, not OS threads.

## Database Schema

**Three-layer architecture**:

| Table | Purpose | Lifetime |
|-------|---------|----------|
| **Redis queue** | Job execution | Ephemeral (1h TTL) |
| **jobs** table | Authorization & tracking | Deleted after retrieval |
| **analysis_runs** table | Permanent results | Persistent (history) |

**Why all three?**
- Redis: Fast queue, no authorization
- Jobs table: Owner verification, API access after Redis expiry
- Analysis runs: Permanent storage, user history

## Configuration

**Environment Variables** ([config.py](../backend/src/core/config.py)):

```bash
# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0

# ARQ Worker Configuration
ARQ_MAX_JOBS=20           # Concurrent job capacity
ARQ_JOB_TIMEOUT=600       # Max seconds per job
ARQ_KEEP_RESULT=3600      # Result retention (seconds)
ARQ_POLL_DELAY=0.1        # Queue polling interval (seconds)

# Job Queue Cleanup
COMPLETED_JOB_RETENTION=21600  # 6 hours
STALE_JOB_THRESHOLD=7200       # 2 hours

# User Limits (DoS Protection)
MAX_CONCURRENT_JOBS_PER_USER=5
DAILY_JOB_LIMIT_PER_USER=100
```

## Running the System

**Required Services** (all must be running):

```bash
# 1. Redis
redis-server

# 2. PostgreSQL
# (via Docker or native install)

# 3. ARQ Worker
cd backend && source .venv/bin/activate
arq src.application.workers.analysis_worker.WorkerSettings

# 4. Backend API
cd backend && source .venv/bin/activate
uvicorn src.api.main:app --reload

# 5. Frontend
cd frontend && npm run dev
```

## Operational Metrics

**Monitor**:
- Queue depth: `redis-cli LLEN arq:queue`
- Worker CPU/memory usage
- Job completion vs. failure rate
- Average job duration
- Cleanup count (orphaned job indicator)

**Logging**:
```bash
# Worker activity
grep "Cleanup complete" worker.log

# User limit violations
grep "exceeded.*limit" api.log
```

## Testing

**Manual Test**:
```bash
# 1. Upload file
curl -X POST http://localhost:8000/api/metrics/analyze/upload/async \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@test.py"
# Returns: {"job_id": "abc123", "status": "pending"}

# 2. Poll status (repeat until completed)
curl http://localhost:8000/api/metrics/jobs/abc123 \
  -H "Authorization: Bearer $TOKEN"
# Returns: {"status": "running"}

# 3. Get result
curl http://localhost:8000/api/metrics/jobs/abc123/result \
  -H "Authorization: Bearer $TOKEN"
# Returns: Full analysis metrics
```

**Integration Tests**:
```bash
cd backend
pytest tests/integration/api/test_async_endpoints.py -v
```

## FAQ

**Q: Why ARQ instead of Celery?**
A: Simpler, Python-native async, sufficient for our needs.

**Q: Why 1-second polling instead of WebSockets?**
A: Simpler implementation, works with any HTTP client, adequate for 1-30s jobs.

**Q: Can I retrieve a job result twice?**
A: No. Job is deleted after first successful retrieval. Client must cache result.

**Q: What happens if a worker crashes mid-job?**
A: ARQ uses "pessimistic execution" - jobs stay in queue until completion. Crashed jobs automatically retry when worker restarts. Tasks are idempotent (safe to rerun).

**Q: How does cleanup prevent database bloat?**
A: Without cleanup, jobs from users who never retrieve results accumulate indefinitely. Cron job removes abandoned and stale jobs every 6 hours.

**Q: Why separate jobs and analysis_runs tables?**
A: Jobs are ephemeral (tracking), analysis_runs are permanent (history). Clean separation of concerns.

---

**Updated**: 2025-11-13
