# Part 89: CI/CD — Steps 881-890

## บทนำ: CI/CD สำหรับ Scala Projects

Continuous Integration (CI) และ Continuous Deployment (CD) ช่วยให้ deliver software ได้เร็วขึ้น ปลอดภัยขึ้น โดยอัตโนมัติ

---

## Step 881: GitHub Actions สำหรับ Scala

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop, 'feature/**', 'hotfix/**']
  pull_request:
    branches: [main, develop]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  SCALA_VERSION: "2.13.12"
  SBT_OPTS: "-Xmx2g -XX:+UseG1GC -XX:MaxMetaspaceSize=512m"

jobs:
  # ===== Compile =====
  compile:
    name: Compile
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Cache sbt
        uses: actions/cache@v3
        with:
          path: |
            ~/.sbt
            ~/.ivy2/cache
            ~/.coursier
          key: ${{ runner.os }}-sbt-${{ hashFiles('**/*.sbt', 'project/**') }}
          restore-keys: ${{ runner.os }}-sbt-
      
      - name: Compile
        run: sbt compile
  
  # ===== Unit Tests =====
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: [compile]
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Cache sbt
        uses: actions/cache@v3
        with:
          path: |
            ~/.sbt
            ~/.ivy2/cache
            ~/.coursier
          key: ${{ runner.os }}-sbt-${{ hashFiles('**/*.sbt', 'project/**') }}
      
      - name: Run Unit Tests
        env:
          DB_URL: jdbc:postgresql://localhost:5432/test
          DB_USER: test
          DB_PASSWORD: test
        run: sbt test
      
      - name: Upload Test Results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-results
          path: target/test-reports/
      
      - name: Test Coverage Report
        run: sbt coverageReport
      
      - name: Upload Coverage
        uses: codecov/codecov-action@v3
        with:
          files: target/scala-2.13/coverage-report/cobertura.xml
  
  # ===== Integration Tests =====
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: [compile]
    
    services:
      kafka:
        image: confluentinc/cp-kafka:7.5.0
        env:
          KAFKA_BROKER_ID: 1
          KAFKA_ZOOKEEPER_CONNECT: localhost:2181
        ports:
          - 9092:9092
    
    steps:
      - uses: actions/checkout@v4
      - name: Setup JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Run Integration Tests
        run: sbt "testOnly **.*IntegrationTest"
  
  # ===== Code Quality =====
  code-quality:
    name: Code Quality
    runs-on: ubuntu-latest
    needs: [compile]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Scalafmt Check
        run: sbt scalafmtCheckAll
      
      - name: Scalafix Lint
        run: sbt "scalafix --check"
      
      - name: Dependency Check
        run: sbt dependencyCheck
  
  # ===== Security Scan =====
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: [compile]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

---

## Step 882: CD Pipeline (Deploy)

