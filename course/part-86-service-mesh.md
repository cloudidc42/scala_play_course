# Part 86: Service Mesh — Steps 851-860

## บทนำ: Service Mesh

Service Mesh คือ infrastructure layer ที่ handle service-to-service communication โดยไม่ต้องแก้ application code เพิ่ม capabilities: traffic management, mTLS, observability, circuit breaking

---

## Step 851: Istio Overview

```yaml
# Istio Architecture:
#
#  ┌─────────────────────────────────────────┐
#  │              Control Plane              │
#  │  istiod (Pilot + Citadel + Galley)     │
#  └─────────────────────────────────────────┘
#          │ config + certificates
#  ┌───────┴────────────────────────────┐
#  │           Data Plane               │
#  │  ┌─────────────┐  ┌─────────────┐ │
#  │  │ Pod A       │  │ Pod B       │ │
#  │  │ ┌─────────┐ │  │ ┌─────────┐ │ │
#  │  │ │ App     │ │  │ │ App     │ │ │
#  │  │ └────┬────┘ │  │ └────┬────┘ │ │
#  │  │ ┌────┴────┐ │  │ ┌────┴────┐ │ │
#  │  │ │Envoy    │ │  │ │Envoy    │ │ │
#  │  │ │Sidecar  │◄├──┤►│Sidecar  │ │ │
#  │  │ └─────────┘ │  │ └─────────┘ │ │
#  │  └─────────────┘  └─────────────┘ │
#  └────────────────────────────────────┘

# install-istio.sh
istioctl install --set profile=production
kubectl label namespace scala-app istio-injection=enabled
```

---

## Step 852: Traffic Management

```yaml
# virtual-service.yaml
# VirtualService: routing rules

apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
  namespace: scala-app
spec:
  hosts:
    - order-service
  http:
    # ===== A/B Testing: 90% v1, 10% v2 =====
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: order-service
            subset: v2
    
    - route:
        - destination:
            host: order-service
            subset: v1
          weight: 90
        - destination:
            host: order-service
            subset: v2
          weight: 10
    
    # Timeout
    timeout: 30s
    
    # Retry policy
    retries:
      attempts: 3
      perTryTimeout: 10s
      retryOn: "5xx,reset,connect-failure,retriable-4xx"
    
    # Fault injection (สำหรับ chaos testing)
    fault:
      delay:
        percentage:
          value: 0.1  # 0.1% of requests
        fixedDelay: 5s

---
# destination-rule.yaml
# DestinationRule: traffic policies per service

apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service
  namespace: scala-app
spec:
  host: order-service
  
  trafficPolicy:
    # Connection pool settings
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 30s
      http:
        h2UpgradePolicy: UPGRADE
        idleTimeout: 10m
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
    
    # Outlier detection (circuit breaker)
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 50
    
    # Load balancing
    loadBalancer:
      simple: LEAST_CONN  # ROUND_ROBIN | LEAST_CONN | RANDOM
  
  subsets:
    - name: v1
      labels:
        version: "1.0"
      trafficPolicy:
        connectionPool:
          http:
            maxRequestsPerConnection: 1
    
    - name: v2
      labels:
        version: "2.0"
```

---

## Step 853: mTLS Configuration

```yaml
# peer-authentication.yaml
# Enable mTLS for all services in namespace

apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: scala-app
spec:
  mtls:
    mode: STRICT  # enforce mTLS

---
# peer-auth-permissive.yaml
# Permissive mode (migration period)

apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: order-service-mtls
  namespace: scala-app
spec:
  selector:
    matchLabels:
      app: order-service
  mtls:
    mode: PERMISSIVE  # accept both TLS and plain

---
# authorization-policy.yaml
# Service-level authorization

apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-service-authz
  namespace: scala-app
spec:
  selector:
    matchLabels:
      app: order-service
  
  action: ALLOW
  
  rules:
    # Product service can call Orders
    - from:
        - source:
            principals:
              - "cluster.local/ns/scala-app/sa/product-service"
      to:
        - operation:
            methods: ["GET"]
            paths: ["/api/v1/orders/*"]
    
    # User service can call Orders
    - from:
        - source:
            principals:
              - "cluster.local/ns/scala-app/sa/user-service"
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/v1/orders", "/api/v1/orders/*"]
    
    # API Gateway can call all methods
    - from:
        - source:
            principals:
              - "cluster.local/ns/scala-app/sa/api-gateway"
```

