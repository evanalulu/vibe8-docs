# Vibe8 Test Suite Documentation

**Purpose:** Documentation of the Vibe8 test suite (backend and frontend).

**Last Updated:** December 4, 2025

---

## Table of Contents
- [Overview](#overview)
- [Backend Tests](#backend-tests)
  - [Test Organization](#test-organization)
  - [Running Backend Tests](#running-backend-tests)
  - [Fixtures](#fixtures)
  - [Test Helpers](#test-helpers)
  - [Integration Test Setup](#integration-test-setup)
- [Frontend Tests](#frontend-tests)
  - [Test Organization](#frontend-test-organization)
  - [Running Frontend Tests](#running-frontend-tests)
  - [Testing Tools](#testing-tools)
- [CI/CD Integration](#cicd-integration)

---

## Overview

The Vibe8 codebase includes comprehensive test coverage across both backend (Python/FastAPI) and frontend (React/Next.js) layers.

### Quick Stats

**Backend:**
- **48 test files** across unit, integration, and e2e categories
- **618 total tests** (as of December 4, 2025)
- **43 unit test files** - Fast tests with mocked external dependencies
- **4 integration test files** - Tests with real PostgreSQL and Redis
- **1 e2e test file** - Complete user workflow testing
- **Current coverage:** 90.94% (exceeds 85% minimum enforced in CI)

**Frontend:**
- **3 test files** (component tests)
- **Test framework:** Vitest + React Testing Library
- **Location:** `frontend/components/dashboard/results-table/table/`

### Technology Stack

**Backend:**
- pytest - Test framework
- pytest-asyncio - Async test support
- responses - HTTP mock library
- SQLAlchemy - Database ORM (real DB in tests)
- FastAPI TestClient - API testing
- cryptography + jose - JWT token generation
- unittest.mock - Mocking utilities

**Frontend:**
- Vitest - Test framework and runner
- @testing-library/react - Component testing utilities
- @testing-library/jest-dom - DOM matchers
- @testing-library/user-event - User interaction simulation
- jsdom - DOM environment

---

## Backend Tests

### Test Organization

The backend test suite is organized into three categories, mirroring the `src/` directory structure:

```
backend/tests/
├── conftest.py                               # Global fixtures and configuration
├── helpers/                                  # Reusable test utilities
│   ├── __init__.py
│   ├── api.py                                # API request helpers
│   ├── auth.py                               # JWT token generation
│   ├── database.py                           # DB record creation
│   └── files.py                              # Python file/project generation
├── unit/                                     # 43 test files
│   ├── api/
│   │   ├── auth/
│   │   │   └── test_auth.py                  # Auth dependency tests
│   │   ├── routes/
│   │   │   ├── test_analysis.py              # Analysis endpoints
│   │   │   ├── test_auth.py                  # Auth endpoints
│   │   │   ├── test_device_auth.py           # Device flow endpoints
│   │   │   └── test_health.py                # Health check
│   │   ├── validators/
│   │   │   └── test_limits.py                # User limit validation
│   │   ├── test_dependencies.py              # Dependency injection
│   │   ├── test_rate_limiting.py             # Rate limiting behavior
│   │   └── test_main.py                      # FastAPI app config
│   ├── application/
│   │   ├── auth/
│   │   │   ├── test_auth.py                  # Auth utilities
│   │   │   └── test_auth0.py                 # Auth0 JWT validation
│   │   └── workers/
│   │       ├── processors/
│   │       │   ├── test_github.py            # GitHub processor
│   │       │   └── test_upload.py            # Upload processor
│   │       ├── test_analysis_worker.py       # ARQ worker
│   │       └── test_orchestrator.py          # Metrics orchestrator
│   ├── core/
│   │   └── test_config.py                    # Configuration management
│   ├── domain/
│   │   ├── analysis/
│   │   │   └── test_schemas.py               # Analysis Pydantic schemas
│   │   ├── auth/
│   │   │   └── test_schemas.py               # Auth Pydantic schemas
│   │   └── jobs/
│   │       └── test_schemas.py               # Job Pydantic schemas
│   ├── infrastructure/
│   │   ├── database/
│   │   │   ├── repositories/
│   │   │   │   ├── test_analysis_repository.py
│   │   │   │   ├── test_job_repository.py
│   │   │   │   └── test_user_repository.py
│   │   │   ├── test_connection.py            # Database connection
│   │   │   ├── test_job_cleanup.py           # Job cleanup cron
│   │   │   └── test_job_counting.py          # Job counting utilities
│   │   ├── github/
│   │   │   └── test_github_utils.py          # GitHub validation/cloning
│   │   └── metrics/
│   │       ├── test_base.py                  # MetricsCollector base class
│   │       ├── test_cc_runner.py             # Cyclomatic complexity (Radon)
│   │       ├── test_churn_runner.py          # Git churn (GitPython)
│   │       ├── test_deadcode_runner.py       # Dead code detection (Vulture)
│   │       ├── test_docs_runner.py           # Docs coverage (docstr_coverage)
│   │       ├── test_dupes_runner.py          # Duplicate code (PMD CPD)
│   │       ├── test_filetree_edge_cases.py   # File tree edge cases
│   │       ├── test_filetree_generation.py   # File tree generation
│   │       ├── test_filetree_metrics.py      # File tree metrics
│   │       ├── test_imports_runner.py        # Import dependencies (AST)
│   │       ├── test_mi_runner.py             # Maintainability Index (Radon)
│   │       ├── test_ruff_runner.py           # Linting (Ruff)
│   │       ├── test_security_runner.py       # Security scanning (Bandit)
│   │       ├── test_severity_utils.py        # Severity calculations
│   │       └── test_size_runner.py           # Size metrics (Radon)
│   └── mcp/
│       ├── test_auth.py                      # MCP OAuth device flow
│       ├── test_server.py                    # MCP server
│       ├── test_token_store.py               # Token persistence
│       └── test_tools.py                     # MCP tools
├── integration/                              # 4 test files
│   ├── api/
│   │   ├── test_async_endpoints.py           # Async job endpoints (real Redis)
│   │   ├── test_file_tree_endpoint.py        # File tree endpoint (real DB)
│   │   └── test_retrieval_with_db.py         # Analysis retrieval (real DB)
│   └── workers/
│       └── test_worker_integration.py        # ARQ worker with real Redis
└── e2e/                                      # 1 test file
    └── test_complete_analysis_workflow.py    # Complete user workflows
```

### Test Categories

#### Unit Tests (43 files)
- **Purpose:** Test isolated components with mocked external dependencies
- **Marker:** `@pytest.mark.unit` or `pytestmark = pytest.mark.unit`
- **Speed:** Very fast (<5 seconds total)
- **Dependencies:** Uses real PostgreSQL via transactional sessions (auto-rollback), mocks external services (Auth0, Redis, GitHub)

**Key areas covered:**
- **API routes** - All FastAPI endpoints (analysis, auth, jobs, health)
- **Auth** - JWT validation, Auth0 integration, dependency injection
- **Workers** - ARQ worker, metrics orchestrator, upload/GitHub processors
- **Metrics collectors** - All 10 metric runners (ruff, cc, docs, deadcode, imports, mi, size, dupes, security, churn)
- **Database repositories** - User, analysis run, job repositories
- **File tree generation** - Tree structure, metrics integration, edge cases
- **Schemas** - Pydantic model validation
- **MCP server** - Claude Desktop integration

#### Integration Tests (4 files)
- **Purpose:** Test component interactions with real infrastructure
- **Marker:** `@pytest.mark.integration` or `pytestmark = pytest.mark.integration`
- **Speed:** Slower (~30 seconds)
- **Dependencies:** Real PostgreSQL + real Redis (Docker container or CI service)

**Test files:**
- `test_async_endpoints.py` - Async job endpoints with real Redis queue
- `test_file_tree_endpoint.py` - File tree endpoint with real database
- `test_retrieval_with_db.py` - Analysis retrieval with real database
- `test_worker_integration.py` - ARQ worker with real Redis

#### E2E Tests (1 file)
- **Purpose:** Complete user workflows end-to-end
- **Marker:** `@pytest.mark.e2e` or `pytestmark = pytest.mark.e2e`
- **Speed:** Slowest (~60 seconds)
- **Dependencies:** Full stack (PostgreSQL + Redis + FastAPI)

**Test file:**
- `test_complete_analysis_workflow.py` - Upload → enqueue → process → retrieve

### Running Backend Tests

All commands run from `backend/` directory with virtual environment activated:

```bash
cd backend && source .venv/bin/activate
```

#### Run All Tests
```bash
pytest
```

Default configuration (from `pytest.ini`) includes:
- Verbose output (`-v`)
- Strict marker checking (`--strict-markers`)
- Coverage reporting (`--cov=src --cov-report=term-missing --cov-report=xml`)

#### Run by Category
```bash
pytest -m unit                    # Unit tests only (~5 seconds)
pytest -m integration             # Integration tests only (~30 seconds)
pytest -m e2e                     # E2E tests only (~60 seconds)
pytest -m "unit or integration"   # Both unit and integration
```

#### Run Specific Tests
```bash
# By directory
pytest tests/unit/api/                                    # All API tests
pytest tests/unit/infrastructure/metrics/                 # All metrics tests
pytest tests/integration/                                 # All integration tests

# By file
pytest tests/unit/infrastructure/metrics/test_ruff_runner.py

# By test function
pytest tests/unit/api/routes/test_health.py::test_health_check_returns_200
```

#### Coverage Reports
```bash
# Terminal report with missing lines (default)
pytest

# HTML report (detailed line-by-line coverage)
pytest --cov=src --cov-report=html
open htmlcov/index.html

# Fail if coverage below threshold
pytest --cov-fail-under=85
```

#### Useful Options
```bash
pytest -v                      # Verbose output (show test names)
pytest -vv                     # Extra verbose (full diffs)
pytest -x                      # Stop on first failure
pytest -k "test_name"          # Run tests matching pattern
pytest --tb=short              # Short traceback format
pytest --pdb                   # Drop into debugger on failure
pytest -s                      # Show print() output
pytest --durations=10          # Show 10 slowest tests
```

### Fixtures

All fixtures are defined in `tests/conftest.py` (consolidated root conftest).

#### Environment Setup
- `pytest_configure()` - Sets default test environment variables (rate limits, DB URL, Auth0 config)

#### Database Fixtures
- **`test_db_engine`** (session scope)
  - Creates PostgreSQL engine for entire test session
  - Creates all tables via SQLAlchemy metadata
  - Drops all tables after tests complete

- **`db_session`** (function scope)
  - Provides clean transactional database session for each test
  - Automatically rolls back changes after test
  - Used in most unit tests for database access

#### Auth Fixtures
- **`test_user_jwt`** (function scope, depends on `mock_auth0_jwks`)
  - Generates real JWT token for test user
  - Creates actual user record in database (not transactional)
  - Returns `TokenWithUser` object with `.token`, `.user_id`, `.auth0_sub` attributes
  - Auto-cleanup: deletes user and related records after test
  - Used in integration tests for authenticated requests

#### Client Fixtures
- **`client_no_auth`** (function scope)
  - FastAPI TestClient with auth dependency overridden
  - Returns mock user without token validation
  - Used in unit tests of API routes

- **`client_integration`** (function scope)
  - FastAPI TestClient with real auth
  - Requires real JWT tokens (use with `test_user_jwt`)
  - Used in integration tests

- **`test_client`** (function scope)
  - Alias for `client_no_auth` (backward compatibility)

- **`client_with_low_rate_limit`** (function scope)
  - Dedicated test app with SlowAPI rate limiter (3 requests/minute)
  - Creates isolated test endpoint `/test-limited` for rate limiting tests
  - Uses explicit memory storage to avoid conflicts with main app
  - Automatically cleans up after test
  - Used in `tests/unit/api/test_rate_limiting.py`

#### Redis Fixtures
- **`redis_test_port`** (session scope)
  - Returns: 6380 (test Redis port)

- **`redis_container`** (session scope)
  - **CI Environment:** Uses GitHub Actions Redis service on port 6380
  - **Local Development:** Spawns Redis Docker container on port 6380
  - Automatically detects environment via `CI` or `GITHUB_ACTIONS` env vars
  - Auto-cleanup: Removes container after tests (local only)

- **`redis_client`** (function scope)
  - Synchronous Redis client connected to port 6380
  - Auto-cleanup: flushes database after test

- **`redis_pool`** (async function scope)
  - Async ARQ Redis pool connected to port 6380
  - Used for async Redis operations in worker tests
  - Auto-cleanup: closes pool after test


#### Data Fixtures
- **`complete_metrics_data`** (function scope)
  - Returns complete metrics dictionary with all 10 metrics
  - Used for tests that need full metrics structure

#### Rate Limiter
- **`reset_rate_limiter`** (function scope, autouse)
  - Automatically resets SlowAPI rate limiter before/after each test
  - Prevents rate limit carryover between tests
  - Skips reset for tests using `client_with_low_rate_limit` (to avoid interference)
  - No need to explicitly use this fixture

### Test Helpers

The `tests/helpers/` directory provides reusable utilities to reduce boilerplate and improve test maintainability.

**Benefits:**
- Reduces code duplication (saves ~500-600 lines across test suite)
- Improves test readability and maintainability
- Centralizes common patterns (single source of truth)
- Makes tests easier to update when implementation changes
- Database helpers alone provide 68% reduction in test setup code

#### `helpers/database.py`

Database record creation helpers:

- **`create_test_user(db_session, email_prefix="test", **overrides)`**
  - Creates unique test user in database
  - Generates unique email/auth0_sub to avoid constraint violations
  - Returns User model instance
  - Usage: `user = create_test_user(db_session, email_prefix="admin")`

- **`create_test_analysis(db_session, user_id, **overrides)`**
  - Creates test analysis run in database with complete default metrics
  - Returns AnalysisRun model instance
  - Reduces ~10 lines of boilerplate per usage (68% code reduction)
  - Used in 11+ tests across unit and integration suites
  - Usage: `analysis = create_test_analysis(db_session, user.id, submission_type="github")`

- **`create_test_job(db_session, user_id, **overrides)`**
  - Creates test job in database
  - Returns Job model instance
  - Usage: `job = create_test_job(db_session, user.id, status="running")`

#### `helpers/files.py`

Python file and project generation:

- **`create_python_project(files: Dict[str, str]) -> Path`**
  - Creates temporary Python project with given files
  - Args: `{"main.py": "def foo(): pass", "lib/utils.py": "..."}`
  - Returns: Path to temporary directory
  - Caller must clean up
  - Used extensively in orchestrator and metric collector tests

- **`create_zip_archive(files: Dict[str, str]) -> Path:`**
  - Creates a temporary ZIP archive containing the provided file paths and contents
  - Args: `{"main.py": "def foo(): pass", "lib/utils.py": "..."}`
  - Returns: Path to temporary .zip file
  - Caller must clean up
  - Used for tests involving ZIP ingestion, unpacking, and archive-based workflows

- **`SAMPLE_CODE`** dictionary
  - Pre-defined code snippets for testing (reduces code duplication):
    - `SAMPLE_CODE["clean"]` - Lint-free, well-documented code
    - `SAMPLE_CODE["complex"]` - High cyclomatic complexity (nested ifs)
    - `SAMPLE_CODE["lint_issues"]` - Code with style issues (bad naming, spacing)
    - `SAMPLE_CODE["no_docs"]` - Undocumented functions
    - `SAMPLE_CODE["minimal"]` - Minimal function for basic tests
    - `SAMPLE_CODE["dead_code"]` - Contains unused functions and variables
    - `SAMPLE_CODE["security_vuln"]` - Security vulnerability (os.system)
    - `SAMPLE_CODE["duplicated"]` - Duplicate code blocks
    - `SAMPLE_CODE["fully_documented"]` - 100% documentation coverage

#### `helpers/api.py`

API request helpers:

- **`make_authenticated_request(client, method, url, token, **kwargs)`**
  - Makes authenticated API request with Bearer token
  - Args: client, HTTP method, URL, JWT token, request kwargs
  - Returns: Response object
  - Usage: `response = make_authenticated_request(client, "get", "/api/metrics/history", token)`

- **`assert_error_response(response, status_code, error_type=None)`**
  - Asserts API error response format
  - Checks status code and presence of error details

- **`assert_success_response(response, status_code=200)`**
  - Asserts successful API response
  - Returns: Parsed JSON response

#### `helpers/auth.py`

JWT token generation and Auth0 mocking:

- **`get_test_private_key_pem()`**
  - Returns PEM-encoded RSA private key for signing JWTs

- **`get_test_public_key_pem()`**
  - Returns PEM-encoded RSA public key for verification

- **`get_test_jwks()`**
  - Returns JWKS JSON response for Auth0 mock
  - Used to mock Auth0 JWKS endpoint

- **`create_test_jwt(sub: str, email: str)`**
  - Generates real, valid JWT token for testing
  - Signs with test private key
  - Returns: JWT token string
  - Usage: `token = create_test_jwt("auth0|user-123", "user@example.com")`

### Integration Test Setup

Integration tests require Redis for ARQ job queue testing. The setup automatically detects CI vs local environments. All Redis fixtures are documented in the [Redis Fixtures](#redis-fixtures) section above.

#### Local Development Requirements
- Docker must be installed and running
- Integration tests will automatically start Redis container
- No manual setup required

#### CI Environment
- No Docker required
- GitHub Actions provides Redis service on port 6380
- Configured in `.github/workflows/backend-ci.yml`

---

## Frontend Tests

### Frontend Test Organization

Frontend tests are located in `frontend/components/dashboard/results-table/table/`:

```
frontend/
├── components/
│   └── dashboard/
│       └── results-table/
│           └── table/
│               ├── index.test.tsx              # ResultsTableContent component
│               ├── result-row.test.tsx         # ResultRow component
│               └── table-header.test.tsx       # Table header component
├── vitest.config.ts                           # Vitest configuration
└── vitest.setup.ts                            # Test setup (jest-dom matchers)
```

**Test files:**
- `index.test.tsx` - Tests for `ResultsTableContent` component (pagination, load more button)
- `result-row.test.tsx` - Tests for `ResultRow` component (data display, actions)
- `table-header.test.tsx` - Tests for table header component (sorting, columns)

**Components tested:**
- Results table pagination and "Load More" functionality
- Individual result rows with metric data
- Table header with sorting controls

### Running Frontend Tests

All commands run from `frontend/` directory:

```bash
cd frontend
```

#### Run All Tests
```bash
npm test                    # Run all tests once
npm run test:watch          # Run in watch mode (re-run on file changes)
```

#### Coverage Report
```bash
npm run test                # Includes coverage by default
```

Coverage reporters configured: `text` and `html` (in `vitest.config.ts`)

### Testing Tools

**Vitest Configuration** (`vitest.config.ts`):
- **Environment:** jsdom (DOM simulation)
- **Setup file:** `vitest.setup.ts` (imports jest-dom matchers)
- **Globals:** Enabled (no need to import `describe`, `it`, `expect`)
- **CSS:** Enabled (processes CSS imports)
- **Coverage reporters:** text (terminal) and html (htmlcov/)
- **Path alias:** `@` resolves to project root

**Setup File** (`vitest.setup.ts`):
- Imports `@testing-library/jest-dom/vitest` for DOM matchers like `toBeInTheDocument()`, `toHaveClass()`, etc.

**Testing Libraries:**
- **Vitest** - Fast unit test framework for Vite projects
- **@testing-library/react** - React component testing utilities (render, screen, queries)
- **@testing-library/user-event** - User interaction simulation (click, type, etc.)
- **@testing-library/jest-dom** - Custom DOM matchers for assertions
- **jsdom** - JavaScript implementation of web standards (DOM, HTML)

**Test Patterns:**
- Component rendering with `render()`
- Query elements with `screen` utilities (`getByRole`, `getByText`, etc.)
- Mock child components with `vi.mock()`
- Simulate user interactions with `userEvent`
- Assert DOM state with jest-dom matchers

---

## CI/CD Integration

### GitHub Actions Workflow

Tests run automatically on all pushes and pull requests to `main` and `dev` branches.

**Workflow file:** `.github/workflows/backend-ci.yml`

#### Frontend CI Job

**Name:** `frontend-test-and-lint`

**Steps:**
1. Checkout code
2. Set up Node.js 18 with npm cache
3. Install dependencies (`npm ci`)
4. Run ESLint (`npm run lint`)
5. Run tests (`npm run test`)

**Triggers:**
- Push to `main` or `dev` branches (only if frontend files changed)
- Pull requests to `main` or `dev` branches (only if frontend files changed)

#### Backend CI Job

**Name:** `backend-test-and-lint`

**Services:**
- **PostgreSQL 15** on port 5432
  - Database: `vibe8_test`
  - User: `postgres`
  - Password: `postgres`
  - Health checks enabled

- **Redis 7** on port 6380 (mapped from 6379)
  - Health checks enabled
  - Used by integration tests

**Steps:**
1. Checkout code
2. Set up Python 3.13
3. Cache pip dependencies
4. Install dependencies from `requirements.txt`
5. Set up test database (`alembic upgrade head`)
6. Install PMD 7.0.0 for duplicate code detection
7. Run Ruff linting on `src` and `tests`
8. Run pytest with coverage (`--cov-fail-under=85`)

**Environment Variables:**
```yaml
CI: "true"                                          # CI flag for test fixtures
DATABASE_URL: postgresql://postgres:postgres@localhost:5432/vibe8_test
REDIS_HOST: localhost
REDIS_PORT: 6380
REDIS_PASSWORD: ""
REDIS_DB: 0
AUTH0_DOMAIN: test-domain.auth0.com
AUTH0_API_AUDIENCE: https://test-api
AUTH0_ISSUER: https://test-domain.auth0.com/
ARQ_MAX_JOBS: 20
ARQ_JOB_TIMEOUT: 600
RATE_LIMIT_PER_MINUTE: "1000"                      # High limit to prevent flaky tests
```

**Coverage Enforcement:**
- Minimum coverage: 85% (enforced via `pytest --cov-fail-under=85`)
- Build fails if coverage drops below threshold
- Coverage report uploaded as XML artifact

**Triggers:**
- Push to `main` or `dev` branches (only if backend files changed)
- Pull requests to `main` or `dev` branches (only if backend files changed)

### Test Markers

Backend tests use pytest markers for selective test execution:

- `unit` - Fast unit tests with mocked dependencies
- `integration` - Tests requiring PostgreSQL or Redis
- `e2e` - End-to-end tests covering complete workflows
- `slow` - Slow tests (e.g., e2e tests)
- `asyncio` - Async tests (requires pytest-asyncio)

Markers are defined in `pytest.ini` with strict checking enabled (`--strict-markers`).

---

## Test Configuration Files

### Backend

**`backend/pytest.ini`:**
```ini
[pytest]
markers =
    unit: Fast unit tests with mocked dependencies
    integration: Tests requiring PostgreSQL database or external services
    e2e: End-to-end tests covering complete user workflows
    slow: Slow tests (e.g., end-to-end tests)
    asyncio: Async tests (requires pytest-asyncio)

addopts = -v --strict-markers --cov=src --cov-report=term-missing --cov-report=xml

testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
```

### Frontend

**`frontend/vitest.config.ts`:**
```typescript
export default defineConfig({
  test: {
    environment: "jsdom",
    setupFiles: "./vitest.setup.ts",
    globals: true,
    css: true,
    coverage: {
      reporter: ["text", "html"],
    },
  },
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./"),
    },
  },
})
```

**`frontend/vitest.setup.ts`:**
```typescript
import "@testing-library/jest-dom/vitest"
```

---

## Related Documentation

- **[CLAUDE.md](../CLAUDE.md)** - Development guidelines and TDD workflow
- **[ARCHITECTURE.md](ARCHITECTURE.md)** - System architecture
- **[API.md](API.md)** - API endpoint specifications
- **[DATABASE.md](DATABASE.md)** - Database schema
- **[ASYNC_API.md](ASYNC_API.md)** - Async job queue architecture
