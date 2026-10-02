# Part 88: Observability — Steps 871-880

## บทนำ: Observability

Observability คือความสามารถในการเข้าใจ state ของ system จากภายนอก โดยดูจาก 3 pillars: Logs, Metrics, Traces

---

## Step 871: Structured Logging with Logback

```scala
// build.sbt
libraryDependencies ++= Seq(
  "ch.qos.logback"        % "logback-classic"         % "1.4.14",
  "net.logstash.logback"  % "logstash-logback-encoder" % "7.4",
  "com.typesafe.scala-logging" %% "scala-logging"      % "3.9.5"
)
```

```xml
<!-- logback.xml -->
<configuration>
  <statusListener class="ch.qos.logback.core.status.NopStatusListener" />
  
  <!-- Console appender for development -->
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
      <providers>
        <timestamp>
          <fieldName>timestamp</fieldName>
          <pattern>yyyy-MM-dd'T'HH:mm:ss.SSS'Z'</pattern>
        </timestamp>
        <logLevel>
          <fieldName>level</fieldName>
        </logLevel>
        <loggerName>
          <fieldName>logger</fieldName>
        </loggerName>
        <message />
        <mdc />
        <arguments />
        <stackTrace>
          <fieldName>stackTrace</fieldName>
        </stackTrace>
      </providers>
    </encoder>
  </appender>

  <!-- File appender for production -->
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>/var/log/app/application.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <fileNamePattern>/var/log/app/application.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
      <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
        <maxFileSize>100MB</maxFileSize>
      </timeBasedFileNamingAndTriggeringPolicy>
      <maxHistory>30</maxHistory>
      <totalSizeCap>5GB</totalSizeCap>
    </rollingPolicy>
    <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
      <providers>
        <timestamp><fieldName>timestamp</fieldName></timestamp>
        <logLevel><fieldName>level</fieldName></logLevel>
        <loggerName><fieldName>logger</fieldName></loggerName>
        <message />
        <mdc />
        <arguments />
        <stackTrace />
      </providers>
    </encoder>
  </appender>

  <root level="INFO">
    <appender-ref ref="${LOG_APPENDER:-STDOUT}" />
  </root>
  
  <!-- Quieter loggers -->
  <logger name="akka" level="WARN" />
  <logger name="io.netty" level="WARN" />
  <logger name="org.apache.kafka" level="WARN" />
</configuration>
```

```scala
// StructuredLogging.scala
import com.typesafe.scalalogging.Logger
import org.slf4j.MDC

object StructuredLogging {
  
  private val logger = Logger("com.example.OrderService")
  
  // ===== Basic structured logging =====
  def logOrderCreated(orderId: String, customerId: String, amount: Double): Unit = {
    // MDC (Mapped Diagnostic Context) สำหรับ thread-local context
    MDC.put("orderId", orderId)
    MDC.put("customerId", customerId)
    MDC.put("service", "order-service")
    
    try {
      logger.info(s"Order created", "orderId" -> orderId, "customerId" -> customerId, "amount" -> amount)
      // JSON output: {"timestamp":"...","level":"INFO","message":"Order created","orderId":"O001","customerId":"C001","amount":999.99}
    } finally {
      MDC.remove("orderId")
      MDC.remove("customerId")
    }
  }
  
  // ===== Correlation ID propagation =====
  case class RequestContext(
    requestId: String,
    traceId: String,
    userId: Option[String]
  )
  
  def withContext[T](ctx: RequestContext)(f: => T): T = {
    MDC.put("requestId", ctx.requestId)
    MDC.put("traceId", ctx.traceId)
    ctx.userId.foreach(MDC.put("userId", _))
    
    try {
      f
    } finally {
      MDC.remove("requestId")
      MDC.remove("traceId")
      MDC.remove("userId")
    }
  }
  
  // ===== Logging wrapper สำหรับ operations =====
  def logOperation[T](operationName: String, metadata: Map[String, Any] = Map.empty)(f: => T): T = {
    val start = System.currentTimeMillis()
    val metaStr = metadata.map { case (k, v) => s"$k=$v" }.mkString(", ")
    
    logger.debug(s"Starting $operationName [$metaStr]")
    
    try {
      val result = f
      val elapsed = System.currentTimeMillis() - start
      logger.info(s"Completed $operationName in ${elapsed}ms [$metaStr]")
      result
    } catch {
      case ex: Exception =>
        val elapsed = System.currentTimeMillis() - start
        logger.error(s"Failed $operationName in ${elapsed}ms [$metaStr]: ${ex.getMessage}", ex)
        throw ex
    }
  }
  
  def main(args: Array[String]): Unit = {
    val ctx = RequestContext(
      requestId = "req-12345",
      traceId   = "trace-abc-def",
      userId    = Some("user-999")
    )
    
    withContext(ctx) {
      logOperation("createOrder", Map("customerId" -> "C001", "amount" -> 999.99)) {
        Thread.sleep(50) // simulate work
        logOrderCreated("O001", "C001", 999.99)
        "success"
      }
    }
  }
}
```