---

## Step 854: Observability with Istio

```yaml
# telemetry.yaml
# Istio observability configuration

apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: scala-app-telemetry
  namespace: scala-app
spec:
  # Tracing
  tracing:
    - providers:
        - name: jaeger
      randomSamplingPercentage: 1.0  # 1% sampling
      disableSpanReporting: false
  
  # Access logging
  accessLogging:
    - providers:
        - name: envoy
  
  # Metrics
  metrics:
    - providers:
        - name: prometheus
      overrides:
        - match:
            metric: ALL_METRICS
          tagOverrides:
            request_id:
              operation: UPSERT
              value: request.headers["x-request-id"] | ""

---
# service-monitor.yaml (Prometheus ServiceMonitor)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istio-proxy-metrics
  namespace: scala-app
spec:
  selector:
    matchLabels:
      app: order-service
  namespaceSelector:
    matchNames:
      - scala-app
  endpoints:
    - port: http-envoy-prom
      path: /stats/prometheus
      interval: 15s
```

```scala
// TraceContext.scala
// Propagate trace headers ใน Scala application

import akka.http.scaladsl.model.HttpRequest
import akka.http.scaladsl.model.headers.RawHeader

// Istio ใช้ B3 headers สำหรับ tracing
// Applications ต้อง forward headers เหล่านี้

object TraceHeaders {
  val B3_TRACE_ID      = "x-b3-traceid"
  val B3_SPAN_ID       = "x-b3-spanid"
  val B3_PARENT_SPAN   = "x-b3-parentspanid"
  val B3_SAMPLED       = "x-b3-sampled"
  val REQUEST_ID       = "x-request-id"
  
  val ALL_HEADERS = List(B3_TRACE_ID, B3_SPAN_ID, B3_PARENT_SPAN, B3_SAMPLED, REQUEST_ID)
  
  def extractFrom(request: HttpRequest): Map[String, String] = {
    ALL_HEADERS.flatMap { headerName =>
      request.headers.find(_.lowercaseName == headerName)
                     .map(h => headerName -> h.value)
    }.toMap
  }
  
  def injectInto(request: HttpRequest, traceCtx: Map[String, String]): HttpRequest = {
    val headers = traceCtx.map { case (k, v) => RawHeader(k, v) }.toList
    request.withHeaders(request.headers ++ headers)
  }
}

// ===== Service with trace propagation =====
class OrderServiceWithTracing(
  userServiceClient: akka.http.scaladsl.HttpExt
)(implicit system: akka.actor.ActorSystem, ec: scala.concurrent.ExecutionContext) {
  
  def createOrder(request: akka.http.scaladsl.model.HttpRequest): scala.concurrent.Future[String] = {
    // Extract trace context from incoming request
    val traceCtx = TraceHeaders.extractFrom(request)
    
    // Forward to user service with same trace context
    val userRequest = TraceHeaders.injectInto(
      akka.http.scaladsl.model.HttpRequest(uri = "http://user-service/api/v1/users/validate"),
      traceCtx
    )
    
    userServiceClient.singleRequest(userRequest).map { response =>
      s"Order created with trace: ${traceCtx.get(TraceHeaders.B3_TRACE_ID).getOrElse("no-trace")}"
    }
  }
}
```

---

## Step 855: Circuit Breaking via Istio

```yaml
# circuit-breaker.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service-cb
  namespace: scala-app
spec:
  host: payment-service
  
  trafficPolicy:
    # Circuit breaker settings
    outlierDetection:
      # Open circuit after 5 consecutive 5xx errors
      consecutiveGatewayErrors: 5
      # Check interval
      interval: 10s
      # Eject for 30 seconds minimum
      baseEjectionTime: 30s
      # Max 100% of hosts can be ejected
      maxEjectionPercent: 100
      # Min healthy hosts before ejection starts
      minHealthPercent: 0
    
    # Connection limits
    connectionPool:
      tcp:
        maxConnections: 50
      http:
        http1MaxPendingRequests: 100
        maxRequestsPerConnection: 10
        maxRetries: 3
        maxRequestsPerConnection: 1

---
# chaos-testing.yaml
# Fault injection สำหรับ Chaos Engineering
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service-chaos
  namespace: scala-app
spec:
  hosts:
    - payment-service
  http:
    - fault:
        # 10% HTTP 500 errors
        abort:
          percentage:
            value: 10
          httpStatus: 500
        # 5% requests get 2s delay
        delay:
          percentage:
            value: 5
          fixedDelay: 2s
      route:
        - destination:
            host: payment-service
```

