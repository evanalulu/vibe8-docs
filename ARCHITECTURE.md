# Vibe8 Architecture

**Purpose:** High-level system architecture, components, and data flows.

**This is the authoritative source for:** System structure, technology choices, and how components connect.

**For related topics, see:** [DESIGN.md](DESIGN.md) (implementation patterns), [API.md](API.md), [DATABASE.md](DATABASE.md)

## Table of Contents
- [System Overview](#system-overview)
- [Technology Stack](#technology-stack)
- [System Diagram](#system-diagram)
- [Project Structure](#project-structure)
- [Layered Architecture](#layered-architecture)
- [Data Flows](#data-flows)
- [Related Documentation](#related-documentation)

## System Overview

Vibe8 is a code quality analysis platform that provides static analysis metrics for Python codebases.

**Core Characteristics:**
- **Static Analysis Only** - No user code execution (security)
- **Async Processing** - Background job queue for long-running analysis
- **Multi-Source Input** - File uploads (`.py`, `.zip`) and GitHub repositories
- **10 Metrics** - Lint, complexity, docs, dead code, dependencies, maintainability, size, duplication, security, churn

## Technology Stack

### Backend
| Component | Technology |
|-----------|------------|
| Language | Python 3.13 |
| Framework | FastAPI (ASGI/Uvicorn) |
| Database | PostgreSQL 16+ with SQLAlchemy 2.0 |
| Job Queue | Redis 7+ with ARQ |
| Auth | Auth0 OAuth 2.0 (JWT/RS256) |

### Analysis Tools
| Metric | Tool |
|--------|------|
| Lint | Ruff |
| Complexity/MI/Size | Radon |
| Documentation | docstr_coverage |
| Dead Code | Vulture |
| Security | Bandit |
| Duplication | PMD CPD |
| Churn | GitPython |
| Dependencies | AST Parser |

### Frontend
| Component | Technology |
|-----------|------------|
| Framework | Next.js 14.2 (App Router) |
| UI | React 18, TypeScript 5, Tailwind CSS 4 |
| Components | Radix UI, shadcn/ui |
| Auth | @auth0/nextjs-auth0 |
| Testing | Vitest, Testing Library |

### Infrastructure
| Component | Technology |
|-----------|------------|
| Containers | Docker, Docker Compose |
| Platform | Google Cloud Platform |

## System Diagram

```
                                                    ┌───────────┐
┌───────────────────────────────────────────────────┤  EXTERNAL │
│              PRESENTATION LAYER                   │           │
│  ┌─────────────────┐    ┌─────────────────┐       │ ┌───────┐ │
│  │ Next.js Frontend│    │   MCP Server    │       │ │ Auth0 │ │
│  │ (Auth0 SDK)     │    │ (Claude Desktop)│◄──────┼─┤OAuth  │ │
│  └────────┬────────┘    └────────┬────────┘       │ └───┬───┘ │
└───────────┼──────────────────────┼────────────────┤     │     │
            │ HTTPS + JWT          │                │     │     │
            ▼                      ▼                │     │     │
┌───────────────────────────────────────────────────┤     │     │
│              APPLICATION LAYER                    │     │     │
│  ┌────────────────────────────────────────────┐   │     │     │
│  │ FastAPI (api/)                             │◄──┼─────┘     │
│  │ • Auth middleware  • Rate limiting  • CORS │   │ JWT verify│
│  └────────────────────────────────────────────┘   │           │
│  ┌──────────────────────┐  ┌──────────────────┐   │           │
│  │ ARQ Workers          │  │ Redis            │   │           │
│  │ (application/)       │◄─┤ (job queue)      │   │           │
│  │ • 20 concurrent jobs │  │                  │   │           │
│  └──────────────────────┘  └──────────────────┘   │           │
└───────────────────────────────────────────────────┴───────────┘
            │
            ▼
┌───────────────────────────────────────────────────────────────┐
│              INFRASTRUCTURE LAYER                             │
│  ┌─────────────┐  ┌─────────────┐  ┌───────────────────────┐  │
│  │ PostgreSQL  │  │ GitHub      │  │ Metrics Collectors    │  │
│  │ (database/) │  │ (github/)   │  │ (metrics/collectors/) │  │
│  └─────────────┘  └─────────────┘  └───────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

## Project Structure

### Backend

```
backend/
├── src/
│   ├── api/              # HTTP layer (routes, middleware, validators)
│   ├── application/      # Business logic (workers, auth services)
│   ├── domain/           # Pydantic schemas (analysis, auth, jobs)
│   ├── infrastructure/   # External integrations (database, metrics, github)
│   ├── core/             # Configuration (config.py)
│   └── mcp/              # Claude Desktop MCP server
├── tests/                # Test suite (unit, integration, e2e)
├── alembic/              # Database migrations
└── tmp-work/             # Ephemeral file processing
```

### Frontend

```
frontend/
├── app/                  # Next.js App Router pages
├── components/           # React components (ui/, dashboard/)
├── context/              # React context providers
├── lib/                  # Utilities (config, API client, hooks)
└── types/                # TypeScript definitions
```

## Layered Architecture

The backend follows Clean Architecture with strict layer separation:

```
┌─────────────────────────────────────┐
│  API Layer (api/)                   │  ← HTTP handling, validation
├─────────────────────────────────────┤
│  Application Layer (application/)   │  ← Business logic, workers
├─────────────────────────────────────┤
│  Domain Layer (domain/)             │  ← Entities, schemas
├─────────────────────────────────────┤
│  Infrastructure Layer (infra/)      │  ← Database, external services
├─────────────────────────────────────┤
│  Core Layer (core/)                 │  ← Configuration, utilities
└─────────────────────────────────────┘
```

### Layer Responsibilities

| Layer | Responsibility | May Depend On |
|-------|---------------|---------------|
| **API** | HTTP handling, validation, response formatting | Application, Domain |
| **Application** | Business logic, orchestration, workers | Domain, Infrastructure |
| **Domain** | Business entities, validation rules | None (pure) |
| **Infrastructure** | Database, metrics collectors, GitHub | Domain |
| **Core** | Configuration, shared utilities | None |

**Dependency Rule:** Dependencies flow inward only. Inner layers never import from outer layers.

## Data Flows

### Analysis Flow (Upload)

```
1. SUBMIT
   User → Frontend → POST /api/metrics/analyze/upload/async
                          │
                          ▼
   API validates file → Creates job (pending) → Enqueues to Redis
                          │
                          ▼
                     Returns job_id (HTTP 202)

2. PROCESS
   ARQ Worker picks up job → Updates status (running)
                                   │
                                   ▼
   Decodes file → Extracts to tmp-work/ → Runs 10 collectors
                                   │
                                   ▼
   Aggregates results → Stores in PostgreSQL → Updates job (completed)

3. RETRIEVE
   Frontend polls GET /jobs/{job_id} → When completed:
                                            │
                                            ▼
   GET /jobs/{job_id}/result → Returns metrics → Deletes job record
```

### Analysis Flow (GitHub)

Same as upload flow with these differences:
- API validates GitHub URL format
- Worker clones repository (public only via MCP, private via `X-GitHub-Token`)
- Worker captures commit SHA and branch metadata
- Worker cleans up cloned repository after analysis

### Authentication Flow

```
1. LOGIN
   User → Click Google/GitHub → Auth0 OAuth flow
                                     │
                                     ▼
   Auth0 Post-Login Action → Checks email allowlist
                                     │
                                     ▼
   On approval → Redirect to frontend → Exchange code for tokens

2. API REQUEST
   Frontend → Authorization: Bearer {token}
                    │
                    ▼
   API → Validates JWT (Auth0 JWKS) → Extracts claims (sub, email)
                    │
                    ▼
   Loads/creates user → Attaches to request context

3. AUTHORIZATION
   API → Checks user_id matches resource owner
            │
            ├─ Mismatch → 403 Forbidden
            └─ Not found → 404 Not Found
```

## Related Documentation

### Implementation
- **[Design Patterns](DESIGN.md)** - Implementation patterns, coding standards
- **[API Reference](API.md)** - Endpoint documentation
- **[Database Schema](DATABASE.md)** - Tables, indexes, migrations

### Configuration
- **[Auth0 Setup](AUTH.md)** - OAuth configuration
- **[Async API](ASYNC_API.md)** - Job queue configuration
- **[Metrics](METRICS.md)** - Metric specifications

### Operations
- **[Deployment](DEPLOYMENT.md)** - Production deployment
- **[Testing](TESTING.md)** - Test structure and practices
- **[MCP Server](MCP.md)** - Claude Desktop integration
- **[Diagrams](diagrams.md)** - Visual architecture diagrams

---

**Last Updated**: 2025-01-15