```yaml
# .github/workflows/cd.yml
name: CD Pipeline

on:
  push:
    tags:
      - 'v*'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deploy to environment'
        required: true
        type: choice
        options:
          - staging
          - production
      version:
        description: 'Version to deploy'
        required: false

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ===== Build Docker Image =====
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    outputs:
      version: ${{ steps.version.outputs.version }}
      image: ${{ steps.meta.outputs.tags }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Extract version
        id: version
        run: |
          if [[ "${{ github.ref }}" == refs/tags/* ]]; then
            VERSION=${GITHUB_REF#refs/tags/v}
          else
            VERSION=$(date +%Y%m%d)-${GITHUB_SHA::8}
          fi
          echo "version=$VERSION" >> $GITHUB_OUTPUT
      
      - name: Setup JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Run Tests
        run: sbt test
      
      - name: Build JAR
        run: sbt assembly
      
      - name: Log in to Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}
      
      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            VERSION=${{ steps.version.outputs.version }}
            GIT_SHA=${{ github.sha }}
      
      - name: Security Scan Image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
  
  # ===== Deploy to Staging =====
  deploy-staging:
    name: Deploy to Staging
    needs: [build]
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.28.0'
      
      - name: Configure kubeconfig
        run: |
          echo "${{ secrets.KUBECONFIG_STAGING }}" | base64 -d > kubeconfig
          export KUBECONFIG=./kubeconfig
      
      - name: Deploy to staging
        env:
          KUBECONFIG: ./kubeconfig
          IMAGE: ${{ needs.build.outputs.image }}
          VERSION: ${{ needs.build.outputs.version }}
        run: |
          kubectl set image deployment/order-service \
            order-service=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:$VERSION \
            -n scala-app-staging
          
          kubectl rollout status deployment/order-service \
            -n scala-app-staging \
            --timeout=300s
      
      - name: Run Smoke Tests
        run: |
          curl -f https://staging.myapp.com/health || exit 1
          curl -f https://staging.myapp.com/api/v1/products?limit=1 || exit 1
  
  # ===== Deploy to Production =====
  deploy-production:
    name: Deploy to Production
    needs: [build, deploy-staging]
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure kubectl
        run: echo "${{ secrets.KUBECONFIG_PROD }}" | base64 -d > kubeconfig
      
      - name: Deploy (Blue-Green)
        env:
          KUBECONFIG: ./kubeconfig
          VERSION: ${{ needs.build.outputs.version }}
        run: |
          # Update deployment
          kubectl set image deployment/order-service \
            order-service=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:$VERSION \
            -n scala-app
          
          # Wait for rollout
          kubectl rollout status deployment/order-service \
            -n scala-app --timeout=600s
      
      - name: Verify Deployment
        run: |
          # Check health
          curl -f https://myapp.com/health
          
          # Check error rate didn't spike (via Prometheus API)
          ERROR_RATE=$(curl -s "https://prometheus.internal/api/v1/query?query=rate(http_requests_total{status=~'5..',job='order-service'}[2m])/rate(http_requests_total{job='order-service'}[2m])" | jq '.data.result[0].value[1]' | tr -d '"')
          
          if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
            echo "Error rate too high: $ERROR_RATE"
            kubectl rollout undo deployment/order-service -n scala-app
            exit 1
          fi
          
          echo "Deployment verified!"
      
      - name: Create GitHub Release
        if: startsWith(github.ref, 'refs/tags/')
        uses: softprops/action-gh-release@v1
        with:
          generate_release_notes: true
          files: |
            target/scala-2.13/*.jar
```

---

## Step 883: Semantic Versioning

```scala
// version.sbt
version := "1.2.3"  // major.minor.patch

// project/plugins.sbt
addSbtPlugin("com.github.sbt" % "sbt-release" % "1.1.0")

// release.sbt (custom release process)
releaseProcess := Seq[ReleaseStep](
  checkSnapshotDependencies,
  inquireVersions,
  runClean,
  runTest,
  setReleaseVersion,
  commitReleaseVersion,
  tagRelease,
  publishArtifacts,
  setNextVersion,
  commitNextVersion,
  pushChanges
)
```

```bash
#!/bin/bash
# scripts/release.sh
# Semantic versioning release automation

set -euo pipefail

CURRENT_VERSION=$(cat version.sbt | grep 'version :=' | sed 's/.*"\(.*\)".*/\1/' | sed 's/-SNAPSHOT//')
TYPE="${1:-patch}"  # major | minor | patch

echo "Current version: $CURRENT_VERSION"

IFS='.' read -r -a parts <<< "$CURRENT_VERSION"
MAJOR="${parts[0]}"
MINOR="${parts[1]}"
PATCH="${parts[2]}"

case "$TYPE" in
  major) NEW_VERSION="$((MAJOR + 1)).0.0" ;;
  minor) NEW_VERSION="${MAJOR}.$((MINOR + 1)).0" ;;
  patch) NEW_VERSION="${MAJOR}.${MINOR}.$((PATCH + 1))" ;;
  *) echo "Unknown type: $TYPE"; exit 1 ;;
esac

echo "New version: $NEW_VERSION"

# Update version files
sed -i "s/version := \"$CURRENT_VERSION\"/version := \"$NEW_VERSION\"/" version.sbt

# Commit and tag
git add version.sbt
git commit -m "chore: bump version to $NEW_VERSION"
git tag "v$NEW_VERSION"

echo "Released v$NEW_VERSION"
echo "Run: git push && git push --tags"
```

---

## Step 884: sbt Release Plugin

