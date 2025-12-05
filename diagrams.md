# High-level system architecture

## Architecture Diagram

```mermaid
graph TB
    subgraph presentation["Presentation Layer (Client)"]
        user([User])
        frontend[Next.js Frontend<br/>TypeScript + React 18]
        mcp[MCP Server<br/>Claude Desktop]
    end

    subgraph application["Application Layer (API Server)"]
        api[FastAPI Application<br/>Python 3.13]
        auth[Auth Middleware<br/>JWT Validation]
    end

    subgraph async["Async Processing Layer"]
        queue[(Redis Queue<br/>Message Broker)]
        worker[ARQ Workers<br/>20 concurrent]
    end

    subgraph domain["Domain Layer (Metrics)"]
        collectors[Metrics Collection Service]
        ruff[Ruff]
        radon[Radon]
        vulture[Vulture]
        bandit[Bandit]
        pmd[PMD CPD]
        ast[AST Parser]
        git[GitPython]
    end

    subgraph persistence["Data Layer"]
        db[(PostgreSQL<br/>Users + Analyses + Jobs)]
    end

    subgraph external["External Services"]
        auth0{{Auth0<br/>OAuth 2.0}}
    end

    %% User interactions
    user -->|HTTPS| frontend
    user -->|Claude Desktop| mcp
    frontend -->|REST API| api
    mcp -->|REST API| api

    %% Auth flow
    frontend -.->|OAuth| auth0
    mcp -.->|Device Flow| auth0
    auth <-->|Verify JWT| auth0
    api --> auth

    %% Sync operations (history)
    api <-->|Read/Write| db

    %% Async operations (analysis)
    api -->|Enqueue Job| queue
    queue -->|Dequeue| worker
    worker -->|Execute| collectors
    collectors --> ruff & radon & vulture & bandit & pmd & ast & git
    worker -->|Store Results| db

    %% Styling - Industry Standard Color Coding
    %% Blue: Client/Presentation
    classDef clientStyle fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#000
    %% Green: Application/Business Logic
    classDef appStyle fill:#C8E6C9,stroke:#2E7D32,stroke-width:2px,color:#000
    %% Orange: Async/Queue Processing
    classDef queueStyle fill:#FFE0B2,stroke:#E65100,stroke-width:2px,color:#000
    %% Purple: Domain/Core Logic
    classDef domainStyle fill:#E1BEE7,stroke:#6A1B9A,stroke-width:2px,color:#000
    %% Teal: Data/Persistence
    classDef dataStyle fill:#B2DFDB,stroke:#00695C,stroke-width:2px,color:#000
    %% Red: External Dependencies
    classDef externalStyle fill:#FFCDD2,stroke:#C62828,stroke-width:2px,color:#000

    class user,frontend,mcp clientStyle
    class api,auth appStyle
    class queue,worker queueStyle
    class collectors,ruff,radon,vulture,bandit,pmd,ast,git domainStyle
    class db dataStyle
    class auth0 externalStyle

    %% Layer styling
    style presentation fill:#F5F5F5,stroke:#1565C0,stroke-width:2px
    style application fill:#F5F5F5,stroke:#2E7D32,stroke-width:2px
    style async fill:#F5F5F5,stroke:#E65100,stroke-width:2px
    style domain fill:#F5F5F5,stroke:#6A1B9A,stroke-width:2px
    style persistence fill:#F5F5F5,stroke:#00695C,stroke-width:2px
    style external fill:#F5F5F5,stroke:#C62828,stroke-width:2px
```

**Legend:**
- **Blue** (Presentation): User-facing components
- **Green** (Application): Business logic and API
- **Orange** (Async): Background job processing
- **Purple** (Domain): Core analysis functionality
- **Teal** (Data): Persistence layer
- **Red** (External): Third-party services

**Data Flow:**
1. User uploads code via frontend or MCP Server (Claude Desktop)
2. API enqueues analysis job to Redis
3. ARQ worker picks up job and executes metrics collection
4. Results stored in PostgreSQL
5. User polls API for completed results

---

# Frontend architecture

