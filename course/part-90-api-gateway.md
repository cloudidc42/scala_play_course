# Part 90: API Gateway — Steps 891-900

## บทนำ: API Gateway

API Gateway เป็น entry point สำหรับ client requests ที่ route ไปยัง microservices โดยเพิ่ม authentication, rate limiting, SSL termination, logging

---

## Step 891: API Gateway Patterns

```scala
// APIGatewayPatterns.scala

/*
===== API Gateway Core Functions =====

1. Routing: /api/v1/orders → order-service
2. Authentication: JWT validation
3. Rate Limiting: 100 req/min per user
4. SSL Termination: HTTPS → HTTP
5. Request/Response Transform
6. Circuit Breaking
7. Load Balancing
8. Caching
9. Logging & Monitoring
10. API Versioning

===== Gateway Topologies =====

Simple:
  Client → Gateway → Services

BFF (Backend For Frontend):
  Mobile App → Mobile BFF → Services
  Web App → Web BFF → Services
  (each BFF optimized for its client)

Multi-tier:
  Internet → Edge Gateway → Internal Gateway → Services
*/

// Gateway configuration model
case class GatewayConfig(
  routes: List[RouteConfig],
  auth: AuthConfig,
  rateLimit: RateLimitConfig
)

case class RouteConfig(
  path: String,
  methods: List[String],
  upstream: String,
  timeout: Int = 30,
  retries: Int = 3,
  authRequired: Boolean = true
)

case class AuthConfig(
  jwtSecret: String,
  issuer: String,
  audience: String
)

case class RateLimitConfig(
  requestsPerMinute: Int = 100,
  burstSize: Int = 20
)
```

---

## Step 892: Scala API Gateway Implementation