```scala
// project/plugins.sbt
addSbtPlugin("com.github.sbt"   % "sbt-release"  % "1.1.0")
addSbtPlugin("org.scalameta"    %% "sbt-scalafmt" % "2.5.2")
addSbtPlugin("ch.epfl.scala"    % "sbt-scalafix"  % "0.11.1")
addSbtPlugin("org.scoverage"    % "sbt-scoverage" % "2.0.9")
addSbtPlugin("net.vonbuchholtz" % "sbt-dependency-check" % "5.1.0")

// build.sbt additions
import sbtrelease.ReleaseStateTransformations._

releaseIgnoreUntrackedFiles := true

releaseProcess := Seq[ReleaseStep](
  checkSnapshotDependencies,
  inquireVersions,
  runClean,
  runTest,
  setReleaseVersion,
  commitReleaseVersion,
  tagRelease,
  ReleaseStep(releaseStepCommand("publishSigned")),
  ReleaseStep(releaseStepCommand("sonatypeBundleRelease")),
  setNextVersion,
  commitNextVersion,
  pushChanges
)

// Scala version cross compilation
crossScalaVersions := Seq("2.13.12", "3.3.1")
```

---

## Step 885: sbt Test Configuration

```scala
// build.sbt สำหรับ test setup
libraryDependencies ++= Seq(
  "org.scalatest"          %% "scalatest"          % "3.2.17" % Test,
  "org.scalatestplus"      %% "scalacheck-1-17"    % "3.2.17.0" % Test,
  "org.scalamock"          %% "scalamock"          % "5.2.0"  % Test,
  "com.typesafe.akka"      %% "akka-testkit"       % "2.9.2"  % Test,
  "com.typesafe.akka"      %% "akka-http-testkit"  % "10.5.3" % Test,
  "com.github.tomakehurst" % "wiremock-jre8"       % "2.35.1" % Test,
  "io.github.embeddedkafka" %% "embedded-kafka"    % "3.6.1"  % Test,
  "com.dimafeng"           %% "testcontainers-scala-postgresql" % "0.41.3" % Test
)

// Test settings
Test / fork             := true
Test / parallelExecution := false

// JVM options for tests
Test / javaOptions ++= Seq(
  "-Xmx2g",
  "-XX:+UseG1GC",
  "-Dconfig.resource=test.conf"
)

// Coverage
coverageEnabled          := true
coverageMinimumStmtTotal := 70
coverageFailOnMinimum    := true
coverageExcludedPackages := ".*Main.*"
```

```scala
// ExampleTest.scala
import org.scalatest.flatspec.AsyncFlatSpec
import org.scalatest.matchers.should.Matchers
import org.scalatest.BeforeAndAfterAll
import com.github.tomakehurst.wiremock.WireMockServer
import com.github.tomakehurst.wiremock.client.WireMock._
import scala.concurrent.Future

class OrderServiceTest extends AsyncFlatSpec with Matchers with BeforeAndAfterAll {
  
  // WireMock สำหรับ mock HTTP services
  private val wireMock = new WireMockServer(9999)
  
  override def beforeAll(): Unit = {
    wireMock.start()
    configureFor("localhost", 9999)
    
    // Mock user service
    stubFor(get(urlEqualTo("/api/v1/users/C001"))
      .willReturn(aResponse()
        .withStatus(200)
        .withHeader("Content-Type", "application/json")
        .withBody("""{"id":"C001","name":"Alice","status":"active"}""")))
  }
  
  override def afterAll(): Unit = wireMock.stop()
  
  "OrderService" should "create order for valid customer" in {
    val service = new OrderService(userServiceUrl = "http://localhost:9999")
    
    service.createOrder("C001", List(("P001", 2, 99.99))).map { result =>
      result.orderId should not be empty
      result.status shouldBe "pending"
    }
  }
  
  it should "reject order for unknown customer" in {
    stubFor(get(urlEqualTo("/api/v1/users/UNKNOWN"))
      .willReturn(aResponse().withStatus(404)))
    
    recoverToSucceededIf[Exception] {
      new OrderService("http://localhost:9999")
        .createOrder("UNKNOWN", List(("P001", 1, 10.0)))
    }
  }
}
```

---

## Step 886: Testcontainers