```mermaid
graph LR
    subgraph routing["App Router"]
        pages[Pages<br/>Dashboard, Login, Signup]
    end

    subgraph auth["Authentication"]
        auth_provider[Auth0 Provider<br/>UserProvider]
        auth_guard[AuthGuard<br/>Route Protection]
    end

    subgraph features["Feature Components"]
        upload[Upload Form<br/>File + GitHub]
        results[Results Display<br/>Charts + Metrics]
        polling[Job Polling<br/>useJobPolling hook]
    end

    subgraph ui["UI Components"]
        metric_cards[Metric Cards<br/>10 Quality Metrics]
        charts[Charts<br/>Spider + Heatmap]
        file_tree[File Tree<br/>Per-file Metrics]
    end

    subgraph services["Services"]
        api_client[API Client<br/>lib/api.ts]
    end

    subgraph external_services["External"]
        auth0{{Auth0<br/>OAuth 2.0}}
        backend[(Backend API<br/>FastAPI)]
    end

    %% Flow
    pages --> auth_provider
    auth_provider --> auth_guard
    auth_guard --> features

    upload & results --> ui
    upload & polling --> api_client

    auth_provider <-.->|OAuth| auth0
    api_client -->|HTTPS + JWT| backend

    %% Styling
    classDef routingStyle fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#000
    classDef authStyle fill:#C8E6C9,stroke:#2E7D32,stroke-width:2px,color:#000
    classDef featureStyle fill:#FFE0B2,stroke:#E65100,stroke-width:2px,color:#000
    classDef uiStyle fill:#F8BBD0,stroke:#AD1457,stroke-width:2px,color:#000
    classDef serviceStyle fill:#E1BEE7,stroke:#6A1B9A,stroke-width:2px,color:#000
    classDef externalStyle fill:#FFCDD2,stroke:#C62828,stroke-width:2px,color:#000

    class pages routingStyle
    class auth_provider,auth_guard authStyle
    class upload,results,polling featureStyle
    class metric_cards,charts,file_tree uiStyle
    class api_client serviceStyle
    class auth0,backend externalStyle

    style routing fill:#F5F5F5,stroke:#1565C0,stroke-width:2px
    style auth fill:#F5F5F5,stroke:#2E7D32,stroke-width:2px
    style features fill:#F5F5F5,stroke:#E65100,stroke-width:2px
    style ui fill:#F5F5F5,stroke:#AD1457,stroke-width:2px
    style services fill:#F5F5F5,stroke:#6A1B9A,stroke-width:2px
    style external_services fill:#F5F5F5,stroke:#C62828,stroke-width:2px
```

**Architecture Layers:**
- 🔵 **Routing** - Next.js 14 App Router pages
- 🟢 **Authentication** - Auth0 OAuth 2.0 (Google + GitHub)
- 🟠 **Features** - Upload, Results, Job Polling
- 🌸 **UI** - Reusable components (shadcn/ui + Radix)
- 🟣 **Services** - API client with JWT authentication
- 🔴 **External** - Auth0 and Backend API

---

# Web Auth Flow (Frontend)

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#E3F2FD','primaryTextColor':'#000','primaryBorderColor':'#1976D2','lineColor':'#1976D2','secondaryColor':'#FFF3E0','tertiaryColor':'#F1F8E9','noteBkgColor':'#E8EAF6','noteTextColor':'#000','noteBorderColor':'#3F51B5','actorBorder':'#1976D2','actorBkg':'#ffffff','actorTextColor':'#000','actorLineColor':'#1976D2','signalColor':'#1976D2','signalTextColor':'#000','labelBoxBkgColor':'#E8EAF6','labelBoxBorderColor':'#3F51B5','labelTextColor':'#000','loopTextColor':'#000','activationBorderColor':'#1976D2','activationBkgColor':'#BBDEFB','sequenceNumberColor':'#000','altBorderColor':'#F57C00','altBkgColor':'#FFF8E1'}}}%%
sequenceDiagram
   participant U as User
   participant F as Frontend
   participant A as Auth0
   participant B as Backend
   participant D as Database

   Note over U,D: OAuth 2.0 Authentication

   U->>F: Login with Google/GitHub
   F->>A: Start OAuth flow
   A->>A: Check email allowlist

   alt Email Allowed
       A-->>F: Access token
       F->>B: Request with token
       B->>A: Verify JWT
       A-->>B: Token valid
       B->>D: Save user
       D-->>B: User data
       B-->>F: 200 OK
       F-->>U: Access granted
   else Email Denied
       A-->>F: Access denied
       F-->>U: Show error
   end

   Note over U,D: Allowlist in Auth0
