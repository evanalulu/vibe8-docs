# Vibe8 Authentication Guide

Complete guide to setting up and configuring Auth0 OAuth 2.0 authentication for Vibe8.

## Table of Contents
- [Quick Setup (Already Configured)](#quick-setup-already-configured)
- [Overview](#overview)
- [Web Authentication Flow](#web-authentication-flow)
- [Device Authentication Flow](#device-authentication-flow)
- [Auth0 Setup](#auth0-setup)
- [Google OAuth Setup](#google-oauth-setup)
- [GitHub OAuth Setup](#github-oauth-setup)
- [Email Allowlist Configuration](#email-allowlist-configuration)
- [Environment Configuration](#environment-configuration)
- [MCP Server Device Flow Configuration](#mcp-server-device-flow-configuration)
- [Token Lifetime and Refresh Configuration](#token-lifetime-and-refresh-configuration)
- [Security Features](#security-features)
- [Frontend Integration](#frontend-integration)

## Quick Setup (Already Configured)

If Auth0 is already set up and you need credentials for a new local machine, follow these steps.

### Finding Your Auth0 Credentials

**Navigate to Auth0 Dashboard**: [https://manage.auth0.com/](https://manage.auth0.com/)

#### 1. Get Application Credentials (Frontend)

1. Go to **Applications** → **Applications**
2. Click on your Vibe8 application (Regular Web Application)
3. Navigate to **Settings** tab
4. Copy the following values:
   - **Domain** → Use for `AUTH0_ISSUER_BASE_URL` (frontend) and `AUTH0_DOMAIN` (backend)
     - Frontend format: `https://your-tenant.auth0.com`
     - Backend format: `your-tenant.auth0.com` (no https://)
   - **Client ID** → Use for `AUTH0_CLIENT_ID` (frontend)
   - **Client Secret** → Click "Show" and copy for `AUTH0_CLIENT_SECRET` (frontend)
     - ⚠️ **Keep this secret!** Never commit to git

#### 2. Get API Audience (Backend)

1. Go to **Applications** → **APIs**
2. Click on your Vibe8 API
3. Copy the **Identifier** → Use for:
   - `AUTH0_API_AUDIENCE` (backend)
   - `AUTH0_AUDIENCE` (frontend)

#### 3. Get MCP Client ID (for Claude Desktop)

1. Go to **Applications** → **Applications**
2. Click on your Vibe8 MCP application (Native Application)
3. Copy the **Client ID** → Use for `AUTH0_MCP_CLIENT_ID` (backend)

#### 4. Verify Callback URLs

While in **Applications** → **Applications** → **Settings** (for the Regular Web Application):

1. Check **Allowed Callback URLs** includes:
   ```
   http://localhost:3000/api/auth/callback
   ```
2. Check **Allowed Logout URLs** includes:
   ```
   http://localhost:3000
   ```
3. Check **Allowed Web Origins** includes:
   ```
   http://localhost:3000
   ```

If missing, add them and click **Save Changes**.

#### 5. Generate New Secrets (Frontend Only)

For `AUTH0_SECRET` (frontend cookie encryption):
```bash
openssl rand -hex 32
```

⚠️ **Generate a NEW secret for each machine** - do not reuse across environments!

### Quick Setup Checklist

Once you have all credentials:

- [ ] Created `.env` in project root with all Auth0 credentials (backend and frontend)
- [ ] Generated new `AUTH0_SECRET` with `openssl rand -hex 32`
- [ ] Verified callback URLs in Auth0 Dashboard
- [ ] Tested login flow with Google/GitHub
- [ ] Verified your email is in Auth0 Action allowlist (see [Email Allowlist Configuration](#email-allowlist-configuration))

**Troubleshooting**: If you encounter "invalid credentials" errors, verify:
- Backend `AUTH0_DOMAIN` has no `https://` prefix
- Backend `AUTH0_ISSUER` has `https://` prefix AND trailing slash
- Frontend `AUTH0_ISSUER_BASE_URL` has `https://` prefix but NO trailing slash
- `AUTH0_AUDIENCE` matches exactly between frontend and backend

---

## Overview

Vibe8 uses **Auth0 OAuth 2.0** for secure user management with the following features:

- **Custom Login/Signup Pages**: Branded authentication pages bypassing Auth0 Universal Login
- **Social Login**: Direct authentication with Google and GitHub providers
- **JWT Token Verification**: RS256 algorithm with JWKS validation
- **Email Allowlist**: Access control managed via Auth0 Post-Login Action
- **User Isolation**: Users can only access their own analysis data

## Web Authentication Flow

```
┌─────────┐
│  User   │
└────┬────┘
     │ 1. Clicks Google/GitHub button
     ▼
┌─────────────────────┐
│ Custom Login Page   │
│  (frontend)         │
└────┬────────────────┘
     │ 2. Redirects to Auth0 with connection parameter
     ▼
┌─────────────────────┐
│     Auth0 OAuth     │ 3. Handles provider authentication
│  (Google/GitHub)    │
└────┬────────────────┘
     │ 4. OAuth callback
     ▼
┌─────────────────────┐
│ Auth0 Post-Login    │ 5. Checks email allowlist
│      Action         │
└────┬────────────────┘
     │ 6a. If email not approved: Deny access
     │ 6b. If email approved: Continue
     ▼
┌─────────────────────┐
│  Frontend Callback  │ 7. Exchanges code for access token
└────┬────────────────┘
     │ 8. Stores token in session
     ▼
┌─────────────────────┐
│   Backend API       │ 9. Validates JWT via JWKS
│  (Authenticated)    │ 10. Loads/creates user in database
└─────────────────────┘
```

### Flow Details

1. **Login Initiation**: User visits `/login` or `/signup` page on frontend
2. **Provider Selection**: User clicks Google or GitHub button
3. **Auth0 OAuth**:
   - Redirect: `/api/auth/login?connection=google-oauth2` (Google)
   - Redirect: `/api/auth/login?connection=GitHub` (GitHub, case-sensitive)
4. **OAuth Flow**: Auth0 handles OAuth with selected provider
5. **Email Allowlist Check**: Auth0 Post-Login Action verifies email is approved
6. **Callback**: User redirected to frontend callback with authorization code
7. **Token Exchange**: Frontend exchanges code for access token
8. **API Authentication**: Backend validates JWT signature via Auth0 JWKS endpoint
9. **User Management**: Backend loads or creates user record in PostgreSQL

## Device Authentication Flow

For CLI/desktop environments (like Claude Desktop via MCP), OAuth Device Flow (RFC 8628) is used:

```
┌─────────────────────┐
│  MCP Server         │
│  (Claude Desktop)   │
└────┬────────────────┘
     │ 1. Request device code
     ▼
┌─────────────────────┐
│  Backend API        │ 2. Forward to Auth0
│  /api/auth/device/* │
└────┬────────────────┘
     │ 3. Return device_code + user_code
     ▼
┌─────────────────────┐
│  MCP Server         │ 4. Display: "Visit URL, enter code"
└────┬────────────────┘
     │
     ▼
┌─────────────────────┐
│      User           │ 5. Visit verification URL
│  (in browser)       │ 6. Enter user_code + login
└────┬────────────────┘
     │
     ▼
┌─────────────────────┐
│  MCP Server         │ 7. Poll /api/auth/device/token
│  (polling)          │
└────┬────────────────┘
     │ 8. Receive access_token + refresh_token
     ▼
┌─────────────────────┐
│  Token Store        │ 9. Save tokens locally
│  ~/.vibe8/          │    (~/.vibe8/mcp_tokens.json)
└─────────────────────┘
```

### Flow Details

1. **Code Request**: MCP server calls `POST /api/auth/device/code`
2. **Auth0 Request**: Backend forwards request to Auth0 Device Code endpoint
3. **Code Response**: Backend returns device_code, user_code, and verification_uri
4. **User Prompt**: MCP displays URL and code to user via Claude Desktop
5. **Browser Auth**: User visits URL in browser and enters code
6. **Login**: User authenticates with Google/GitHub (same email allowlist applies)
7. **Token Polling**: MCP polls `POST /api/auth/device/token` until authorized
8. **Token Exchange**: Once authorized, Auth0 returns access and refresh tokens
9. **Token Storage**: Tokens saved to `~/.vibe8/mcp_tokens.json` with 600 permissions
10. **API Access**: Subsequent requests use JWT for authentication

### Token Refresh

- Tokens auto-refresh using `POST /api/auth/device/refresh`
- Refresh happens transparently before token expiration
- Users only re-authenticate if refresh token expires

## Auth0 Setup

### 1. Create Auth0 Account
- Sign up at [auth0.com](https://auth0.com)
- Choose your region (affects latency and data residency)

### 2. Create Application
1. Navigate to **Applications** → **Create Application**
2. Select **Regular Web Application**
3. Note the following credentials:
   - **Domain**: `your-tenant.auth0.com`
   - **Client ID**: `abc123...`
   - **Client Secret**: `xyz789...` (keep secure!)

### 3. Create API
1. Navigate to **Applications** → **APIs** → **Create API**
2. Set **API Identifier** (audience): `https://your-api-audience`
3. Set **Signing Algorithm**: `RS256`
4. Note the **Identifier** (used as `AUTH0_API_AUDIENCE`)

### 4. Configure Allowed Callback URLs
Add the following URLs to your Application settings:

**Development**:
```
http://localhost:3000/api/auth/callback
```

**Production**:
```
https://your-frontend-domain.com/api/auth/callback
```

### 5. Configure Allowed Logout URLs
**Development**:
```
http://localhost:3000
```

**Production**:
```
https://your-frontend-domain.com
```

### 6. Configure Allowed Web Origins
Same as logout URLs (for silent authentication):
```
http://localhost:3000
https://your-frontend-domain.com
```

## Google OAuth Setup

### 1. Google Cloud Console
1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Create a new project or select existing project

### 2. Enable Google+ API
1. Navigate to **APIs & Services** → **Library**
2. Search for "Google+ API"
3. Click **Enable**

### 3. Create OAuth Credentials
1. Navigate to **APIs & Services** → **Credentials**
2. Click **Create Credentials** → **OAuth 2.0 Client ID**
3. Select **Web application**
4. Set **Name**: "Vibe8 Auth0"

### 4. Configure OAuth Consent Screen
1. Navigate to **OAuth consent screen**
2. Set **User type**: External
3. Fill in required fields:
   - App name: "Vibe8"
   - User support email: your email
   - Developer contact: your email
4. Add **Scopes**:
   - `email`
   - `profile`
   - `openid`

### 5. Set Authorized Redirect URIs
Add Auth0 callback URL:
```
https://YOUR-AUTH0-DOMAIN/login/callback
```
Example: `https://vibe8-dev.auth0.com/login/callback`

### 6. Get Credentials
- Copy **Client ID**
- Copy **Client Secret**

### 7. Configure in Auth0
1. Navigate to **Authentication** → **Social** → **Google**
2. Enable the connection
3. Paste **Client ID** and **Client Secret**
4. Ensure connection name is exactly: `google-oauth2` (case-sensitive)
5. Select attributes to request: `email`, `profile`
6. Enable connection for your Application

## GitHub OAuth Setup

### 1. GitHub Developer Settings
1. Go to [GitHub Developer Settings](https://github.com/settings/developers)
2. Click **New OAuth App**

### 2. Register OAuth Application
**Application name**: `Vibe8`

**Homepage URL**:
- Development: `http://localhost:3000`
- Production: `https://your-frontend-domain.com`

**Authorization callback URL**:
```
https://YOUR-AUTH0-DOMAIN/login/callback
```
Example: `https://vibe8-dev.auth0.com/login/callback`

### 3. Get Credentials
- Copy **Client ID**
- Generate and copy **Client Secret**

### 4. Configure in Auth0
1. Navigate to **Authentication** → **Social** → **GitHub**
2. Enable the connection
3. Paste **Client ID** and **Client Secret**
4. **IMPORTANT**: Ensure connection name is exactly: `GitHub` (capital G, capital H)
5. Select scopes: `user:email`, `read:user`
6. Enable connection for your Application

## Email Allowlist Configuration

Access control is managed via an Auth0 Post-Login Action. No admin UI is needed.

### Setup Email Allowlist Action

1. Navigate to **Auth0 Dashboard** → **Actions** → **Library**
2. Click **Create Action** → **Build from scratch**
3. Select trigger: **Login / Post-Login**
4. Name: `Email Allowlist`
5. Add the following code:

```javascript
exports.onExecutePostLogin = async (event, api) => {
  // Define allowed emails
  const allowedEmails = [
    'user1@example.com',
    'user2@example.com',
    'admin@yourcompany.com',
    // Add more approved emails here
  ];

  // Check if user's email is in allowlist
  if (!allowedEmails.includes(event.user.email)) {
    api.access.deny('Access denied. Contact administrator to request access.');
  }
};
```

6. Click **Deploy**
7. Navigate to **Actions** → **Flows** → **Login**
8. Drag the "Email Allowlist" action into the flow (between "Start" and "Complete")
9. Click **Apply**

### Managing Access

**To add a user**:
1. Edit the Action code
2. Add email to `allowedEmails` array
3. Click **Deploy**
4. Changes take effect immediately on next login

**To remove a user**:
1. Edit the Action code
2. Remove email from `allowedEmails` array
3. Click **Deploy**
4. User will be denied on next login attempt

### Testing the Allowlist
1. Try logging in with an unapproved email
2. Should see error: "Access denied. Contact administrator to request access."
3. Add email to allowlist and deploy
4. Try logging in again - should succeed

## Environment Configuration

**Single .env file:** All configuration is in the **project root** `.env` file (not in backend/ or frontend/ directories).

Create `.env` file in project root:

```bash
# ==============================================================================
# AUTH0 CONFIGURATION (REQUIRED)
# ==============================================================================
# Backend Auth0
AUTH0_DOMAIN=your-tenant.auth0.com
AUTH0_API_AUDIENCE=https://your-api-audience
AUTH0_ISSUER=https://your-tenant.auth0.com/
AUTH0_ALGORITHMS=RS256

# Frontend Auth0
AUTH0_ISSUER_BASE_URL=https://your-tenant.auth0.com
AUTH0_CLIENT_ID=your-client-id
AUTH0_CLIENT_SECRET=your-client-secret
AUTH0_SECRET=your-random-secret-generate-with-openssl
AUTH0_BASE_URL=http://localhost:3000
AUTH0_AUDIENCE=https://your-api-audience
AUTH0_SCOPE=openid profile email offline_access

# ==============================================================================
# DATABASE & REDIS
# ==============================================================================
DATABASE_URL=postgresql://localhost:5432/vibe8_dev
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0

# ==============================================================================
# API CONFIGURATION
# ==============================================================================
NEXT_PUBLIC_API_URL=http://localhost:8000
ENVIRONMENT=development
PORT=8000
CORS_ORIGINS=http://localhost:3000
MAX_UPLOAD_MB=5
RATE_LIMIT_PER_MINUTE=60
LOG_LEVEL=INFO

# ==============================================================================
# ARQ WORKER & JOB QUEUE
# ==============================================================================
ARQ_MAX_JOBS=20
ARQ_JOB_TIMEOUT=600
ARQ_KEEP_RESULT=3600
ARQ_POLL_DELAY=0.1

# ==============================================================================
# OTHER SETTINGS
# ==============================================================================
GITHUB_CLONE_TIMEOUT=300
NEXT_PUBLIC_JOB_POLL_INTERVAL=1000
```

**Generate AUTH0_SECRET**:
```bash
openssl rand -hex 32
```

**Important Notes**:
- `AUTH0_DOMAIN`: Must NOT include `https://` protocol
- `AUTH0_ISSUER`: Must INCLUDE `https://` protocol and trailing slash
- `AUTH0_ISSUER_BASE_URL`: Must INCLUDE `https://` but NO trailing slash
- `AUTH0_ALGORITHMS`: Use `RS256` (asymmetric signing)
- `AUTH0_SCOPE` must include `offline_access` for refresh tokens
- `AUTH0_SECRET` is used for cookie encryption (keep secure!)
- Variables with `NEXT_PUBLIC_` prefix are exposed to browser

See [.env.example](.env.example) in the project root for the complete list of configuration options.

## MCP Server Device Flow Configuration

The MCP server uses OAuth Device Flow for authentication in CLI/desktop environments. This requires a separate **Native Application** in Auth0.

### 1. Create Native Application

1. Navigate to **Applications** → **Create Application**
2. Select **Native** as the application type
3. Name it something like "Vibe8 MCP"
4. Note the **Client ID** (used as `AUTH0_MCP_CLIENT_ID`)

### 2. Enable Device Code Grant

**Location**: Dashboard > Applications > Applications > [Your Native App] > Settings > Advanced Settings > Grant Types

**Required**: Enable:
- Device Code
- Refresh Token

### 3. Configure API Access

**Location**: Dashboard > Applications > APIs > [Your API] > Settings

**Required**: Enable "Allow Offline Access" to allow refresh token issuance.

### 4. Usage

Users configure the MCP server with these Auth0 values in their Claude Desktop config:
- `AUTH0_DOMAIN`: Your Auth0 tenant domain
- `AUTH0_MCP_CLIENT_ID`: The Native app's Client ID
- `AUTH0_API_AUDIENCE`: Your API audience

See [MCP.md](MCP.md) for complete MCP server setup instructions.

## Token Lifetime and Refresh Configuration

Auth0 token lifetimes and refresh token policies are configured in the **Auth0 Dashboard**, not via environment variables. These settings control session duration, automatic token refresh, and security policies.

### Required Dashboard Configuration

Navigate to your Auth0 Dashboard at [https://manage.auth0.com/](https://manage.auth0.com/) and configure the following settings:

#### 1. Access Token Lifetime

**Location**: Dashboard > Applications > APIs > [Your API] > Token Settings

**Setting**: Token Expiration
- **Recommended**: 3600 seconds (1 hour)
- **Default**: 86400 seconds (24 hours)
- **Range**: Any value up to 2,592,000 seconds (30 days)
- **Security**: Shorter lifetimes limit exposure from stolen tokens

**Purpose**: Controls how long access tokens remain valid before requiring refresh.

#### 2. Refresh Token Rotation (Required for Security)

**Location**: Dashboard > Applications > Applications > [Your App] > Settings > Refresh Token Rotation

**Configuration**:
- **Toggle ON**: "Rotation"
- **Rotation Overlap Period**: 10 seconds (prevents concurrency issues)

**Purpose**: Prevents stolen refresh tokens from being reused. Each refresh token can only be used once to obtain a new access token.

**Security Benefit**: If a refresh token is compromised and used by an attacker, the legitimate user's next refresh attempt will fail, alerting you to potential token theft.

#### 3. Refresh Token Expiration

**Location**: Dashboard > Applications > Applications > [Your App] > Settings > Refresh Token Expiration

##### a) Inactivity Timeout (Idle Lifetime)

- **Toggle ON**: "Inactivity Expiration"
- **Recommended**: 7200 seconds (2 hours)
- **Default**: 2,592,000 seconds (30 days)
- **Purpose**: Logs out users after a period of inactivity

##### b) Absolute Timeout (Maximum Lifetime)

- **Toggle ON**: "Absolute Expiration"
- **Recommended**: 604800 seconds (7 days)
- **Purpose**: Forces re-authentication after maximum time
- **Security**: Ensures periodic password/MFA verification

#### 4. Offline Access (Required for Refresh Tokens)

**Location**: Dashboard > Applications > APIs > [Your API] > Settings

**Setting**: Allow Offline Access
- **Toggle ON**: Enable this setting

**Purpose**: Enables refresh token issuance for this API. Without this setting, refresh tokens will not be issued and users will need to re-authenticate when access tokens expire.

#### 5. Application Grant Types (Required)

**Location**: Dashboard > Applications > Applications > [Your App] > Settings > Advanced Settings > Grant Types

**Required**: Ensure "Refresh Token" is checked

**Purpose**: Allows your application to use refresh tokens to obtain new access tokens without user re-authentication.

### Frontend Session Management

The Auth0 Next.js SDK manages frontend sessions automatically with these defaults:

- **Rolling sessions**: Enabled (session extends on activity)
- **Absolute timeout**: 7 days (maximum login duration)
- **Inactivity timeout**: 1 day (logout after 24 hours of inactivity)

These defaults work well for most applications. To customize, set these environment variables in the root `.env` file:

```bash
# Optional: Customize session behavior
AUTH0_SESSION_ROLLING=true                    # Extend session on activity
AUTH0_SESSION_ABSOLUTE_DURATION=604800        # 7 days max session
AUTH0_SESSION_ROLLING_DURATION=86400          # 1 day inactivity timeout
```

### Automatic Token Refresh

The SDK's `getAccessToken()` method handles automatic token refresh:

1. **Before Expiration**: SDK automatically refreshes access tokens using the refresh token
2. **Transparent**: Happens behind the scenes without user interaction
3. **Requires Scope**: Must include `offline_access` scope (already configured in this project)

**Current Configuration** ([frontend/app/api/auth/[auth0]/route.ts:18](frontend/app/api/auth/[auth0]/route.ts#L18)):
```typescript
scope: 'openid profile email offline_access'
```

The `offline_access` scope is **required** for refresh tokens. Without it, users must re-authenticate when access tokens expire.

### Post-Configuration Steps

**Important**: After configuring these settings in the Auth0 Dashboard:

1. Users will need to **log in again** for new token lifetimes to take effect
2. Existing sessions will continue using old token lifetimes until they expire
3. Test the configuration by logging in and monitoring token refresh behavior

### Monitoring Token Expiration

You can verify token expiration behavior by:

1. Checking browser developer tools > Application > Cookies
2. Decoding the JWT access token at [jwt.io](https://jwt.io)
3. Inspecting the `exp` claim (expiration timestamp)
4. Monitoring Auth0 logs for token refresh events

## Security Features

### JWT Token Validation

The backend validates JWT tokens using Auth0's JWKS (JSON Web Key Set):

1. **Token Extraction**: Extract Bearer token from `Authorization` header
2. **JWKS Fetch**: Fetch public keys from Auth0 JWKS endpoint
3. **Signature Verification**: Verify JWT signature using RS256 algorithm
4. **Claims Validation**:
   - `iss` (issuer): Must match `AUTH0_ISSUER`
   - `aud` (audience): Must match `AUTH0_API_AUDIENCE`
   - `exp` (expiration): Token must not be expired
5. **User Loading**: Load or create user in database based on `sub` (subject) claim

### Token Security

**Backend**:
- Tokens validated on every request
- Public key cached from JWKS endpoint
- Invalid tokens result in HTTP 401 Unauthorized
- No token storage (stateless authentication)

**Frontend**:
- Tokens stored in encrypted HTTP-only cookies (via Auth0 SDK)
- Automatic token refresh via `getAccessToken()` method (requires `offline_access` scope)
- Refresh tokens rotated on each use (configured in Auth0 Dashboard)
- Session validation on protected page load
- Automatic redirect to login on session expiration
- Secure transmission (HTTPS in production)

### Access Control

**Application-Level Authorization**:
- Email allowlist enforced by Auth0 Action (before token issuance)
- Resource ownership checked in API endpoints
- Users can only access their own analysis data
- HTTP 403 Forbidden returned for unauthorized access

**No Database-Level RLS**:
- Authorization enforced in application layer (repositories)
- Simpler to understand and debug
- Flexible for future changes

## Frontend Integration

### Auth Context Provider

The frontend uses a custom Auth context wrapping Auth0's SDK:

```typescript
// context/auth-context.tsx
import { UserProvider } from '@auth0/nextjs-auth0/client';

export const AuthProvider = ({ children }) => {
  return (
    <UserProvider>
      {children}
    </UserProvider>
  );
};
```

### Social Login Methods

Custom login/signup pages implement direct social provider authentication:

```typescript
// Login with Google
const loginWithGoogle = () => {
  window.location.href = '/api/auth/login?connection=google-oauth2';
};

// Login with GitHub
const loginWithGitHub = () => {
  window.location.href = '/api/auth/login?connection=GitHub';
};

// Signup with Google
const signupWithGoogle = () => {
  window.location.href = '/api/auth/login?connection=google-oauth2&screen_hint=signup';
};

// Signup with GitHub
const signupWithGitHub = () => {
  window.location.href = '/api/auth/login?connection=GitHub&screen_hint=signup';
};
```

**Important**: Connection names are case-sensitive:
- Google: `google-oauth2`
- GitHub: `GitHub` (capital G and H)

### Getting Access Token for API Requests

```typescript
// Create token endpoint in frontend
// app/api/auth/token/route.ts
import { getAccessToken } from '@auth0/nextjs-auth0';

export async function GET() {
  const { accessToken } = await getAccessToken();
  return Response.json({ accessToken });
}

// Use in components
const getToken = async (): Promise<string> => {
  const response = await fetch('/api/auth/token');
  const data = await response.json();
  return data.accessToken;
};

// Make authenticated API request
const uploadFile = async (file: File) => {
  const token = await getToken();
  const formData = new FormData();
  formData.append('file', file);

  const response = await fetch(`${API_BASE_URL}/api/metrics/analyze/upload/async`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`
    },
    body: formData
  });

  return await response.json();
};
```

### Protected Routes and Session Validation

The `AuthGuard` component protects routes and validates sessions on page load:

**Features**:
- Redirects unauthenticated users to login
- Validates session on component mount
- Attempts automatic token refresh
- Redirects to login if session has expired

**Implementation** ([frontend/components/auth-guard.tsx](frontend/components/auth-guard.tsx)):

```typescript
export function AuthGuard({ children }: { children: React.ReactNode }) {
  const { user, isLoading, getToken } = useAuth();

  // Validate session on mount by attempting to get a token
  // This triggers automatic refresh if needed or redirects if session is expired
  useEffect(() => {
    const validateSession = async () => {
      if (isLoading || !user) return;

      try {
        await getToken(); // Triggers refresh or redirect
      } catch (error) {
        console.log('Session validation failed, redirecting to login');
      }
    };

    validateSession();
  }, [user, isLoading, getToken]);

  // Redirect if no user
  useEffect(() => {
    if (!isLoading && !user) {
      router.push('/login');
    }
  }, [user, isLoading]);

  if (isLoading || !user) return <div>Loading...</div>;
  return <>{children}</>;
}
```

**Usage**:
```typescript
export default function DashboardPage() {
  return (
    <AuthGuard>
      <DashboardContent />
    </AuthGuard>
  );
}
```

**Behavior**:
1. User lands on protected page
2. AuthGuard validates session by calling `getToken()`
3. If token is expired but refresh token is valid → automatic refresh
4. If refresh token is expired → redirect to `/api/auth/login?returnTo=/current-page`
5. After login → user returns to original page

## Troubleshooting

### Common Issues

**"Invalid token" errors**:
- Verify `AUTH0_DOMAIN` matches your Auth0 tenant
- Ensure `AUTH0_ISSUER` has `https://` and trailing slash
- Check `AUTH0_API_AUDIENCE` matches API identifier

**"Access denied" on login**:
- Verify user's email is in Auth0 Action allowlist
- Check Action is deployed and added to Login flow
- Check browser console for error messages

**"Connection not found" errors**:
- Verify connection names are exact: `google-oauth2`, `GitHub`
- Ensure connections are enabled for your Application
- Check Auth0 Application settings

**Token not refreshing**:
- Verify `AUTH0_SECRET` is set and valid
- Check cookie settings in browser (must allow HTTP-only cookies)
- Ensure `AUTH0_BASE_URL` matches frontend URL
- Verify `offline_access` scope is included in `AUTH0_SCOPE`

**"Access denied" after app startup**:
- This issue has been fixed with automatic session validation
- AuthGuard now validates sessions on mount
- Expired sessions trigger automatic redirect to login
- Users are returned to original page after re-authentication

**CORS errors**:
- Add frontend URL to `CORS_ORIGINS` in backend
- Verify frontend URL format (no trailing slash)
- Check browser console for specific CORS error

## Related Documentation

- **[API Reference](API.md)**: Authentication requirements for endpoints
- **[Architecture Guide](ARCHITECTURE.md)**: Authentication flow details
- **[Database Schema](DATABASE.md)**: User table structure
- **[Deployment Guide](DEPLOYMENT.md)**: Production Auth0 configuration
- **[MCP Server Guide](MCP.md)**: MCP server setup and Device Flow authentication

---

**Last Updated**: 2025-01-07