```scala
// TestcontainersSpec.scala
import com.dimafeng.testcontainers.{DockerComposeContainer, ExposedService}
import com.dimafeng.testcontainers.scalatest.TestContainerForAll
import org.scalatest.flatspec.AsyncFlatSpec
import org.scalatest.matchers.should.Matchers
import java.io.File

class DatabaseIntegrationTest extends AsyncFlatSpec with Matchers with TestContainerForAll {
  
  override val containerDef = DockerComposeContainer.Def(
    composeFiles = new File("src/test/resources/docker-compose-test.yml"),
    exposedServices = Seq(
      ExposedService("postgres_1", 5432),
      ExposedService("redis_1", 6379)
    )
  )
  
  "OrderRepository" should "persist and retrieve orders" in withContainers { containers =>
    val pgHost = containers.getServiceHost("postgres_1", 5432)
    val pgPort = containers.getServicePort("postgres_1", 5432)
    
    val dbUrl = s"jdbc:postgresql://$pgHost:$pgPort/test"
    
    val repo = new OrderRepository(dbUrl, "test", "test")
    
    for {
      orderId <- repo.create(OrderRecord("O001", "C001", "pending", 99.99))
      found   <- repo.findById(orderId)
    } yield {
      found.isDefined shouldBe true
      found.get.customerId shouldBe "C001"
    }
  }
}
```

---

## Step 887: ArgoCD Deployment

```yaml
# argocd-application.yaml
# GitOps: ArgoCD watches Git repo and syncs to K8s

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/mycompany/k8s-manifests
    targetRevision: HEAD
    path: services/order-service/overlays/production
  
  destination:
    server: https://kubernetes.default.svc
    namespace: scala-app
  
  syncPolicy:
    automated:
      prune: true      # delete resources removed from Git
      selfHeal: true   # revert manual changes
      allowEmpty: false
    
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - ApplyOutOfSyncOnly=true
    
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  revisionHistoryLimit: 10

---
# kustomization.yaml (production overlay)
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: scala-app

resources:
  - ../../base
  - hpa.yaml
  - pdb.yaml

patches:
  - path: deployment-patch.yaml
    target:
      kind: Deployment
      name: order-service

images:
  - name: order-service
    newName: ghcr.io/mycompany/order-service
    newTag: v1.2.3
```

---

## Step 888: Feature Flags

```scala
// FeatureFlags.scala
import com.typesafe.config.ConfigFactory

class FeatureFlags {
  private val config = ConfigFactory.load()
  
  def isEnabled(flagName: String): Boolean = {
    val envVar = s"FEATURE_${flagName.toUpperCase.replace('-', '_')}"
    sys.env.get(envVar).map(_.toBoolean)
      .getOrElse(
        Try(config.getBoolean(s"features.$flagName")).getOrElse(false)
      )
  }
}

object FeatureFlags extends FeatureFlags {
  val NEW_CHECKOUT   = "new-checkout"
  val REAL_TIME_SYNC = "real-time-sync"
  val AI_RECOMMENDATIONS = "ai-recommendations"
}

// Usage in routes
def checkoutRoute: Route = {
  if (FeatureFlags.isEnabled(FeatureFlags.NEW_CHECKOUT)) {
    newCheckoutHandler
  } else {
    legacyCheckoutHandler
  }
}

// LaunchDarkly integration (production feature flags)
/*
libraryDependencies += "com.launchdarkly" % "launchdarkly-java-server-sdk" % "7.2.3"

import com.launchdarkly.sdk._
import com.launchdarkly.sdk.server._

val ldClient = new LDClient(sys.env("LAUNCHDARKLY_SDK_KEY"))

def isFeatureEnabled(key: String, userId: String): Boolean = {
  val user = LDContext.builder(userId).build()
  ldClient.boolVariation(key, user, false)
}
*/
```

---

## Step 889: Database Migration

```scala
// build.sbt
libraryDependencies += "org.flywaydb" % "flyway-core"              % "9.22.3"
libraryDependencies += "org.flywaydb" % "flyway-database-postgresql" % "9.22.3"

// src/main/resources/db/migration/
// V1__create_orders_table.sql
// V2__add_order_index.sql
// V3__add_audit_columns.sql
```

```sql
-- V1__create_orders_table.sql
CREATE TABLE orders (
  id          VARCHAR(36)    PRIMARY KEY,
  customer_id VARCHAR(36)    NOT NULL,
  status      VARCHAR(20)    NOT NULL DEFAULT 'pending',
  amount      DECIMAL(10, 2) NOT NULL,
  currency    VARCHAR(3)     NOT NULL DEFAULT 'USD',
  created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- V2__add_order_items.sql
CREATE TABLE order_items (
  id         VARCHAR(36)    PRIMARY KEY,
  order_id   VARCHAR(36)    NOT NULL REFERENCES orders(id),
  product_id VARCHAR(36)    NOT NULL,
  quantity   INTEGER        NOT NULL,
  unit_price DECIMAL(10, 2) NOT NULL,
  total      DECIMAL(10, 2) GENERATED ALWAYS AS (quantity * unit_price) STORED
);

-- V3__add_audit_columns.sql
ALTER TABLE orders 
  ADD COLUMN created_by VARCHAR(36),
  ADD COLUMN updated_by VARCHAR(36),
  ADD COLUMN version INTEGER NOT NULL DEFAULT 1;
```