```

# Device Auth Flow (MCP)

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#FCE4EC','primaryTextColor':'#000','primaryBorderColor':'#C2185B','lineColor':'#C2185B','secondaryColor':'#FFF3E0','tertiaryColor':'#F1F8E9','noteBkgColor':'#FCE4EC','noteTextColor':'#000','noteBorderColor':'#C2185B','actorBorder':'#C2185B','actorBkg':'#ffffff','actorTextColor':'#000','actorLineColor':'#C2185B','signalColor':'#C2185B','signalTextColor':'#000','labelBoxBkgColor':'#FCE4EC','labelBoxBorderColor':'#C2185B','labelTextColor':'#000','loopTextColor':'#000','activationBorderColor':'#C2185B','activationBkgColor':'#FCE4EC','sequenceNumberColor':'#000','altBorderColor':'#F57C00','altBkgColor':'#FFF8E1'}}}%%
sequenceDiagram
   participant U as User
   participant M as MCP Server
   participant B as Backend
   participant A as Auth0

   Note over U,A: OAuth Device Flow (RFC 8628)

   M->>B: POST /api/auth/device/code
   B->>A: Request device code
   A-->>B: device_code, user_code, verification_uri
   B-->>M: Return codes to display

   M->>U: Display: "Visit URL, enter code"
   U->>A: Visit verification URL
   U->>A: Enter user_code + login

   loop Poll for token
       M->>B: POST /api/auth/device/token
       B->>A: Poll with device_code
       alt Pending
           A-->>B: authorization_pending
           B-->>M: status: pending
       else Authorized
           A-->>B: access_token, refresh_token
           B-->>M: Return tokens
       end
   end

   M->>M: Store tokens locally
   M->>B: API requests with JWT
   B->>A: Verify JWT
   A-->>B: Token valid
   B-->>M: Response

   M->>B: POST /api/auth/device/refresh
   B->>A: Refresh request
   A-->>B: New access_token
   B-->>M: Return token
```

# Code analysis flow

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#E3F2FD','primaryTextColor':'#000','primaryBorderColor':'#1976D2','lineColor':'#1976D2','secondaryColor':'#FFF3E0','tertiaryColor':'#F1F8E9','edgeLabelBackground':'#ffffff','clusterBkg':'#ffffff','clusterBorder':'#1976D2','titleColor':'#000'}}}%%
flowchart LR
    A1["Upload File"]
    A2["GitHub URL"]
    B{"Authenticated?"}
    C1["Extract Files"]
    C2["Clone Repo"]
    D["Run Metrics"]
    E["Aggregate"]
    F[("Save to DB")]
    G["Return JSON"]
    H["Display"]
    L["❌ Error"]

    A1 --> B
    A2 --> B
    B -->|"Yes"| C1
    B -->|"Yes"| C2
    B -->|"No"| L
    C1 --> D
    C2 --> D
    D --> E
    E --> F
    F --> G
    G --> H

    style A1 fill:#E3F2FD,stroke:#1976D2,stroke-width:2px,color:#000
    style A2 fill:#E3F2FD,stroke:#1976D2,stroke-width:2px,color:#000
    style B fill:#FFF3E0,stroke:#F57C00,stroke-width:2px,color:#000
    style C1 fill:#E3F2FD,stroke:#1976D2,stroke-width:2px,color:#000
    style C2 fill:#E3F2FD,stroke:#1976D2,stroke-width:2px,color:#000
    style D fill:#EDE7F6,stroke:#673AB7,stroke-width:2px,color:#000
    style E fill:#EDE7F6,stroke:#673AB7,stroke-width:2px,color:#000
    style F fill:#FFF3E0,stroke:#F57C00,stroke-width:2px,color:#000
    style G fill:#E8F5E9,stroke:#388E3C,stroke-width:2px,color:#000
    style H fill:#E8F5E9,stroke:#388E3C,stroke-width:2px,color:#000
    style L fill:#FFEBEE,stroke:#C62828,stroke-width:2px,color:#000

    linkStyle default stroke:#1976D2,stroke-width:2px
