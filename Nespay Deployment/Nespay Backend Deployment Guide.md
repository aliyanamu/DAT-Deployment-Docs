---
tags:
  - nespay
  - webtiga
  - deployment
  - system-administration
---

# Nespay Backend Deployment Guide

## Overview

Nespay Backend is a cryptocurrency payment platform running as three microservices from a single codebase:

- **Main API** (port 8000): Public and backoffice endpoints
- **Webhook Service** (port 8001): External payment provider callbacks
- **Job Service** (port 8002): Background processing and scheduled tasks

All services run on Node.js 20+ with NestJS, connecting to PostgreSQL, and Redis.

## Prerequisites

### Infrastructure Requirements

For server infrastructure setup (PostgreSQL, Redis, Grafana stack), refer to `server-setup.md`.

### Required Software
- Docker 20.10+
- Docker Compose 2.0+
- At least 2GB free RAM

### Required External Services
You'll need API credentials for:
- **AWS Cognito** - User and admin authentication
- **Privy.io** - Wallet management and MPC services
- **Rain Cards** - Card issuance (required for card features)
- **Grafana Loki** - Application log storage (required)

## Environment Configuration

A `.env.example` file is provided as a template. Copy it to create your environment file:

```bash
cp .env.example .env
```

Then edit `.env` with your actual configuration values. Key variables to configure:

```bash
# ============================================
# CORE CONFIGURATION
# ============================================
NODE_ENV=production
API_PORT=8000
WEBHOOK_PORT=8001
JOB_PORT=8002
BASE_URL=https://api.yourdomain.com
PUBLIC_API_PREFIX=api/v1
INTERNAL_API_PREFIX=api/backoffice/v1

# ============================================
# DATABASE
# ============================================
DATABASE_URL=postgresql://nespay_user:yourpassword@localhost:5432/nespay?schema=public

# Database monitoring settings
DB_MONITORING_ENABLED=true
DB_MONITORING_LEVEL=ALL
DB_SLOW_QUERY_THRESHOLD=100

# ============================================
# REDIS - Optional performance cache
# Application runs in degraded mode without Redis
# ============================================
REDIS_ENABLED=false
REDIS_USER=default
REDIS_PASSWORD=yourpassword

# Standalone mode
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=0

# Advanced settings (optional, sensible defaults provided)
REDIS_MAX_RETRIES=3
REDIS_RETRY_DELAY=100
REDIS_MAX_RETRY_DELAY=2000
REDIS_READY_CHECK=true
REDIS_OFFLINE_QUEUE=false
REDIS_LAZY_CONNECT=false

# Cluster mode
REDIS_CLUSTER_ENABLED=false
REDIS_CLUSTER_NODES=localhost:7001,localhost:7002,localhost:7003
REDIS_CLUSTER_NAT_MAP=redis-node-1:7001=>localhost:7001,redis-node-2:7002=>localhost:7002,redis-node-3:7003=>localhost:7003

# Sentinel mode
REDIS_SENTINEL_ENABLED=false
REDIS_SENTINEL_NAME=mymaster
REDIS_SENTINEL_NODES=localhost:26379,localhost:26380,localhost:26381
REDIS_SENTINEL_NAT_MAP=redis-master:6379=>localhost:6379

# ============================================
# AWS COGNITO - Authentication (Required)
# ============================================
COGNITO_USER_POOL_ID=us-east-1_XXXXXXXXX
COGNITO_USER_CLIENT_ID=xxxxxxxxxxxxxxxxxxxx
COGNITO_ADMIN_POOL_ID=us-east-1_XXXXXXXXX
COGNITO_ADMIN_CLIENT_ID=xxxxxxxxxxxxxxxxxxxx

# ============================================
# PRIVY - Wallet Management (Required)
# ============================================
PRIVY_APP_ID=your-app-id
PRIVY_APP_SECRET=your-app-secret
PRIVY_API=https://auth.privy.io

# ============================================
# RAIN CARDS - Card Issuance (Required for card features)
# ============================================
RAIN_API_URL=https://api.raincards.xyz
RAIN_API_KEY=your-rain-api-key
CARD_ADMIN_EVM_WALLET_ADDRESS=0x...

# ============================================
# SECURITY
# ============================================
# Generate with: openssl rand -hex 64
ENCRYPTION_MASTER_KEY=your-64-byte-hex-key

# Webhook IP validation
WEBHOOK_ENABLE_IP_VALIDATION=true
WEBHOOK_ENABLE_SIGNATURE_VALIDATION=true
WEBHOOK_ALLOWED_IPS=
WEBHOOK_RAIN_ALLOWED_IPS=34.67.3.60,34.66.230.233

# ============================================
# OBSERVABILITY - Logs and Monitoring (Required)
# OTLP_LOGS_ENDPOINT points to Grafana Alloy instance
# ============================================
OTLP_LOGS_ENDPOINT=http://localhost:4318/v1/logs
SERVICE_NAME=nespay-backend
SERVICE_VERSION=1.0.0
ENVIRONMENT=production
CLUSTER_NAME=production

# Loki for log storage (required)
LOKI_URL=http://localhost:3100

# ============================================
# EMAIL - Transactional emails
# ============================================
EMAIL_APP_NAME=Nespay
EMAIL_SUPPORT_EMAIL=support@yourdomain.com

# AWS SES configuration
AWS_REGION=us-east-1
AWS_SES_FROM_EMAIL=noreply@yourdomain.com

# ============================================
# BACKGROUND JOBS
# ============================================
CONSOLIDATION_JOB_ENABLED=false
CONSOLIDATION_JOB_SCHEDULE="0 */6 * * *"
CONSOLIDATION_JOB_MAX_CONCURRENT=3
CONSOLIDATION_JOB_TIMEOUT_MS=300000

# ============================================
# OPTIONAL FEATURES
# ============================================
SWAGGER_ENABLE=1
ENABLE_CORS_URLS=
HEALTH_TOKEN=
APP_SCHEME=nespay
```