```scala
// FlywayMigration.scala
import org.flywaydb.core.Flyway

object FlywayMigration {
  
  def migrate(dbUrl: String, user: String, password: String): Unit = {
    val flyway = Flyway.configure()
      .dataSource(dbUrl, user, password)
      .locations("classpath:db/migration")
      .baselineOnMigrate(true)  // เพิ่มสำหรับ existing DB
      .validateMigrationNaming(true)
      .load()
    
    val result = flyway.migrate()
    println(s"Applied ${result.migrationsExecuted} migrations")
  }
  
  def validate(dbUrl: String, user: String, password: String): Unit = {
    val flyway = Flyway.configure()
      .dataSource(dbUrl, user, password)
      .load()
    
    flyway.validate()  // throws if migrations don't match
  }
}
```

---

## Step 890: Production Deployment Checklist

```markdown
# Pre-Deployment Checklist

## Code Review
- [ ] PR approved by 2+ engineers
- [ ] No failing tests
- [ ] Coverage > 70%
- [ ] No critical security vulnerabilities (Trivy scan)
- [ ] API backwards compatible (or migration plan)
- [ ] Database migrations tested

## Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Load testing results reviewed (P99 < SLO)
- [ ] Smoke tests on staging pass

## Infrastructure
- [ ] Database migrations ready (rollback script)
- [ ] Feature flags configured
- [ ] ConfigMap/Secret changes reviewed
- [ ] Resource limits appropriate for new load

## Monitoring
- [ ] Dashboards show healthy baselines
- [ ] Alerts configured for new features
- [ ] Runbook updated
- [ ] On-call informed of deployment

## Rollback Plan
- [ ] Docker image tagged with specific version
- [ ] Previous version known and ready
- [ ] Database migration rollback script ready
- [ ] Rollback procedure documented and tested

## Post-Deployment
- [ ] Monitor error rate for 30 min
- [ ] Monitor latency P99 for 30 min
- [ ] Verify key business metrics
- [ ] Update incident response runbook if needed
```

---

## สรุป Part 89: CI/CD

| Stage | Tool | Purpose |
|-------|------|---------|
| CI | GitHub Actions | Build, test, lint |
| Artifacts | Docker + GHCR | Container registry |
| Security | Trivy | Vulnerability scan |
| Deploy | kubectl/ArgoCD | K8s deployment |
| Migrations | Flyway | DB schema changes |
| Feature Flags | LaunchDarkly | Safe rollout |
| GitOps | ArgoCD | Declarative sync |

### Deployment Strategies

| Strategy | Risk | Speed | Use Case |
|---------|------|-------|----------|
| Recreate | High | Fast | Dev only |
| Rolling | Medium | Medium | Most services |
| Blue-Green | Low | Slow | Critical services |
| Canary | Very Low | Very Slow | High-risk changes |

---

## แบบฝึกหัด Part 89

1. **Full CI Pipeline**: สร้าง GitHub Actions workflow ที่มี compile, test, lint, coverage, security scan ทำงานใน parallel

2. **CD with ArgoCD**: setup GitOps pipeline ด้วย ArgoCD ที่ auto-deploy เมื่อ image tag update ใน Git

3. **Database Migration**: implement Flyway migrations ที่มี forward migrations, rollback scripts, และ test ใน CI

4. **Feature Flags**: implement feature flag system ที่ support percentage rollouts, user targeting, และ environment-specific flags

5. **Deployment Metrics**: สร้าง script ที่ verify deployment success โดย check error rate, latency, และ key metrics ใน Prometheus

---

## ไปต่อ: Part 90 — API Gateway
ใน Part ถัดไปจะเรียน API Gateway patterns, Nginx, Traefik, service discovery

[→ Part 90: API Gateway](./part-90-api-gateway.md)
