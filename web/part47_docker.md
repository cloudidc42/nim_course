# Part 47: Docker และ CI/CD สำหรับ Nim

## Dockerfile สำหรับ Nim Application

```dockerfile
# Dockerfile - multi-stage build
# Stage 1: Build
FROM nimlang/nim:2.0.0 AS builder

WORKDIR /app

# Copy nimble files first (cache dependencies)
COPY *.nimble ./
RUN nimble install -y --depsOnly

# Copy source
COPY src/ ./src/

# Build release binary
RUN nim c -d:release --opt:speed \
    --passL:"-static" \
    -o:bin/app \
    src/main.nim

# Stage 2: Runtime (minimal image)
FROM scratch

# Copy only the binary
COPY --from=builder /app/bin/app /app

# Copy SSL certs if needed
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

EXPOSE 8080
CMD ["/app"]
```

```dockerfile
# Dockerfile.dev - development with hot reload
FROM nimlang/nim:2.0.0

WORKDIR /app

# Install development tools
RUN nimble install -g nimlangserver

COPY *.nimble ./
RUN nimble install -y --depsOnly

COPY . .

# Development server
CMD ["nim", "c", "-r", "src/main.nim"]
```

## Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://postgres:password@db:5432/myapp
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=change-in-production
      - SERVER_PORT=8080
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped
    volumes:
      - ./data:/app/data
    networks:
      - app-network

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: myapp
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes
    networks:
      - app-network

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - app
    networks:
      - app-network

volumes:
  postgres-data:
  redis-data:

networks:
  app-network:
    driver: bridge
```

## nginx.conf

```nginx
# nginx.conf - reverse proxy with SSL
events {
  worker_connections 1024;
}

http {
  upstream app {
    server app:8080;
    keepalive 32;
  }

  server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
  }

  server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    location / {
      proxy_pass http://app;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection 'upgrade';
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_cache_bypass $http_upgrade;
    }

    location /api/ {
      limit_req zone=api burst=20 nodelay;
      proxy_pass http://app;
    }
  }
}
```

## GitHub Actions CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Nim
        uses: jiro4989/setup-nim-action@v1
        with:
          nim-version: 'stable'
      
      - name: Cache Nimble packages
        uses: actions/cache@v3
        with:
          path: ~/.nimble
          key: ${{ runner.os }}-nimble-${{ hashFiles('*.nimble') }}
      
      - name: Install dependencies
        run: nimble install -y
      
      - name: Run tests
        run: nimble test
      
      - name: Type check
        run: nim check src/main.nim
  
  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Log in to registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha
            type=ref,event=branch
            type=semver,pattern={{version}}
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/myapp
            docker-compose pull
            docker-compose up -d --no-deps app
            docker-compose exec -T app /app --healthcheck
```

## Nim สำหรับ Docker Health Check

```nim
# src/main.nim - with health check endpoint
import asynchttpserver, asyncdispatch, json, os, strutils

var startTime = now()
var requestCount = 0

proc handler(req: Request) {.async.} =
  inc requestCount
  
  if req.url.path == "/health":
    let uptime = (now() - startTime).inSeconds()
    await req.respond(Http200,
      $(%*{"status": "ok", "uptime": uptime, "requests": requestCount}),
      newHttpHeaders({"Content-Type": "application/json"}))
    return
  
  if req.url.path == "/ready":
    # Check DB connection, etc.
    await req.respond(Http200, """{"ready":true}""",
      newHttpHeaders({"Content-Type": "application/json"}))
    return
  
  await req.respond(Http200, "Hello World")

let port = parseInt(getEnv("SERVER_PORT", "8080"))
echo "Starting on port ", port
let server = newAsyncHttpServer()
waitFor server.serve(Port(port), handler)
```

## สรุป Part 47

- ␅ Multi-stage Dockerfile
- ␅ Docker Compose สำหรับ full stack
- ␅ Nginx reverse proxy
- ␅ GitHub Actions CI/CD pipeline
- ␅ Health check endpoints

---
**Next**: [Part 48 - Redis Caching](part48_redis.md)