## Build and Deploy

### Prerequisites

Ensure infrastructure services are running (see `server-setup.md`):
- PostgreSQL 17 with `nespay` database and `pg_trgm` extension
- Redis (optional but recommended for performance)

### Using Docker Compose

The `docker-compose.yml` deploys only the application services (backend, webhook, jobs). Infrastructure dependencies (PostgreSQL, Redis) must be installed separately on the host system.

```bash
# Build all services
docker-compose build

# Start services in background
docker-compose up -d

# View logs
docker-compose logs -f

# Check service status
docker-compose ps

# Stop services
docker-compose down
```

Services will be available at:
- Main API: http://localhost:8000
- Webhook: http://localhost:8001
- Job Service: http://localhost:8002

**Note**: The `.env` file must contain valid connection strings for PostgreSQL, and Redis running on the host or remote servers.

## Database Migration

**Before first run**, apply database migrations and seed initial data:

```bash
# Run migrations
docker-compose exec backend npx prisma migrate deploy

# Seed blockchain networks, currencies, and RPC endpoints
docker-compose exec backend pnpm prisma:seed:prod
```

**Note**: Migrations must be run after the backend service is built and running.

## Service Verification

### Health Checks

```bash
# Main API
curl http://localhost:8000/api/v1/health

# Webhook Service
curl http://localhost:8001/api/v1/health

# Job Service
curl http://localhost:8002/api/v1/health
```

Expected response:
```json
{
  "status": "ok",
  "info": {
    "database": { "status": "up" },
    "redis": { "status": "up" }
  }
}
```

### View Logs

```bash
# Docker Compose - all services
docker-compose logs -f

# Specific service
docker-compose logs -f backend
docker-compose logs -f webhook
docker-compose logs -f jobs

# Individual container
docker logs -f nespay-backend
```

### Verify Database Connection

```bash
docker-compose exec backend pnpm prisma db pull
```

## Port Reference

| Service     | Port  | Purpose                     |
| ----------- | ----- | --------------------------- |
| Main API    | 8000  | Public + Backoffice APIs    |
| Webhook     | 8001  | External provider callbacks |
| Jobs        | 8002  | Background processing       |
| PostgreSQL  | 5432  | Database                    |
| Redis       | 6379  | Cache                       |
| Loki        | 3100  | Log aggregation             |
| Alloy       | 4318  | OTLP receiver               |

## Troubleshooting

### Port Already in Use

```bash
# Find process using port 8000
lsof -i :8000

# Kill process
kill -9 <PID>
```

### Database Connection Failed

Verify `DATABASE_URL` format:
```bash
postgresql://username:password@host:port/database?schema=public
```

Check PostgreSQL is running:
```bash
docker ps | grep postgres
# OR if using system PostgreSQL
sudo systemctl status postgresql-17
```

### Prisma Client Not Found

Regenerate Prisma client:
```bash
docker-compose exec backend pnpm prisma:generate
docker-compose restart backend
```

### Redis Connection Issues

The application runs without Redis but with degraded performance. Verify Redis status:

```bash
# Check Redis is running
docker ps | grep redis
# OR
sudo systemctl status redis

# Test connection
redis-cli ping  # Should return PONG
```


### Container Crashes on Startup

Check environment variables:
```bash
docker-compose exec backend env | grep DATABASE_URL
```

View detailed logs:
```bash
docker logs nespay-backend --tail 100
```

### Migration Failures

Reset and reapply migrations (WARNING: deletes all data):
```bash
docker-compose exec backend pnpm prisma:migrate:reset
docker-compose exec backend npx prisma migrate deploy
docker-compose exec backend pnpm prisma:seed:prod
```

### Build Failures

Clear Docker cache and rebuild:
```bash
docker-compose down
docker system prune -a
docker-compose build --no-cache
docker-compose up -d
```

### Logs Not Appearing in Loki

Verify Grafana Alloy is running and accessible:
```bash
curl http://localhost:4318/v1/logs
```

Check `OTLP_LOGS_ENDPOINT` in `.env` points to Alloy instance.

## Security Considerations

### Environment Variables in Docker

The current Dockerfile copies `.env` into the image. For production:
- Use Docker secrets
- Use AWS Secrets Manager
- Use Kubernetes secrets
- Use environment variables injected at runtime

### Webhook Security

Configure allowed IPs for each provider in `.env`:
```bash
WEBHOOK_RAIN_ALLOWED_IPS=34.67.3.60,34.66.230.233
WEBHOOK_INTRAJASA_TOKEN=your-shared-secret
```

Enable signature validation:
```bash
WEBHOOK_ENABLE_IP_VALIDATION=true
WEBHOOK_ENABLE_SIGNATURE_VALIDATION=true
```

## API Documentation

When `SWAGGER_ENABLE=1`, Swagger UI is available at:
- http://localhost:8000/api (Main API)
- http://localhost:8001/api (Webhook service)