```scala
// ScalaAPIGateway.scala
import akka.actor.typed.ActorSystem
import akka.actor.typed.scaladsl.Behaviors
import akka.http.scaladsl.Http
import akka.http.scaladsl.model._
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.server.{Directive0, Route}
import akka.http.scaladsl.model.headers._
import spray.json._
import spray.json.DefaultJsonProtocol._
import scala.concurrent.{ExecutionContext, Future}
import java.util.concurrent.atomic.{AtomicLong, AtomicInteger}
import scala.collection.concurrent.TrieMap

// ===== Rate Limiter =====
class TokenBucketRateLimiter(tokensPerSecond: Int, bucketSize: Int) {
  private val tokens = TrieMap[String, (AtomicLong, AtomicLong)]()
  
  def tryConsume(key: String): Boolean = {
    val now = System.currentTimeMillis()
    val entry = tokens.getOrElseUpdate(key, (new AtomicLong(bucketSize), new AtomicLong(now)))
    val (bucket, lastRefill) = entry
    
    // Refill tokens based on elapsed time
    val elapsed = now - lastRefill.get()
    val newTokens = (elapsed * tokensPerSecond / 1000).toInt
    
    if (newTokens > 0) {
      bucket.updateAndGet(t => math.min(t + newTokens, bucketSize))
      lastRefill.set(now)
    }
    
    // Try to consume 1 token
    val current = bucket.get()
    if (current > 0) {
      bucket.compareAndSet(current, current - 1)
    } else {
      false
    }
  }
}

// ===== JWT Validator =====
class JWTValidator(secret: String) {
  import javax.crypto.spec.SecretKeySpec
  import javax.crypto.Mac
  import java.util.Base64
  
  def validate(token: String): Option[String] = {
    // Simple HS256 JWT validation
    val parts = token.split("\\.")
    if (parts.length != 3) return None
    
    val (header, payload, signature) = (parts(0), parts(1), parts(2))
    
    // Verify signature
    val mac = Mac.getInstance("HmacSHA256")
    mac.init(new SecretKeySpec(secret.getBytes, "HmacSHA256"))
    val expectedSig = Base64.getUrlEncoder.withoutPadding()
      .encodeToString(mac.doFinal(s"$header.$payload".getBytes))
    
    if (expectedSig != signature) return None
    
    // Decode payload
    val payloadJson = new String(Base64.getUrlDecoder.decode(payload))
    val json = payloadJson.parseJson.asJsObject
    
    // Extract subject
    json.fields.get("sub").map(_.convertTo[String])
  }
}

// ===== Circuit Breaker =====
class SimpleCircuitBreaker(maxFailures: Int = 5, resetSeconds: Int = 30) {
  sealed trait State
  case object Closed extends State
  case object Open extends State
  case object HalfOpen extends State
  
  private var state: State = Closed
  private var failures = 0
  private var lastFailureTime = 0L
  
  def execute[T](f: => Future[T])(implicit ec: ExecutionContext): Future[T] = {
    state match {
      case Open =>
        if (System.currentTimeMillis() - lastFailureTime > resetSeconds * 1000) {
          state = HalfOpen
          call(f)
        } else {
          Future.failed(new RuntimeException("Circuit breaker OPEN"))
        }
      case _ => call(f)
    }
  }
  
  private def call[T](f: => Future[T])(implicit ec: ExecutionContext): Future[T] = {
    f.map { result =>
      if (state == HalfOpen) { state = Closed; failures = 0 }
      result
    }.recoverWith { case ex =>
      failures += 1
      lastFailureTime = System.currentTimeMillis()
      if (failures >= maxFailures) state = Open
      Future.failed(ex)
    }
  }
}

// ===== Main Gateway =====
class APIGateway(config: GatewayConfig)(implicit system: ActorSystem[_]) {
  implicit val ec: ExecutionContext = system.executionContext
  
  private val http = Http()
  private val rateLimiter = new TokenBucketRateLimiter(
    config.rateLimit.requestsPerMinute / 60,
    config.rateLimit.burstSize
  )
  private val jwtValidator = new JWTValidator(config.auth.jwtSecret)
  private val circuitBreakers = TrieMap[String, SimpleCircuitBreaker]()
  
  // CORS headers
  private val corsHeaders = List(
    `Access-Control-Allow-Origin`.*,
    `Access-Control-Allow-Methods`(HttpMethods.GET, HttpMethods.POST, HttpMethods.PUT, HttpMethods.DELETE, HttpMethods.OPTIONS),
    `Access-Control-Allow-Headers`("Authorization", "Content-Type", "X-Request-ID")
  )
  
  // Authentication directive
  def authenticate: Directive1[String] = {
    optionalHeaderValueByName("Authorization").flatMap {
      case Some(token) if token.startsWith("Bearer ") =>
        jwtValidator.validate(token.drop(7)) match {
          case Some(userId) => provide(userId)
          case None         => complete(StatusCodes.Unauthorized, """{"error":"Invalid token"}""")
        }
      case _ => complete(StatusCodes.Unauthorized, """{"error":"Missing token"}""")
    }
  }
  
  // Rate limit directive
  def rateLimit(userId: String): Directive0 = {
    if (rateLimiter.tryConsume(userId)) pass
    else complete(StatusCodes.TooManyRequests, """{"error":"Rate limit exceeded","retryAfter":60}""")
  }
  
  // Proxy to upstream
  def proxyRequest(request: HttpRequest, upstream: String): Future[HttpResponse] = {
    val cb = circuitBreakers.getOrElseUpdate(upstream, new SimpleCircuitBreaker())
    
    cb.execute {
      val forwardUri = s"$upstream${request.uri.path}${request.uri.rawQueryString.map("?" + _).getOrElse("")}"
      val forwardRequest = request.withUri(forwardUri)
        .withHeaders(
          request.headers.filterNot(_.is("host")) ++
          List(RawHeader("X-Forwarded-By", "api-gateway"))
        )
      
      http.singleRequest(forwardRequest)
    }
  }
  
  // Build routes
  val routes: Route = {
    respondWithHeaders(corsHeaders) {
      // CORS preflight
      options { complete(StatusCodes.OK) } ~
      
      // Health check (no auth)
      path("health") {
        get { complete("""{"status":"healthy"}""") }
      } ~
      
      // API routes
      pathPrefix("api" / "v1") {
        authenticate { userId =>
          rateLimit(userId) {
            // Match configured routes
            config.routes.foldLeft[Route](reject) { case (acc, route) =>
              acc ~ {
                val upstream = route.upstream
                
                pathPrefix(route.path.split("/").filter(_.nonEmpty).map(pm => pm: PathMatcher0).reduceLeft(_ / _)) {
                  extractRequest { request =>
                    if (route.methods.contains(request.method.value)) {
                      onSuccess(proxyRequest(request, upstream)) { response =>
                        complete(response)
                      }
                    } else {
                      reject
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
  
  def start(host: String = "0.0.0.0", port: Int = 8080): Future[Http.ServerBinding] = {
    Http().newServerAt(host, port).bind(routes)
  }
}

object APIGatewayMain {
  def main(args: Array[String]): Unit = {
    implicit val system: ActorSystem[Nothing] = ActorSystem(Behaviors.empty, "api-gateway")
    
    val config = GatewayConfig(
      routes = List(
        RouteConfig("/orders",   List("GET", "POST"), "http://order-service:8080"),
        RouteConfig("/products", List("GET"),          "http://product-service:8080"),
        RouteConfig("/users",    List("GET", "PUT"),   "http://user-service:8080", authRequired = false)
      ),
      auth      = AuthConfig("my-jwt-secret", "myapp", "myapp-api"),
      rateLimit = RateLimitConfig(requestsPerMinute = 100, burstSize = 20)
    )
    
    val gateway = new APIGateway(config)
    
    import scala.concurrent.ExecutionContext.Implicits.global
    gateway.start().foreach { binding =>
      println(s"API Gateway started on ${binding.localAddress}")
    }
    
    system.whenTerminated.foreach(_ => println("Gateway stopped"))
  }
}
```

