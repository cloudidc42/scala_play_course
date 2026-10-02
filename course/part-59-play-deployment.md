# Part 59: Play Deployment

## Steps 581-590: Production Config, SSL, Nginx, Docker, JVM Tuning

---

## Step 581: Production Configuration

```hocon
# conf/application.conf (production overrides)
include "reference.conf"

# Production settings
play.http.secret.key = ${?APPLICATION_SECRET}

# Database
db.default {
  driver   = org.postgresql.Driver
  url      = ${?DATABASE_URL}
  username = ${?DATABASE_USER}
  password = ${?DATABASE_PASSWORD}
  hikaricp {
    maximumPoolSize = 20
    minimumIdle     = 5
    connectionTimeout = 30000
    idleTimeout     = 600000
    maxLifetime     = 1800000
  }
}

# Disable dev features
play.evolutions.db.default.autoApply = false
play.evolutions.db.default.autoApplyDowns = false

# Security
play.filters.https.redirectEnabled = true
play.filters.headers.contentSecurityPolicy = "default-src 'self'"

# Logging
play.logger.root = WARN
play.logger.play  = INFO
play.logger.application = INFO
```

```bash
# สร้าง Application Secret
sbt playGenerateSecret
# หรือ
openssl rand -base64 32
```

---

## Step 582: Environment-Specific Configs

```
conf/
├── application.conf        # Base config
├── application.dev.conf    # Development overrides
├── application.prod.conf   # Production overrides
├── application.test.conf   # Test overrides
└── reference.conf          # Module defaults
```

```hocon
# conf/application.prod.conf
include "application.conf"

play.http.secret.key = ${APPLICATION_SECRET}
play.filters.https.redirectEnabled = true

db.default.url = ${DATABASE_URL}

# Caching
play.cache.ehcache.maxSize = 10000

# Thread pool tuning
pekko {
  actor {
    default-dispatcher {
      fork-join-executor {
        parallelism-min  = 8
        parallelism-max  = 64
        parallelism-factor = 3.0
      }
    }
  }
}
```

```bash
# Run with production config
sbt -Dconfig.file=conf/application.prod.conf run

# Or using environment variable
CONFIG_FILE=conf/application.prod.conf sbt run
```

---

## Step 583: Building for Production

```bash
# build.sbt
name         := "my-app"
version      := "1.0.0"
scalaVersion := "3.3.3"

lazy val root = (project in file("."))
  .enablePlugins(PlayScala)

# Build production distribution
sbt dist

# Creates: target/universal/my-app-1.0.0.zip
# Contains:
#   bin/my-app         (Unix start script)
#   bin/my-app.bat     (Windows start script)
#   lib/               (All JARs)
#   conf/              (Config files)
```

```bash
# Start production app
unzip my-app-1.0.0.zip
cd my-app-1.0.0

bin/my-app \
  -Dplay.http.secret.key="<secret>" \
  -Dconfig.file=conf/application.prod.conf \
  -Dhttp.port=9000 \
  -J-Xmx2g \
  -J-Xms512m
```

---

## Step 584: JVM Tuning

```bash
# Production JVM options
# conf/jvm.options (หรือใส่ใน start script)

# Memory
-Xmx2g              # Max heap
-Xms512m            # Initial heap
-XX:MetaspaceSize=128m
-XX:MaxMetaspaceSize=256m

# GC — G1GC for low latency
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=16m
-XX:+ParallelRefProcEnabled

# JIT Compilation
-XX:+TieredCompilation
-XX:CompileThreshold=1000

# Monitoring
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:/var/log/myapp/gc.log
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=5
-XX:GCLogFileSize=20m

# Crash dump
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/myapp/heapdump.hprof

# String deduplication (reduces memory)
-XX:+UseStringDeduplication
```

```scala
// build.sbt — JVM options
javaOptions in Universal ++= Seq(
  "-Dpidfile.path=/dev/null",
  "-J-Xmx2g",
  "-J-Xms512m",
  "-J-XX:+UseG1GC",
  "-J-XX:MaxGCPauseMillis=200"
)

// scriptClasspath +=  ฯลฯ
```

---

## Step 585: Docker

```dockerfile
# Dockerfile
# Stage 1: Build
FROM sbtscala/scala-sbt:eclipse-temurin-jammy-21.0.1_12_1.9.7_3.3.1 AS builder

WORKDIR /app
COPY . .

# Build production distribution
RUN sbt dist

# Extract distribution
RUN unzip target/universal/my-app-*.zip -d /app/dist && \
    mv /app/dist/my-app-* /app/dist/app

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-jammy

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Copy built app
COPY --from=builder /app/dist/app .

# Set ownership
RUN chown -R appuser:appuser /app

USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:9000/health || exit 1

EXPOSE 9000

ENV JAVA_OPTS="-Xmx1g -Xms256m -XX:+UseG1GC"

CMD ["bin/my-app", \
     "-Dplay.http.secret.key=${APPLICATION_SECRET}", \
     "-Dconfig.file=conf/application.prod.conf", \
     "-Dhttp.port=9000"]
```

```yaml
# docker-compose.yml
version: "3.9"

services:
  app:
    build: .
    ports:
      - "9000:9000"
    environment:
      APPLICATION_SECRET: ${APPLICATION_SECRET}
      DATABASE_URL: jdbc:postgresql://db:5432/myapp
      DATABASE_USER: ${DB_USER}
      DATABASE_PASSWORD: ${DB_PASSWORD}
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - app-network

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - app
    networks:
      - app-network

volumes:
  postgres_data:

networks:
  app-network:
    driver: bridge
```

