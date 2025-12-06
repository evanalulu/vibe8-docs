# Vibe8 Design

**Purpose:** Implementation patterns, coding standards, and interface contracts.

**This is the authoritative source for:** How to implement features, design patterns, and coding conventions.

**For related topics, see:** [ARCHITECTURE.md](ARCHITECTURE.md) (system overview), [API.md](API.md), [DATABASE.md](DATABASE.md)

## Table of Contents
- [Design Principles](#design-principles)
- [Design Patterns](#design-patterns)
- [Error Handling](#error-handling)
- [Frontend Patterns](#frontend-patterns)
- [Coding Standards](#coding-standards)
- [Related Documentation](#related-documentation)

## Design Principles

1. **Static Analysis Only** - Never execute user code. All analysis uses external tools that parse but don't run code.

2. **Fail Gracefully** - Tool failures are captured in an `errors` array. Never crash the service; return partial results.

3. **Owner-Only Access** - Users can only access their own data. Enforced at repository and route level.

4. **Async-First** - Long operations (>1s) use background job queue. API returns job_id immediately.

5. **Configuration as Single Source of Truth**:
   - Backend: `backend/src/core/config.py`
   - Frontend: `frontend/lib/config.ts`

   Never access `os.getenv()` or `process.env` directly in application code.

## Design Patterns

### Repository Pattern

Database operations are abstracted through repository classes in `infrastructure/database/repositories/`.

```python
class AnalysisRepository:
    def __init__(self, db: Session):
        self.db = db

    def create(self, user_id: UUID, metrics: dict) -> AnalysisRun:
        # Ownership set at creation
        ...

    def get_by_id(self, run_id: UUID) -> AnalysisRun | None:
        # Returns None if not found
        ...

    def get_user_analyses(self, user_id: UUID, limit: int) -> list[AnalysisRun]:
        # Filtered by user_id for owner-only access
        ...
```

**Why:**
- Encapsulates all database queries
- Centralizes access control
- Enables query optimization in one place
- Simplifies testing through mocking

### Collector Pattern

Each metric uses a collector implementing a common interface in `infrastructure/metrics/collectors/`.

```python
class MetricsCollector(ABC):
    @abstractmethod
    def collect(self, code_dir: Path) -> dict:
        """Collect metrics from code directory.

        Returns:
            dict with metric-specific keys

        Raises:
            Exception on failure (caught by orchestrator)
        """
        pass
```

**Error Contract:**
- Collectors may raise exceptions
- Orchestrator catches and logs failures
- Failures recorded in `errors` array
- Partial results returned to user

**Adding a new metric:**
1. Create collector in `infrastructure/metrics/collectors/`
2. Implement `MetricsCollector.collect()`
3. Add to orchestrator's collector list
4. Add processor in `application/workers/processors/`

### Dependency Injection

FastAPI's DI system manages cross-cutting concerns:

```python
@router.get("/api/metrics/history")
async def get_history(
    user: User = Depends(get_current_user),  # Auth
    db: Session = Depends(get_db),           # Database
):
    ...
```

**Standard Dependencies:**

| Dependency | Purpose | Location |
|------------|---------|----------|
| `get_current_user` | JWT validation, user loading | `api/auth/auth.py` |
| `get_db` | Database session (auto-closes) | `api/dependencies.py` |
| `RateLimiter` | Per-user request throttling | `api/limiter.py` |

### Authorization Pattern

All resource access checks ownership at the route level:

```python
def get_analysis(run_id: UUID, user_id: UUID) -> AnalysisRun:
    analysis = repo.get_by_id(run_id)
    if not analysis:
        raise HTTPException(404)
    if analysis.user_id != user_id:
        raise HTTPException(403, "Access denied")
    return analysis
```

## Error Handling

### HTTP Response Contract

| HTTP Code | Meaning | Response Body |
|-----------|---------|---------------|
| 400 | Invalid request | `{"detail": "description"}` |
| 401 | Not authenticated | `{"detail": "Not authenticated"}` |
| 403 | Not authorized | `{"detail": "Access denied"}` |
| 404 | Not found | `{"detail": "Resource not found"}` |
| 413 | File too large | `{"detail": "File exceeds limit"}` |
| 422 | Validation error | `{"detail": [validation errors]}` |
| 429 | Rate limited | `{"detail": "Rate limit exceeded"}` |

### Analysis Errors

Metric collector failures don't crash the service:

```json
{
  "metrics": {
    "lint": { ... },
    "complexity": { ... }
  },
  "errors": [
    "Security collector failed: bandit not found",
    "Churn collector failed: not a git repository"
  ]
}
```

## Frontend Patterns

### Component Model

Next.js 14 App Router uses two component types:

**Server Components (default):**
- Data fetching
- Static content
- No React hooks

**Client Components (`"use client"`):**
- Interactive UI
- Browser APIs
- React hooks (useState, useEffect)
- Event handlers

### Context Providers

Application-wide state via React Context:

```typescript
<AuthProvider>      {/* Auth0 integration */}
  <ThemeProvider>   {/* Dark/light mode */}
    <App />
  </ThemeProvider>
</AuthProvider>
```

### Custom Hooks

Encapsulate reusable logic:

```typescript
const { status, result, error } = useJobPolling(jobId, {
  interval: 2000,
  onComplete: (result) => handleResult(result),
});
```

**Key Hooks:**

| Hook | Purpose | Location |
|------|---------|----------|
| `useJobPolling` | Polls job status, handles completion | `lib/useJobPolling.ts` |
| `useAuth` | Auth0 state access | `context/auth-context.tsx` |

### Component Organization

```
components/
├── ui/              # Reusable primitives (Button, Card)
├── dashboard/       # Feature-specific components
├── auth-guard.tsx   # Protected route wrapper
└── theme-provider.tsx
```

**Rules:**
- UI primitives: Generic, no business logic
- Feature components: Specific to one feature
- Use context to avoid prop drilling

## Coding Standards

### Naming Conventions

| Context | Convention | Example |
|---------|------------|---------|
| Python modules | snake_case | `analysis_worker.py` |
| Python classes | PascalCase | `AnalysisRepository` |
| Python functions | snake_case | `get_analysis()` |
| TypeScript files | kebab-case | `auth-guard.tsx` |
| TypeScript components | PascalCase | `AuthGuard` |
| TypeScript functions | camelCase | `handleSubmit()` |
| Test files | `test_*.py` / `*.test.tsx` | `test_analysis.py` |

### Import Order

```python
# 1. Standard library
import os
from pathlib import Path

# 2. Third-party
from fastapi import APIRouter, Depends

# 3. Local
from src.core.config import settings
```

### Configuration Access

```python
# ✅ Correct
from src.core.config import settings
max_size = settings.MAX_UPLOAD_MB

# ❌ Wrong - direct env access
max_size = os.getenv("MAX_UPLOAD_MB")
```

```typescript
// ✅ Correct
import { config } from '@/lib/config';
const apiUrl = config.API_BASE_URL;

// ❌ Wrong - direct env access
const apiUrl = process.env.NEXT_PUBLIC_API_BASE_URL;
```

## Related Documentation

- **[Architecture](ARCHITECTURE.md)** - System overview, data flows
- **[API Reference](API.md)** - Endpoint contracts
- **[Database Schema](DATABASE.md)** - Data model, migrations
- **[Testing Guide](TESTING.md)** - Test patterns, fixtures

---

**Last Updated**: 2025-01-15