---

## Step 893: Nginx Configuration

```nginx
# nginx.conf — Production API Gateway

worker_processes auto;
worker_rlimit_nofile 65535;

events {
  worker_connections 10000;
  multi_accept on;
  use epoll;
}

http {
  # Connection settings
  keepalive_timeout 65;
  keepalive_requests 100;
  
  # Buffer settings
  client_body_buffer_size 128k;
  client_max_body_size 10m;
  
  # Compression
  gzip on;
  gzip_types text/plain application/json application/javascript;
  gzip_min_length 1024;
  
  # Logging
  log_format json_combined escape=json
    '{'
      '"time":"$time_iso8601",'
      '"remote_addr":"$remote_addr",'
      '"method":"$request_method",'
      '"uri":"$request_uri",'
      '"status":"$status",'
      '"request_time":"$request_time",'
      '"upstream_response_time":"$upstream_response_time",'
      '"request_id":"$http_x_request_id"'
    '}';
  
  access_log /var/log/nginx/access.log json_combined;
  
  # Rate limiting zones
  limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
  limit_req_zone $http_authorization zone=authenticated:10m rate=1000r/m;
  
  # Upstream definitions
  upstream order_service {
    least_conn;
    server order-service-1:8080 weight=1;
    server order-service-2:8080 weight=1;
    server order-service-3:8080 weight=1;
    keepalive 32;
  }
  
  upstream product_service {
    server product-service:8080;
    keepalive 16;
  }
  
  # Main server
  server {
    listen 443 ssl http2;
    server_name api.myapp.com;
    
    # SSL
    ssl_certificate     /etc/nginx/certs/api.myapp.com.crt;
    ssl_certificate_key /etc/nginx/certs/api.myapp.com.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_session_cache   shared:SSL:10m;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;
    add_header X-XSS-Protection "1; mode=block";
    
    # CORS
    add_header Access-Control-Allow-Origin "$http_origin" always;
    add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
    add_header Access-Control-Allow-Headers "Authorization, Content-Type, X-Request-ID" always;
    
    if ($request_method = OPTIONS) {
      return 204;
    }
    
    # Request ID
    add_header X-Request-ID $request_id;
    proxy_set_header X-Request-ID $request_id;
    
    # Health check
    location /health {
      access_log off;
      return 200 '{"status":"ok"}';
      add_header Content-Type application/json;
    }
    
    # Orders API
    location /api/v1/orders {
      limit_req zone=authenticated burst=20 nodelay;
      
      proxy_pass http://order_service;
      proxy_http_version 1.1;
      proxy_set_header Connection "";
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      
      proxy_connect_timeout 5s;
      proxy_send_timeout 30s;
      proxy_read_timeout 30s;
      
      # Retry on failure
      proxy_next_upstream error timeout http_503;
      proxy_next_upstream_tries 3;
    }
    
    # Products API with caching
    location /api/v1/products {
      proxy_pass http://product_service;
      proxy_cache product_cache;
      proxy_cache_valid 200 5m;
      proxy_cache_key "$host$request_uri";
      add_header X-Cache-Status $upstream_cache_status;
    }
  }
  
  # HTTP → HTTPS redirect
  server {
    listen 80;
    return 301 https://$host$request_uri;
  }
  
  # Response cache
  proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=product_cache:10m max_size=1g;
}
```