---

## Step 586: Nginx Reverse Proxy

```nginx
# nginx.conf
worker_processes auto;

events {
  worker_connections 1024;
  use epoll;
  multi_accept on;
}

http {
  # Gzip compression
  gzip on;
  gzip_vary on;
  gzip_proxied any;
  gzip_types text/plain text/css application/json application/javascript;
  gzip_min_length 1000;

  # Rate limiting
  limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
  limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

  # Upstream Play app
  upstream play_app {
    server app:9000;
    keepalive 32;
  }

  # Redirect HTTP to HTTPS
  server {
    listen 80;
    server_name myapp.com www.myapp.com;
    return 301 https://$server_name$request_uri;
  }

  # Main HTTPS server
  server {
    listen 443 ssl http2;
    server_name myapp.com www.myapp.com;

    # SSL/TLS
    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    # HSTS
    add_header Strict-Transport-Security "max-age=63072000" always;

    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";

    # Buffer settings
    client_max_body_size 50M;
    proxy_buffer_size    128k;
    proxy_buffers        4 256k;
    proxy_busy_buffers_size 256k;

    # Static assets with caching
    location /assets/ {
      proxy_pass http://play_app;
      proxy_cache_valid 200 30d;
      add_header Cache-Control "public, max-age=2592000, immutable";
      expires 30d;
    }

    # API rate limiting
    location /api/ {
      limit_req zone=api burst=20 nodelay;
      limit_conn conn_limit 20;
      proxy_pass http://play_app;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket support
    location /ws/ {
      proxy_pass http://play_app;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_read_timeout 86400;
    }

    # Default proxy
    location / {
      proxy_pass http://play_app;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_read_timeout 60s;
      proxy_connect_timeout 10s;
    }
  }
}
```

---

## Step 587: SSL Setup with Let's Encrypt

```bash
# Install certbot
apt-get install -y certbot python3-certbot-nginx

# Get SSL certificate
certbot --nginx -d myapp.com -d www.myapp.com

# Auto-renewal
crontab -e
# 0 0 * * * /usr/bin/certbot renew --quiet
```

---

## Step 588: Kubernetes Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: play-app
  labels:
    app: play-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: play-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: play-app
    spec:
      containers:
      - name: play-app
        image: myregistry/play-app:1.0.0
        ports:
        - containerPort: 9000
        env:
        - name: APPLICATION_SECRET
          valueFrom:
            secretKeyRef:
              name: play-secrets
              key: application-secret
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: play-secrets
              key: database-url
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 9000
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 9000
          initialDelaySeconds: 30
          periodSeconds: 10

---
apiVersion: v1
kind: Service
metadata:
  name: play-app-service
spec:
  selector:
    app: play-app
  ports:
  - port: 80
    targetPort: 9000
  type: LoadBalancer
```

---

## Step 589: CI/CD with GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-java@v4
      with:
        java-version: '21'
        distribution: 'temurin'
        cache: 'sbt'

    - name: Run tests
      run: sbt test

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}

    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: myuser/play-app:${{ github.sha }},myuser/play-app:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/play-app \
          play-app=myuser/play-app:${{ github.sha }}
        kubectl rollout status deployment/play-app
```

---

## Step 590: Monitoring & Observability

```scala
// app/controllers/MetricsController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*

@Singleton
class MetricsController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  def metrics(): Action[AnyContent] = Action {
    val runtime = Runtime.getRuntime
    val totalMem = runtime.totalMemory()
    val freeMem  = runtime.freeMemory()
    val usedMem  = totalMem - freeMem

    Ok(Json.obj(
      "memory" -> Json.obj(
        "total" -> totalMem,
        "used"  -> usedMem,
        "free"  -> freeMem,
        "max"   -> runtime.maxMemory()
      ),
      "threads" -> Thread.activeCount(),
      "uptime"  -> java.lang.management.ManagementFactory.getRuntimeMXBean.getUptime
    ))
  }
}
```

---

## สรุป Part 59

| Topic | Tool/Approach | Key Config |
|-------|--------------|------------|
| Production build | `sbt dist` | Creates zip distribution |
| Docker | Multi-stage Dockerfile | Minimal JRE runtime image |
| Nginx | Reverse proxy | SSL termination, rate limiting |
| JVM Tuning | G1GC, heap sizing | `-Xmx2g -XX:+UseG1GC` |
| Kubernetes | Deployment + Service | Rolling update, health probes |
| CI/CD | GitHub Actions | Test → Build → Deploy |
| SSL | Let's Encrypt | Auto-renewal with certbot |
| Monitoring | `/health`, `/metrics` | K8s liveness/readiness probes |

---

## แบบฝึกหัด Part 59

1. **Docker Multi-stage Build**: สร้าง optimized Docker image ที่ใช้ multi-stage build และ minimal JRE base image ให้ image size < 200MB

2. **Zero-Downtime Deploy**: Implement blue-green deployment ใน Kubernetes ที่ทำ rolling update โดยไม่มี downtime ด้วย proper health probes

3. **JVM Profiling**: Setup JVM monitoring ด้วย JMX หรือ async-profiler แล้ว identify และ fix memory leak ใน sample application

4. **Nginx Load Balancing**: Configure Nginx upstream ที่ load balance ระหว่าง 3 Play instances ด้วย sticky sessions และ health checking

5. **Full CI/CD Pipeline**: สร้าง complete CI/CD pipeline ที่ run tests, build Docker image, push to registry, และ deploy to staging แล้ว production ด้วย manual approval

---

[→ ไปยัง Part 60: Play GraphQL](part-60-play-graphql.md)