```

# Backend API endpoints

```mermaid
---
config:
 theme: base
 themeVariables:
   primaryColor: '#E3F2FD'
   primaryTextColor: '#000'
   primaryBorderColor: '#1976D2'
   lineColor: '#1976D2'
   secondaryColor: '#FFF3E0'
   tertiaryColor: '#F1F8E9'
   noteBkgColor: '#E8EAF6'
   noteTextColor: '#000'
   noteBorderColor: '#3F51B5'
   edgeLabelBackground: '#ffffff'
   clusterBkg: '#ffffff'
   clusterBorder: '#1976D2'
   titleColor: '#000'
---
flowchart TB
subgraph Async["Async Job Queue (Auth Required)"]
       A1["POST /api/metrics/analyze/upload/async"]
       A2["POST /api/metrics/analyze/github/async"]
       A3["GET /api/metrics/jobs/{job_id}"]
       A4["GET /api/metrics/jobs/{job_id}/result"]
 end
subgraph History["History (Auth Required)"]
       B2["GET /api/metrics/history"]
       B3["GET /api/metrics/history/{run_id}"]
       B4["DELETE /api/metrics/history/{run_id}"]
       B5["GET /api/metrics/file-tree/{run_id}"]
 end
subgraph Auth2["Web Auth (Frontend)"]
       D1["GET /api/auth/me"]
       D2["GET /api/auth/login"]
       D3["GET /api/auth/logout"]
 end
subgraph Device["Device Auth (MCP)"]
       E1["POST /api/auth/device/code"]
       E2["POST /api/auth/device/token"]
       E3["POST /api/auth/device/refresh"]
 end
subgraph Public["Public"]
       C1["GET /health"]
 end
   style A1 fill:#EDE7F6,stroke:#673AB7,stroke-width:2px,color:#000
   style A2 fill:#EDE7F6,stroke:#673AB7,stroke-width:2px,color:#000
   style A3 fill:#EDE7F6,stroke:#673AB7,stroke-width:2px,color:#000
   style A4 fill:#EDE7F6,stroke:#673AB7,stroke-width:2px,color:#000
   style B2 fill:#E3F2FD,stroke:#1976D2,stroke-width:2px,color:#000
   style B3 fill:#E3F2FD,stroke:#1976D2,stroke-width:2px,color:#000
   style B4 fill:#E3F2FD,stroke:#1976D2,stroke-width:2px,color:#000
   style B5 fill:#E3F2FD,stroke:#1976D2,stroke-width:2px,color:#000
   style D1 fill:#FFF3E0,stroke:#F57C00,stroke-width:2px,color:#000
   style D2 fill:#FFF3E0,stroke:#F57C00,stroke-width:2px,color:#000
   style D3 fill:#FFF3E0,stroke:#F57C00,stroke-width:2px,color:#000
   style E1 fill:#FCE4EC,stroke:#C2185B,stroke-width:2px,color:#000
   style E2 fill:#FCE4EC,stroke:#C2185B,stroke-width:2px,color:#000
   style E3 fill:#FCE4EC,stroke:#C2185B,stroke-width:2px,color:#000
   style C1 fill:#E8F5E9,stroke:#388E3C,stroke-width:2px,color:#000
   style Async fill:transparent,stroke:#673AB7,stroke-width:2px,color:#000
   style History fill:transparent,stroke:#1976D2,stroke-width:2px,color:#000
   style Auth2 fill:transparent,stroke:#F57C00,stroke-width:2px,color:#000
   style Device fill:transparent,stroke:#C2185B,stroke-width:2px,color:#000
   style Public fill:transparent,stroke:#388E3C,stroke-width:2px,color:#000
