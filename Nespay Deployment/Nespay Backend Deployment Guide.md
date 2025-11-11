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

All services run on Node.js 20+ with NestJS, connecting to PostgreSQL, Redis, and RabbitMQ.
## Prerequisites

**Required:**
- Docker 20.10+
- Docker Compose 2.0+
- PostgreSQL 17+ with `pg_trgm` extension
- At least 2GB free RAM

**Recommended:**
- Redis 6+ (standalone, cluster, or sentinel)
- RabbitMQ 3.11+ (for invoice callbacks)

**External Services:**
You'll need API credentials for any features you want to use:
- AWS Cognito (authentication)
- Rain Cards (card issuance)

## Environment Configuration

Create a `.env` file in the project root. Here are the critical variables:

### Core Configuration
```bash
NODE_ENV=production
API_PORT=8000
WEBHOOK_PORT=8001
JOB_PORT=8002
BASE_URL=https://api.yourdomain.com

PUBLIC_API_PREFIX=api/v1
INTERNAL_API_PREFIX=api/backoffice/v1
```

### Database
```bash
DATABASE_URL=postgresql://nespay:yourpassword@localhost:5432/nespay?schema=public
```

### Redis
```bash
# Core settings
REDIS_USER=default
REDIS_PASSWORD=password
REDIS_ENABLED=false

# Standalone mode
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=0

# Advanced settings (all optional with sensible defaults)
REDIS_MAX_RETRIES=3                  # Number of retry attempts before fallback
REDIS_RETRY_DELAY=100                # Base delay between retries (ms)
REDIS_MAX_RETRY_DELAY=2000           # Maximum retry delay (ms)
REDIS_READY_CHECK=true               # Enable ready check
REDIS_OFFLINE_QUEUE=false            # Queue commands when disconnected
REDIS_LAZY_CONNECT=false              # Connect on first command
```

For cluster mode:
```bash
REDIS_CLUSTER_ENABLED=false
REDIS_SENTINEL_ENABLED=true

# Cluster mode
REDIS_CLUSTER_NODES=localhost:7001,localhost:7002,localhost:7003
REDIS_CLUSTER_NAT_MAP=redis-node-1:7001=>localhost:7001,redis-node-2:7002=>localhost:7002,redis-node-3:7003=>localhost:7003

# Sentinel mode
REDIS_SENTINEL_NAME=mymaster
REDIS_SENTINEL_NODES=localhost:26379,localhost:26380,localhost:26381
REDIS_SENTINEL_NAT_MAP=redis-master:6379=>localhost:6379

```

### RabbitMQ (Required for Invoice Callbacks)
```bash
RABBITMQ_URL=amqp://nespay:yourpassword@localhost:5672
```

### Authentication

**AWS Cognito:**
```bash
COGNITO_USER_POOL_ID=us-east-1_...
COGNITO_USER_CLIENT_ID=...
COGNITO_ADMIN_POOL_ID=us-east-1_...
COGNITO_ADMIN_CLIENT_ID=...
```

**Privy (Wallet Management):**
```bash
PRIVY_APP_ID=...
PRIVY_APP_SECRET=...
```

### Security
```bash
# Generate with: openssl rand -hex 64
ENCRYPTION_MASTER_KEY=your-64-byte-hex-key

# Webhook validation
WEBHOOK_ENABLE_IP_VALIDATION=true
WEBHOOK_ENABLE_SIGNATURE_VALIDATION=true
```

### External Services (Optional)
```bash
# Rain Cards
RAIN_API_URL=https://api.raincards.xyz
RAIN_API_KEY=...

# Email
EMAIL_SUPPORT_EMAIL=support@yourdomain.com
```

### Monitoring (Optional)

`OTLP_LOGS_ENDPOINT` value is where `Grafana Alloy` is located.

```bash
## OpenTelemetry 
OTLP_LOGS_ENDPOINT=http://localhost:4318/v1/logs
SERVICE_NAME=nespay-backend
ENVIRONMENT=production

# Loki
LOKI_URL=http://localhost:3100
```

## Database Setup

### 1. Install PostgreSQL Extension

Connect to your database and run:
```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

This extension is required for full-text search functionality.

### 2. Create Database
```bash
createdb nespay
```

Or via SQL:
```sql
CREATE DATABASE nespay OWNER nespay;
```

## Build Instructions

### Method 1: Docker Compose (Recommended)

The `docker-compose.yml` includes backend, webhook, and database services.

```bash
# Build all services
docker-compose build

# Start services
docker-compose up -d

# View logs
docker-compose logs -f
```

### Method 2: Manual Docker Build

```bash
# Build main application image
docker build -t nespay-backend:latest .
```

### Method 3: Local Build (No Docker)

Requires pnpm 9.15.2+:

```bash
# Enable pnpm
corepack enable pnpm

