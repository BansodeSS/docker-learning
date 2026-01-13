# Day 10: Production Best Practices - Complete Guide

## 🎯 Development vs Production

### The Problem

**Development (What we've been doing):**
```yaml
services:
  app:
    image: myapp:latest           # ❌ "latest" is vague
    # No user specified           # ❌ Runs as root
    # No health checks            # ❌ Can't detect failures
    # No resource limits          # ❌ Can consume all memory
    # Passwords in plain text     # ❌ Security risk
```

**Production (What we need):**
```yaml
services:
  app:
    image: myapp:1.2.3            # ✅ Specific version
    user: "1000:1000"             # ✅ Non-root user
    healthcheck: ...              # ✅ Monitors health
    deploy:
      resources:
        limits: ...               # ✅ Resource limits
    secrets:
      - db_password               # ✅ Secure secrets
```

---

## 🔒 Security Best Practices

### 1️⃣ Use Specific Versions (Not "latest")

**❌ Bad (Development):**
```dockerfile
FROM python:latest
FROM node:latest
FROM nginx:latest
```

**Problem:** 
- "latest" changes over time
- Your app might break with new version
- Hard to reproduce issues
- Different team members get different versions

**✅ Good (Production):**
```dockerfile
FROM python:3.11.5-slim
FROM node:18.17.0-alpine
FROM nginx:1.25.2-alpine
```

**Benefits:**
- Reproducible builds
- Predictable behavior
- Can test updates before deploying

**Example:**
```powershell
# Create production Dockerfile
@"
# ✅ Specific versions
FROM python:3.11.5-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ✅ Even more specific
FROM python:3.11.5-slim
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY . .

# ✅ Non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8000
CMD [""python"", ""app.py""]
"@ | Out-File Dockerfile.prod -Encoding UTF8
```

---

### 2️⃣ Don't Run as Root

**Why is running as root dangerous?**

```
Container runs as root (UID 0)
    ↓
If attacker breaks out of container
    ↓
They have root access on host system!
    ↓
💥 Complete system compromise
```

**❌ Bad:**
```dockerfile
FROM python:3.11-slim
COPY . .
CMD ["python", "app.py"]
# ↑ Runs as root (UID 0)
```

**✅ Good:**
```dockerfile
FROM python:3.11-slim

# Create non-root user
RUN useradd -m -u 1000 appuser
# -m: Create home directory
# -u 1000: User ID (standard non-root UID)

# Change ownership of files
RUN chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

CMD ["python", "app.py"]
# ↑ Now runs as appuser (UID 1000)
```

**Complete Example:**
```powershell
# Create secure Dockerfile
@"
FROM python:3.11-slim

WORKDIR /app

# Install dependencies as root (needed)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application files
COPY . .

# Create non-root user
RUN groupadd -r appgroup && \
    useradd -r -g appgroup -u 1000 appuser && \
    chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

EXPOSE 8000
CMD [""python"", ""app.py""]
"@ | Out-File Dockerfile.secure -Encoding UTF8

# Build and verify
docker build -f Dockerfile.secure -t secure-app .

# Check what user it runs as
docker run --rm secure-app whoami
# Output: appuser ✅ (not root!)
```

---

### 3️⃣ Use Multi-Stage Builds (Security + Size)

**Why?**
- Build tools not needed in production
- Smaller image = less attack surface
- Faster deployments

**Example:**
```dockerfile
# Stage 1: Build (with all tools)
FROM node:18 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
# ↑ Has: npm, build tools, dev dependencies (500+ MB)

# Stage 2: Production (minimal)
FROM node:18-alpine
WORKDIR /app

# Create non-root user
RUN addgroup -g 1000 appgroup && \
    adduser -D -u 1000 -G appgroup appuser

# Copy only production files
COPY --from=build --chown=appuser:appgroup /app/dist ./dist
COPY --from=build --chown=appuser:appgroup /app/node_modules ./node_modules

USER appuser
EXPOSE 3000
CMD ["node", "dist/server.js"]
# ↑ Has: Only runtime + app (50 MB)
```

---

### 4️⃣ Scan for Vulnerabilities

**Check images for security issues:**

```powershell
# Scan an image
docker scan myapp:latest

# Or use Trivy (better)
# Install: choco install trivy
trivy image myapp:latest

# Example output:
# ┌───────────────┬────────────────┬──────────┬───────────────────┐
# │   Library     │ Vulnerability  │ Severity │ Fixed Version     │
# ├───────────────┼────────────────┼──────────┼───────────────────┤
# │ openssl       │ CVE-2023-1234  │ HIGH     │ 1.1.1t-r0        │
# │ curl          │ CVE-2023-5678  │ MEDIUM   │ 8.4.0-r2         │
# └───────────────┴────────────────┴──────────┴───────────────────┘
```

**Automated scanning in CI/CD:**
```yaml
# .github/workflows/security-scan.yml
name: Security Scan
on: [push]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build image
        run: docker build -t myapp .
      - name: Scan for vulnerabilities
        run: |
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy image myapp
```

---

### 5️⃣ Secrets Management

**❌ NEVER do this:**
```dockerfile
ENV DATABASE_PASSWORD=supersecret123
ENV API_KEY=abc123xyz789
# ↑ Secrets are IN THE IMAGE!
# Anyone with access to image can see them!
```

**✅ Do this instead:**

#### Option 1: Environment Files (Development)
```powershell
# Create .env file
@"
DATABASE_PASSWORD=supersecret123
API_KEY=abc123xyz789
"@ | Out-File .env -Encoding UTF8

# Add to .gitignore
Add-Content .gitignore ".env"

# Use in docker-compose.yml
@"
version: '3.8'
services:
  app:
    env_file:
      - .env
"@ | Out-File docker-compose.yml -Encoding UTF8
```

#### Option 2: Docker Secrets (Production)
```powershell
# Create secret
echo "supersecret123" | docker secret create db_password -

# Use in docker-compose.yml
@"
version: '3.8'

services:
  app:
    secrets:
      - db_password

secrets:
  db_password:
    external: true
"@ | Out-File docker-compose.yml -Encoding UTF8

# In application, read from:
# /run/secrets/db_password
```

#### Option 3: Pass at Runtime (Best for CI/CD)
```powershell
# Pass secrets as environment variables
docker run -e DATABASE_PASSWORD=$env:DB_PASS myapp

# Or from secret management system
docker run -e DATABASE_PASSWORD=$(vault read -field=password secret/db) myapp
```

---

## 🏥 Health Checks

### Why Health Checks?

**Without health check:**
```
Container starts → Appears "running"
    ↓
But app might be:
• Crashed
• Deadlocked  
• Out of memory
• Database connection lost
    ↓
Docker thinks it's fine! ❌
```

**With health check:**
```
Container starts → Health check runs
    ↓
App responds? 
    ✅ Healthy
    ❌ Unhealthy → Docker can restart
```

### Dockerfile Health Check

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY . .

# Health check configuration
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=40s \
            --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

# --interval=30s     Check every 30 seconds
# --timeout=10s      Wait max 10 seconds for response
# --start-period=40s Don't check for first 40 seconds (startup time)
# --retries=3        3 failures = unhealthy

CMD ["python", "app.py"]
```

### Health Check Endpoint

```python
# app.py
from flask import Flask
app = Flask(__name__)

@app.route('/health')
def health_check():
    # Check database connection
    try:
        db.execute('SELECT 1')
        return {'status': 'healthy'}, 200
    except:
        return {'status': 'unhealthy'}, 503

@app.route('/')
def home():
    return 'Hello World!'
```

### docker-compose.yml Health Check

```yaml
version: '3.8'

services:
  app:
    build: .
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### Check Health Status

```powershell
# View health status
docker ps
# Shows: (healthy) or (unhealthy) in STATUS column

# Inspect health
docker inspect --format='{{.State.Health.Status}}' myapp

# View health check logs
docker inspect --format='{{json .State.Health}}' myapp | ConvertFrom-Json
```

---

## 📊 Logging Best Practices

### Configure Log Rotation

**Problem:** Logs grow infinitely, fill up disk

```yaml
services:
  app:
    image: myapp
    logging:
      driver: "json-file"
      options:
        max-size: "10m"      # Each log file max 10MB
        max-file: "3"        # Keep max 3 log files
        # Total: 30MB max per container
```

### Structured Logging

**❌ Bad:**
```python
print("User logged in")
print("Error occurred")
```

**✅ Good:**
```python
import logging
import json

logger = logging.getLogger(__name__)

# Structured log entry
logger.info(json.dumps({
    'event': 'user_login',
    'user_id': 123,
    'timestamp': '2024-01-06T10:30:00Z',
    'ip': '192.168.1.100'
}))
```

### Centralized Logging (Production)

```yaml
version: '3.8'

services:
  app:
    image: myapp
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: myapp

  # Log aggregator
  fluentd:
    image: fluent/fluentd
    ports:
      - "24224:24224"
    volumes:
      - ./fluent.conf:/fluentd/etc/fluent.conf
```

---

## ⚙️ Production docker-compose.yml

### Complete Production Example

```yaml
version: '3.8'

services:
  # Application service
  app:
    image: myapp:${VERSION:-1.0.0}  # ✅ Version from env var
    container_name: myapp
    restart: unless-stopped         # ✅ Auto-restart
    
    # User configuration
    user: "1000:1000"              # ✅ Non-root
    
    # Resource limits
    deploy:
      replicas: 3                   # ✅ Multiple instances
      resources:
        limits:
          cpus: '0.5'              # ✅ Max 50% of one CPU
          memory: 512M             # ✅ Max 512MB RAM
        reservations:
          cpus: '0.25'             # ✅ Guaranteed 25% CPU
          memory: 256M             # ✅ Guaranteed 256MB RAM
      
      # Update strategy
      update_config:
        parallelism: 1              # ✅ Update one at a time
        delay: 10s                  # ✅ Wait 10s between updates
        failure_action: rollback    # ✅ Rollback on failure
      
      # Restart policy
      restart_policy:
        condition: on-failure       # ✅ Restart on crash
        delay: 5s                   # ✅ Wait 5s before restart
        max_attempts: 3             # ✅ Max 3 restart attempts
    
    # Health check
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
    # Logging
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        labels: "app=myapp,env=production"
    
    # Environment
    environment:
      - NODE_ENV=production
      - LOG_LEVEL=info
    
    # Secrets
    secrets:
      - db_password
      - api_key
    
    # Networks
    networks:
      - frontend
      - backend
    
    # Volumes
    volumes:
      - app-data:/app/data
    
    # Ports
    ports:
      - "8000:8000"
    
    # Dependencies
    depends_on:
      db:
        condition: service_healthy

  # Database service
  db:
    image: postgres:15.3-alpine
    restart: unless-stopped
    
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    
    secrets:
      - db_password
    
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./backups:/backups
    
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser"]
      interval: 10s
      timeout: 5s
      retries: 5
    
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    
    networks:
      - backend
    
    # No ports exposed! Only accessible via backend network

# Volumes
volumes:
  app-data:
    driver: local
  postgres-data:
    driver: local

# Networks
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # ✅ No external access

# Secrets
secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    file: ./secrets/api_key.txt
```

---

## 🚀 CI/CD Integration

### GitHub Actions Example

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [ main ]

env:
  REGISTRY: docker.io
  IMAGE_NAME: myorg/myapp

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      # Checkout code
      - name: Checkout repository
        uses: actions/checkout@v3
      
      # Set up Docker Buildx
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      # Login to Docker Hub
      - name: Log in to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      # Extract metadata
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha
      
      # Build and push
      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          file: ./Dockerfile.prod
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      # Scan for vulnerabilities
      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      # Run tests
      - name: Run tests
        run: |
          docker run --rm \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            npm test
      
      # Deploy (example using SSH)
      - name: Deploy to production
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.PROD_SERVER }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /opt/myapp
            docker compose pull
            docker compose up -d
            docker system prune -f
