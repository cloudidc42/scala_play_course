# Part 83: Docker and Scala — Steps 821-830

## บทนำ: Containerizing Scala Applications

Docker ช่วยให้ deploy Scala applications ได้อย่าง consistent ในทุก environment การใช้ multi-stage builds ช่วยลด image size และ sbt-native-packager ช่วย generate Docker images อัตโนมัติ

---

## Step 821: Dockerfile สำหรับ Scala

```dockerfile
# Dockerfile — Production-ready Scala app
# Multi-stage build

# ===== Stage 1: Build =====
FROM eclipse-temurin:17-jdk-jammy AS builder

# Install sbt
RUN apt-get update && apt-get install -y curl gnupg && \
    echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" | tee /etc/apt/sources.list.d/sbt.list && \
    curl -sL "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x2EE0EA64E40A89B84B2DF73499E82A75642AC823" | apt-key add && \
    apt-get update && \
    apt-get install -y sbt && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy build files first (for layer caching)
COPY build.sbt .
COPY project/ project/

# Pre-download dependencies (cached layer)
RUN sbt update

# Copy source code
COPY src/ src/

# Build the application
RUN sbt "set test in assembly := {}" assembly

# ===== Stage 2: Runtime =====
FROM eclipse-temurin:17-jre-jammy AS runtime

# Security: non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Copy JAR from build stage
COPY --from=builder /app/target/scala-2.13/myapp-assembly-*.jar app.jar

# Application config
COPY src/main/resources/application.conf .
COPY src/main/resources/logback.xml .

# Set ownership
RUN chown -R appuser:appuser /app

USER appuser

# JVM options สำหรับ container
ENV JAVA_OPTS="\
  -server \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:+UseG1GC \
  -XX:+OptimizeStringConcat \
  -Dfile.encoding=UTF-8 \
  -Dlogback.configurationFile=logback.xml"

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

---

## Step 822: sbt-native-packager

```scala
// build.sbt สำหรับ Docker packaging
import com.typesafe.sbt.packager.docker._

name := "scala-microservice"
version := "1.0.0"
scalaVersion := "2.13.12"

// เพิ่ม plugin
// project/plugins.sbt:
// addSbtPlugin("com.github.sbt" % "sbt-native-packager" % "1.9.16")

enablePlugins(JavaAppPackaging, DockerPlugin, AshScriptPlugin)

// Docker settings
Docker / packageName := "mycompany/scala-microservice"
Docker / version     := version.value

dockerBaseImage := "eclipse-temurin:17-jre-jammy"

dockerExposedPorts ++= Seq(8080, 9443)

dockerLabels ++= Map(
  "maintainer"   -> "team@mycompany.com",
  "version"      -> version.value,
  "description"  -> "Scala Microservice"
)

// JVM options
Universal / javaOptions ++= Seq(
  "-J-XX:+UseContainerSupport",
  "-J-XX:MaxRAMPercentage=75.0",
  "-J-XX:+UseG1GC",
  "-J-Dfile.encoding=UTF-8"
)

// Custom Docker commands
dockerCommands ++= Seq(
  Cmd("USER", "root"),
  ExecCmd("RUN", "apt-get", "update", "-y"),
  ExecCmd("RUN", "apt-get", "install", "-y", "curl"),
  Cmd("USER", "1001:0")
)

// Build: sbt docker:publishLocal
// Push:  sbt docker:publish
```

---

## Step 823: Docker Compose สำหรับ Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ===== Application =====
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    image: scala-microservice:latest
    container_name: scala-app
    ports:
      - "8080:8080"
    environment:
      - APP_ENV=development
      - DB_URL=jdbc:postgresql://postgres:5432/myapp
      - DB_USER=myapp
      - DB_PASSWORD=secret
      - KAFKA_BROKERS=kafka:9092
      - REDIS_HOST=redis
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_healthy
    volumes:
      - ./config:/app/config:ro
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
    networks:
      - app-network

  # ===== PostgreSQL =====
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  # ===== Kafka =====
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - app-network

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    healthcheck:
      test: ["CMD", "kafka-topics", "--bootstrap-server", "localhost:9092", "--list"]
      interval: 30s
      timeout: 10s
      retries: 5
    networks:
      - app-network

  # ===== Redis =====
  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes
    networks:
      - app-network

  # ===== Observability =====
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    networks:
      - app-network

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - app-network

volumes:
  postgres-data:
  redis-data:
  grafana-data:

networks:
  app-network:
    driver: bridge
```