---

## Step 872: Prometheus Metrics

```scala
// build.sbt
libraryDependencies ++= Seq(
  "io.prometheus" % "client"                  % "0.16.0",
  "io.prometheus" % "client_hotspot"          % "0.16.0",
  "io.prometheus" % "simpleclient_httpserver" % "0.16.0"
)
```

```scala
// PrometheusMetrics.scala
import io.prometheus.client._
import io.prometheus.client.hotspot.DefaultExports
import io.prometheus.client.exporter.HTTPServer

object Metrics {
  
  // ===== Counter =====
  val ordersCreated: Counter = Counter.build()
    .name("orders_created_total")
    .help("Total number of orders created")
    .labelNames("status", "category")
    .register()
  
  val paymentsFailed: Counter = Counter.build()
    .name("payments_failed_total")
    .help("Total payment failures")
    .labelNames("reason")
    .register()
  
  // ===== Gauge =====
  val activeOrders: Gauge = Gauge.build()
    .name("active_orders")
    .help("Current active orders count")
    .register()
  
  val inventoryLevel: Gauge = Gauge.build()
    .name("inventory_level")
    .help("Current inventory by product")
    .labelNames("product_id", "category")
    .register()
  
  // ===== Histogram =====
  val orderProcessingTime: Histogram = Histogram.build()
    .name("order_processing_seconds")
    .help("Order processing duration")
    .labelNames("operation")
    .buckets(0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0, 5.0)
    .register()
  
  val httpRequestDuration: Histogram = Histogram.build()
    .name("http_request_duration_seconds")
    .help("HTTP request duration")
    .labelNames("method", "path", "status")
    .buckets(0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10)
    .register()
  
  // ===== Summary =====
  val orderAmountSummary: Summary = Summary.build()
    .name("order_amount_dollars")
    .help("Order amount distribution")
    .labelNames("category")
    .quantile(0.5, 0.05)   // P50
    .quantile(0.9, 0.01)   // P90
    .quantile(0.99, 0.001) // P99
    .register()
  
  // ===== Initialize JVM metrics =====
  def init(): Unit = {
    DefaultExports.initialize()  // GC, heap, threads, etc.
  }
  
  // ===== Start metrics server =====
  def startMetricsServer(port: Int = 9095): HTTPServer = {
    init()
    new HTTPServer(port)
  }
}

// ===== Business Metrics Recording =====
object BusinessMetrics {
  import Metrics._
  
  def recordOrderCreated(category: String, amount: Double): Unit = {
    ordersCreated.labels("created", category).inc()
    activeOrders.inc()
    orderAmountSummary.labels(category).observe(amount)
  }
  
  def recordOrderCompleted(category: String): Unit = {
    ordersCreated.labels("completed", category).inc()
    activeOrders.dec()
  }
  
  def recordOrderFailed(category: String, reason: String): Unit = {
    ordersCreated.labels("failed", category).inc()
    activeOrders.dec()
    paymentsFailed.labels(reason).inc()
  }
  
  // Timing wrapper
  def timed[T](operation: String)(f: => T): T = {
    val timer = orderProcessingTime.labels(operation).startTimer()
    try {
      val result = f
      result
    } finally {
      timer.observeDuration()
    }
  }
  
  def timedHTTP[T](method: String, path: String)(f: => (Int, T)): T = {
    val timer = httpRequestDuration.labels(method, path, "").startTimer()
    try {
      val (status, result) = f
      timer.observeDuration()
      result
    } catch {
      case ex: Exception =>
        timer.observeDuration()
        throw ex
    }
  }
  
  def updateInventory(productId: String, category: String, qty: Int): Unit = {
    inventoryLevel.labels(productId, category).set(qty.toDouble)
  }
}

// ===== Akka HTTP Middleware =====
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.server.Route

object MetricsMiddleware {
  
  def metricsRoute: Route =
    path("metrics") {
      get {
        import io.prometheus.client.exporter.common.TextFormat
        import java.io.StringWriter
        
        val writer = new StringWriter()
        TextFormat.write004(writer, CollectorRegistry.defaultRegistry.metricFamilySamples())
        complete(writer.toString)
      }
    }
  
  def withMetrics(route: Route): Route = {
    extractMethod { method =>
      extractUri { uri =>
        val path = uri.path.toString
        val start = System.currentTimeMillis()
        
        mapResponse { response =>
          val elapsed = (System.currentTimeMillis() - start) / 1000.0
          Metrics.httpRequestDuration
            .labels(method.value, path, response.status.intValue.toString)
            .observe(elapsed)
          response
        } {
          route
        }
      }
    }
  }
}

object MetricsDemo {
  def main(args: Array[String]): Unit = {
    // Start metrics server
    val metricsServer = Metrics.startMetricsServer(9095)
    
    println("Metrics server started on :9095")
    println("Visit: http://localhost:9095/metrics")
    
    // Simulate some business activity
    for (i <- 1 to 100) {
      val category = if (i % 3 == 0) "Electronics" else if (i % 3 == 1) "Clothing" else "Food"
      val amount = 100.0 + (i % 900)
      
      BusinessMetrics.timed("createOrder") {
        BusinessMetrics.recordOrderCreated(category, amount)
        Thread.sleep(10 + (i % 50))
      }
      
      if (i % 10 == 0) {
        BusinessMetrics.recordOrderFailed(category, "payment_declined")
      } else {
        BusinessMetrics.recordOrderCompleted(category)
      }
    }
    
    println("Done! Check metrics at http://localhost:9095/metrics")
    Thread.sleep(30000)
    metricsServer.stop()
  }
}
```