# Install dependencies
pnpm install

# Generate Prisma client
pnpm prisma:generate

# Build TypeScript
pnpm build
```

## Running Database Migrations

**Before first run**, apply database migrations:

### Run the migration
```bash
npx prisma migrate deploy
```

### Seed Initial Data
```bash
# Seeds blockchain networks, currencies, and RPC endpoints
pnpm prisma:seed:prod
```

## Running Services

### Using Docker Compose
```bash
# Start all services
docker-compose up -d

# Check status
docker-compose ps

# Stop services
docker-compose down
```

Services will be available at:
- Main API: http://localhost:8000
- Webhook: http://localhost:8001

### Using Deployment Scripts

**Development:**
```bash
chmod +x deploy-dev.sh
./deploy-dev.sh
```
Starts containers named `nespay-backend-dev` and `nespay-webhook-dev`.

**Staging:**
```bash
chmod +x deploy-staging.sh
./deploy-staging.sh
```
Uses ports 8200 (backend) and 8201 (webhook).

### Manual Docker Run
```bash
# Main API
docker run -d \
  --name nespay-backend \
  -p 8000:8000 \
  --env-file .env \
  nespay-backend:latest \
  node dist/src/main.js

# Webhook Service
docker run -d \
  --name nespay-webhook \
  -p 8001:8001 \
  --env-file .env \
  nespay-backend:latest \
  node dist/src/webhook-main.js

# Job Service
docker run -d \
  --name nespay-jobs \
  -p 8002:8002 \
  --env-file .env \
  nespay-backend:latest \
  node dist/src/job-main.js
```

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

### Check Logs
```bash
# Docker Compose
docker-compose logs -f backend

# Individual container
docker logs -f nespay-backend
```

### Verify Database Connection
```bash
# Check Prisma can connect
docker exec -it nespay-backend pnpm prisma db pull
```

## Port Reference

| Service     | Port  | Purpose                     |
| ----------- | ----- | --------------------------- |
| Main API    | 8000  | Public + Backoffice APIs    |
| Webhook     | 8001  | External provider callbacks |
| Jobs        | 8002  | Background processing       |
| PostgreSQL  | 5432  | Database                    |
| Redis       | 6379  | Cache                       |
| RabbitMQ    | 5672  | Message queue               |
| RabbitMQ UI | 15672 | Management interface        |

## Troubleshooting

### Port Already in Use
```bash
# Find process using port 8000
lsof -i :8000

# Kill it
kill -9 <PID>
```

### Database Connection Failed
Check your `DATABASE_URL` format:
```bash
postgresql://username:password@host:port/database?schema=public
```

Verify PostgreSQL is running:
```bash
docker ps | grep postgres
```

### Prisma Client Not Found
Regenerate the Prisma client:
```bash
pnpm prisma:generate
pnpm build
```

### Container Crashes on Startup
Check environment variables are set:
```bash
docker exec nespay-backend env | grep DATABASE_URL
```

View detailed logs:
```bash
docker logs nespay-backend --tail 100
```

### Migration Failures
Reset and reapply migrations:
```bash
# WARNING: Deletes all data
pnpm prisma:migrate:reset
pnpm prisma:migrate:deploy
pnpm prisma:db:seed
```

### Build Failures
Clear Docker cache and rebuild:
```bash
docker-compose down
docker system prune -a
docker-compose build --no-cache
docker-compose up -d
```

## Security Considerations

**Production Checklist:**
- [ ] Change all default passwords
- [ ] Use secrets management (not .env in image)
- [ ] Enable webhook IP validation
- [ ] Use HTTPS for all external endpoints
- [ ] Rotate `ENCRYPTION_MASTER_KEY` regularly
- [ ] Enable Redis authentication
- [ ] Restrict database access by IP
- [ ] Set up proper firewall rules
- [ ] Enable audit logging
- [ ] Configure rate limiting

**Environment Variables in Docker:**
The current Dockerfile copies `.env` into the image. For production, use:
- Docker secrets
- Kubernetes secrets
- AWS Parameter Store
- HashiCorp Vault

**Webhook Security:**
Configure allowed IPs for each provider:
```bash
WEBHOOK_RAIN_ALLOWED_IPS=34.67.3.60,34.66.230.233
WEBHOOK_INTRAJASA_TOKEN=your-shared-secret
```

## Next Steps

1. Configure external service credentials
2. Set up monitoring and alerting
3. Configure backup strategy for PostgreSQL
4. Set up reverse proxy (nginx/Traefik)
5. Configure SSL certificates
6. Set up log aggregation
7. Configure autoscaling (if needed)

For development commands and detailed configuration, see the project README.
