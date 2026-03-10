# Phase 7 V1 Deployment Guide

**Version**: 1.0  
**Date**: 2026-03-09  
**Scope**: Portfolio Service, Analytics Service, Optimization Service

---

## Table of Contents

1. [Overview](#overview)
2. [Service Deployment Units](#service-deployment-units)
3. [Environment Variables](#environment-variables)
4. [Health Checks](#health-checks)
5. [Scaling Considerations](#scaling-considerations)
6. [Docker Deployment](#docker-deployment)
7. [Kubernetes Deployment (Staging/Prod)](#kubernetes-deployment-stagingprod)
8. [Monitoring & Logging](#monitoring--logging)
9. [Rolling Updates & rollbacks](#rolling-updates--rollbacks)
10. [Service Dependencies](#service-dependencies)

---

## Overview

Phase 7 V1 introduces **three new backend services** that run as independent processes/containers behind the API Gateway. All services are stateless and share common infrastructure (PostgreSQL, Redis, RabbitMQ/Redis Queue).

### Architecture

```
[Client] → [API Gateway] → [Portfolio Service:8001]
                                [Analytics Service:8002]
                                [Optimization Service:8003]
                                   ↓
                        [Shared Infrastructure]
                     - PostgreSQL (DB)
                     - Redis (Cache + Queue)
                     - Event Log
```

### Port Assignments

| Service | Port | Health Check Path |
|---------|------|-------------------|
| Portfolio Service | `8001` | `GET /health` |
| Analytics Service | `8002` | `GET /health` |
| Optimization Service | `8003` | `GET /health` |

---

## Service Deployment Units

Each service is deployed as a separate container/pod with its own:

- **Codebase**: Python module (`services/portfolio/`, `services/analytics/`, `services/optimization/`)
- **Docker image**: `tradermate/portfolio:v1.0`, etc.
- **K8s Deployment**: Individual deployments with replica count configuration
- **Service Discovery**: Registered in K8s service mesh or K8s Service object

### Service Configuration

All services read configuration from:
1. **Environment variables** (primary)
2. `.env` file (development only, git-ignored)
3. ConfigMap/Secret (K8s production)

---

## Environment Variables

### Common Variables (All Services)

| Variable | Description | Required | Default |
|----------|-------------|----------|---------|
| `DATABASE_URL` | PostgreSQL connection string | ✅ | - |
| `REDIS_URL` | Redis connection string | ✅ | - |
| `JWT_SECRET` | JWT signing secret (matches API Gateway) | ✅ | - |
| `LOG_LEVEL` | Logging level (DEBUG/INFO/WARNING/ERROR) | ❌ | `INFO` |
| `ENV` | Environment name (dev/staging/prod) | ❌ | `dev` |
| `SERVICE_NAME` | Service identifier (auto-set) | ✅ | - |
| `SERVICE_PORT` | HTTP port to bind | ✅ | `8000` (overridden per service) |

### Service-Specific Variables

#### Portfolio Service

| Variable | Description | Required | Default |
|----------|-------------|----------|---------|
| `CACHE_TTL_POSITIONS` | Position cache TTL in seconds | ❌ | `300` |
| `PRICE_FEED_URL` | External price API endpoint (optional) | ❌ | - |
| `PRICE_FEED_KEY` | API key for price feed | ❌ (if using internal feed) | - |

#### Analytics Service

| Variable | Description | Required | Default |
|----------|-------------|----------|---------|
| `CACHE_TTL_DASHBOARD` | Dashboard cache TTL in seconds | ❌ | `300` |
| `RISK_FREE_RATE` | Risk-free rate for Sharpe calculation | ❌ | `0.02` (2%) |
| `ROLLING_WINDOW_DAYS` | Performance calculation window (days) | ❌ | `252` |

#### Optimization Service

| Variable | Description | Required | Default |
|----------|-------------|----------|---------|
| `QUEUE_URL` | Queue connection (Redis/RabbitMQ) | ✅ | `redis://localhost:6379/0` |
| `MAX_JOBS_PER_USER` | Max concurrent jobs per user | ❌ | `3` |
| `OPTIMIZATION_TIMEOUT` | Job timeout in seconds | ❌ | `3600` |
| `WORKER_CONCURRENCY` | Number of worker threads/processes | ❌ | `2` |
| `RESULT_STORAGE_PATH` | Optional S3/local path for large results | ❌ | - |

### Example `.env` (Development)

```bash
# Common
DATABASE_URL=mysql+pymysql://trader:password@10.0.0.73:3306/tradermate
REDIS_URL=redis://localhost:6379/0
JWT_SECRET=super-secret-jwt-key-change-in-prod
LOG_LEVEL=DEBUG
ENV=dev

# Portfolio Service (port 8001)
SERVICE_NAME=portfolio
SERVICE_PORT=8001
CACHE_TTL_POSITIONS=300

# Analytics Service (port 8002)
SERVICE_NAME=analytics
SERVICE_PORT=8002
CACHE_TTL_DASHBOARD=300
RISK_FREE_RATE=0.02

# Optimization Service (port 8003)
SERVICE_NAME=optimization
SERVICE_PORT=8003
QUEUE_URL=redis://localhost:6379/0/trader-queue
MAX_JOBS_PER_USER=3
OPTIMIZATION_TIMEOUT=3600
WORKER_CONCURRENCY=2
```

---

## Health Checks

All services expose `GET /health` endpoint that returns JSON:

**Success Response (200 OK)**:
```json
{
  "status": "healthy",
  "service": "portfolio",
  "timestamp": "2026-03-09T04:30:00Z",
  "version": "1.0.0",
  "uptime_seconds": 3600,
  "checks": {
    "database": {"status": "healthy", "latency_ms": 5},
    "redis": {"status": "healthy", "latency_ms": 2},
    "queue": {"status": "healthy", "latency_ms": 3}
  }
}
```

**Failure Response (503 Service Unavailable)**:
```json
{
  "status": "unhealthy",
  "service": "portfolio",
  "timestamp": "2026-03-09T04:30:00Z",
  "checks": {
    "database": {"status": "unhealthy", "error": "Connection refused"}
  }
}
```

### Health Check Logic

- **Database**: Execute `SELECT 1` with 2s timeout
- **Redis**: `PING` command with 2s timeout
- **Queue**: Connection test (Redis/RabbitMQ) if applicable
- If any critical dependency fails, return `503`

### K8s Liveness/Readiness Probes

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: service-port
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health
    port: service-port
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 2
```

---

## Scaling Considerations

### Horizontal Scaling

All services are **stateless** and can be horizontally scaled:

- Portfolio Service: Scale out behind load balancer (multiple replicas)
- Analytics Service: Cache-aware; ensure Redis cache shared across replicas
- Optimization Service: Queue-based workers are naturally scalable; add more worker pods for higher throughput

### Replica Count Recommendations

| Environment | Portfolio | Analytics | Optimization (web) | Optimization (worker) |
|-------------|-----------|-----------|-------------------|---------------------|
| Dev | 1 | 1 | 1 | 1 |
| Staging | 2 | 2 | 2 | 2 |
| Production (small) | 3 | 3 | 3 | 3 |
| Production (large) | 5+ | 5+ | 5+ | 5+ |

---

## Docker Deployment

### Dockerfile Template (common for all services)

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy service code
COPY services/${SERVICE_NAME}/ ./services/${SERVICE_NAME}/
COPY shared_libs/ ./shared_libs/
COPY main.py .

# Create non-root user
RUN useradd --uid 1001 --gid 0 appuser && chown -R appuser:0 /app
USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health').read()"

CMD ["python", "main.py"]
```

### Build Commands

```bash
# Build portfolio service
docker build \
  --build-arg SERVICE_NAME=portfolio \
  -t tradermate/portfolio:v1.0 \
  -f Dockerfile .

# Build analytics
docker build \
  --build-arg SERVICE_NAME=analytics \
  -t tradermate/analytics:v1.0 \
  -f Dockerfile .

# Build optimization
docker build \
  --build-arg SERVICE_NAME=optimization \
  -t tradermate/optimization:v1.0 \
  -f Dockerfile .
```

### Docker Compose (Dev)

```yaml
version: '3.8'

services:
  api-gateway:
    image: tradermate/api-gateway:latest
    ports:
      - "8000:8000"
    depends_on:
      - portfolio
      - analytics
      - optimization
      - redis
      - db

  portfolio:
    image: tradermate/portfolio:v1.0
    environment:
      - DATABASE_URL=mysql+pymysql://trader:password@db:3306/tradermate
      - REDIS_URL=redis://redis:6379/0
      - JWT_SECRET=dev-secret-key
      - SERVICE_NAME=portfolio
      - SERVICE_PORT=8001
      - CACHE_TTL_POSITIONS=300
    ports:
      - "8001:8001"
    depends_on:
      - db
      - redis

  analytics:
    image: tradermate/analytics:v1.0
    environment:
      - DATABASE_URL=mysql+pymysql://trader:password@db:3306/tradermate
      - REDIS_URL=redis://redis:6379/0
      - JWT_SECRET=dev-secret-key
      - SERVICE_NAME=analytics
      - SERVICE_PORT=8002
      - CACHE_TTL_DASHBOARD=300
    ports:
      - "8002:8002"
    depends_on:
      - db
      - redis

  optimization:
    image: tradermate/optimization:v1.0
    environment:
      - DATABASE_URL=mysql+pymysql://trader:password@db:3306/tradermate
      - REDIS_URL=redis://redis:6379/0/trader-queue
      - JWT_SECRET=dev-secret-key
      - SERVICE_NAME=optimization
      - SERVICE_PORT=8003
      - QUEUE_URL=redis://redis:6379/0/trader-queue
      - MAX_JOBS_PER_USER=3
      - WORKER_CONCURRENCY=2
    ports:
      - "8003:8003"
    depends_on:
      - db
      - redis
    # Worker process runs in same container (simplified for dev)
    command: python -m services.optimization.worker &

  db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=rootpassword
      - MYSQL_DATABASE=tradermate
      - MYSQL_USER=trader
      - MYSQL_PASSWORD=password
    volumes:
      - mysql_data:/var/lib/mysql
      - ./init-scripts:/docker-entrypoint-initdb.d
    ports:
      - "3306:3306"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  mysql_data:
```

---

## Kubernetes Deployment (Staging/Prod)

### Deployment Manifests (YAML snippets)

#### Common Labels/Annotations

```yaml
labels:
  app: tradermate
  component: backend
  service: portfolio  # or analytics/optimization
  environment: staging  # or prod
```

#### Portfolio Service Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portfolio-service
  namespace: tradermate
spec:
  replicas: 3
  selector:
    matchLabels:
      service: portfolio
  template:
    metadata:
      labels:
        service: portfolio
    spec:
      containers:
        - name: portfolio
          image: tradermate/portfolio:v1.0
          ports:
            - containerPort: 8001
          envFrom:
            - configMapRef:
                name: common-config
            - secretRef:
                name: secrets
          env:
            - name: SERVICE_NAME
              value: "portfolio"
            - name: SERVICE_PORT
              value: "8001"
            - name: CACHE_TTL_POSITIONS
              valueFrom:
                configMapKeyRef:
                  name: service-config
                  key: CACHE_TTL_POSITIONS
          resources:
            requests:
              memory: "256Mi"
              cpu: "200m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 8001
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /health
              port: 8001
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: portfolio-service
  namespace: tradermate
spec:
  selector:
    service: portfolio
  ports:
    - port: 8001
      targetPort: 8001
      protocol: TCP
  type: ClusterIP
```

#### Analytics Service Deployment (similar)

Change `service: analytics`, `name: analytics-service`, `containerPort: 8002`, env `SERVICE_NAME=analytics`, `SERVICE_PORT=8002`.

#### Optimization Service Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: optimization-service
  namespace: tradermate
spec:
  replicas: 3
  selector:
    matchLabels:
      service: optimization
  template:
    metadata:
      labels:
        service: optimization
    spec:
      containers:
        - name: optimization
          image: tradermate/optimization:v1.0
          ports:
            - containerPort: 8003
          envFrom:
            - configMapRef:
                name: common-config
            - secretRef:
                name: secrets
          env:
            - name: SERVICE_NAME
              value: "optimization"
            - name: SERVICE_PORT
              value: "8003"
            - name: QUEUE_URL
              value: "redis://redis-master:6379/0/trader-queue"
            - name: MAX_JOBS_PER_USER
              value: "3"
            - name: WORKER_CONCURRENCY
              value: "2"
          command: ["python", "-m", "services.optimization"]
          # Worker runs as separate deployment (see below)
          resources:
            requests:
              memory: "512Mi"
              cpu: "400m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          livenessProbe:
            httpGet:
              path: /health
              port: 8003
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /health
              port: 8003
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: optimization-service
  namespace: tradermate
spec:
  selector:
    service: optimization
  ports:
    - port: 8003
      targetPort: 8003
      protocol: TCP
  type: ClusterIP
```

#### Optimization Worker Deployment (separate from web service)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: optimization-worker
  namespace: tradermate
spec:
  replicas: 5
  selector:
    matchLabels:
      app: optimization-worker
  template:
    metadata:
      labels:
        app: optimization-worker
    spec:
      containers:
        - name: worker
          image: tradermate/optimization:v1.0
          command: ["python", "-m", "services.optimization.worker"]
          envFrom:
            - configMapRef:
                name: common-config
            - secretRef:
                name: secrets
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DATABASE_URL
            - name: QUEUE_URL
              value: "redis://redis-master:6379/0/trader-queue"
          resources:
            requests:
              memory: "1Gi"
              cpu: "1000m"
            limits:
              memory: "2Gi"
              cpu: "2000m"
```

---

## Monitoring & Logging

### Structured Logging

All services log in JSON format (compatible with Loki/ELK):

```json
{
  "timestamp": "2026-03-09T04:30:00.123Z",
  "level": "INFO",
  "service": "portfolio",
  "trace_id": "req_abc123",
  "user_id": 42,
  "endpoint": "GET /api/portfolio/positions",
  "latency_ms": 15,
  "status_code": 200,
  "message": "Successfully retrieved positions"
}
```

### Metrics to Export (Prometheus)

- `http_requests_total` (labels: service, method, path, status)
- `http_request_duration_seconds` (histogram)
- `database_query_duration_seconds` (histogram)
- `cache_hits_total` / `cache_misses_total`
- `optimization_jobs_total` (by status)
- `optimization_job_duration_seconds`
- `active_connections` (DB, Redis)

### Alerting Thresholds (SLOs per NFR)

| Metric | Target | Alert if... |
|--------|--------|-------------|
| API success rate | > 99.9% | < 99% for 5m |
| API latency p95 | < 200ms (read) / < 500ms (write) | p95 > 2x target for 10m |
| DB connection pool | < 80% utilized | > 80% for 5m |
| Redis cache hit rate | > 80% for portfolio | < 60% for 15m |
| Optimization job queue | < 100 jobs pending | > 1000 jobs pending |

---

## Rolling Updates & Rollbacks

### Deployment Strategy

Use **RollingUpdate** with:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 25%
```

### Pre-stop Hook

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 30"]
```

Gives in-flight requests 30s to complete before pod termination.

### Rollback Command

```bash
kubectl rollout undo deployment/portfolio-service -n tradermate
kubectl rollout undo deployment/analytics-service -n tradermate
kubectl rollout undo deployment/optimization-service -n tradermate
```

---

## Service Dependencies

### Startup Order (Dev only)

In production, K8s handles dependency via readiness probes. In Docker Compose, services declare `depends_on`.

| Service | Depends On |
|---------|------------|
| Portfolio | PostgreSQL, Redis |
| Analytics | PostgreSQL, Redis |
| Optimization | PostgreSQL, Redis, Task Queue |

### Retry Logic

Services must implement exponential backoff for:
- Database connection (5 retries, 1s→10s delay)
- Redis connection (5 retries, 1s→10s delay)
- Queue connection (3 retries, 2s→30s delay)

Fail fast if dependencies unavailable after 30s total wait.

---

## Testing Health

Once deployed, verify:

```bash
# Test health endpoints
curl http://portfolio:8001/health
curl http://analytics:8002/health
curl http://optimization:8003/health

# Test API through gateway
curl -H "Authorization: Bearer <token>" \
  http://gateway:8000/api/portfolio/positions

curl -H "Authorization: Bearer <token>" \
  http://gateway:8000/api/analytics/dashboard

curl -H "Authorization: Bearer <token>" \
  -X POST http://gateway:8000/api/optimization \
  -H "Content-Type: application/json" \
  -d '{"strategy_class":"TripleMAStrategy","symbol":"000001.SZ","start_date":"2023-01-01","end_date":"2023-12-31","initial_capital":100000,"param_ranges":{"fast_window":{"min":5,"max":20,"step":1}}}'
```

---

## End of Deployment V1

**Next**: Final integration checklist before handing off to Coder for implementation.