---

## Step 873: OpenTelemetry Tracing

```scala
// build.sbt
libraryDependencies ++= Seq(
  "io.opentelemetry"           % "opentelemetry-api"                      % "1.32.0",
  "io.opentelemetry"           % "opentelemetry-sdk"                      % "1.32.0",
  "io.opentelemetry"           % "opentelemetry-sdk-trace"                % "1.32.0",
  "io.opentelemetry"           % "opentelemetry-exporter-otlp"            % "1.32.0",
  "io.opentelemetry"           % "opentelemetry-exporter-jaeger"          % "1.28.0",
  "io.opentelemetry.instrumentation" % "opentelemetry-instrumentation-api" % "1.32.0-alpha"
)
```

```scala
// OpenTelemetrySetup.scala
import io.opentelemetry.api.GlobalOpenTelemetry
import io.opentelemetry.api.trace.{Span, SpanKind, StatusCode, Tracer}
import io.opentelemetry.context.Context
import io.opentelemetry.exporter.otlp.http.trace.OtlpHttpSpanExporter
import io.opentelemetry.sdk.OpenTelemetrySdk
import io.opentelemetry.sdk.resources.Resource
import io.opentelemetry.sdk.trace.SdkTracerProvider
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor
import io.opentelemetry.semconv.resource.attributes.ResourceAttributes
import io.opentelemetry.api.common.Attributes
import scala.concurrent.{ExecutionContext, Future}
import scala.util.{Try, Success, Failure}

object OpenTelemetrySetup {
  
  def initialize(serviceName: String, jaegerEndpoint: String): OpenTelemetrySdk = {
    // Resource attributes
    val resource = Resource.getDefault.merge(
      Resource.create(Attributes.of(
        ResourceAttributes.SERVICE_NAME, serviceName,
        ResourceAttributes.SERVICE_VERSION, "1.0.0",
        ResourceAttributes.DEPLOYMENT_ENVIRONMENT, "production"
      ))
    )
    
    // Exporter (to Jaeger/Tempo/Collector)
    val exporter = OtlpHttpSpanExporter.builder()
      .setEndpoint(jaegerEndpoint)
      .build()
    
    // Tracer provider
    val tracerProvider = SdkTracerProvider.builder()
      .setResource(resource)
      .addSpanProcessor(BatchSpanProcessor.builder(exporter).build())
      .build()
    
    // Build OpenTelemetry
    OpenTelemetrySdk.builder()
      .setTracerProvider(tracerProvider)
      .buildAndRegisterGlobal()
  }
  
  val tracer: Tracer = GlobalOpenTelemetry.getTracer("com.example.order-service")
}

// ===== Tracing Utilities =====
object Tracing {
  import OpenTelemetrySetup._
  
  // Trace a synchronous operation
  def trace[T](operationName: String, attributes: Map[String, String] = Map.empty)(f: Span => T): T = {
    val span = tracer.spanBuilder(operationName)
      .setSpanKind(SpanKind.INTERNAL)
      .startSpan()
    
    val scope = span.makeCurrent()
    
    try {
      attributes.foreach { case (k, v) => span.setAttribute(k, v) }
      val result = f(span)
      span.setStatus(StatusCode.OK)
      result
    } catch {
      case ex: Exception =>
        span.setStatus(StatusCode.ERROR, ex.getMessage)
        span.recordException(ex)
        throw ex
    } finally {
      scope.close()
      span.end()
    }
  }
  
  // Trace an async operation
  def traceAsync[T](operationName: String, attributes: Map[String, String] = Map.empty)(
    f: Span => Future[T]
  )(implicit ec: ExecutionContext): Future[T] = {
    
    val span = tracer.spanBuilder(operationName)
      .setSpanKind(SpanKind.INTERNAL)
      .startSpan()
    
    val scope = span.makeCurrent()
    
    try {
      attributes.foreach { case (k, v) => span.setAttribute(k, v) }
      
      f(span).transform(
        result => {
          span.setStatus(StatusCode.OK)
          scope.close()
          span.end()
          result
        },
        ex => {
          span.setStatus(StatusCode.ERROR, ex.getMessage)
          span.recordException(ex)
          scope.close()
          span.end()
          ex
        }
      )
    } catch {
      case ex: Exception =>
        span.setStatus(StatusCode.ERROR, ex.getMessage)
        scope.close()
        span.end()
        Future.failed(ex)
    }
  }
  
  // HTTP span (outgoing)
  def traceHTTP[T](method: String, url: String)(f: Span => T): T = {
    val span = tracer.spanBuilder(s"HTTP $method")
      .setSpanKind(SpanKind.CLIENT)
      .setAttribute("http.method", method)
      .setAttribute("http.url", url)
      .startSpan()
    
    val scope = span.makeCurrent()
    
    try {
      f(span)
    } finally {
      scope.close()
      span.end()
    }
  }
  
  // Database span
  def traceDB[T](dbSystem: String, operation: String, table: String)(f: => T): T = {
    val span = tracer.spanBuilder(s"$operation $table")
      .setSpanKind(SpanKind.CLIENT)
      .setAttribute("db.system", dbSystem)
      .setAttribute("db.operation", operation)
      .setAttribute("db.sql.table", table)
      .startSpan()
    
    val scope = span.makeCurrent()
    
    try {
      val result = f
      span.setStatus(StatusCode.OK)
      result
    } catch {
      case ex: Exception =>
        span.setStatus(StatusCode.ERROR, ex.getMessage)
        span.recordException(ex)
        throw ex
    } finally {
      scope.close()
      span.end()
    }
  }
}

// ===== Usage Example =====
class OrderServiceWithTracing(implicit ec: ExecutionContext) {
  import Tracing._
  
  def createOrder(customerId: String, amount: Double): Future[String] = {
    traceAsync("createOrder", Map("customerId" -> customerId, "amount" -> amount.toString)) { span =>
      
      // Validate customer (child span)
      val validateFuture = traceAsync("validateCustomer") { _ =>
        Future.successful(true)
      }
      
      validateFuture.flatMap { valid =>
        if (!valid) Future.failed(new Exception("Invalid customer"))
        else {
          // Process payment (child span)
          traceAsync("processPayment", Map("amount" -> amount.toString)) { _ =>
            Thread.sleep(50)
            Future.successful("payment-123")
          }.flatMap { paymentId =>
            
            // Save to DB (child span)
            traceDB("postgresql", "INSERT", "orders") {
              Thread.sleep(20)
              val orderId = java.util.UUID.randomUUID().toString
              span.setAttribute("orderId", orderId)
              orderId
            }.let(Future.successful)
          }
        }
      }
    }
  }
}

// Extension method helper
implicit class AnyOps[A](val a: A) extends AnyVal {
  def let[B](f: A => B): B = f(a)
}
```