---

## Step 856: Traffic Mirroring

```yaml
# traffic-mirroring.yaml
# Shadow traffic: send copy to v2 without affecting users

apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service-mirror
  namespace: scala-app
spec:
  hosts:
    - order-service
  http:
    - route:
        - destination:
            host: order-service
            subset: v1
          weight: 100
      # Mirror 100% of traffic to v2 (shadow)
      mirror:
        host: order-service
        subset: v2
      mirrorPercentage:
        value: 100.0

---
# Blue-Green Deployment
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service-bluegreen
spec:
  hosts:
    - order-service
  http:
    - route:
        # 100% → green (new version)
        - destination:
            host: order-service
            subset: green
          weight: 100
        # Keep blue at 0% (easy rollback)
        - destination:
            host: order-service
            subset: blue
          weight: 0
```

---

## Step 857: Linkerd Alternative

```yaml
# Linkerd: simpler alternative to Istio

# Install
# curl -sL run.linkerd.io/install | sh
# linkerd install | kubectl apply -f -

# Annotate namespace
# kubectl annotate namespace scala-app linkerd.io/inject=enabled

# ServiceProfile (traffic management in Linkerd)
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: order-service.scala-app.svc.cluster.local
  namespace: scala-app
spec:
  routes:
    - name: "POST /api/v1/orders"
      condition:
        method: POST
        pathRegex: /api/v1/orders
      responseClasses:
        - condition:
            status:
              min: 500
              max: 599
          isFailure: true
      timeout: 30s
      retryBudget:
        retryRatio: 0.2
        minRetriesPerSecond: 10
        ttl: 10s
    
    - name: "GET /api/v1/orders/{id}"
      condition:
        method: GET
        pathRegex: /api/v1/orders/[^/]*
      isRetryable: true
      timeout: 10s
```

---

## Step 858: Istio Gateway

```yaml
# gateway.yaml
# Istio Gateway: เข้ามาจาก external traffic

apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: api-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      tls:
        httpsRedirect: true
      hosts:
        - api.myapp.com
    
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: api-tls-cert
      hosts:
        - api.myapp.com

---
# virtual-service-gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-gateway-routing
  namespace: scala-app
spec:
  hosts:
    - api.myapp.com
  gateways:
    - istio-system/api-gateway
  http:
    - match:
        - uri:
            prefix: /api/v1/orders
      route:
        - destination:
            host: order-service.scala-app.svc.cluster.local
            port:
              number: 80
    
    - match:
        - uri:
            prefix: /api/v1/products
      route:
        - destination:
            host: product-service.scala-app.svc.cluster.local
            port:
              number: 80
    
    # Rate limiting via Envoy filter
    - match:
        - headers:
            x-api-tier:
              exact: free
      fault:
        delay:
          percentage:
            value: 100
          fixedDelay: 50ms  # slow down free tier
      route:
        - destination:
            host: order-service.scala-app.svc.cluster.local
```

---

## Step 859: Monitoring Service Mesh

```scala
// ServiceMeshMetrics.scala
// Query Prometheus metrics exposed by Istio/Envoy

/*
Key Istio Metrics:

1. Request Rate:
   rate(istio_requests_total{destination_service="order-service"}[5m])

2. Error Rate:
   rate(istio_requests_total{
     destination_service="order-service",
     response_code=~"5.."
   }[5m]) / rate(istio_requests_total{
     destination_service="order-service"
   }[5m])

3. P99 Latency:
   histogram_quantile(0.99, 
     rate(istio_request_duration_milliseconds_bucket{
       destination_service="order-service"
     }[5m])
   )

4. Throughput:
   sum(rate(istio_request_bytes_sum{
     destination_service="order-service"
   }[5m]))

5. mTLS Success Rate:
   sum(rate(istio_requests_total{
     destination_service="order-service",
     connection_security_policy="mutual_tls"
   }[5m])) / sum(rate(istio_requests_total{
     destination_service="order-service"
   }[5m]))
*/

// Kiali dashboard: visualize service mesh topology
// kubectl port-forward svc/kiali -n istio-system 20001:20001

// Jaeger tracing: distributed traces
// kubectl port-forward svc/tracing -n istio-system 80:80

// Grafana dashboards: pre-built Istio dashboards
// kubectl port-forward svc/grafana -n istio-system 3000:3000
```