```

---

## 📋 Production Checklist

### Pre-Deployment Checklist

```powershell
# Create checklist script
@"
Write-Host "🔍 Production Readiness Checklist`n" -ForegroundColor Cyan

# 1. Version tags
Write-Host "✓ Checking image versions..." -ForegroundColor Yellow
docker compose config | Select-String "image:" | Select-String -NotMatch "latest"

# 2. Non-root users
Write-Host "`n✓ Checking for non-root users..." -ForegroundColor Yellow
docker compose exec app whoami

# 3. Health checks
Write-Host "`n✓ Checking health checks..." -ForegroundColor Yellow
docker compose config | Select-String "healthcheck"

# 4. Resource limits
Write-Host "`n✓ Checking resource limits..." -ForegroundColor Yellow
docker compose config | Select-String "limits"

# 5. Logging configuration
Write-Host "`n✓ Checking logging..." -ForegroundColor Yellow
docker compose config | Select-String "logging"

# 6. Secrets
Write-Host "`n✓ Checking secrets..." -ForegroundColor Yellow
docker compose config | Select-String "secrets"

# 7. Restart policies
Write-Host "`n✓ Checking restart policies..." -ForegroundColor Yellow
docker compose config | Select-String "restart"

Write-Host "`n✅ Checklist complete!`n" -ForegroundColor Green
"@ | Out-File check-production.ps1 -Encoding UTF8