---

## Step 824: Container Optimization

```dockerfile
# Dockerfile.optimized — Optimized multi-stage build

# ===== Dependency Cache Stage =====
FROM eclipse-temurin:17-jdk-jammy AS deps

RUN apt-get update && \
    apt-get install -y --no-install-recommends curl gnupg && \
    echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" | \
      tee /etc/apt/sources.list.d/sbt.list && \
    curl -sL "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x2EE0EA64E40A89B84B2DF73499E82A75642AC823" | \
      apt-key add - && \
    apt-get update && \
    apt-get install -y --no-install-recommends sbt && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Only copy files needed for dependency resolution
COPY build.sbt project/build.properties project/plugins.sbt ./project/

# Download dependencies (this layer cached until build.sbt changes)
RUN sbt update

# ===== Build Stage =====
FROM deps AS build

COPY . .
RUN sbt assembly

# ===== JLink Stage: Custom JRE =====
FROM eclipse-temurin:17-jdk-jammy AS jlink

# Create minimal JRE with only needed modules
RUN jlink \
    --add-modules java.base,java.logging,java.xml,java.sql,java.management,java.net.http \
    --strip-debug \
    --no-man-pages \
    --no-header-files \
    --compress=2 \
    --output /opt/minimal-jre

# ===== Final Stage =====
FROM debian:bookworm-slim AS final

# Copy custom JRE
COPY --from=jlink /opt/minimal-jre /opt/jre

ENV PATH="/opt/jre/bin:$PATH"
ENV JAVA_HOME="/opt/jre"

# Non-root user
RUN groupadd -r app && useradd --no-log-init -r -g app app
WORKDIR /app

COPY --from=build /app/target/scala-2.13/*-assembly-*.jar app.jar
RUN chown -R app:app /app

USER app

EXPOSE 8080

# JVM tuning for containers
ENTRYPOINT ["java", \
  "-server", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:InitialRAMPercentage=50.0", \
  "-XX:+UseG1GC", \
  "-XX:G1HeapRegionSize=16m", \
  "-XX:+ExitOnOutOfMemoryError", \
  "-jar", "app.jar"]
```

---

## Step 825: Docker Build Script

```bash
#!/bin/bash
# scripts/docker-build.sh

set -euo pipefail

APP_NAME="scala-microservice"
REGISTRY="registry.mycompany.com"
VERSION=$(cat version.sbt | grep 'version :=' | sed 's/.*"\(.*\)".*/\1/')
GIT_SHA=$(git rev-parse --short HEAD)

echo "Building $APP_NAME version $VERSION (git: $GIT_SHA)"

# Build with multiple tags
docker build \
  --target runtime \
  --tag "$APP_NAME:latest" \
  --tag "$APP_NAME:$VERSION" \
  --tag "$APP_NAME:$GIT_SHA" \
  --tag "$REGISTRY/$APP_NAME:$VERSION" \
  --build-arg VERSION="$VERSION" \
  --build-arg GIT_SHA="$GIT_SHA" \
  --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  --cache-from "$REGISTRY/$APP_NAME:latest" \
  .

echo "Build successful!"

# Security scan
if command -v trivy &> /dev/null; then
  echo "Running security scan..."
  trivy image --exit-code 1 --severity HIGH,CRITICAL "$APP_NAME:latest"
fi

# Push to registry (if --push flag)
if [ "${1:-}" = "--push" ]; then
  echo "Pushing to registry..."
  docker push "$REGISTRY/$APP_NAME:$VERSION"
  docker push "$REGISTRY/$APP_NAME:latest"
  echo "Pushed successfully!"
fi
```

---

## Step 826: Environment Variables และ Secrets