---

## Step 894: Traefik Configuration

```yaml
# traefik.yml
api:
  dashboard: true
  insecure: false

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
  websecure:
    address: ":443"
    http:
      tls:
        certResolver: letsencrypt

certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@mycompany.com
      storage: /data/acme.json
      httpChallenge:
        entryPoint: web

providers:
  docker:
    exposedByDefault: false
    network: web

---
# docker-compose-traefik.yml
version: '3.8'

services:
  traefik:
    image: traefik:v3.0
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"  # dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./traefik.yml:/traefik.yml
      - ./acme.json:/data/acme.json
    networks:
      - web

  order-service:
    image: registry.mycompany.com/order-service:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.orders.rule=Host(`api.myapp.com`) && PathPrefix(`/api/v1/orders`)"
      - "traefik.http.routers.orders.entrypoints=websecure"
      - "traefik.http.routers.orders.tls=true"
      - "traefik.http.services.orders.loadbalancer.server.port=8080"
      
      # Middlewares
      - "traefik.http.routers.orders.middlewares=auth-middleware,rate-limit-middleware"
      
      # Auth middleware
      - "traefik.http.middlewares.auth-middleware.forwardauth.address=http://auth-service:8080/verify"
      
      # Rate limit
      - "traefik.http.middlewares.rate-limit-middleware.ratelimit.average=100"
      - "traefik.http.middlewares.rate-limit-middleware.ratelimit.burst=50"
      - "traefik.http.middlewares.rate-limit-middleware.ratelimit.period=1m"
    networks:
      - web

networks:
  web:
    external: true
```

---

## Step 895: API Versioning

```scala
// APIVersioning.scala
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.server.Route

// ===== Versioning Strategies =====

object APIVersioning {
  
  // 1. URL versioning: /api/v1/orders, /api/v2/orders
  def urlVersioningRoutes: Route = {
    pathPrefix("api") {
      concat(
        pathPrefix("v1") {
          // V1 routes
          path("orders") {
            get { complete("""{"version":"v1","orders":[]}""") }
          }
        },
        pathPrefix("v2") {
          // V2 routes (new features)
          path("orders") {
            get { complete("""{"version":"v2","orders":[],"pagination":{"page":1}}""") }
          }
        }
      )
    }
  }
  
  // 2. Header versioning: Accept: application/vnd.myapp.v2+json
  def headerVersioningRoute: Route = {
    optionalHeaderValueByName("Accept") { acceptHeader =>
      val version = acceptHeader.flatMap { accept =>
        if (accept.contains("v2")) Some("v2")
        else if (accept.contains("v1")) Some("v1")
        else None
      }.getOrElse("v1")
      
      path("orders") {
        get { complete(s"""{"version":"$version","orders":[]}""") }
      }
    }
  }
  
  // 3. Query param versioning: /orders?version=2
  def queryVersioningRoute: Route = {
    path("orders") {
      get {
        parameter("version".withDefault("1")) { version =>
          complete(s"""{"version":"v$version","orders":[]}""")
        }
      }
    }
  }
}

// ===== API Version Management =====
case class APIVersion(major: Int, minor: Int) {
  override def toString: String = s"v$major.$minor"
  def isCompatibleWith(other: APIVersion): Boolean = major == other.major
}

object APIVersion {
  val CURRENT = APIVersion(2, 0)
  val SUPPORTED = List(APIVersion(1, 0), APIVersion(2, 0))
  val DEPRECATED = List(APIVersion(1, 0))
  
  def from(versionStr: String): Option[APIVersion] = {
    versionStr.stripPrefix("v").split("\\.") match {
      case Array(major, minor) => Some(APIVersion(major.toInt, minor.toInt))
      case Array(major)        => Some(APIVersion(major.toInt, 0))
      case _                   => None
    }
  }
}
```