---

## Step 860: Production Service Mesh Checklist

```yaml
# production-mesh-config.yaml
# Complete production configuration

# 1. Strict mTLS for all services
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: enforce-mtls
  namespace: scala-app
spec:
  mtls:
    mode: STRICT

---
# 2. Authorization policies
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: scala-app
spec:
  # Deny all unless explicitly allowed

---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-order-service
  namespace: scala-app
spec:
  selector:
    matchLabels:
      app: order-service
  action: ALLOW
  rules:
    - from:
        - source:
            namespaces: ["scala-app", "istio-system"]

---
# 3. Global traffic policy
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: global-tls
  namespace: istio-system
spec:
  host: "*.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL  # use Istio-managed certs

---
# 4. Retry and timeout defaults
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service-defaults
  namespace: scala-app
spec:
  hosts:
    - order-service
  http:
    - timeout: 30s
      retries:
        attempts: 3
        perTryTimeout: 10s
        retryOn: "gateway-error,connect-failure,refused-stream"
      route:
        - destination:
            host: order-service
```

```markdown
# Production Service Mesh Checklist

## Security
- [ ] mTLS STRICT mode enabled for all services
- [ ] AuthorizationPolicy: deny-all + explicit allows
- [ ] Network policies (K8s level)
- [ ] No service accounts with cluster-admin

## Traffic Management
- [ ] VirtualService with timeout configured
- [ ] Retry policy with appropriate retryOn
- [ ] Circuit breaker (outlierDetection)
- [ ] PodDisruptionBudget for critical services

## Observability
- [ ] Distributed tracing (Jaeger/Zipkin)
- [ ] Metrics collection (Prometheus)
- [ ] Dashboards (Grafana/Kiali)
- [ ] Access logging enabled
- [ ] Alert rules for error rate > 1%, p99 > 1s

## Deployment
- [ ] Canary deployment process documented
- [ ] Traffic mirroring for new versions
- [ ] Rollback procedure tested
- [ ] Load testing through mesh before go-live
```

---

## สรุป Part 86: Service Mesh

| Feature | Istio | Linkerd |
|---------|-------|---------|
| Complexity | High | Low |
| Features | Very rich | Essential |
| Resource usage | High | Low |
| Learning curve | Steep | Gentle |
| gRPC support | Yes | Yes |
| mTLS | Yes | Automatic |

### Key Istio Resources

| Resource | Purpose |
|---------|---------|
| VirtualService | Routing rules |
| DestinationRule | Traffic policy |
| Gateway | External traffic entry |
| PeerAuthentication | mTLS enforcement |
| AuthorizationPolicy | RBAC for services |
| ServiceEntry | External services |

---

## แบบฝึกหัด Part 86

1. **Canary Deployment**: configure Istio VirtualService เพื่อ route 5% traffic ไปยัง v2 แล้ว monitor metrics ก่อน promote เป็น 100%

2. **mTLS Setup**: enable strict mTLS ใน namespace และ verify ด้วย `istioctl authn tls-check`

3. **Authorization Policy**: implement RBAC ที่ service A เรียก B ได้เฉพาะ specific endpoints

4. **Chaos Engineering**: inject fault (delay + abort) ใน payment service และ verify ว่า order service handle failures gracefully

5. **Observability**: สร้าง Grafana dashboard สำหรับ service mesh ที่แสดง request rate, error rate, P99 latency per service

---

## ไปต่อ: Part 87 — Cloud AWS
ใน Part ถัดไปจะเรียน AWS SDK for Scala, S3, SQS, DynamoDB, Lambda

[→ Part 87: Cloud AWS](./part-87-cloud-aws.md)
