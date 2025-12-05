# Vibe8 MCP Server

Connect Vibe8 code quality metrics to Claude Desktop via the Model Context Protocol (MCP).

**For related topics, see:** [ARCHITECTURE.md](ARCHITECTURE.md) (system design), [AUTH.md](AUTH.md) (Auth0 setup), [API.md](API.md) (endpoints), [METRICS.md](METRICS.md) (metric details)

## Overview

The Vibe8 MCP Server exposes 6 intent-based tools that allow Claude to analyze Python code quality:

| Tool | Description |
|------|-------------|
| `analyze_code` | Submit Python code for analysis (supports files, archives, or GitHub repos) |
| `check_job_status` | Monitor analysis job progress |
| `get_job_result` | Retrieve complete analysis results with all 10 metrics |
| `find_analysis` | Browse analysis history or get a specific past analysis |
| `get_file_tree` | Get per-file metrics and health scores |
| `finish_login` | Complete authentication after visiting the verification URL |

## Prerequisites

Before setting up the MCP server, ensure you have:

1. **Vibe8 backend running** - The MCP server connects to your Vibe8 API
2. **Auth0 configured** - MCP uses OAuth Device Flow for authentication
3. **Claude Desktop installed** - The MCP client application

**Full setup instructions:** See [README.md](../README.md) for complete application setup including:
- Backend server setup (Python 3.13, PostgreSQL, Redis)
- Auth0 configuration
- Running the API server and worker

## Quick Start

### 1. Install Dependencies

```bash
cd backend
pip install -r requirements.txt
```

### 2. Start the Vibe8 Backend

The MCP server connects to your Vibe8 API. Make sure the backend is running:

```bash
# Terminal 1: API server
uvicorn src.api.main:app --reload

# Terminal 2: Background worker (for analysis jobs)
arq src.application.workers.analysis_worker.WorkerSettings
```

### 3. Configure Claude Desktop

Create or edit `~/.claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "vibe8": {
      "command": "/path/to/vibe8/backend/.venv/bin/python",
      "args": ["-m", "src.mcp.server"],
      "cwd": "/path/to/vibe8/backend",
      "env": {
        "PYTHONPATH": "/path/to/vibe8/backend",
        "VIBE8_API_URL": "http://localhost:8000",
        "AUTH0_DOMAIN": "your-tenant.auth0.com",
        "AUTH0_MCP_CLIENT_ID": "your-native-app-client-id",
        "AUTH0_API_AUDIENCE": "your-api-audience"
      }
    }
  }
}
```

Replace:
- `/path/to/vibe8/backend` with the actual path to your backend directory (appears 3 times)
- Auth0 values with your actual Auth0 configuration

**Important**: Use the full path to Python in your virtual environment, not just `python`.

### 4. Restart Claude Desktop

After saving the config, restart Claude Desktop for changes to take effect.

### 5. Authenticate (First Time Only)

On first tool use, Claude will display an authentication prompt:

```json
{
  "auth_required": true,
  "message": "🔐 Authentication required!\n\nPlease visit: https://your-tenant.auth0.com/activate?user_code=ABCD-1234\n\nOr go to: https://your-tenant.auth0.com/activate\nEnter code: ABCD-1234\n\nAfter completing authentication, call the `finish_login` tool.",
  "verification_url": "https://your-tenant.auth0.com/activate?user_code=ABCD-1234",
  "user_code": "ABCD-1234"
}
```

1. Visit the URL shown in the message
2. Log in with your Vibe8 account
3. Enter the code shown
4. Ask Claude to call the `finish_login` tool
5. Authorization is saved for future sessions (auto-refreshes)

## Usage Examples

Once configured, you can ask Claude to analyze code:

### Analyze a Local File

> "Analyze /Users/me/projects/myapp/main.py for code quality"

Claude will:
1. Use `analyze_code(source="/Users/me/projects/myapp/main.py", source_type="file")` to submit the file
2. Poll with `check_job_status(job_id)` every 2 seconds
3. Retrieve results with `get_job_result(job_id)`
4. Optionally drill down with `get_file_tree(run_id)`