---

## Step 896: GraphQL as API Gateway

```scala
// GraphQLGateway.scala
// ใช้ Sangria สำหรับ GraphQL

// build.sbt
// "org.sangria-graphql" %% "sangria" % "4.0.2"
// "org.sangria-graphql" %% "sangria-spray-json" % "2.0.2"
// "org.sangria-graphql" %% "sangria-akka-http-core" % "0.0.3"

import sangria.schema._
import sangria.macros._
import sangria.execution._
import sangria.marshalling.sprayJson._
import spray.json._
import DefaultJsonProtocol._
import scala.concurrent.{ExecutionContext, Future}

// ===== Schema Stitching =====
// GraphQL Federation-like pattern: combine multiple services

case class OrderGQL(id: String, customerId: String, total: Double, status: String)
case class CustomerGQL(id: String, name: String, email: String)

class OrderRepository(implicit ec: ExecutionContext) {
  def findById(id: String): Future[Option[OrderGQL]] =
    Future.successful(Some(OrderGQL(id, "C001", 99.99, "pending")))
  
  def findByCustomer(customerId: String): Future[List[OrderGQL]] =
    Future.successful(List(OrderGQL("O001", customerId, 99.99, "pending")))
}

class CustomerRepository(implicit ec: ExecutionContext) {
  def findById(id: String): Future[Option[CustomerGQL]] =
    Future.successful(Some(CustomerGQL(id, "Alice", "alice@example.com")))
}

// ===== GraphQL Types =====
object GraphQLSchema {
  
  val OrderType: ObjectType[Unit, OrderGQL] = ObjectType("Order",
    fields[Unit, OrderGQL](
      Field("id",         StringType,  resolve = _.value.id),
      Field("customerId", StringType,  resolve = _.value.customerId),
      Field("total",      FloatType,   resolve = _.value.total),
      Field("status",     StringType,  resolve = _.value.status)
    )
  )
  
  val CustomerType: ObjectType[Unit, CustomerGQL] = ObjectType("Customer",
    fields[Unit, CustomerGQL](
      Field("id",    StringType, resolve = _.value.id),
      Field("name",  StringType, resolve = _.value.name),
      Field("email", StringType, resolve = _.value.email)
    )
  )
  
  val QueryType: ObjectType[GraphQLContext, Unit] = ObjectType("Query",
    fields[GraphQLContext, Unit](
      Field("order",
        OptionType(OrderType),
        arguments = Argument("id", StringType) :: Nil,
        resolve = ctx => ctx.ctx.orderRepo.findById(ctx.arg("id"))
      ),
      Field("orders",
        ListType(OrderType),
        arguments = Argument("customerId", StringType) :: Nil,
        resolve = ctx => ctx.ctx.orderRepo.findByCustomer(ctx.arg("customerId"))
      ),
      Field("customer",
        OptionType(CustomerType),
        arguments = Argument("id", StringType) :: Nil,
        resolve = ctx => ctx.ctx.customerRepo.findById(ctx.arg("id"))
      )
    )
  )
  
  val schema: Schema[GraphQLContext, Unit] = Schema(QueryType)
}

case class GraphQLContext(
  orderRepo: OrderRepository,
  customerRepo: CustomerRepository
)
```

---

## Step 897: WebSocket Gateway

