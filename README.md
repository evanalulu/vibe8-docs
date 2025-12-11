# Vibe8

A full-stack code quality analysis platform that provides comprehensive static analysis for Python projects through REST API with secure OAuth 2.0 authentication.

## Technology Stack

**Backend**: Python 3.13 • FastAPI • PostgreSQL • Redis • SQLAlchemy
**Frontend**: Next.js • React • TypeScript • Tailwind CSS
**Auth**: Auth0 OAuth 2.0
**Infrastructure**: Docker • Docker Compose

For complete technology stack and project structure, see [ARCHITECTURE.md](ARCHITECTURE.md).

## Features

- **10 Comprehensive Metrics**: Linting, complexity, documentation, dead code, dependencies, maintainability, size, duplicates, security, and git churn
- **Multiple Input Sources**: Upload Python files/archives or analyze GitHub repositories (public and private)
- **Async Job Queue**: Long-running analysis via background workers with automated cleanup
- **Auth0 OAuth 2.0**: Secure authentication with Google and GitHub social providers
- **Analysis History**: Track and compare code quality over time
- **RESTful API**: Clean JSON responses with comprehensive error handling

## Table of Contents

- [Quick Start](#quick-start)
- [Docker Compose Setup](#docker-compose-setup)
- [Local Development Setup](#local-development-setup)
- [Testing](#testing)
- [Usage](#usage)
- [Documentation](#documentation)

## Quick Start

Choose your preferred setup method:

- **[Docker Compose Setup (Recommended)](#docker-compose-setup)** - Minimal dependencies, everything runs in isolated containers
- **[Local Development Setup](#local-development-setup)** - All services run directly on your machine

---

## Docker Compose Setup

**Best for:** Quick setup with minimal host dependencies, consistent environment across machines.

### Prerequisites

- **Docker Desktop**: Container platform ([download](https://www.docker.com/products/docker-desktop/))
- **Auth0 Account**: For authentication ([setup guide](AUTH.md))

### 1. Install Docker Desktop

**macOS:**
```bash
brew install --cask docker-desktop
# OR download from https://www.docker.com/products/docker-desktop/

# Launch Docker Desktop from Applications
open -a Docker
```

**Linux:**
```bash
# Follow instructions at https://docs.docker.com/engine/install/
```

**Windows:**
```bash
# Download installer from https://www.docker.com/products/docker-desktop/
```

**Verify Docker is running:**
```bash
docker --version
docker compose version
```

### 2. Configuration

Create `.env` file in the project root (used by Docker Compose):

```bash
cp .env.example .env
```

Edit `.env` and set your Auth0 credentials (get from [Auth0 Dashboard](https://manage.auth0.com/) — see [Setup Guide](AUTH.md) or [Quick Setup](AUTH.md#quick-setup-already-configured) if already configured):

```bash
# Required Auth0 settings
AUTH0_DOMAIN=your-tenant.auth0.com
AUTH0_API_AUDIENCE=https://your-api-audience
AUTH0_ISSUER=https://your-tenant.auth0.com/
AUTH0_CLIENT_ID=your-client-id
AUTH0_CLIENT_SECRET=your-client-secret
AUTH0_SECRET=your-random-secret  # Generate with: openssl rand -hex 32
```

> **Note**: All other settings (database URLs, Redis config, etc.) have defaults configured for Docker Compose.

📄 **Full Configuration Reference**: See [.env.example](.env.example) for all available options with detailed descriptions.

### 3. Start Services

```bash
# Build and start all services (PostgreSQL, Redis, Backend, Worker, Frontend)
docker compose up -d

# View logs
docker compose logs -f

# Check service status
docker compose ps
```

### 4. Access the Application

- **Frontend**: http://localhost:3000
- **Backend API**: http://localhost:8000
- **API Docs**: http://localhost:8000/docs

#### Managing Docker Services

```bash
# Stop all services
docker compose down

# Restart services
docker compose restart

# View logs for specific service
docker compose logs backend
docker compose logs frontend

# Rebuild after code changes
docker compose up -d --build
```

---

## Local Development Setup

**Best for:** Direct access to all services, full control over dependencies.

### Prerequisites

Before you begin, you'll need:

- **Python 3.13+**: Programming language for the backend
- **Node.js 18+**: JavaScript runtime for the frontend
- **PostgreSQL 15+**: Database to store analysis results
- **Redis 7+**: In-memory data store for background job processing
- **PMD 7.0+**: Tool for duplicate code detection
- **Auth0 Account**: For secure user authentication ([setup guide](AUTH.md))

### 1. Install Dependencies

#### Python 3.13+

**macOS (using Homebrew):**
```bash
brew install python@3.13

# Add to PATH (required) - use your shell's config file
# For zsh (default on macOS):
echo 'export PATH="/opt/homebrew/opt/python@3.13/bin:$PATH"' >> ~/.zshrc
# For bash:
# echo 'export PATH="/opt/homebrew/opt/python@3.13/bin:$PATH"' >> ~/.bashrc

# Reload shell configuration
source ~/.zshrc  # or source ~/.bashrc for bash

# Verify installation
python3.13 --version
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt update && sudo apt install python3.13 python3.13-venv

# Verify installation
python3.13 --version
```

**Windows:**
- Download from https://www.python.org/downloads/
- During installation, check "Add Python to PATH"
- Restart terminal after installation
- Verify: `python --version`

#### Node.js 18+

**macOS (using Homebrew):**
```bash
brew install node@18

# Add to PATH (required for Homebrew's node@18)
# For zsh (default on macOS):
echo 'export PATH="/opt/homebrew/opt/node@18/bin:$PATH"' >> ~/.zshrc
# For bash:
# echo 'export PATH="/opt/homebrew/opt/node@18/bin:$PATH"' >> ~/.bashrc

# Reload shell configuration
source ~/.zshrc  # or source ~/.bashrc for bash

# Verify installation
node --version
npm --version
```

**Linux (Ubuntu/Debian):**
```bash
# Using NodeSource repository
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify installation
node --version
npm --version
```

**Windows:**
- Download from https://nodejs.org/
- Installer automatically adds to PATH
- Restart terminal after installation
- Verify: `node --version` and `npm --version`

#### PostgreSQL 15+

**macOS (using Homebrew):**
```bash
brew install postgresql@15

# Add to PATH (required)
# For zsh (default on macOS):
echo 'export PATH="/opt/homebrew/opt/postgresql@15/bin:$PATH"' >> ~/.zshrc
# For bash:
# echo 'export PATH="/opt/homebrew/opt/postgresql@15/bin:$PATH"' >> ~/.bashrc

# Reload shell configuration
source ~/.zshrc  # or source ~/.bashrc for bash

# Start PostgreSQL service
brew services start postgresql@15

# Create database
psql postgres -c "CREATE DATABASE vibe8_dev;"
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt update && sudo apt install postgresql-15
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Create database
sudo -u postgres psql -c "CREATE DATABASE vibe8_dev;"
```

**Windows:**
- Download installer from https://www.postgresql.org/download/windows/
- Installer automatically configures PATH
- Create database using pgAdmin or: `psql postgres -c "CREATE DATABASE vibe8_dev;"`

#### Redis 7+

**macOS (using Homebrew):**
```bash
brew install redis

# Start Redis service
brew services start redis

# Verify installation
redis-cli ping  # Should return "PONG"
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt update && sudo apt install redis-server
sudo systemctl start redis-server
sudo systemctl enable redis-server

# Verify installation
redis-cli ping  # Should return "PONG"
```

**Windows:**
- Download from https://github.com/microsoftarchive/redis/releases
- Follow installation instructions
- Verify: `redis-cli ping`

#### PMD 7.0+

**macOS (using Homebrew):**
```bash
brew install pmd

# PMD is automatically added to PATH by Homebrew

# Verify installation
pmd --version
```

**Linux/Windows:**
- Download from https://pmd.github.io/latest/pmd_userdocs_installation.html
- Extract and add to PATH
- Verify: `pmd --version`

> **⚠️ Important**: After installing dependencies via Homebrew on macOS, **restart your terminal** or reload your shell configuration file for PATH changes to take effect.

### 2. Backend Setup

```bash
cd backend

# Create virtual environment (isolated Python environment)
python3.13 -m venv .venv

# Activate virtual environment
source .venv/bin/activate  # macOS/Linux
# OR
.venv\Scripts\activate  # Windows

# Install dependencies
pip install --upgrade pip
pip install -r requirements.txt

# Edit .env and add your Auth0 and database cNredentials (see step 4 below)
# Apply database migrations (creates all tables and schema)
alembic upgrade head

# NOTE: After pulling updates with new migrations, run this again:
# alembic upgrade head
```

### 3. Frontend Setup

```bash
cd frontend

# Install dependencies
npm install
```

### 4. Configuration

Create a `.env` file in the **project root**:

```bash
# From project root
cp .env.example .env
```

Edit `.env` with your Auth0 credentials and other settings:

**Required Auth0 Settings** (get from [Auth0 Dashboard](https://manage.auth0.com/) — see [Setup Guide](AUTH.md) or [Quick Setup](AUTH.md#quick-setup-already-configured) if already configured):
```bash
# Backend Auth0
AUTH0_DOMAIN=your-tenant.auth0.com
AUTH0_API_AUDIENCE=https://your-api-audience
AUTH0_ISSUER=https://your-tenant.auth0.com/

# Frontend Auth0
AUTH0_ISSUER_BASE_URL=https://your-tenant.auth0.com
AUTH0_CLIENT_ID=your-client-id
AUTH0_CLIENT_SECRET=your-client-secret
AUTH0_SECRET=your-random-secret  # Generate with: openssl rand -hex 32
AUTH0_BASE_URL=http://localhost:3000
AUTH0_AUDIENCE=https://your-api-audience
```

**Database Connection**:
```bash
DATABASE_URL=postgresql://localhost:5432/vibe8_dev
```

> **Note**: For local development, most PostgreSQL installations don't require username/password. If your setup requires authentication, use `postgresql://user:password@localhost:5432/vibe8_dev` instead (create a user with `createuser -s myuser`).

**Other Key Settings** (these have sensible defaults):
```bash
# Backend API URL (for frontend)
NEXT_PUBLIC_API_URL=http://localhost:8000

# Server Configuration
ENVIRONMENT=development
CORS_ORIGINS=http://localhost:3000

# Security
MAX_UPLOAD_MB=5
RATE_LIMIT_PER_MINUTE=60

# Background Jobs
MAX_CONCURRENT_JOBS_PER_USER=5
DAILY_JOB_LIMIT_PER_USER=100
```

📄 **Full Configuration Reference**: See [.env.example](.env.example) in the project root for all available options with detailed descriptions.

### 5. Run Application

**Terminal 1 - Backend API Server**:
```bash
cd backend
source .venv/bin/activate  # macOS/Linux (.venv\Scripts\activate on Windows)
uvicorn src.api.main:app --reload --port 8000  # Start the backend server
```

**Terminal 2 - Background Job Processor** (required):
```bash
cd backend
source .venv/bin/activate  # macOS/Linux (.venv\Scripts\activate on Windows)
arq src.application.workers.analysis_worker.WorkerSettings  # Process analysis jobs in the background
```

**Terminal 3 - Frontend Web Application**:
```bash
cd frontend
npm run dev  # Start the web interface
```

**Access the Application**:
- **Frontend**: http://localhost:3000
- **Backend API**: http://localhost:8000
- **API Docs**: http://localhost:8000/docs

### 6. Shutdown Services

To stop the application when you're done:

**Stop the running servers** (in each terminal):
- Press `Ctrl+C` in each of the 3 terminals to stop the backend, worker, and frontend

**Stop background services** (optional - only if you want to free up resources):
```bash
# Stop PostgreSQL
brew services stop postgresql@15

# Stop Redis
brew services stop redis
```

**Note**: You can leave PostgreSQL and Redis running between sessions. Only stop them if you need to free up system resources.

---

## Testing

See **[TESTING.md](TESTING.md)** for complete testing guide including:
- Test structure and organization
- Writing unit and integration tests
- Using fixtures and helpers
- Coverage targets and reports
- Troubleshooting common issues

### Backend Tests

```bash
cd backend
source .venv/bin/activate

# Run all tests with coverage
pytest

# Run unit tests only (fast, ~5 seconds)
pytest -m unit

# Run integration tests only (~30 seconds)
pytest -m integration

# Run e2e tests only (~15 seconds)
pytest -m e2e

# Run with HTML coverage report
pytest --cov=src --cov-report=html
open htmlcov/index.html

# Lint code
ruff check src
ruff check --fix src  # Auto-fix issues
```

**Test Organization:**
- `tests/unit/` - Unit tests for individual components (~547 tests)
- `tests/integration/` - Integration tests requiring database and Redis (~10 tests)
- `tests/e2e/` - End-to-end workflow tests (~6 tests)
- `tests/helpers/` - Reusable test utilities (database, files, API, auth helpers)
- `tests/conftest.py` - Global fixtures (DB, auth, clients, file helpers)

**Test Coverage:**
- Total tests: ~560 tests
- Coverage: 85%+ (target: 85%+)

### Frontend Tests

```bash
cd frontend

# Run tests
npm run test

# Run tests in watch mode
npm run test:watch

# Lint code
npm run lint
```

## Usage

### Upload File for Analysis

```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
     -F "file=@example.py" \
     http://localhost:8000/api/metrics/analyze/upload/async
```

### Analyze GitHub Repository

```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
     -F "repo_url=https://github.com/owner/repo" \
     http://localhost:8000/api/metrics/analyze/github/async
```

### Poll Job Status

```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
     http://localhost:8000/api/metrics/jobs/{job_id}
```

### Get Analysis Results

```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
     http://localhost:8000/api/metrics/jobs/{job_id}/result
```

### Analysis Results

When you submit code for analysis, you'll receive comprehensive metrics covering 10 quality dimensions:

- **Linting** - PEP 8 violations and code style issues (Ruff)
- **Complexity** - Cyclomatic complexity scores per function (Radon)
- **Documentation** - Missing docstrings percentage (docstr_coverage)
- **Dead Code** - Unused functions, classes, and variables (Vulture)
- **Dependencies** - Import graph and package health analysis
- **Maintainability** - Maintainability Index scores (Radon)
- **Size** - SLOC counts and file statistics (Radon)
- **Duplicates** - Repeated code blocks (PMD CPD)
- **Security** - Vulnerability scan results (Bandit)
- **Git Churn** - Code change frequency analysis (GitPython)

Analysis runs asynchronously - you'll receive a job ID immediately and can poll for results.

### Response Format

Results are returned as JSON with this structure:

```json
{
  "run_id": "uuid",
  "created_at": "ISO-8601",
  "source": { /* upload or github metadata */ },
  "metrics": {
    "lint": { "total": 12, "linting_score": 92.5 },
    "cyclomatic": { "avg_cc": 3.2, "complexity_score": 84.0 },
    "docs": { "coverage_pct": 85.0 },
    "dead_code": { "unused_symbols": 3 },
    "deps": { "total_modules": 5, "package_health": [...] },
    "mi": { "mi_avg": 65.5 },
    "size": { "files": 1, "sloc": 150, "avg_file_sloc": 150 },
    "dupes": { "duplicate_lines": 10, "duplication_score": 87.5 },
    "security": { "high": 0, "medium": 1, "low": 2, "security_score": 55.0 },
    "churn": { "total_commits": 125, "totalChurn": 11700, "churn_score": 0.0 }
  },
  "errors": []  // Any metric collection failures
}

Note: Per-file metrics are available via GET /api/metrics/file-tree/{run_id}
```

Full schema documentation: [API.md](API.md)

## Documentation

- **[Architecture Guide](ARCHITECTURE.md)** - System design, tech stack, project structure
- **[Design Patterns](DESIGN.md)** - Design patterns, coding standards
- **[API Reference](API.md)** - Complete endpoint documentation
- **[Authentication Setup](AUTH.md)** - Auth0 OAuth 2.0 configuration
- **[Database Schema](DATABASE.md)** - PostgreSQL tables and migrations
- **[Metrics Reference](METRICS.md)** - Detailed metrics specifications
- **[MCP Server](MCP.md)** - Claude Desktop integration via Model Context Protocol
- **[Deployment Guide](DEPLOYMENT.md)** - Production deployment (Google Cloud Platform)


## License

This project is part of the Vibe8 code quality platform.

---

**Need help?** Check the [documentation]() or open an issue on GitHub.