---

## Step 874: Grafana Dashboard Configuration

```json
// grafana-dashboard.json (simplified)
{
  "title": "Order Service Dashboard",
  "uid": "order-service-v1",
  "panels": [
    {
      "title": "Request Rate",
      "type": "stat",
      "targets": [
        {
          "expr": "sum(rate(http_request_duration_seconds_count{job=\"order-service\"}[5m]))",
          "legendFormat": "Requests/sec"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "stat",
      "targets": [
        {
          "expr": "sum(rate(http_request_duration_seconds_count{job=\"order-service\",status=~\"5..\"}[5m])) / sum(rate(http_request_duration_seconds_count{job=\"order-service\"}[5m])) * 100",
          "legendFormat": "Error %"
        }
      ]
    },
    {
      "title": "P99 Latency",
      "type": "stat",
      "targets": [
        {
          "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{job=\"order-service\"}[5m])) by (le)) * 1000",
          "legendFormat": "P99 ms"
        }
      ]
    },
    {
      "title": "Order Processing Time",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.50, rate(order_processing_seconds_bucket[5m]))",
          "legendFormat": "P50"
        },
        {
          "expr": "histogram_quantile(0.90, rate(order_processing_seconds_bucket[5m]))",
          "legendFormat": "P90"
        },
        {
          "expr": "histogram_quantile(0.99, rate(order_processing_seconds_bucket[5m]))",
          "legendFormat": "P99"
        }
      ]
    },
    {
      "title": "Active Orders",
      "type": "gauge",
      "targets": [
        {
          "expr": "active_orders",
          "legendFormat": "Active"
        }
      ]
    },
    {
      "title": "JVM Heap Usage",
      "type": "graph",
      "targets": [
        {
          "expr": "jvm_memory_bytes_used{area=\"heap\"} / jvm_memory_bytes_max{area=\"heap\"} * 100",
          "legendFormat": "Heap %"
        }
      ]
    }
  ]
}
```