**Note**: File paths must be absolute paths on your local machine (e.g., `/Users/...` on macOS).

### Analyze a Zip Archive

> "Analyze /Users/me/Downloads/project.zip for code quality"

The zip archive should contain Python files. Uses the same `analyze_code` tool with `source_type="file"`.

### Analyze a GitHub Repository

> "Analyze the Python code quality of https://github.com/owner/repo"

Claude will use `analyze_code(source="https://github.com/owner/repo", source_type="github")`.

Note: Only public repositories are supported via MCP.

### View Past Analyses

> "Show me my recent code analyses"

Claude will use `find_analysis()` (without run_id) to retrieve your history.

> "Get details for analysis run abc-123"

Claude will use `find_analysis(run_id="abc-123")` to retrieve specific analysis details.

## Security

- **User-scoped data**: You can only access your own analyses
- **No Git credentials**: GitHub analysis only works with public repos
- **Read-only**: The DELETE endpoint is not exposed via MCP
- **Secure token storage**: Tokens stored at `~/.vibe8/mcp_tokens.json` with 600 permissions
- **Auto-refresh**: Tokens automatically refresh using OAuth refresh tokens

## Troubleshooting

### "Not authenticated" Error

Delete the token file and re-authenticate:

```bash
rm ~/.vibe8/mcp_tokens.json
```

Then restart Claude Desktop and use any Vibe8 tool to trigger re-authentication.

### Token Expired

Tokens should auto-refresh. If they don't:
1. Delete `~/.vibe8/mcp_tokens.json`
2. Restart Claude Desktop
3. Re-authenticate

### Connection Refused

Ensure your Vibe8 API server is running:

```bash
cd backend
uvicorn src.api.main:app --reload
```

### MCP Server Not Found

Verify the path in your Claude Desktop config points to the correct `backend` directory.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Your Computer                                              │
│                                                             │
│  ┌──────────────┐    stdio    ┌──────────────────────────┐ │
│  │Claude Desktop│◀──────────▶│ MCP Server (subprocess)   │ │
│  └──────────────┘             │ - Handles auth            │ │
│                               │ - Calls FastAPI           │ │
│                               └────────────┬─────────────┘ │
└────────────────────────────────────────────┼───────────────┘
                                             │ HTTPS + JWT
                                             ▼
                              ┌──────────────────────────────┐
                              │  Vibe8 API Server            │
                              │  - Validates JWT             │
                              │  - Returns user's data only  │
                              └──────────────────────────────┘
```

## Auth0 Setup Requirements

To use MCP, you need a **Native Application** in Auth0:

1. Go to Auth0 Dashboard → Applications → Create Application
2. Select "Native" as the application type
3. In Settings → Advanced Settings → Grant Types, enable:
   - Device Code
   - Refresh Token
4. In your API settings, enable "Allow Offline Access"

## Available Metrics

The analysis returns 10 code quality metrics:

1. **Lint** - Ruff linting issues
2. **Cyclomatic Complexity** - Code complexity scores
3. **Documentation** - Docstring coverage
4. **Dead Code** - Unused code detection
5. **Dependencies** - Import analysis
6. **Maintainability Index** - Radon MI scores
7. **Size** - Lines of code metrics
8. **Churn** - Git history analysis
9. **Duplication** - Duplicate code detection
10. **Security** - Bandit security findings

**For detailed metric specifications:** See [METRICS.md](METRICS.md)

## Related Documentation

- **[README.md](../README.md)** - Full application setup and quickstart
- **[Architecture Guide](ARCHITECTURE.md)** - System design and component overview
- **[API Reference](API.md)** - REST API endpoint documentation
- **[Auth0 Setup](AUTH.md)** - Complete Auth0 configuration guide
- **[Metrics Reference](METRICS.md)** - Detailed metric specifications and interpretations
- **[Visual Diagrams](diagrams.md)** - MCP architecture diagram

---

**Last Updated**: 2025-01-15
