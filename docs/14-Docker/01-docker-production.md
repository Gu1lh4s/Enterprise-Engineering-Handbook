# Docker for Production

> **Category:** Infrastructure
> **Version:** 1.0.0

---

## Table of Contents

1. [Production Dockerfiles](#1-production-dockerfiles)
2. [Multi-Stage Builds](#2-multi-stage-builds)
3. [Security Hardening](#3-security-hardening)
4. [Docker Compose for Development](#4-docker-compose-for-development)
5. [Image Optimization](#5-image-optimization)
6. [Health Checks](#6-health-checks)
7. [Secrets and Configuration](#7-secrets-and-configuration)
8. [Networking](#8-networking)
9. [Logging and Monitoring](#9-logging-and-monitoring)
10. [Container Security Best Practices](#10-container-security-best-practices)

---

## 1. Production Dockerfiles

### Node.js API Production Dockerfile

```dockerfile
# ━━━ Stage 1: Dependencies ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FROM node:20-alpine AS deps
WORKDIR /app

# Copy only package files first (cache layer — only rebuilds if deps change)
COPY package.json package-lock.json ./

# Install production dependencies only
RUN npm ci --only=production

# ━━━ Stage 2: Build ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FROM node:20-alpine AS builder
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci  # Full deps (includes devDependencies for build)

COPY . .
RUN npm run build  # TypeScript compile → dist/

# ━━━ Stage 3: Production ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FROM node:20-alpine AS production

# Security: non-root user
RUN addgroup -g 1001 -S nodejs && adduser -S nodeapp -u 1001 -G nodejs

WORKDIR /app

# Copy only what's needed in production (no source, no devDeps)
COPY --from=deps    --chown=nodeapp:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodeapp:nodejs /app/dist         ./dist
COPY --from=builder --chown=nodeapp:nodejs /app/package.json ./package.json

# Remove write permission from app files (defense in depth)
RUN chmod -R a-w /app

# Drop to non-root user
USER nodeapp

# Port declaration (documentation only — doesn't actually expose)
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => r.statusCode === 200 ? process.exit(0) : process.exit(1)).on('error', () => process.exit(1))"

# Start application
CMD ["node", "dist/server.js"]
```

### Python FastAPI Dockerfile

```dockerfile
FROM python:3.12-slim AS production

# Security: create non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*  # Clean apt cache (smaller image)

WORKDIR /app

# Copy and install Python dependencies
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY --chown=appuser:appgroup . .

# Set ownership and switch to non-root
USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", \
     "--workers", "4", "--no-access-log"]
```

### Go Binary (Distroless)

```dockerfile
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
# CGO_ENABLED=0: static binary, no C dependencies
# -ldflags="-s -w": strip debug info (smaller binary)
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w" -o /app/server ./cmd/server

# Distroless: no shell, no package manager, no OS utilities
# Attack surface is minimal
FROM gcr.io/distroless/static-debian12:nonroot AS production

COPY --from=builder /app/server /server

EXPOSE 8080

ENTRYPOINT ["/server"]
```

---

## 2. Multi-Stage Builds

```dockerfile
# Next.js production build
FROM node:20-alpine AS base
WORKDIR /app

# ─── Dependencies ────────────────────────────────────────────────────────────
FROM base AS deps
COPY package.json package-lock.json ./
RUN npm ci

# ─── Builder ─────────────────────────────────────────────────────────────────
FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Build-time environment variables (not secrets)
ENV NEXT_TELEMETRY_DISABLED=1
ARG NEXT_PUBLIC_API_URL
ENV NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}

RUN npm run build

# ─── Runner ──────────────────────────────────────────────────────────────────
FROM base AS runner

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN addgroup -g 1001 -S nodejs && adduser -S nextjs -u 1001 -G nodejs

# Next.js standalone output (includes only necessary files)
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --from=builder --chown=nextjs:nodejs /app/public ./public

USER nextjs

EXPOSE 3000

CMD ["node", "server.js"]
```

### .dockerignore

```
# Always have a .dockerignore (equivalent of .gitignore for Docker)
.git
.gitignore
.github
*.md
Dockerfile*
docker-compose*
.env
.env.*
node_modules  # Never copy — rebuilt in container
dist          # Rebuilt in build stage
.next
coverage
*.test.ts
*.spec.ts
__tests__
.nyc_output
*.log
.DS_Store
```

---

## 3. Security Hardening

### Non-Root User

```dockerfile
# NEVER run as root in production
# Default containers run as root — full filesystem access if container escapes

# Method 1: Create user in Dockerfile
RUN addgroup -g 1001 -S appgroup && adduser -u 1001 -S appuser -G appgroup
USER appuser

# Method 2: Use Dockerfile USER with UID directly (Kubernetes prefers this)
USER 1001  # Numeric UID — works when UName not defined

# Verify: docker run --rm my-image id
# Expected: uid=1001(appuser) gid=1001(appgroup) groups=1001(appgroup)
# BAD: uid=0(root) gid=0(root) groups=0(root)
```

### Read-Only Filesystem

```yaml
# docker-compose.yml — read-only root filesystem
services:
  api:
    image: my-api:latest
    read_only: true  # Root filesystem is read-only
    tmpfs:
      - /tmp          # App can still write to /tmp (in-memory)
      - /var/run      # PID files, sockets
    volumes:
      - uploads:/app/uploads  # Explicit writable volume for uploads only
```

```yaml
# Kubernetes — securityContext
spec:
  containers:
  - name: api
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 1001
      runAsGroup: 1001
      capabilities:
        drop: [ALL]  # Drop all Linux capabilities
        add: [NET_BIND_SERVICE]  # Add only what's needed (binding port < 1024)
```

### No New Privileges

```dockerfile
# Prevent privilege escalation via setuid binaries
# Add to Dockerfile
RUN chmod a-s /bin/su /usr/bin/sudo  # Remove setuid bit from common tools

# Or set in docker run / compose
# --security-opt=no-new-privileges:true

# Scan for setuid files
RUN find / -perm /4000 -type f 2>/dev/null | grep -v '^/$'
```

---

## 4. Docker Compose for Development

```yaml
# docker-compose.yml — full local development stack
version: '3.9'

networks:
  app-network:
    driver: bridge

volumes:
  postgres-data:
  redis-data:
  node-modules:  # Named volume for node_modules (prevents override by host mount)

services:
  # ─── Application ───────────────────────────────────────────────────────────
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev  # Different from production Dockerfile
    volumes:
      - .:/app                    # Mount source for hot reload
      - node-modules:/app/node_modules  # Use container's node_modules
    ports:
      - "3000:3000"
      - "9229:9229"  # Node.js debug port
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://dev:dev@postgres:5432/devdb
      - REDIS_URL=redis://redis:6379
    env_file:
      - .env.local  # Local developer secrets (gitignored)
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks: [app-network]
    restart: unless-stopped
    
    # Health check for compose depends_on
    healthcheck:
      test: ["CMD", "node", "-e", "require('http').get('http://localhost:3000/health', r => process.exit(r.statusCode === 200 ? 0 : 1))"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
  
  # ─── Database ──────────────────────────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: devdb
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro  # Init scripts
    ports:
      - "5432:5432"  # Expose to host for pgAdmin/DataGrip
    networks: [app-network]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev -d devdb"]
      interval: 5s
      timeout: 5s
      retries: 5
  
  # ─── Cache ─────────────────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    command: redis-server --save 60 1 --loglevel warning
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"
    networks: [app-network]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3
  
  # ─── Development Tools ─────────────────────────────────────────────────────
  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "8025:8025"  # Web UI
      - "1025:1025"  # SMTP
    networks: [app-network]
  
  # PgAdmin for database management
  pgadmin:
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: dev@local.dev
      PGADMIN_DEFAULT_PASSWORD: devpassword
    ports:
      - "5050:80"
    networks: [app-network]
    profiles: [tools]  # Only start with: docker compose --profile tools up
```

---

## 5. Image Optimization

```bash
# Analyze image layers
docker history my-image:latest --no-trunc

# Detailed analysis with dive tool
dive my-image:latest

# Check image size
docker images my-image:latest

# Optimization techniques:
# 1. Use alpine or distroless base (alpine: ~5MB vs ubuntu: ~80MB)
# 2. Combine RUN commands (fewer layers)
# 3. Clean up in same RUN layer (cleanup in next layer doesn't reduce size)
# 4. Use .dockerignore
# 5. Use multi-stage builds (build tools not in final image)
# 6. Use --no-cache-dir for pip (don't cache wheels in image)
# 7. Use --no-install-recommends for apt (skip optional packages)

# BAD: cleanup in next layer (doesn't reduce size)
RUN apt-get update && apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*  # This layer adds 0 bytes in terms of size reduction

# GOOD: cleanup in same layer
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*  # Size saved because it's same layer
```

### Layer Caching Strategy

```dockerfile
# Order matters for cache efficiency
# Most frequently changing files → go last

# Rarely changes (dependencies)
COPY package.json package-lock.json ./
RUN npm ci  # Only re-runs if package files change

# Sometimes changes (config)
COPY tsconfig.json .eslintrc.js ./

# Frequently changes (source code)
COPY src/ ./src/

# Build
RUN npm run build
```

---

## 6. Health Checks

```typescript
// Application health endpoint
app.get('/health', (req, res) => res.json({ status: 'alive', timestamp: new Date() }))

// Ready endpoint (deeper check: DB + external deps)
app.get('/health/ready', async (req, res) => {
  const checks = await Promise.allSettled([
    db.query('SELECT 1'),
    redis.ping(),
  ])
  
  const results = {
    database: checks[0].status === 'fulfilled' ? 'ok' : 'failing',
    cache: checks[1].status === 'fulfilled' ? 'ok' : 'failing',
  }
  
  const healthy = Object.values(results).every(v => v === 'ok')
  
  res.status(healthy ? 200 : 503).json({
    status: healthy ? 'ready' : 'not_ready',
    checks: results,
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
  })
})
```

```dockerfile
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=30s \
            --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1
  # OR: curl -f http://localhost:3000/health
  # OR: node healthcheck.js  (custom script)
```

---

## 7. Secrets and Configuration

```yaml
# NEVER bake secrets into images
# Use runtime injection:

# docker-compose (development): env_file or environment
services:
  api:
    env_file:
      - .env.local  # Not committed to git

# docker run (simple)
docker run \
  -e DATABASE_URL="$(aws ssm get-parameter --name /prod/db-url --with-decryption --query Parameter.Value --output text)" \
  my-image:latest

# Docker Secrets (Docker Swarm)
# File at /run/secrets/db_password inside container
services:
  api:
    secrets:
      - db_password
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password  # App reads from file

secrets:
  db_password:
    external: true  # Managed by Docker Swarm secret store
```

```typescript
// Read secret from file (Docker Secrets pattern)
import { readFileSync } from 'fs'

function readSecret(name: string): string {
  const filePath = `/run/secrets/${name}`
  try {
    return readFileSync(filePath, 'utf8').trim()
  } catch {
    // Fallback to environment variable (development)
    const envVar = process.env[name.toUpperCase()]
    if (!envVar) throw new Error(`Secret ${name} not found`)
    return envVar
  }
}

const dbPassword = readSecret('db_password')
```

---

## 8. Networking

```yaml
# Custom networks for service isolation
networks:
  frontend:   # Web → API
  backend:    # API → Database (DB not exposed to frontend)
  monitoring: # Metrics scraping

services:
  nginx:
    networks: [frontend]  # Only frontend network
  
  api:
    networks: [frontend, backend]  # Both
  
  postgres:
    networks: [backend]  # Only backend — not accessible from frontend
  
  prometheus:
    networks: [monitoring, backend]  # Scrapes from backend services
```

---

## 9. Logging and Monitoring

```yaml
# docker-compose logging drivers
services:
  api:
    logging:
      driver: "json-file"       # Default: local JSON files
      options:
        max-size: "100m"        # Rotate at 100MB
        max-file: "3"           # Keep 3 rotated files
    
    # OR: forward to centralized logging
    logging:
      driver: "awslogs"
      options:
        awslogs-group: "/prod/api"
        awslogs-region: "us-east-1"
        awslogs-stream-prefix: "api"
    
    # OR: forward to Loki (Grafana)
    logging:
      driver: "loki"
      options:
        loki-url: "http://loki:3100/loki/api/v1/push"
        loki-labels: "service=api,environment=production"
```

---

## 10. Container Security Best Practices

```
1. Run as non-root (USER instruction)
2. Read-only root filesystem
3. No new privileges (--security-opt=no-new-privileges)
4. Drop all Linux capabilities (--cap-drop=all)
5. Use minimal base images (alpine, distroless)
6. Scan images for CVEs (Trivy, Grype, Snyk)
7. Sign images (cosign)
8. No secrets in ENV vars in Dockerfile (use runtime injection)
9. No secrets in image layers (use BuildKit secrets for build-time)
10. Pin base image to specific digest (not mutable tag)
11. Enable seccomp and AppArmor profiles
12. Limit CPU and memory (prevent container from consuming all host resources)

# Resource limits in docker-compose
services:
  api:
    deploy:
      resources:
        limits:
          cpus: '2.0'      # Max 2 CPU cores
          memory: '512M'   # Max 512MB RAM
        reservations:
          cpus: '0.25'     # Reserve 0.25 CPU
          memory: '256M'   # Reserve 256MB

# Seccomp profile (restrict system calls)
docker run --security-opt seccomp=my-seccomp-profile.json my-image

# AppArmor profile
docker run --security-opt apparmor=my-profile my-image
```