# Run checklist
.\check-production.ps1
```

### Monitoring Setup

```yaml
version: '3.8'

services:
  # Your application
  app:
    image: myapp:1.0.0
    # ... app configuration

  # Prometheus (metrics)
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'

  # Grafana (dashboards)
  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana

  # cAdvisor (container metrics)
  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

volumes:
  prometheus-data:
  grafana-data:
```

---

## ✅ Complete Production Setup Example

```powershell
# Create production-ready project
mkdir C:\myapp-production
cd C:\myapp-production

# Create directory structure
New-Item -ItemType Directory -Path "secrets", "backups", ".github\workflows" -Force

# Create secrets
"supersecret123" | Out-File secrets\db_password.txt -Encoding UTF8 -NoNewline
"api-key-xyz789" | Out-File secrets\api_key.txt -Encoding UTF8 -NoNewline

# Create production Dockerfile
@"
FROM python:3.11.5-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11.5-slim
WORKDIR /app

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

# Copy dependencies
COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appuser . .

# Switch to non-root
USER appuser
ENV PATH=/home/appuser/.local/bin:`$PATH

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000
CMD [""python"", ""app.py""]
"@ | Out-File Dockerfile.prod -Encoding UTF8

# Create production docker-compose.yml
@"
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.prod
    image: myapp:1.0.0
    restart: unless-stopped
    user: ""1000:1000""
    
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    
    healthcheck:
      test: [""CMD"", ""curl"", ""-f"", ""http://localhost:8000/health""]
      interval: 30s
      timeout: 10s
      retries: 3
    
    logging:
      driver: ""json-file""
      options:
        max-size: ""10m""
        max-file: ""3""
    
    secrets:
      - db_password
    
    ports:
      - ""8000:8000""

secrets:
  db_password:
    file: ./secrets/db_password.txt

"@ | Out-File docker-compose.prod.yml -Encoding UTF8

Write-Host "✅ Production setup complete!" -ForegroundColor Green
```

---

## 🎓 Summary

**Production vs Development:**

| Aspect | Development | Production |
|--------|-------------|-----------|
| Image tags | `latest` | `1.2.3` |
| User | root | Non-root (UID 1000) |
| Health checks | None | Required |
| Resource limits | None | CPU + Memory limits |
| Secrets | Plain text | Docker secrets/vault |
| Logging | Console | Rotated JSON logs |
| Restart | Manual | Auto-restart |
| Monitoring | None | Prometheus + Grafana |

**Key Takeaways:**
1. **Never use `latest` in production**
2. **Always run as non-root user**
3. **Implement health checks**
4. **Set resource limits**
5. **Use proper secrets management**
6. **Configure log rotation**
7. **Enable monitoring**
8. **Automate with CI/CD**

**You're now ready for production Docker deployments!** 🚀

Want help setting up any specific part? Monitoring? CI/CD? Security scanning? Let me know! 😊