```scala
// WebSocketGateway.scala
import akka.actor.typed.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.model.ws._
import akka.stream.scaladsl._
import scala.concurrent.ExecutionContext

class WebSocketGateway()(implicit system: ActorSystem[_]) {
  implicit val ec: ExecutionContext = system.executionContext
  
  // ===== WebSocket Proxy =====
  def wsProxyFlow(upstream: String): Flow[Message, Message, _] = {
    Flow[Message].collect { case TextMessage.Strict(text) =>
      println(s"WS message: $text")
      TextMessage(text) // echo for demo
    }
  }
  
  // ===== Server-Sent Events (SSE) =====
  import akka.http.scaladsl.model.sse.ServerSentEvent
  
  def sseStream: Source[ServerSentEvent, _] = {
    Source.tick(
      scala.concurrent.duration.Duration.Zero,
      scala.concurrent.duration.Duration(5, "seconds"),
      ()
    ).map { _ =>
      ServerSentEvent(s"""{"time":"${java.time.Instant.now()}","status":"ok"}""", "status")
    }
  }
  
  val routes = {
    concat(
      // WebSocket endpoint
      path("ws" / "orders" / Segment) { customerId =>
        handleWebSocketMessages(wsProxyFlow(s"ws://order-service:8080/ws/$customerId"))
      },
      
      // SSE endpoint
      path("events" / "status") {
        get {
          import akka.http.scaladsl.marshalling.sse.EventStreamMarshalling._
          complete(sseStream)
        }
      }
    )
  }
}
```

---

## Step 898: Request Transformation

```scala
// RequestTransformation.scala
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.model._
import akka.http.scaladsl.server.Route
import spray.json._
import DefaultJsonProtocol._

object RequestTransformation {
  
  // ===== Request enrichment =====
  def enrichRequest(request: HttpRequest, userId: String): HttpRequest = {
    request.withHeaders(
      request.headers ++
      List(
        headers.RawHeader("X-User-ID", userId),
        headers.RawHeader("X-Request-Time", java.time.Instant.now().toString),
        headers.RawHeader("X-Gateway-Version", "1.0")
      )
    )
  }
  
  // ===== Response transformation =====
  def wrapResponse(body: String, status: Int, requestId: String): String = {
    s"""{
      "requestId": "$requestId",
      "status": $status,
      "data": $body,
      "timestamp": "${java.time.Instant.now()}"
    }"""
  }
  
  // ===== Field filtering (remove sensitive fields) =====
  def filterSensitiveFields(json: JsValue): JsValue = {
    json match {
      case obj: JsObject =>
        val filtered = obj.fields
          .filterNot { case (key, _) =>
            key == "password" || key == "cardNumber" || key == "ssn"
          }
          .mapValues(filterSensitiveFields)
        JsObject(filtered)
      
      case arr: JsArray =>
        JsArray(arr.elements.map(filterSensitiveFields))
      
      case other => other
    }
  }
  
  // ===== Schema adaptation =====
  // V1 response → V2 response
  def adaptV1toV2(v1Json: JsObject): JsObject = {
    // Example: rename "total_price" to "totalAmount"
    val adapted = v1Json.fields.map {
      case ("total_price", v) => "totalAmount" -> v
      case ("customer_id", v) => "customerId"  -> v
      case other              => other
    }
    JsObject(adapted)
  }
}
```

---

## Step 899: Gateway Monitoring

```scala
// GatewayMonitoring.scala
import io.prometheus.client._

object GatewayMetrics {
  
  val requestsTotal: Counter = Counter.build()
    .name("gateway_requests_total")
    .help("Total gateway requests")
    .labelNames("method", "path", "status", "service")
    .register()
  
  val requestDuration: Histogram = Histogram.build()
    .name("gateway_request_duration_seconds")
    .help("Gateway request duration")
    .labelNames("method", "path", "service")
    .buckets(0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5)
    .register()
  
  val upstreamErrors: Counter = Counter.build()
    .name("gateway_upstream_errors_total")
    .help("Upstream service errors")
    .labelNames("service", "error_type")
    .register()
  
  val circuitBreakerStatus: Gauge = Gauge.build()
    .name("gateway_circuit_breaker_status")
    .help("Circuit breaker status (0=closed, 1=open)")
    .labelNames("service")
    .register()
  
  val activeConnections: Gauge = Gauge.build()
    .name("gateway_active_connections")
    .help("Active connections")
    .register()
  
  val rateLimitExceeded: Counter = Counter.build()
    .name("gateway_rate_limit_exceeded_total")
    .help("Rate limit exceeded count")
    .labelNames("user_id")
    .register()
  
  def recordRequest(method: String, path: String, status: Int, service: String, duration: Double): Unit = {
    requestsTotal.labels(method, path, status.toString, service).inc()
    requestDuration.labels(method, path, service).observe(duration)
  }
}
```