```scala
// ConfigFromEnvironment.scala
import com.typesafe.config.{Config, ConfigFactory}
import scala.util.{Try, Success, Failure}

// ===== Environment-aware Config =====
object AppConfig {
  
  private val config: Config = {
    val env = sys.env.getOrElse("APP_ENV", "development")
    
    ConfigFactory.systemEnvironment()
      .withFallback(ConfigFactory.parseResources(s"application-$env.conf"))
      .withFallback(ConfigFactory.load())
      .resolve()
  }
  
  // Database
  val dbUrl: String     = config.getString("database.url")
  val dbUser: String    = config.getString("database.user")
  val dbPassword: String = config.getString("database.password")
  
  // Kafka
  val kafkaBrokers: String = config.getString("kafka.brokers")
  
  // Feature flags
  val enableNewFeature: Boolean = config.getBoolean("features.enable-new-feature")
  
  // Server
  val serverPort: Int = config.getInt("server.port")
  
  // ===== Validate config on startup =====
  def validate(): Either[List[String], Unit] = {
    val errors = scala.collection.mutable.ListBuffer[String]()
    
    if (dbUrl.isEmpty)       errors += "DATABASE_URL is required"
    if (dbUser.isEmpty)      errors += "DATABASE_USER is required"
    if (kafkaBrokers.isEmpty) errors += "KAFKA_BROKERS is required"
    
    if (errors.nonEmpty) Left(errors.toList)
    else                 Right(())
  }
}

// ===== Secrets Management =====
// ใน production ใช้ AWS Secrets Manager, HashiCorp Vault, ฯลฯ

import software.amazon.awssdk.services.secretsmanager.SecretsManagerClient
import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueRequest

object SecretsManager {
  
  private lazy val client = SecretsManagerClient.builder()
    .region(software.amazon.awssdk.regions.Region.AP_SOUTHEAST_1)
    .build()
  
  def getSecret(secretName: String): String = {
    val request = GetSecretValueRequest.builder()
      .secretId(secretName)
      .build()
    
    client.getSecretValue(request).secretString()
  }
  
  // Database credentials from Secrets Manager
  def getDatabaseCredentials(): (String, String) = {
    import spray.json._
    import DefaultJsonProtocol._
    
    val secretJson = getSecret("myapp/database/credentials")
    val parsed = secretJson.parseJson.asJsObject
    
    val username = parsed.fields("username").convertTo[String]
    val password = parsed.fields("password").convertTo[String]
    
    (username, password)
  }
}
```

---

## Step 827: Docker Health Checks

```scala
// HealthCheck.scala สำหรับ Docker HEALTHCHECK
import akka.actor.typed.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.model._
import spray.json._
import spray.json.DefaultJsonProtocol._
import scala.concurrent.{Future, ExecutionContext}

case class HealthResponse(
  status: String,
  version: String,
  uptime: Long,
  checks: Map[String, String]
)

object HealthResponse {
  implicit val format: RootJsonFormat[HealthResponse] = jsonFormat4(HealthResponse)
}

class HealthRoutes()(implicit system: ActorSystem[_], ec: ExecutionContext) {
  
  private val startTime = System.currentTimeMillis()
  
  val routes =
    pathPrefix("health") {
      concat(
        // Liveness: is the app running?
        pathEnd {
          get {
            complete(HealthResponse(
              status  = "healthy",
              version = "1.0.0",
              uptime  = (System.currentTimeMillis() - startTime) / 1000,
              checks  = Map("status" -> "ok")
            ).toJson.toString)
          }
        },
        
        // Readiness: is the app ready to serve traffic?
        path("ready") {
          get {
            onSuccess(checkReadiness()) { ready =>
              if (ready) complete("""{"status":"ready"}""")
              else       complete(StatusCodes.ServiceUnavailable, """{"status":"not ready"}""")
            }
          }
        },
        
        // Liveness probe (minimal)
        path("live") {
          get {
            complete("""{"status":"alive"}""")
          }
        }
      )
    }
  
  private def checkReadiness(): Future[Boolean] = {
    // Check database connection, Kafka, etc.
    Future.successful(true) // simplified
  }
}
```

---

## Step 828: Container Resource Limits