```

# Metrics Collection pipeline

```mermaid
---
config:
 theme: base
 themeVariables:
   primaryColor: '#E3F2FD'
   primaryTextColor: '#000'
   primaryBorderColor: '#1976D2'
   lineColor: '#1976D2'
   secondaryColor: '#FFF3E0'
   tertiaryColor: '#F1F8E9'
   edgeLabelBackground: '#ffffff'
   clusterBkg: '#ffffff'
   clusterBorder: '#1976D2'
   titleColor: '#000'
---
flowchart LR
subgraph Analysis[" "]
       B["Lint"]
       C["Complexity"]
       D["Docs"]
       E["Security"]
       F["Dead Code"]
       G["Dependencies"]
       H1["Churn"]
       I1["Duplication"]
       J1["Maintainability"]
       K1["Size"]
 end
   A["Python Code"] --> Analysis
   Analysis --> H["Aggregate"]
   H --> I["JSON"]
   I --> J["Dashboard"]
   style A fill:#E3F2FD,stroke:#1976D2,stroke-width:2px,color:#000
   style B fill:#EDE7F6,stroke:#673AB7,stroke-width:2px,color:#000
   style C fill:#EDE7F6,stroke:#673AB7,stroke-width:2px,color:#000
   style D fill:#EDE7F6,stroke:#673AB7,stroke-width:2px,color:#000
   style E fill:#EDE7F6,stroke:#673AB7,stroke-width:2px,color:#000
   style F fill:#EDE7F6,stroke:#673AB7,stroke-width:2px,color:#000
   style G fill:#EDE7F6,stroke:#673AB7,stroke-width:2px,color:#000
   style H1 fill:#EDE7F6,stroke:#673AB7,stroke-width:2px,color:#000
   style I1 fill:#EDE7F6,stroke:#673AB7,stroke-width:2px,color:#000
   style J1 fill:#EDE7F6,stroke:#673AB7,stroke-width:2px,color:#000
   style K1 fill:#EDE7F6,stroke:#673AB7,stroke-width:2px,color:#000
   style H fill:#FFF3E0,stroke:#F57C00,stroke-width:2px,color:#000
   style I fill:#FFF3E0,stroke:#F57C00,stroke-width:2px,color:#000
   style J fill:#E8F5E9,stroke:#388E3C,stroke-width:2px,color:#000
   style Analysis fill:transparent,stroke:#673AB7,stroke-width:2px,color:#000
```

# Security Flow

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#E3F2FD','primaryTextColor':'#000','primaryBorderColor':'#1976D2','lineColor':'#1976D2','secondaryColor':'#FFF3E0','tertiaryColor':'#F1F8E9','edgeLabelBackground':'#ffffff','clusterBkg':'#ffffff','clusterBorder':'#1976D2','titleColor':'#000'}}}%%
flowchart TB
   A["API Request"]
   B["CORS Check"]
   C["Rate Limit"]
   D["Email Allowlist"]
   E["JWT Verification"]
   F["Input Validation"]
   G["Owner Check"]
   H["✓ Request Authorized"]

   A --> B
   B --> C
   C --> D
   D --> E
   E --> F
   F --> G
   G --> H

   style A fill:#E3F2FD,stroke:#1976D2,stroke-width:2px,color:#000
   style B fill:#FFF3E0,stroke:#F57C00,stroke-width:2px,color:#000
   style C fill:#FFF3E0,stroke:#F57C00,stroke-width:2px,color:#000
   style D fill:#FFF3E0,stroke:#F57C00,stroke-width:2px,color:#000
   style E fill:#FFF3E0,stroke:#F57C00,stroke-width:2px,color:#000
   style F fill:#FFF3E0,stroke:#F57C00,stroke-width:2px,color:#000
   style G fill:#FFF3E0,stroke:#F57C00,stroke-width:2px,color:#000
   style H fill:#E8F5E9,stroke:#388E3C,stroke-width:2px,color:#000

   linkStyle default stroke:#1976D2,stroke-width:2px
```