---

## Step 900: Production Gateway Config

```yaml
# kong/kong.yml
# Kong API Gateway configuration

_format_version: "3.0"
_transform: true

services:
  - name: order-service
    url: http://order-service.scala-app.svc.cluster.local:8080
    retries: 3
    connect_timeout: 5000
    write_timeout: 30000
    read_timeout: 30000
    
    routes:
      - name: orders-route
        paths:
          - /api/v1/orders
        methods:
          - GET
          - POST
          - PUT
          - DELETE
        strip_path: false
    
    plugins:
      # Authentication
      - name: jwt
        config:
          secret_is_base64: false
          key_claim_name: kid
      
      # Rate limiting
      - name: rate-limiting
        config:
          minute: 100
          hour: 1000
          policy: local
          fault_tolerant: true
          hide_client_headers: false
      
      # Request transformer
      - name: request-transformer
        config:
          add:
            headers:
              - X-Gateway-Version:1.0
              - X-Service-Name:order-service
      
      # Response transformer (remove sensitive)
      - name: response-transformer
        config:
          remove:
            headers:
              - X-Internal-ID
      
      # Logging
      - name: http-log
        config:
          http_endpoint: http://log-collector:8080/logs
      
      # Prometheus metrics
      - name: prometheus
        config:
          status_code_metrics: true
          latency_metrics: true
          bandwidth_metrics: true

  - name: product-service
    url: http://product-service.scala-app.svc.cluster.local:8080
    
    routes:
      - name: products-route
        paths:
          - /api/v1/products
    
    plugins:
      # Cache for GET requests
      - name: proxy-cache
        config:
          response_code: [200]
          request_method: [GET]
          content_type: [application/json]
          cache_ttl: 300  # 5 minutes
          strategy: memory

consumers:
  - username: mobile-app
    jwt_secrets:
      - secret: super-secret-key
  
  - username: web-app
    jwt_secrets:
      - secret: web-app-secret
```

---

## สรุป Part 90: API Gateway

| Feature | Nginx | Traefik | Kong | Custom |
|---------|-------|---------|------|--------|
| Config | File | Docker labels | Declarative | Code |
| Dynamic | Limited | Auto | Yes | Yes |
| Plugins | Modules | Middleware | Rich | Unlimited |
| Performance | Highest | High | High | Depends |
| Learning | Medium | Easy | Medium | High |

### API Gateway Decisions

| Use | When |
|-----|------|
| Custom | Need full control, complex logic |
| Nginx | High performance, simple routing |
| Traefik | Kubernetes-native, auto-discovery |
| Kong | Plugin ecosystem, enterprise features |

---

## แบบฝึกหัด Part 90

1. **Full Gateway**: implement Scala API Gateway ที่มี JWT auth, rate limiting (token bucket), circuit breaking, และ request logging

2. **Nginx Config**: configure Nginx เป็น API Gateway ที่ handle SSL termination, load balancing ให้ 3 backend services, rate limiting, และ caching

3. **BFF Pattern**: implement Backend-for-Frontend pattern โดย mobile BFF aggregates 3 service calls เป็น single response

4. **GraphQL Federation**: setup GraphQL gateway ที่ stitch schemas จาก Order Service และ Product Service เป็น unified graph

5. **Gateway Metrics**: implement comprehensive metrics สำหรับ gateway ที่ track per-route latency, error rates, upstream health

---

## ไปต่อ: Part 91 — Performance Tuning
ใน Part ถัดไปจะเรียน JVM tuning, GC, Scala-specific optimizations, profiling

[→ Part 91: Performance Tuning](./part-91-performance-tuning.md)