```scala
// JVM tuning สำหรับ container resource limits

object ContainerJVMConfig {
  
  /*
  ===== JVM Container Settings =====
  
  1. UseContainerSupport (Java 11+):
     JVM reads cgroup limits แทน host memory
     
  2. MaxRAMPercentage:
     ใช้ 75% ของ container memory limit
     Container: 512MB → JVM heap: ~384MB
     
  3. InitialRAMPercentage:
     Start JVM with 50% of limit
     ลด startup time, ค่อยๆ grow
     
  4. ExitOnOutOfMemoryError:
     Container restart แทน hung process
     
  ===== Example Resource Limits =====
  
  Kubernetes:
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
  
  JVM Flags:
  -XX:+UseContainerSupport
  -XX:MaxRAMPercentage=75.0
  -XX:InitialRAMPercentage=50.0
  -XX:+ExitOnOutOfMemoryError
  -XX:+CrashOnOutOfMemoryError
  
  ===== GC Tuning by Container Size =====
  
  < 512MB: -XX:+UseSerialGC (single-threaded, less overhead)
  512MB-4GB: -XX:+UseG1GC (default, balanced)
  > 4GB: -XX:+UseZGC (low-latency, Java 15+)
  
  ===== Thread Pool Sizing =====
  
  IO threads = availableProcessors * 2
  CPU threads = availableProcessors
  
  In container: docker --cpus=2 → availableProcessors = 2
  */
  
  def printJVMInfo(): Unit = {
    val runtime = Runtime.getRuntime
    println(s"Available processors: ${runtime.availableProcessors()}")
    println(s"Max memory: ${runtime.maxMemory() / 1024 / 1024}MB")
    println(s"Total memory: ${runtime.totalMemory() / 1024 / 1024}MB")
    println(s"Free memory: ${runtime.freeMemory() / 1024 / 1024}MB")
    
    val memBean = java.lang.management.ManagementFactory.getMemoryMXBean
    val heapUsage = memBean.getHeapMemoryUsage
    println(s"Heap used: ${heapUsage.getUsed / 1024 / 1024}MB")
    println(s"Heap max: ${heapUsage.getMax / 1024 / 1024}MB")
  }
  
  def main(args: Array[String]): Unit = printJVMInfo()
}
```

---

## Step 829: Docker Networking

```yaml
# docker-compose-network.yml
version: '3.8'

networks:
  # Frontend network: Nginx → App
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24

  # Backend network: App ↔ Database, Kafka
  backend:
    driver: bridge
    internal: true  # ไม่สามารถเข้าถึงจากภายนอก
    ipam:
      config:
        - subnet: 172.21.0.0/24

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - frontend  # เข้าถึงได้จาก internet

  app:
    build: .
    networks:
      - frontend   # Nginx เรียกได้
      - backend    # เข้า database ได้
    # ไม่ expose port ออกไปข้างนอก

  postgres:
    image: postgres:15
    networks:
      - backend  # ปลอดภัย: ไม่เปิด public
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    networks:
      - backend

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

---

## Step 830: CI/CD Docker Pipeline

```yaml
# .github/workflows/docker.yml
name: Build and Push Docker Image

on:
  push:
    branches: [main, develop]
    tags: ['v*']
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'sbt'

      - name: Run Tests
        run: sbt test

      - name: Log in to Registry
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Security Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
```

---

## สรุป Part 83: Docker and Scala

| Topic | Key Points |
|-------|-----------|
| Multi-stage Build | Builder → Runtime (เล็กลง 5-10x) |
| sbt-native-packager | `sbt docker:publishLocal` |
| Resource Limits | `-XX:+UseContainerSupport` |
| Health Checks | `/health`, `/ready`, `/live` |
| Secrets | ENV vars, Secrets Manager, Docker secrets |
| Networking | Frontend/backend network separation |
| CI/CD | GitHub Actions + Docker layer cache |

### Image Size Comparison

| Base Image | Size |
|-----------|------|
| openjdk:17 | ~470MB |
| eclipse-temurin:17-jre-jammy | ~230MB |
| eclipse-temurin:17-jre-alpine | ~145MB |
| Custom JRE (jlink) | ~80MB |

---

## แบบฝึกหัด Part 83

1. **Multi-stage Dockerfile**: สร้าง Dockerfile ที่มี 3 stages (deps, build, runtime) โดย final image < 200MB และใช้ non-root user

2. **sbt-native-packager**: configure sbt-native-packager สำหรับ Docker ที่มี custom base image, labels, health check, และ JVM options

3. **Docker Compose Dev**: สร้าง docker-compose.yml สำหรับ development environment ที่มี app, PostgreSQL, Kafka, Redis, Prometheus, Grafana

4. **Container Optimization**: profile และ optimize Docker image โดยใช้ `docker history` และ `dive` เพื่อ reduce layers และ size

5. **CI/CD Pipeline**: สร้าง GitHub Actions workflow ที่ test, build, scan, และ push Docker image ไปยัง registry โดยมี layer caching

---

## ไปต่อ: Part 84 — Kubernetes
ใน Part ถัดไปจะเรียน Kubernetes deployment, ConfigMap, Secrets, HPA

[→ Part 84: Kubernetes](./part-84-kubernetes.md)
