# Vibe8 Production Deployment Guide

This guide covers deploying Vibe8 to production. The current deployment uses **Docker Compose on Google Cloud Platform** with Terraform for infrastructure provisioning.

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Docker Compose Deployment](#docker-compose-deployment)
- [Configuration](#configuration)
- [Monitoring and Troubleshooting](#monitoring-and-troubleshooting)
- [Security Recommendations](#security-recommendations)

## Architecture Overview

### Current Stack

- **Frontend**: Docker container (Next.js)
- **Backend**: Docker container (FastAPI API server)
- **ARQ Worker**: Docker container (background job processor)
- **Redis**: Docker container with persistent volume (job queue)
- **PostgreSQL**: Docker container with persistent volume
- **Platform**: Google Cloud Platform (GCP)
- **Infrastructure as Code**: Terraform

### Infrastructure Components

**Known Configuration**:
- Backend API: Port 8000 (HTTP)
- Frontend: Port 3000 (HTTP)
- ARQ Worker: No exposed ports (internal)
- Redis: Port 6379 (internal)
- PostgreSQL: Port 5432 (internal)

**TODO - GCP Infrastructure** (handled by deployment team):
- [ ] Document GCP compute service (Cloud Run, GCE, GKE, etc.)
- [ ] Document VPC and networking configuration
- [ ] Document load balancing strategy (if applicable)
- [ ] Document Cloud SQL vs containerized PostgreSQL decision
- [ ] Document DNS/domain configuration

## Prerequisites

### Required Tools

```bash
# Docker and Docker Compose
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Google Cloud SDK (for GCP deployments)
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init
```

### Required Accounts

1. **Google Cloud Platform Account** with billing enabled
2. **Auth0 Account** with application configured ([setup guide](AUTH.md))
3. **Domain name** (optional, for custom domain)

## Docker Compose Deployment

### 1. Clone Repository

```bash
git clone <your-repo-url>
cd vibe8
```

### 2. Configure Environment

**For Production Deployments:**
Environment variables are managed via **Google Secret Manager** (configured by deployment team with Terraform).

**For Local Development with Docker Compose:**
Environment variables are configured in the root `.env` file. See `.env.example` in the project root for all available configuration options.

**Critical Configuration Requirements**:

**Backend**:
- `DATABASE_URL`: Must use `postgresql://` format (NOT `postgresql+psycopg2://`)
- `CORS_ORIGINS`: Must include your frontend URL (e.g., `http://YOUR_IP:3000`)
- `ALLOWED_HOSTS`: Must include your server IP or domain
- Auth0 configuration (domain, audience, issuer)

**Frontend**:
- `NEXT_PUBLIC_API_URL`: Backend URL (must be set at build time)
- `AUTH0_BASE_URL`: Frontend URL
- Auth0 configuration (issuer, client ID, secret, audience)

**Backend environment variables**:
```yaml
backend:
  environment:
    - ENVIRONMENT=production
    - DEBUG=false
    - DATABASE_URL=postgresql://vibe8:password@db:5432/vibe8_prod
    - CORS_ORIGINS=http://YOUR_IP:3000
    - ALLOWED_HOSTS=YOUR_IP,localhost,127.0.0.1
    - REDIS_HOST=redis
    - REDIS_PORT=6379
    - REDIS_PASSWORD=
    - REDIS_DB=0
    - MAX_CONCURRENT_JOBS_PER_USER=5
    - DAILY_JOB_LIMIT_PER_USER=100
    - COMPLETED_JOB_RETENTION=21600
    - STALE_JOB_THRESHOLD=7200
  depends_on:
    - db
    - redis
```

**ARQ Worker service** (⚠️ REQUIRED - add to docker-compose.yml):
```yaml
worker:
  build:
    context: ./backend
    dockerfile: Dockerfile
  image: vibe8-backend:latest
  container_name: vibe8_arq_worker
  command: arq src.application.workers.analysis_worker.WorkerSettings
  environment:
    - DATABASE_URL=postgresql://vibe8:password@db:5432/vibe8_prod
    - REDIS_HOST=redis
    - REDIS_PORT=6379
    - REDIS_PASSWORD=
    - REDIS_DB=0
    - COMPLETED_JOB_RETENTION=21600
    - STALE_JOB_THRESHOLD=7200
    - LOG_LEVEL=INFO
  depends_on:
    - db
    - redis
  restart: always
```

**Note**: Worker uses same backend image but runs ARQ command. Without this service, async jobs won't process.

### 3. Build and Deploy

```bash
# Build and start all services
docker-compose up -d --build

# Check service status
docker-compose ps

# View logs
docker-compose logs -f backend
docker-compose logs -f frontend
```

### 4. Verify Deployment

```bash
# Check health endpoint
curl http://YOUR_IP:8000/health

# Check API docs (disabled in production)
curl http://YOUR_IP:8000/docs
```

**TODO - GCP Deployment Steps** (handled by deployment team):
- [ ] Document GCP infrastructure provisioning with Terraform
- [ ] Document firewall rules and VPC configuration
- [ ] Document Memorystore Redis setup (if used instead of containerized Redis)
- [ ] Document backup and recovery procedures
- [ ] Document auto-scaling configuration

## Configuration

### Redis Configuration

**IMPORTANT**: Configure Redis persistence to prevent job loss on crashes.

Add Redis service to `docker-compose.yml`:
```yaml
redis:
  image: redis:7-alpine
  command: redis-server --appendonly yes --appendfsync everysec
  ports:
    - "6379:6379"
  volumes:
    - redis_data:/data
  restart: always
```

Add volume to volumes section:
```yaml
volumes:
  postgres_data:
  redis_data:
```

**GCP Memorystore Alternative**: For production, consider Google Cloud Memorystore for Redis with high availability and automatic backups. Update `REDIS_HOST` to Memorystore endpoint.

See [ASYNC_API.md](ASYNC_API.md) for detailed Redis persistence options.

### Database Persistence

PostgreSQL data must be backed up regularly since Docker containers are ephemeral:

```bash
# Create backup
docker-compose exec db pg_dump -U vibe8 vibe8_prod > backup-$(date +%Y%m%d).sql

# TODO: Document automated backup strategy
# - Backup schedule
# - Retention policy
# - S3 storage configuration
```

### Frontend Rebuild

Next.js bakes `NEXT_PUBLIC_*` environment variables at build time. After changing these, rebuild:

```bash
docker-compose stop frontend
docker-compose rm -f frontend
docker-compose build --no-cache frontend
docker-compose up -d frontend
```

## Monitoring and Troubleshooting

### View Logs

```bash
# Backend API logs
docker-compose logs -f backend

# ARQ worker logs (job processing)
docker-compose logs -f worker

# Frontend logs
docker-compose logs -f frontend

# Database logs
docker-compose logs -f db

# Redis logs
docker-compose logs -f redis
```

### Common Issues

**400/500 Errors**:
1. Check backend logs for errors
2. Verify Auth0 configuration
3. Verify DATABASE_URL format (`postgresql://` not `postgresql+psycopg2://`)
4. Check CORS settings

**Connection Refused**:
1. Verify firewall/security group rules allow ports 3000 and 8000
2. Check docker-compose port mappings
3. Ensure services are running: `docker-compose ps`

**CORS Errors**:
1. Update `CORS_ORIGINS` in docker-compose.yml
2. Restart backend: `docker-compose restart backend`
3. Check browser console for specific errors

**Job Queue Not Processing**:
1. Verify Redis is running: `docker-compose ps redis`
2. Verify ARQ worker is running: `docker-compose ps worker`
3. Check worker logs for errors: `docker-compose logs -f worker`
4. Verify Redis connection: `docker-compose exec worker env | grep REDIS`
5. Check queue depth: `docker-compose exec redis redis-cli LLEN arq:queue`

**TODO - GCP-Specific Monitoring** (handled by deployment team):
- [ ] Document Cloud Monitoring integration
- [ ] Document metrics collection
- [ ] Document Cloud Alerting setup
- [ ] Document Cloud Logging aggregation

## Security Recommendations

### Critical Security Items

1. **Change Default Credentials**:
   - Update `POSTGRES_PASSWORD` in docker-compose.yml
   - Use strong, randomly generated passwords

2. **Auth0 Configuration**:
   - Update allowed callback URLs: `https://YOUR_DOMAIN:3000/api/auth/callback`
   - Update allowed logout URLs: `https://YOUR_DOMAIN:3000`
   - Update allowed web origins: `https://YOUR_DOMAIN:3000`

3. **Environment Variables**:
   - Never commit secrets to git
   - Production uses **Google Secret Manager** (managed by deployment team via Terraform)
   - Rotate credentials regularly

4. **Network Security**:
   - Restrict database access (port 5432 should not be publicly accessible)
   - Use HTTPS in production (TODO: document SSL setup)
   - Configure security groups to allow only necessary traffic

5. **Redis Security**:
   - Use Redis password authentication
   - Enable encryption in transit (if using ElastiCache)
   - Restrict network access

**TODO - Security Hardening** (handled by deployment team):
- [ ] Document SSL/TLS certificate setup (Cloud Load Balancing + managed certificates)
- [ ] Document Cloud Armor configuration (WAF and DDoS protection)
- [ ] Document firewall rules
- [ ] Document IAM roles and service account permissions
- [ ] Document VPC Service Controls (if applicable)

## Related Documentation

- **[Architecture Guide](ARCHITECTURE.md)**: System design overview
- **[API Reference](API.md)**: Endpoint specifications
- **[Authentication Guide](AUTH.md)**: Auth0 configuration
- **[Database Schema](DATABASE.md)**: Schema and migrations
- **[Async API Guide](ASYNC_API.md)**: Job queue and Redis resilience

---

**Last Updated**: 2025-01-15
**Platform**: Google Cloud Platform (Docker Compose + Terraform deployment)
**Status**: Production - Documentation in progress
**Infrastructure**: Managed by deployment team
