---
name: devops-patterns
description: DevOps patterns for CI/CD pipelines (GitHub Actions, GitLab CI), Docker containerization, monitoring, observability, infrastructure-as-code, and production deployment strategies.
---

# DevOps & Infrastructure Patterns

Patterns for building reliable CI/CD pipelines, container deployments, and observable production systems.

## When to Activate

- Setting up CI/CD pipelines (GitHub Actions, GitLab CI, Jenkins)
- Writing Dockerfiles and docker-compose configurations
- Configuring monitoring, logging, and alerting
- Designing deployment strategies (blue-green, canary, rolling)
- Managing infrastructure as code (Terraform, Pulumi)
- Setting up secret management and environment configuration

## Core Principles

### 1. Everything as Code

Infrastructure, pipelines, monitoring — all versioned in git.

### 2. Fail Fast, Recover Fast

Detect issues in CI before production. Automate rollback when production fails.

### 3. Shift Left Security

Security scanning in CI, not after deployment.

## CI/CD Pipeline Patterns

### GitHub Actions Standard Pipeline

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  PYTHON_VERSION: "3.11"
  REGISTRY: ghcr.io

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: pip
      - run: pip install ruff mypy
      - run: ruff check .
      - run: mypy --strict src/

  test:
    runs-on: ubuntu-latest
    needs: lint
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: pip
      - run: pip install -r requirements.txt
      - run: pytest --cov=app --cov-report=xml --cov-fail-under=80
      - uses: codecov/codecov-action@v4

  security:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - run: pip install bandit safety
      - run: bandit -r src/ -f json -o bandit-report.json || true
      - run: safety check --full-report

  build:
    runs-on: ubuntu-latest
    needs: [test, security]
    if: github.ref == 'refs/heads/main'
    permissions:
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ env.REGISTRY }}/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Pipeline Caching Strategy

```yaml
# Python dependencies
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: pip-${{ hashFiles('requirements*.txt') }}
    restore-keys: pip-

# Node.js dependencies
- uses: actions/cache@v4
  with:
    path: node_modules
    key: node-${{ hashFiles('package-lock.json') }}

# Docker layer cache
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

## Docker Patterns

### Production Dockerfile (Python)

```dockerfile
# ---- Builder stage ----
FROM python:3.11-slim AS builder
WORKDIR /app

# Install dependencies first (cache layer)
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ---- Runtime stage ----
FROM python:3.11-slim
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    libgl1-mesa-glx libglib2.0-0 \
    && rm -rf /var/lib/apt/lists/*

# Copy installed Python packages
COPY --from=builder /install /usr/local

# Copy application code
COPY . .

# Non-root user
RUN useradd --create-home --shell /bin/bash appuser
USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Docker Compose for Development

```yaml
version: "3.9"

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: builder  # Use builder stage for dev
    volumes:
      - .:/app
      - /app/__pycache__  # Exclude bytecode
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://dev:dev@db:5432/app
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    command: uvicorn main:app --host 0.0.0.0 --reload

  db:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

## Monitoring & Observability

### Health Check Endpoint

```python
from fastapi import FastAPI
from datetime import datetime

@app.get("/health")
async def health_check():
    checks = {
        "database": await check_database(),
        "redis": await check_redis(),
        "model_loaded": check_model(),
    }
    status = "healthy" if all(checks.values()) else "unhealthy"
    return {
        "status": status,
        "checks": checks,
        "timestamp": datetime.utcnow().isoformat(),
        "version": APP_VERSION,
    }
```

### Structured Logging

```python
import logging
import json

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno,
        }
        if hasattr(record, "request_id"):
            log_data["request_id"] = record.request_id
        if record.exc_info:
            log_data["exception"] = self.formatException(record.exc_info)
        return json.dumps(log_data)

# Usage
handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logger = logging.getLogger("app")
logger.addHandler(handler)
```

### Prometheus Metrics

```python
from prometheus_client import Counter, Histogram, generate_latest

REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status"],
)
REQUEST_LATENCY = Histogram(
    "http_request_duration_seconds",
    "HTTP request latency",
    ["method", "endpoint"],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0],
)
DETECTION_COUNT = Counter(
    "plate_detections_total",
    "Total license plate detections",
    ["region", "confidence_bucket"],
)

@app.get("/metrics")
async def metrics():
    return Response(generate_latest(), media_type="text/plain")
```

## Deployment Strategies

```
| Strategy | Downtime | Risk | Rollback Speed |
|----------|----------|------|----------------|
| Rolling | None | Medium | Slow |
| Blue-Green | None | Low | Instant |
| Canary | None | Lowest | Fast |
| Recreate | Yes | High | Slow |
```

### Blue-Green with Health Check

```bash
# Deploy new version (green)
docker compose -f docker-compose.green.yml up -d

# Health check new version
for i in {1..10}; do
  curl -sf http://green:8000/health && break
  sleep 5
done

# Switch traffic (update reverse proxy)
# If healthy: point nginx to green
# If unhealthy: keep routing to blue
```

## Secret Management

```python
# Load secrets from environment (NEVER hardcode)
import os

class Config:
    DATABASE_URL = os.environ["DATABASE_URL"]
    JWT_SECRET = os.environ["JWT_SECRET"]
    API_KEY = os.environ["API_KEY"]

    @classmethod
    def validate(cls):
        """Fail fast if required secrets missing."""
        required = ["DATABASE_URL", "JWT_SECRET", "API_KEY"]
        missing = [key for key in required if not os.environ.get(key)]
        if missing:
            raise RuntimeError(f"Missing required env vars: {', '.join(missing)}")
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Manual deployments | Automate with CI/CD pipeline |
| Secrets in git | Environment variables or secret manager |
| Root Docker containers | Add USER in Dockerfile |
| No health checks | /health endpoint + Docker HEALTHCHECK |
| No pipeline caching | Cache dependencies and Docker layers |
| Logs to stdout only | Structured JSON + centralized collection |
| No rollback plan | Blue-green or version-tagged deployments |
| Monolithic pipeline | Parallel stages for independent jobs |