---

## Step 875: Alerting Rules

```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: order-service-alerts
  namespace: monitoring
spec:
  groups:
    - name: order-service
      interval: 30s
      rules:
        # High Error Rate
        - alert: HighErrorRate
          expr: |
            sum(rate(http_request_duration_seconds_count{
              job="order-service", status=~"5.."
            }[5m])) / 
            sum(rate(http_request_duration_seconds_count{
              job="order-service"
            }[5m])) > 0.01
          for: 2m
          labels:
            severity: critical
            team: backend
          annotations:
            summary: "High error rate: {{ $value | humanizePercentage }}"
            description: "Order service error rate > 1% for 2 minutes"
            runbook: "https://wiki.company.com/runbooks/order-service-errors"

        # High Latency
        - alert: HighP99Latency
          expr: |
            histogram_quantile(0.99, 
              rate(order_processing_seconds_bucket[5m])
            ) > 2.0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "P99 latency {{ $value }}s > 2s"

        # Low Throughput (possible outage)
        - alert: LowThroughput
          expr: |
            sum(rate(orders_created_total[5m])) < 0.1
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Order throughput very low"

        # JVM Memory Pressure
        - alert: JVMMemoryPressure
          expr: |
            jvm_memory_bytes_used{area="heap"} / 
            jvm_memory_bytes_max{area="heap"} > 0.85
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "JVM heap usage > 85%"

        # Service Down
        - alert: ServiceDown
          expr: up{job="order-service"} == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Order service is down"
```

---

## Step 876: Log Aggregation

```yaml
# fluentbit-config.yaml
# Fluentbit: collect logs from containers → Elasticsearch

apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentbit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
      Flush         1
      Daemon        Off
      Log_Level     info
      Parsers_File  parsers.conf

    [INPUT]
      Name              tail
      Tag               kube.*
      Path              /var/log/containers/*.log
      Parser            docker
      DB                /var/log/flb_kube.db
      Mem_Buf_Limit     50MB
      Skip_Long_Lines   On
      Refresh_Interval  10

    [FILTER]
      Name                kubernetes
      Match               kube.*
      Kube_URL            https://kubernetes.default.svc:443
      Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
      Merge_Log           On
      K8S-Logging.Parser  On
      K8S-Logging.Exclude Off

    [FILTER]
      Name    record_modifier
      Match   kube.*
      Record  cluster production
      Record  region ap-southeast-1

    [OUTPUT]
      Name            es
      Match           kube.*
      Host            elasticsearch.logging.svc.cluster.local
      Port            9200
      Index           app-logs
      Type            _doc
      Logstash_Format On
      Logstash_Prefix app-logs
      Time_Key        @timestamp
      Replace_Dots    On

  parsers.conf: |
    [PARSER]
      Name        docker
      Format      json
      Time_Key    time
      Time_Format %Y-%m-%dT%H:%M:%S.%L

    [PARSER]
      Name        scala-json
      Format      json
      Time_Key    timestamp
      Time_Format %Y-%m-%dT%H:%M:%S.%L%z
```

