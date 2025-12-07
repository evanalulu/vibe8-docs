# Vibe8 API Reference

Complete API documentation for the Vibe8 backend REST API.

## Table of Contents
- [Base URL](#base-url)
- [Authentication](#authentication)
- [Rate Limiting](#rate-limiting)
- [Async Job Architecture](#async-job-architecture)
- [Endpoints](#endpoints)
  - [Health Check](#health-check)
  - [Analysis Endpoints](#analysis-endpoints)
  - [Job Status Endpoints](#job-status-endpoints)
  - [History Endpoints](#history-endpoints)
  - [User Endpoints](#user-endpoints)
  - [Device Auth Endpoints](#device-auth-endpoints)
- [Response Schemas](#response-schemas)
- [Error Handling](#error-handling)

## Base URL

- **Development**: `http://127.0.0.1:8000`
- **Production**: `https://vibe8.me`

## Authentication

All endpoints except `/health` require Auth0 Bearer token authentication.

**Header Format**:
```
Authorization: Bearer <auth0_access_token>
```

**Token Acquisition**:
- Frontend: Use Auth0 Next.js SDK to obtain access token
- Testing: Use Auth0 Management API or frontend login flow

See [Authentication Guide](AUTH.md) for detailed setup instructions.

## Rate Limiting

- **Default Limit**: 60 requests per minute per user
- **Configurable**: Set via `RATE_LIMIT_PER_MINUTE` environment variable
- **Scope**: Per-user (based on authenticated identity)
- **Response on Exceed**: HTTP 429 Too Many Requests

## Async Job Architecture

All analysis endpoints are **asynchronous** to handle long-running tasks without gateway timeouts.

**Workflow**:
1. Submit analysis request → Receive `job_id` (HTTP 202 Accepted)
2. Poll `/api/metrics/jobs/{job_id}` every 1-2 seconds
3. When status is `completed`, retrieve result via `/api/metrics/jobs/{job_id}/result`
4. Job record auto-deletes after result retrieval

**Requirements**:
- Redis server must be running
- ARQ worker must be running (`arq src.application.workers.analysis_worker.WorkerSettings`)

See [Architecture Guide](ARCHITECTURE.md#data-flows) for detailed workflow.

---

## Endpoints

### Health Check

#### `GET /health`

Service readiness check for monitoring.

**Authentication**: Not required

**Response**: `200 OK`
```json
{
  "status": "ok"
}
```

**Example**:
```bash
curl http://127.0.0.1:8000/health
```

---

### Analysis Endpoints

All analysis endpoints require authentication and return job IDs for async processing.

#### `POST /api/metrics/analyze/upload/async`

Enqueue uploaded Python code for async analysis.

**Authentication**: Required (Auth0 Bearer token)

**Content-Type**: `multipart/form-data`

**Parameters**:
| Name | Type | Required | Description |
|------|------|----------|-------------|
| `file` | File | Yes | Single `.py` file or `.zip` archive containing Python files |
| `project_name` | String | No | User-assigned project name for identification |
| `author_type` | String | No | Code authorship type: "human" or "ai" |
| `author_model` | String | No | Code authorship model (e.g., "developer", "chatgpt", "gpt-4", "claude-3.5-sonnet") |

**Constraints**:
- Max file size: Configurable via `MAX_UPLOAD_MB` (default: 5 MB)
- Allowed extensions: `.py`, `.zip`
- ZIP archives must contain at least one `.py` file

**Response**: `202 Accepted`
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "pending",
  "created_at": "2025-01-07T10:30:00Z"
}
```

**Side Effects**:
- Creates job record in `jobs` PostgreSQL table
- Enqueues job to Redis for ARQ worker processing
- File encoded to base64 and stored in job metadata

**Example**:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
     -F "file=@example.py" \
     -F "project_name=My Project" \
     -F "author_type=human" \
     -F "author_model=developer" \
     http://127.0.0.1:8000/api/metrics/analyze/upload/async
```

**Errors**:
- `400 Bad Request`: Invalid file type or missing file
- `401 Unauthorized`: Missing or invalid authentication token
- `413 Request Entity Too Large`: File exceeds size limit
- `429 Too Many Requests`: Rate limit exceeded

---

#### `POST /api/metrics/analyze/github/async`

Enqueue GitHub repository for async analysis.

**Authentication**: Required (Auth0 Bearer token)

**Content-Type**: `multipart/form-data`

**Parameters**:
| Name | Type | Required | Description |
|------|------|----------|-------------|
| `repo_url` | String | Yes | GitHub repository URL (e.g., `https://github.com/owner/repo`) |
| `branch` | String | No | Branch name (defaults to repository's default branch) |
| `project_name` | String | No | User-assigned project name for identification |
| `author_type` | String | No | Code authorship type: "human" or "ai" |
| `author_model` | String | No | Code authorship model (e.g., "developer", "chatgpt", "gpt-4", "claude-3.5-sonnet") |

**Headers**:
| Name | Required | Description |
|------|----------|-------------|
| `X-GitHub-Token` | No | GitHub Personal Access Token for private repositories |

**GitHub Token Requirements**:
- **Fine-grained PAT** (recommended): Minimum permissions: `Contents: Read`, `Metadata: Read`
- **Classic PAT**: `repo` scope (broader access)
- Generate at: https://github.com/settings/tokens
- **Not required** for public repositories

**GitHub Token Security**:
- ✅ **Ephemeral**: Token used only for clone operation, never persisted
- ✅ **No logging**: Token sanitized from all error messages and logs
- ✅ **Memory-only**: Exists only in function scope, garbage collected after use
- ✅ **HTTPS-only**: Transmitted securely via custom header

**Response**: `202 Accepted`
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440001",
  "status": "pending",
  "created_at": "2025-01-07T10:35:00Z"
}
```

**Side Effects**:
- Creates job record in `jobs` PostgreSQL table
- Enqueues job to Redis for ARQ worker processing
- Repository metadata stored in job metadata

**Example (Public Repo)**:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
     -F "repo_url=https://github.com/owner/repo" \
     -F "branch=main" \
     http://127.0.0.1:8000/api/metrics/analyze/github/async
```

**Example (Private Repo)**:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
     -H "X-GitHub-Token: ghp_your_token_here" \
     -F "repo_url=https://github.com/owner/private-repo" \
     http://127.0.0.1:8000/api/metrics/analyze/github/async
```

**Errors**:
- `400 Bad Request`: Invalid GitHub URL format
- `401 Unauthorized`: Missing/invalid authentication token
- `403 Forbidden`: GitHub token lacks required permissions
- `404 Not Found`: Repository doesn't exist or not accessible
- `429 Too Many Requests`: Rate limit exceeded

---

### Job Status Endpoints

#### `GET /api/metrics/jobs/{job_id}`

Poll status of an async analysis job.

**Authentication**: Required (Auth0 Bearer token)

**Authorization**: Owner only (returns 403 if not owner)

**Path Parameters**:
| Name | Type | Description |
|------|------|-------------|
| `job_id` | String | Job identifier returned from async endpoint |

**Response**: `200 OK`
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "running",
  "created_at": "2025-01-07T10:30:00Z",
  "updated_at": "2025-01-07T10:30:15Z"
}
```

**Status Values**:
- `pending`: Job queued, not yet picked up by worker
- `running`: Worker currently processing job
- `completed`: Analysis finished successfully (retrieve result)
- `failed`: Analysis failed (retrieve error message)

**Polling Recommendation**: Poll every 1-2 seconds until status is `completed` or `failed`.

**Example**:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
     http://127.0.0.1:8000/api/metrics/jobs/{job_id}
```

**Errors**:
- `401 Unauthorized`: Missing/invalid authentication token
- `403 Forbidden`: User is not the job owner
- `404 Not Found`: Job doesn't exist

---

#### `GET /api/metrics/jobs/{job_id}/result`

Retrieve results of a completed job (auto-deletes job record).

**Authentication**: Required (Auth0 Bearer token)

**Authorization**: Owner only (returns 403 if not owner)

**Path Parameters**:
| Name | Type | Description |
|------|------|-------------|
| `job_id` | String | Job identifier returned from async endpoint |

**Response (Pending/Running)**: `200 OK`
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "running",
  "result": null,
  "error_message": null
}
```

**Response (Completed)**: `200 OK`
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "completed",
  "result": {
    "run_id": "650e8400-e29b-41d4-a716-446655440000",
    "created_at": "2025-01-07T10:30:45Z",
    "source": {
      "kind": "upload",
      "author_type": "human",
      "author_model": "developer",
      "upload_filename": "example.py",
      "project_name": "My Project"
    },
    "metrics": {
      "lint": { "total": 5, "issues_per_100_loc": 1.5, "linting_score": 92.5 },
      "cyclomatic": { "avg_cc": 3.5, "complexity_score": 84.0 },
      "docs": { "coverage_pct": 75.0 },
      "dead_code": { "unused_symbols": 2 },
      "deps": { "total_modules": 5, "package_health": [...] },
      "mi": { "mi_avg": 65.5 },
      "size": { "files": 1, "sloc": 150, "avg_file_sloc": 150 },
      "dupes": { "duplicate_lines": 10, "duplicate_pct": 12.5, "duplication_score": 87.5 },
      "security": { "high": 0, "medium": 1, "low": 2, "security_score": 55.0 },
      "churn": { "total_commits": 125, "totalChurn": 11700, "churn_score": 0.0 }
    },
    "errors": []
  },
  "error_message": null
}
```

**Response (Failed)**: `200 OK`
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "failed",
  "result": null,
  "error_message": "Failed to clone repository: Authentication required"
}
```

**Side Effects**:
- **Auto-deletes job record** from database after retrieval (one-time access)
- For completed jobs: Analysis result remains in `analysis_runs` table

**Example**:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
     http://127.0.0.1:8000/api/metrics/jobs/{job_id}/result
```

**Errors**:
- `401 Unauthorized`: Missing/invalid authentication token
- `403 Forbidden`: User is not the job owner
- `404 Not Found`: Job doesn't exist (may have been deleted after previous retrieval)

---

### History Endpoints

#### `GET /api/metrics/history`

Retrieve all analysis runs for the authenticated user.

**Authentication**: Required (Auth0 Bearer token)

**Query Parameters**:
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `limit` | Integer | No | 10 | Maximum number of results to return |

**Response**: `200 OK`
```json
{
  "analyses": [
    {
      "run_id": "650e8400-e29b-41d4-a716-446655440000",
      "user_id": "auth0|123456",
      "created_at": "2025-01-07T10:30:45Z",
      "upload_filename": "example.py",
      "project_name": "My Project",
      "submission_type": "upload",
      "author_type": "human",
      "author_model": "developer",
      "metrics": { /* Full metrics object */ },
      "errors": []
    },
    {
      "run_id": "650e8400-e29b-41d4-a716-446655440001",
      "user_id": "auth0|123456",
      "created_at": "2025-01-06T15:20:30Z",
      "git_repo_url": "https://github.com/owner/repo",
      "git_branch": "main",
      "git_commit_sha": "abc123def456",
      "project_name": "Research Project",
      "submission_type": "github",
      "author_type": "ai",
      "author_model": "chatgpt",
      "metrics": { /* Full metrics object */ },
      "errors": []
    }
  ]
}
```

**Ordering**: Results ordered by `created_at` descending (most recent first)

**Example**:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
     "http://127.0.0.1:8000/api/metrics/history?limit=20"
```

**Errors**:
- `401 Unauthorized`: Missing/invalid authentication token

---

#### `GET /api/metrics/history/{run_id}`

Retrieve detailed results for a specific analysis run.

**Authentication**: Required (Auth0 Bearer token)

**Authorization**: Owner only (returns 403 if not owner)

**Path Parameters**:
| Name | Type | Description |
|------|------|-------------|
| `run_id` | UUID | Analysis run identifier |

**Response**: `200 OK`
```json
{
  "run_id": "650e8400-e29b-41d4-a716-446655440000",
  "user_id": "auth0|123456",
  "created_at": "2025-01-07T10:30:45Z",
  "upload_filename": "example.py",
  "submission_type": "upload",
  "project_name": "My Project",
  "author_type": "human",
  "author_model": "developer",
  "metrics": {
    "lint": { /* ... */ },
    "cyclomatic": { /* ... */ },
    "docs": { /* ... */ },
    "dead_code": { /* ... */ },
    "deps": { /* ... */ },
    "mi": { /* ... */ },
    "size": { /* ... */ },
    "dupes": { /* ... */ },
    "security": { /* ... */ }
  },
  "errors": []
}
```

**Example**:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
     http://127.0.0.1:8000/api/metrics/history/650e8400-e29b-41d4-a716-446655440000
```

**Errors**:
- `401 Unauthorized`: Missing/invalid authentication token
- `403 Forbidden`: User is not the analysis owner
- `404 Not Found`: Analysis run doesn't exist

---

#### `DELETE /api/metrics/history/{run_id}`

Permanently delete a specific analysis run.

**Authentication**: Required (Auth0 Bearer token)

**Authorization**: Owner only (returns 403 if not owner)

**Path Parameters**:
| Name | Type | Description |
|------|------|-------------|
| `run_id` | UUID | Analysis run identifier |

**Response**: `204 No Content`

**Side Effects**:
- Permanently removes analysis run from `analysis_runs` table
- Cannot be undone

**Example**:
```bash
curl -X DELETE \
     -H "Authorization: Bearer YOUR_TOKEN" \
     http://127.0.0.1:8000/api/metrics/history/650e8400-e29b-41d4-a716-446655440000
```

**Errors**:
- `401 Unauthorized`: Missing/invalid authentication token
- `403 Forbidden`: User is not the analysis owner
- `404 Not Found`: Analysis run doesn't exist
- `500 Internal Server Error`: Database deletion failed

---

#### `GET /api/metrics/file-tree/{run_id}`

Retrieve file structure visualization with per-file metrics and severity scores for a specific analysis run.

**Note:** This is the **only** endpoint that provides per-file metrics. All other analysis responses contain aggregate metrics only.

**Authentication**: Required (Auth0 Bearer token)

**Authorization**: Owner only (returns 403 if not owner)

**Path Parameters**:
| Name | Type | Description |
|------|------|-------------|
| `run_id` | UUID | Analysis run identifier |

**Response**: `200 OK`
```json
{
  "run_id": "650e8400-e29b-41d4-a716-446655440000",
  "created_at": "2025-11-19T10:30:00Z",
  "source_info": {
    "kind": "upload",
    "author_type": "human",
    "author_model": "developer",
    "upload_filename": "example.py",
    "project_name": "My Project"
  },
  "root_directory": {
    "name": "root",
    "type": "directory",
    "path": "",
    "children": [
      {
        "name": "src",
        "type": "directory",
        "path": "src",
        "children": [
          {
            "name": "main.py",
            "type": "file",
            "path": "src/main.py",
            "extension": ".py",
            "size": 2048,
            "metrics": {
              "complexity": {"value": 15.0, "severity": "high"},
              "lint": {"count": 8, "severity": "high"},
              "security": {"high": 1, "medium": 2, "low": 0, "severity": "high"},
              "docs": {"value": 75.0, "severity": "medium"},
              "maintainability": {"value": 65.0, "severity": "medium"},
              "dead_code": {"count": 0, "severity": "low"},
              "duplication": {"count": 0, "severity": "low"},
              "health_score": 58
            }
          },
          {
            "name": "utils.py",
            "type": "file",
            "path": "src/utils.py",
            "extension": ".py",
            "size": 1024,
            "metrics": {
              "complexity": {"value": 7.0, "severity": "medium"},
              "lint": {"count": 2, "severity": "low"},
              "security": {"high": 0, "medium": 0, "low": 1, "severity": "low"},
              "docs": {"value": 85.0, "severity": "low"},
              "maintainability": {"value": 80.0, "severity": "low"},
              "dead_code": {"count": 0, "severity": "low"},
              "duplication": {"count": 0, "severity": "low"},
              "health_score": 82
            }
          }
        ]
      },
      {
        "name": "tests",
        "type": "directory",
        "path": "tests",
        "children": [
          {
            "name": "test_main.py",
            "type": "file",
            "path": "tests/test_main.py",
            "extension": ".py",
            "size": 512,
            "metrics": {
              "complexity": {"value": 3.0, "severity": "low"},
              "lint": {"count": 0, "severity": "low"},
              "security": {"high": 0, "medium": 0, "low": 0, "severity": "low"},
              "docs": {"value": 100.0, "severity": "low"},
              "maintainability": {"value": 90.0, "severity": "low"},
              "dead_code": {"count": 0, "severity": "low"},
              "duplication": {"count": 0, "severity": "low"},
              "health_score": 95
            }
          }
        ]
      }
    ]
  }
}
```

**Severity Levels**: `low`, `medium`, `high`

Severity is calculated automatically from metric values using a weighted scoring system (see `backend/src/infrastructure/metrics/severity/`). Each file receives per-metric severity scores and an overall health score (0-100).

**Health Score**: Weighted aggregate of all file metrics, where 100 is perfect and lower scores indicate more issues.

**Example**:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
     http://127.0.0.1:8000/api/metrics/file-tree/650e8400-e29b-41d4-a716-446655440000
```

**Errors**:
- `401 Unauthorized`: Missing/invalid authentication token
- `403 Forbidden`: User is not the analysis owner
- `404 Not Found`: Analysis run doesn't exist

---

### User Endpoints

#### `GET /me`

Get current authenticated user information.

**Authentication**: Required (Auth0 Bearer token)

**Response**: `200 OK`
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com"
}
```

**Example**:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
     http://127.0.0.1:8000/me
```

**Errors**:
- `401 Unauthorized`: Missing/invalid authentication token

---

### Device Auth Endpoints

These endpoints support OAuth Device Flow authentication for the MCP server.

#### `POST /api/auth/device/code`

Request a device code for OAuth Device Flow authentication.

**Authentication**: Not required

**Response**: `200 OK`
```json
{
  "device_code": "ABC123...",
  "user_code": "ABCD-1234",
  "verification_uri": "https://your-tenant.auth0.com/activate",
  "verification_uri_complete": "https://your-tenant.auth0.com/activate?user_code=ABCD-1234",
  "expires_in": 900,
  "interval": 5
}
```

**Example**:
```bash
curl -X POST http://127.0.0.1:8000/api/auth/device/code
```

---

#### `POST /api/auth/device/token`

Poll for access token after user completes authorization.

**Authentication**: Not required

**Request Body**:
```json
{
  "device_code": "ABC123..."
}
```

**Response (Pending)**: `200 OK`
```json
{
  "status": "pending"
}
```

**Response (Success)**: `200 OK`
```json
{
  "access_token": "eyJ...",
  "refresh_token": "v1...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**Example**:
```bash
curl -X POST http://127.0.0.1:8000/api/auth/device/token \
     -H "Content-Type: application/json" \
     -d '{"device_code": "ABC123..."}'
```

---

#### `POST /api/auth/device/refresh`

Refresh an expired access token using a refresh token.

**Authentication**: Not required

**Request Body**:
```json
{
  "refresh_token": "v1..."
}
```

**Response**: `200 OK`
```json
{
  "access_token": "eyJ...",
  "refresh_token": "v1...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**Example**:
```bash
curl -X POST http://127.0.0.1:8000/api/auth/device/refresh \
     -H "Content-Type: application/json" \
     -d '{"refresh_token": "v1..."}'
```

---

## Response Schemas

### MetricsResponseSummary

Complete analysis result structure returned by `/api/metrics/jobs/{job_id}/result` endpoint:

```json
{
  "run_id": "UUID",
  "created_at": "ISO-8601 timestamp",
  "source": {
    "kind": "upload | github",
    "author_type": "string (nullable) - 'human' or 'ai'",
    "author_model": "string (nullable) - e.g., 'developer', 'chatgpt', 'gpt-4'",
    "upload_filename": "string (nullable, upload only)",
    "project_name": "string (nullable)",
    "git_repo_url": "string (nullable, github only)",
    "git_branch": "string (nullable, github only)",
    "git_commit_sha": "string (nullable, github only)"
  },
  "metrics": {
    "lint": {
      "total": "integer",
      "issues_per_100_loc": "float (transient, not persisted)",
      "linting_score": "float (0-100, higher=better)"
    },
    "cyclomatic": {
      "avg_cc": "float",
      "complexity_score": "float (0-100, higher=better)"
    },
    "docs": {
      "coverage_pct": "float"
    },
    "dead_code": {
      "unused_symbols": "integer"
    },
    "deps": {
      "total_modules": "integer",
      "package_health": [
        {
          "package": "string",
          "installed": "boolean",
          "outdated": "boolean",
          "current_version": "string | null",
          "latest_version": "string | null"
        }
      ]
    },
    "mi": {
      "mi_avg": "float"
    },
    "size": {
      "files": "integer",
      "sloc": "integer",
      "avg_file_sloc": "integer"
    },
    "dupes": {
      "duplicate_lines": "integer",
      "duplicate_pct": "float (calculated)",
      "duplication_score": "float (0-100, higher=better)"
    },
    "security": {
      "high": "integer",
      "medium": "integer",
      "low": "integer",
      "security_score": "float (0-100, higher=better)"
    },
    "churn": {
      "total_commits": "integer | null",
      "totalChurn": "integer | null (sum of lines added + deleted)",
      "churn_score": "float (0-100, higher=better)"
    }
  },
  "errors": ["string"]
}
```

**Note on Calculated vs Persisted Fields**:
- `issues_per_100_loc` is calculated at response time (not stored in database)
- All other score fields are persisted in the database JSONB column
- **Per-file metrics** are stored separately in the `file_tree` column and accessed via `GET /api/metrics/file-tree/{run_id}`

### JobResponse

Job status structure:

```json
{
  "job_id": "string",
  "status": "pending | running | completed | failed",
  "created_at": "ISO-8601 timestamp",
  "updated_at": "ISO-8601 timestamp",
  "error_message": "string (nullable)"
}
```

### JobResultResponse

Job result structure:

```json
{
  "job_id": "string",
  "status": "pending | running | completed | failed",
  "result": "MetricsResponse | null",
  "error_message": "string | null"
}
```

---

## Error Handling

### HTTP Status Codes

| Code | Meaning | Description |
|------|---------|-------------|
| 200 | OK | Request succeeded |
| 202 | Accepted | Async job created successfully |
| 204 | No Content | Deletion succeeded |
| 400 | Bad Request | Invalid request parameters |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 413 | Request Entity Too Large | File exceeds size limit |
| 422 | Unprocessable Entity | Validation error |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server-side error |

### Error Response Format

```json
{
  "detail": "Human-readable error message"
}
```

### Common Error Scenarios

**Authentication Errors**:
```json
{
  "detail": "Not authenticated"
}
```

**Authorization Errors**:
```json
{
  "detail": "Access denied: You don't have permission to access this resource"
}
```

**Validation Errors**:
```json
{
  "detail": [
    {
      "loc": ["body", "file"],
      "msg": "field required",
      "type": "value_error.missing"
    }
  ]
}
```

**Rate Limit Errors**:
```json
{
  "detail": "Rate limit exceeded"
}
```

### Analysis Errors

Even on successful analysis (HTTP 200), individual metric collectors may fail. Check the `errors` array:

```json
{
  "metrics": { /* ... */ },
  "errors": [
    "Bandit security scan failed: Tool not installed",
    "PMD CPD duplicate detection failed: Out of memory"
  ]
}
```

---

## Related Documentation

- **[Architecture Guide](ARCHITECTURE.md)**: System overview and data flows
- **[Authentication Guide](AUTH.md)**: Auth0 setup and token management
- **[Database Schema](DATABASE.md)**: PostgreSQL tables for jobs and analyses
- **[Metrics Reference](METRICS.md)**: Detailed metric specifications
- **[MCP Server Guide](MCP.md)**: MCP server for Claude Desktop integration

---

**Last Updated**: 2025-01-07