**Legend:** 🔵 Blue = Entry Point | 🟠 Orange = Security Checks | 🟢 Green = Authorized

# Async Job Queue Flow

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant Redis
    participant Worker
    participant DB

    Client->>+API: Submit Analysis
    API->>DB: Create Job (status: pending)
    API->>Redis: Enqueue Job
    API-->>-Client: job_id (202 Accepted)

    Worker->>+Redis: Poll for Jobs
    Redis-->>-Worker: Job Data
    Worker->>DB: Update Status: 'running'
    Worker->>Worker: Run Analysis
    Worker->>DB: Save Results
    Worker->>DB: Update Status: 'completed'

    Client->>+API: Poll Job Status
    API->>DB: Query Job Status
    API-->>-Client: Status: 'completed'

    Client->>+API: Fetch Result
    API->>DB: Get Analysis Result
    API->>DB: Delete Job
    API-->>-Client: Analysis Results
```

This sequence diagram shows the chronological interactions between all actors in the async job queue system.

# Testing

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#E3F2FD','primaryTextColor':'#000','primaryBorderColor':'#1976D2','lineColor':'#1976D2','secondaryColor':'#FFF3E0','tertiaryColor':'#F1F8E9','edgeLabelBackground':'#ffffff','clusterBkg':'#ffffff','clusterBorder':'#1976D2','titleColor':'#000'}}}%%
flowchart TD
   PR["Pull Request"]
   GH["GitHub Actions"]

   subgraph Backend["Backend (Python)"]
      BLint["Ruff Lint"]
      BTest["Pytest<br/>618 tests, 85%+ coverage"]
   end

   subgraph Frontend["Frontend (TypeScript)"]
      FLint["ESLint"]
      FTest["Vitest<br/>Component tests"]
   end

   Merge["Merge"]
   Deploy["Deploy"]
   Prod["Production"]

   PR --> GH
   GH --> Backend
   GH --> Frontend
   BLint -->|Pass| Merge
   BTest -->|Pass| Merge
   FLint -->|Pass| Merge
   FTest -->|Pass| Merge
   Merge --> Deploy
   Deploy --> Prod

   style PR fill:#E3F2FD,stroke:#1976D2,stroke-width:2px,color:#000
   style GH fill:#E3F2FD,stroke:#1976D2,stroke-width:2px,color:#000
   style BLint fill:#FFF3E0,stroke:#F57C00,stroke-width:2px,color:#000
   style BTest fill:#FFF3E0,stroke:#F57C00,stroke-width:2px,color:#000
   style FLint fill:#FCE4EC,stroke:#C2185B,stroke-width:2px,color:#000
   style FTest fill:#FCE4EC,stroke:#C2185B,stroke-width:2px,color:#000
   style Merge fill:#E8F5E9,stroke:#388E3C,stroke-width:2px,color:#000
   style Deploy fill:#E8F5E9,stroke:#388E3C,stroke-width:2px,color:#000
   style Prod fill:#E8F5E9,stroke:#388E3C,stroke-width:2px,color:#000
   style Backend fill:#F5F5F5,stroke:#F57C00,stroke-width:2px
   style Frontend fill:#F5F5F5,stroke:#C2185B,stroke-width:2px

   linkStyle default stroke:#1976D2,stroke-width:2px
```

**Legend:** 🔵 Blue = Trigger | 🟠 Orange = Backend | 🌸 Pink = Frontend | 🟢 Green = Deployment

# Database Schema