---

## Step 877: Distributed Tracing with Jaeger

```yaml
# jaeger-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: tracing
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/all-in-one:1.52
          ports:
            - containerPort: 16686  # UI
            - containerPort: 4317   # OTLP gRPC
            - containerPort: 4318   # OTLP HTTP
          env:
            - name: COLLECTOR_OTLP_ENABLED
              value: "true"
          resources:
            limits:
              memory: "1Gi"
              cpu: "1000m"

---
apiVersion: v1
kind: Service
metadata:
  name: jaeger
  namespace: tracing
spec:
  selector:
    app: jaeger
  ports:
    - name: ui
      port: 16686
    - name: otlp-grpc
      port: 4317
    - name: otlp-http
      port: 4318
```

---

## Step 878: SLIs, SLOs, Error Budgets

```scala
// SLOTracking.scala

object SLOTracking {
  
  /*
  ===== Service Level Indicators (SLIs) =====
  
  1. Availability SLI:
     Successful requests / Total requests
     
  2. Latency SLI:
     Requests < 500ms / Total requests
     
  3. Error Rate SLI:
     1 - (Error requests / Total requests)
  
  ===== Service Level Objectives (SLOs) =====
  
  Order Service:
  - Availability: 99.9% (43.8 min/month downtime allowed)
  - P99 Latency < 1000ms: 99%
  - Error Rate < 0.1%
  
  Payment Service:
  - Availability: 99.99%
  - P99 Latency < 500ms: 99.9%
  - Error Rate < 0.01%
  
  ===== Error Budget =====
  
  Monthly error budget = 100% - SLO
  
  Order Service:
  - Availability budget = 0.1% = 43.8 min/month
  - Latency budget = 1% of requests can be > 1s
  
  If error budget < 5% remaining:
  → Freeze non-critical deployments
  → Focus on reliability improvements
  
  ===== Prometheus Queries for SLOs =====
  
  Availability SLI:
  1 - (
    sum(rate(http_requests_total{job="order-service",status=~"5.."}[30d]))
    /
    sum(rate(http_requests_total{job="order-service"}[30d]))
  )
  
  Error Budget Remaining:
  (
    1 - (
      sum(rate(http_requests_total{job="order-service",status=~"5.."}[30d]))
      /
      sum(rate(http_requests_total{job="order-service"}[30d]))
    )
  ) / 0.001  # = 0.1% SLO budget
  */
  
  case class SLO(
    name: String,
    target: Double, // 0.999 = 99.9%
    windowDays: Int = 30
  )
  
  case class ErrorBudget(
    slo: SLO,
    currentSLI: Double,
    budgetRemaining: Double, // percentage
    budgetRemainingMinutes: Double
  ) {
    def isBurnRateCritical: Boolean = budgetRemaining < 5.0
    def isExhausted: Boolean = budgetRemaining <= 0.0
  }
  
  def calculateErrorBudget(slo: SLO, currentSLI: Double): ErrorBudget = {
    val totalMinutes = slo.windowDays * 24 * 60.0
    val allowedDowntimeMinutes = totalMinutes * (1 - slo.target)
    
    val usedBudget = (slo.target - currentSLI) / (1 - slo.target)
    val remainingPercent = (1 - usedBudget) * 100
    val remainingMinutes = remainingPercent / 100 * allowedDowntimeMinutes
    
    ErrorBudget(slo, currentSLI, remainingPercent, remainingMinutes)
  }
  
  def main(args: Array[String]): Unit = {
    val orderSLO = SLO("order-service-availability", 0.999, 30)
    
    // Simulate: current month measured 99.85% availability
    val budget = calculateErrorBudget(orderSLO, 0.9985)
    
    println(s"SLO: ${orderSLO.target * 100}%")
    println(s"Current: ${budget.currentSLI * 100}%")
    println(f"Budget remaining: ${budget.budgetRemaining}%.1f%%")
    println(f"Budget minutes remaining: ${budget.budgetRemainingMinutes}%.1f min")
    
    if (budget.isBurnRateCritical) println("WARNING: Error budget critically low!")
    if (budget.isExhausted) println("ALERT: Error budget exhausted!")
  }
}
```

---

## Step 879: Dashboards และ Runbooks

```markdown
# Order Service Runbook

## Overview
Order Service handles order creation, updates, and queries.

## Alerts

### HighErrorRate
**Trigger**: Error rate > 1% for 2 minutes  
**Impact**: Orders failing, customer experience degraded  
**Actions**:
1. Check recent deployments (`kubectl rollout history deployment/order-service`)
2. Check application logs (`kubectl logs -l app=order-service --tail=100`)
3. Check DynamoDB errors in CloudWatch
4. Check downstream service health (payment-service, product-service)
5. If recent deploy caused it: `kubectl rollout undo deployment/order-service`

### HighP99Latency
**Trigger**: P99 latency > 2 seconds  
**Actions**:
1. Check database slow queries in DynamoDB console
2. Check Kafka consumer lag
3. Check JVM heap usage (Grafana > JVM dashboard)
4. Scale up if CPU/memory bound: `kubectl scale deployment/order-service --replicas=6`

### JVMMemoryPressure
**Trigger**: JVM heap > 85%  
**Actions**:
1. Check for memory leaks (thread dump, heap dump)
2. Increase memory limit: edit deployment.yaml
3. Check for cache size issues
4. Restart pod as temporary fix: `kubectl delete pod -l app=order-service`

## Common Operations
- Scale: `kubectl scale deployment/order-service --replicas=N -n scala-app`
- Restart: `kubectl rollout restart deployment/order-service -n scala-app`
- Logs: `kubectl logs -l app=order-service -n scala-app --tail=200 -f`
- Debug: `kubectl exec -it <pod> -- /bin/sh`
```

---

## Step 880: Full Observability Stack

```yaml
# observability-stack.yaml — Complete stack

# Prometheus
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      team: backend
  retention: 30d
  storage:
    volumeClaimTemplate:
      spec:
        resources:
          requests:
            storage: 50Gi

---
# Grafana
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:10.2.0
          env:
            - name: GF_SECURITY_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana-secrets
                  key: admin-password
            - name: GF_INSTALL_PLUGINS
              value: "grafana-piechart-panel"
          volumeMounts:
            - name: dashboards
              mountPath: /var/lib/grafana/dashboards
      volumes:
        - name: dashboards
          configMap:
            name: grafana-dashboards
```

---

## สรุป Part 88: Observability

| Pillar | Tool | Purpose |
|--------|------|---------|
| Logs | Logback + JSON | Debug, audit |
| Metrics | Prometheus + Grafana | Performance, SLOs |
| Traces | OpenTelemetry + Jaeger | Request flow |
| Alerts | Prometheus Rules | Incident detection |
| Dashboards | Grafana | Visualization |

### Golden Signals (SRE)
1. **Latency**: P50/P90/P99 response time
2. **Traffic**: Requests/second
3. **Errors**: Error rate %
4. **Saturation**: CPU, memory, disk utilization

---

## แบบฝึกหัด Part 88

1. **Structured Logging**: implement structured logging framework ที่ automatic add request context (trace ID, user ID) ทุก log line

2. **Custom Metrics**: สร้าง business metrics สำหรับ e-commerce ที่ track orders per category, revenue per region, conversion funnel

3. **Distributed Tracing**: implement end-to-end tracing across 3 services โดย trace spans propagate correctly ผ่าน HTTP headers

4. **SLO Dashboard**: สร้าง Grafana dashboard ที่แสดง availability SLI, error budget remaining, และ burn rate

5. **Alert Tuning**: ตั้ง Prometheus alerts ที่ minimise false positives โดยใช้ `for` duration และ appropriate thresholds

---

## ไปต่อ: Part 89 — CI/CD
ใน Part ถัดไปจะเรียน GitHub Actions, automated testing, Docker push, semantic versioning

[→ Part 89: CI/CD](./part-89-cicd.md)