```mermaid
erDiagram
    USERS ||--o{ ANALYSIS_RUNS : creates
    USERS ||--o{ JOBS : submits
    JOBS }o--|| ANALYSIS_RUNS : "results in"

    USERS {
        uuid id PK "Primary key"
        string email UK "Unique email, indexed"
        string auth0_sub UK "Auth0 subject, unique, indexed, nullable"
        timestamp last_login "Last login timestamp, nullable"
        timestamp created_at "Account creation, default now()"
    }

    JOBS {
        string job_id PK "Primary key (ARQ job ID)"
        uuid user_id FK "Foreign key to users.id, indexed"
        string status "Job status: pending, running, completed, failed"
        string source_kind "Source type: upload or github"
        jsonb source_metadata "Source details (filename, repo_url, etc)"
        uuid result_run_id FK "Foreign key to analysis_runs.run_id, nullable"
        string error_message "Error message if failed, nullable"
        timestamp created_at "Job creation, default now(), indexed"
        timestamp updated_at "Last status update, default now()"
    }

    ANALYSIS_RUNS {
        uuid run_id PK "Primary key"
        uuid user_id FK "Foreign key to users.id, indexed"
        timestamp created_at "Analysis timestamp, default now(), indexed"
        string upload_filename "Uploaded file name, nullable"
        string submission_type "Source type: upload or github"
        string project_name "User-assigned project name, nullable"
        string author_type "Code authorship: human or ai, nullable"
        string author_model "AI model if applicable, nullable"
        string git_repo_url "GitHub repository URL, nullable"
        string git_branch "Git branch name, nullable"
        string git_commit_sha "Git commit SHA, nullable"
        jsonb metrics "Aggregate metrics from all 10 collectors"
        jsonb file_tree "Per-file metrics and severity scores, nullable"
        jsonb errors "Error messages array, default []"
    }
```

# MCP Server Architecture

```mermaid
---
config:
 theme: base
 themeVariables:
   primaryColor: '#FCE4EC'
   primaryTextColor: '#000'
   primaryBorderColor: '#C2185B'
   lineColor: '#C2185B'
   secondaryColor: '#FFF3E0'
   tertiaryColor: '#F1F8E9'
   edgeLabelBackground: '#ffffff'
   clusterBkg: '#ffffff'
   clusterBorder: '#C2185B'
   titleColor: '#000'
---
flowchart TB
    subgraph claude["Claude Desktop"]
        desktop[Claude Desktop App]
    end

    subgraph mcp["MCP Server (subprocess)"]
        server[FastMCP Server<br/>server.py]
        tools[Tools Module<br/>6 tools]
        client[HTTP Client<br/>client.py]
        auth_mod[Auth Module<br/>auth.py]
        tokens[Token Store<br/>~/.vibe8/mcp_tokens.json]
    end

    subgraph backend["Vibe8 Backend"]
        api[FastAPI REST API<br/>includes /api/auth/device/*]
    end

    subgraph external["External"]
        auth0{{Auth0}}
    end

    desktop <-->|stdio| server
    server --> tools
    tools --> client
    client --> auth_mod
    auth_mod <--> tokens
    client -->|HTTPS + JWT| api
    auth_mod -->|Device Flow| api
    api <-->|OAuth| auth0

    style desktop fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#000
    style server fill:#FCE4EC,stroke:#C2185B,stroke-width:2px,color:#000
    style tools fill:#FCE4EC,stroke:#C2185B,stroke-width:2px,color:#000
    style client fill:#FCE4EC,stroke:#C2185B,stroke-width:2px,color:#000
    style auth_mod fill:#FCE4EC,stroke:#C2185B,stroke-width:2px,color:#000
    style tokens fill:#B2DFDB,stroke:#00695C,stroke-width:2px,color:#000
    style api fill:#C8E6C9,stroke:#2E7D32,stroke-width:2px,color:#000
    style auth0 fill:#FFCDD2,stroke:#C62828,stroke-width:2px,color:#000

    style claude fill:#F5F5F5,stroke:#1565C0,stroke-width:2px
    style mcp fill:#F5F5F5,stroke:#C2185B,stroke-width:2px
    style backend fill:#F5F5F5,stroke:#2E7D32,stroke-width:2px
    style external fill:#F5F5F5,stroke:#C62828,stroke-width:2px
```

**MCP Components:**
- **FastMCP Server** - Entry point, registers tools with Claude Desktop
- **Tools Module** - 6 tools: analyze_code, check_job_status, get_job_result, find_analysis, get_file_tree, finish_login
- **HTTP Client** - Makes authenticated requests to Vibe8 API
- **Auth Module** - Handles OAuth Device Flow authentication
- **Token Store** - Persists tokens locally (~/.vibe8/mcp_tokens.json)
