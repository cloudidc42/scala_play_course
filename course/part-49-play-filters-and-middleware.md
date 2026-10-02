# Part 49: Play Filters and Middleware

## Steps 481-490: Filters, Logging Filter, CORS Filter, Compression, Custom Filters

---

## Step 481: Play Filters คืออะไร?

Filters ใน Play คือ middleware ที่ intercept HTTP requests และ responses ก่อนที่จะถึง Controller หรือหลังจาก Controller ตอบกลับ

```
Request Flow:
Browser → Filter 1 → Filter 2 → Filter 3 → Controller → View

Response Flow:
View → Controller → Filter 3 → Filter 2 → Filter 1 → Browser
```

### Filter Interface

```scala
trait Filter {
  def apply(
    nextFilter: RequestHeader => Future[Result]
  )(requestHeader: RequestHeader): Future[Result]
}
```

### build.sbt

```scala
libraryDependencies ++= Seq(
  guice,
  filters  // Play filters module
)
```

---

## Step 482: Built-in Filters

```hocon
# conf/application.conf - Enable built-in filters

play.filters.enabled = [
  # Security headers (XSS protection, HSTS, etc.)
  "play.filters.headers.SecurityHeadersFilter",

  # CORS (Cross-Origin Resource Sharing)
  "play.filters.cors.CORSFilter",

  # CSRF protection
  "play.filters.csrf.CSRFFilter",

  # Allowed hosts filter
  "play.filters.hosts.AllowedHostsFilter",

  # GZip compression
  "play.filters.gzip.GzipFilter",

  # Redirect HTTP to HTTPS
  # "play.filters.https.RedirectHttpsFilter"
]

# Security Headers configuration
play.filters.headers {
  contentSecurityPolicy = "default-src 'self'"
  xFrameOptions = "DENY"
  xssProtection = "1; mode=block"
  contentTypeOptions = "nosniff"
  permittedCrossDomainPolicies = "master-only"
  referrerPolicy = "origin-when-cross-origin, strict-origin-when-cross-origin"
}

# Allowed Hosts
play.filters.hosts {
  allowed = ["localhost", "127.0.0.1", ".example.com"]
}

# CORS configuration
play.filters.cors {
  pathPrefixes = ["/api"]
  allowedOrigins = ["http://localhost:3000", "https://myapp.com"]
  allowedHttpMethods = ["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"]
  allowedHttpHeaders = ["Accept", "Content-Type", "Authorization"]
  exposedHeaders = ["X-Total-Count", "X-Request-Id"]
  supportsCredentials = true
  preflightMaxAge = 3 days
}

# GZip filter
play.filters.gzip {
  contentType {
    whiteList = ["text/*", "application/json", "application/javascript"]
    blackList = ["image/*", "video/*", "audio/*"]
  }
  chunkedThreshold = 102400  # 100KB
  bufferSize = 8192
}
```

---

## Step 483: Custom Logging Filter

```scala
// app/filters/LoggingFilter.scala
package filters

import javax.inject.*
import org.apache.pekko.stream.Materializer
import play.api.Logger
import play.api.mvc.*
import scala.concurrent.*

@Singleton
class LoggingFilter @Inject()(implicit
  val mat: Materializer,
  ec: ExecutionContext
) extends Filter {

  private val logger = Logger("access")

  override def apply(
    next: RequestHeader => Future[Result]
  )(request: RequestHeader): Future[Result] = {
    val startTime = System.currentTimeMillis()
    val requestId = java.util.UUID.randomUUID().toString.take(8)

    // Log request
    logger.info(s"[$requestId] ${request.method} ${request.uri} " +
                s"from ${request.remoteAddress}")

    // Execute next filter/controller
    next(request).map { result =>
      val duration = System.currentTimeMillis() - startTime
      val status = result.header.status

      // Log response
      val logLevel = if (status >= 500) "ERROR"
                     else if (status >= 400) "WARN"
                     else "INFO"

      logger.info(s"[$requestId] ${status} ${duration}ms " +
                  s"${request.method} ${request.path}")

      // เพิ่ม request ID header ใน response
      result.withHeaders(
        "X-Request-Id"    -> requestId,
        "X-Response-Time" -> s"${duration}ms"
      )
    }.recover {
      case ex: Exception =>
        val duration = System.currentTimeMillis() - startTime
        logger.error(s"[$requestId] ERROR ${duration}ms " +
                     s"${request.method} ${request.path}: ${ex.getMessage}")
        throw ex
    }
  }
}
```

### Structured Logging Filter

```scala
// app/filters/StructuredLoggingFilter.scala
package filters

import javax.inject.*
import org.apache.pekko.stream.Materializer
import play.api.libs.json.*
import play.api.mvc.*
import scala.concurrent.*

@Singleton
class StructuredLoggingFilter @Inject()(implicit
  val mat: Materializer,
  ec: ExecutionContext
) extends Filter {

  private val logger = play.api.Logger("structured-access")

  override def apply(
    next: RequestHeader => Future[Result]
  )(request: RequestHeader): Future[Result] = {
    val startTime = System.nanoTime()
    val requestId = java.util.UUID.randomUUID().toString

    next(request).map { result =>
      val durationMs = (System.nanoTime() - startTime) / 1_000_000

      // Structured log as JSON
      val logEntry = Json.obj(
        "timestamp"   -> java.time.Instant.now().toString,
        "requestId"   -> requestId,
        "method"      -> request.method,
        "path"        -> request.path,
        "queryString" -> request.rawQueryString,
        "status"      -> result.header.status,
        "durationMs"  -> durationMs,
        "remoteAddr"  -> request.remoteAddress,
        "userAgent"   -> request.headers.get("User-Agent"),
        "userId"      -> request.session.get("userId")
      )

      logger.info(Json.stringify(logEntry))
      result.withHeaders("X-Request-Id" -> requestId)
    }
  }
}
```

---

## Step 484: Authentication Filter

```scala
// app/filters/AuthFilter.scala
package filters

import javax.inject.*
import org.apache.pekko.stream.Materializer
import play.api.mvc.*
import play.api.libs.json.*
import scala.concurrent.*

@Singleton
class AuthFilter @Inject()(implicit
  val mat: Materializer,
  ec: ExecutionContext
) extends Filter {

  // Paths ที่ต้องการ authentication
  private val protectedPaths = Set(
    "/api/users",
    "/api/articles/create",
    "/dashboard"
  )

  // Paths ที่ skip auth check
  private val publicPaths = Set(
    "/api/auth/login",
    "/api/auth/register",
    "/assets",
    "/public"
  )

  override def apply(
    next: RequestHeader => Future[Result]
  )(request: RequestHeader): Future[Result] = {
    val path = request.path

    // Skip auth สำหรับ public paths
    if (publicPaths.exists(p => path.startsWith(p))) {
      next(request)
    } else if (protectedPaths.exists(p => path.startsWith(p))) {
      // ตรวจสอบ authentication
      val authenticated = checkAuth(request)

      if (authenticated) {
        next(request)
      } else if (path.startsWith("/api")) {
        // API: return JSON error
        Future.successful(
          Unauthorized(Json.obj(
            "error"   -> "Unauthorized",
            "message" -> "Authentication required"
          ))
        )
      } else {
        // Web: redirect to login
        Future.successful(
          play.api.mvc.Results.Redirect("/login")
            .flashing("error" -> "กรุณาเข้าสู่ระบบก่อน")
        )
      }
    } else {
      next(request)
    }
  }

  private def checkAuth(request: RequestHeader): Boolean = {
    // ตรวจสอบ session สำหรับ web requests
    request.session.get("userId").isDefined ||
    // ตรวจสอบ Bearer token สำหรับ API requests
    request.headers.get("Authorization").exists(_.startsWith("Bearer "))
  }
}
```

---

## Step 485: Rate Limit Filter

```scala
// app/filters/RateLimitFilter.scala
package filters

import javax.inject.*
import org.apache.pekko.stream.Materializer
import play.api.mvc.*
import play.api.libs.json.*
import scala.concurrent.*
import java.util.concurrent.ConcurrentHashMap
import java.util.concurrent.atomic.AtomicLong

case class RateLimitConfig(
  maxRequests: Int,
  windowSeconds: Long
)

@Singleton
class RateLimitFilter @Inject()(implicit
  val mat: Materializer,
  ec: ExecutionContext
) extends Filter {

  // Rate limit configs สำหรับ different paths
  private val configs = Map(
    "/api" -> RateLimitConfig(maxRequests = 100, windowSeconds = 60),
    "/"    -> RateLimitConfig(maxRequests = 1000, windowSeconds = 60)
  )

  private case class Counter(count: AtomicLong, windowStart: Long)
  private val counters = new ConcurrentHashMap[String, Counter]()

  override def apply(
    next: RequestHeader => Future[Result]
  )(request: RequestHeader): Future[Result] = {
    val config = configs.find { case (prefix, _) =>
      request.path.startsWith(prefix)
    }.map(_._2).getOrElse(configs("/"))

    val key = s"${request.remoteAddress}:${request.path.split("/")(1)}"
    val now = System.currentTimeMillis() / 1000

    val counter = counters.computeIfAbsent(key, _ => Counter(new AtomicLong(0), now))

    // Reset หากหมดช่วงเวลา
    if (now - counter.windowStart >= config.windowSeconds) {
      counters.put(key, Counter(new AtomicLong(0), now))
    }

    val currentCount = counter.count.incrementAndGet()

    if (currentCount > config.maxRequests) {
      val retryAfter = config.windowSeconds - (now - counter.windowStart)
      Future.successful(
        TooManyRequests(Json.obj(
          "error"       -> "Too Many Requests",
          "retryAfter"  -> retryAfter,
          "limit"       -> config.maxRequests,
          "window"      -> config.windowSeconds
        )).withHeaders(
          "X-RateLimit-Limit"     -> config.maxRequests.toString,
          "X-RateLimit-Remaining" -> "0",
          "X-RateLimit-Reset"     -> (counter.windowStart + config.windowSeconds).toString,
          "Retry-After"           -> retryAfter.toString
        )
      )
    } else {
      next(request).map { result =>
        result.withHeaders(
          "X-RateLimit-Limit"     -> config.maxRequests.toString,
          "X-RateLimit-Remaining" -> (config.maxRequests - currentCount).toString
        )
      }
    }
  }
}
```

---

## Step 486: Request Timeout Filter

```scala
// app/filters/TimeoutFilter.scala
package filters

import javax.inject.*
import org.apache.pekko.stream.Materializer
import org.apache.pekko.actor.ActorSystem
import play.api.mvc.*
import play.api.libs.json.*
import scala.concurrent.*
import scala.concurrent.duration.*

@Singleton
class TimeoutFilter @Inject()(implicit
  val mat: Materializer,
  system: ActorSystem,
  ec: ExecutionContext
) extends Filter {

  private val defaultTimeout = 30.seconds
  private val apiTimeout = 10.seconds

  override def apply(
    next: RequestHeader => Future[Result]
  )(request: RequestHeader): Future[Result] = {
    val timeout = if (request.path.startsWith("/api")) apiTimeout
                  else defaultTimeout

    val responseFuture = next(request)

    // สร้าง timeout future
    val timeoutFuture = org.apache.pekko.pattern.after(timeout, system.scheduler) {
      Future.successful(
        ServiceUnavailable(Json.obj(
          "error"   -> "Request timeout",
          "message" -> s"Request timed out after ${timeout.toSeconds}s"
        ))
      )
    }

    // Return whichever finishes first
    Future.firstCompletedOf(Seq(responseFuture, timeoutFuture))
  }
}
```

---

## Step 487: Header Manipulation Filter

```scala
// app/filters/ApiVersionFilter.scala
package filters

import javax.inject.*
import org.apache.pekko.stream.Materializer
import play.api.mvc.*
import scala.concurrent.*

@Singleton
class ApiVersionFilter @Inject()(implicit
  val mat: Materializer,
  ec: ExecutionContext
) extends Filter {

  override def apply(
    next: RequestHeader => Future[Result]
  )(request: RequestHeader): Future[Result] = {
    next(request).map { result =>
      // เพิ่ม API version headers ใน response
      result.withHeaders(
        "X-API-Version"     -> "1.0",
        "X-Powered-By"      -> "Play Framework 3.0",
        "X-Request-Time"    -> java.time.Instant.now().toString
      )
    }
  }
}

// Cache Control Filter
@Singleton
class CacheControlFilter @Inject()(implicit
  val mat: Materializer,
  ec: ExecutionContext
) extends Filter {

  override def apply(
    next: RequestHeader => Future[Result]
  )(request: RequestHeader): Future[Result] = {
    next(request).map { result =>
      val path = request.path

      // กำหนด cache policy ตาม path
      val cacheControl = if (path.startsWith("/assets")) {
        "public, max-age=31536000, immutable"  // 1 year สำหรับ static assets
      } else if (path.startsWith("/api")) {
        "no-store"  // ไม่ cache API responses
      } else {
        "no-cache, must-revalidate"  // HTML pages
      }

      result.withHeaders("Cache-Control" -> cacheControl)
    }
  }
}
```

---

## Step 488: Filters Configuration

```scala
// app/filters/Filters.scala
package filters

import javax.inject.*
import play.api.http.DefaultHttpFilters
import play.filters.cors.CORSFilter
import play.filters.csrf.CSRFFilter
import play.filters.gzip.GzipFilter
import play.filters.headers.SecurityHeadersFilter

/**
 * Configure all filters สำหรับ application
 * filters จะทำงานตามลำดับที่กำหนด
 */
@Singleton
class Filters @Inject()(
  loggingFilter: LoggingFilter,
  rateLimitFilter: RateLimitFilter,
  corsFilter: CORSFilter,
  csrfFilter: CSRFFilter,
  securityHeadersFilter: SecurityHeadersFilter,
  gzipFilter: GzipFilter
) extends DefaultHttpFilters(
  // ลำดับของ filters สำคัญมาก
  // 1. Logging ก่อน (capture all requests)
  loggingFilter,
  // 2. Rate limiting ก่อน auth (ป้องกัน brute force)
  rateLimitFilter,
  // 3. CORS headers
  corsFilter,
  // 4. CSRF check
  csrfFilter,
  // 5. Security headers
  securityHeadersFilter,
  // 6. GZip (last - compress final response)
  gzipFilter
)
```

```hocon
# conf/application.conf
# Register custom Filters class
play.http.filters = "filters.Filters"
```

---

## Step 489: Environment-specific Filters

```scala
// app/filters/DevFilters.scala
package filters

import javax.inject.*
import play.api.http.DefaultHttpFilters
import play.api.{Environment, Mode}
import play.filters.cors.CORSFilter
import play.filters.headers.SecurityHeadersFilter

@Singleton
class DevFilters @Inject()(
  loggingFilter: LoggingFilter,
  corsFilter: CORSFilter
) extends DefaultHttpFilters(
  loggingFilter,
  corsFilter
)

// app/filters/ProdFilters.scala
@Singleton
class ProdFilters @Inject()(
  loggingFilter: LoggingFilter,
  rateLimitFilter: RateLimitFilter,
  corsFilter: CORSFilter,
  securityHeadersFilter: SecurityHeadersFilter
) extends DefaultHttpFilters(
  loggingFilter,
  rateLimitFilter,
  corsFilter,
  securityHeadersFilter
)
```

```scala
// app/Module.scala
package app

import com.google.inject.AbstractModule
import play.api.{Environment, Mode}
import play.api.http.HttpFilters

class Module(environment: Environment, configuration: play.api.Configuration)
  extends AbstractModule {

  override def configure(): Unit = {
    environment.mode match {
      case Mode.Dev  => bind(classOf[HttpFilters]).to(classOf[filters.DevFilters])
      case Mode.Test => bind(classOf[HttpFilters]).to(classOf[filters.DevFilters])
      case Mode.Prod => bind(classOf[HttpFilters]).to(classOf[filters.ProdFilters])
    }
  }
}
```

---

## Step 490: Custom Filter Testing

```scala
// test/filters/LoggingFilterSpec.scala
package filters

import org.scalatestplus.play.*
import org.scalatestplus.play.guice.GuiceOneAppPerTest
import play.api.test.*
import play.api.test.Helpers.*

class LoggingFilterSpec extends PlaySpec with GuiceOneAppPerTest {

  "LoggingFilter" should {
    "add X-Request-Id header to responses" in {
      val request = FakeRequest(GET, "/")
      val result = route(app, request).get

      // ตรวจสอบว่ามี X-Request-Id header
      header("X-Request-Id", result) mustBe defined
    }

    "add X-Response-Time header" in {
      val request = FakeRequest(GET, "/")
      val result = route(app, request).get

      val responseTime = header("X-Response-Time", result)
      responseTime mustBe defined
      responseTime.get must endWith("ms")
    }
  }
}

// test/filters/RateLimitFilterSpec.scala
class RateLimitFilterSpec extends PlaySpec with GuiceOneAppPerTest {

  "RateLimitFilter" should {
    "allow requests within limit" in {
      val request = FakeRequest(GET, "/api/articles")
      val result = route(app, request).get

      status(result) mustNot be(TOO_MANY_REQUESTS)
      header("X-RateLimit-Limit", result) mustBe defined
    }
  }
}
```

---

## สรุป Part 49

| Filter Type | Class | Purpose |
|-------------|-------|---------|
| Logging | Custom `Filter` | Log all requests/responses |
| Authentication | Custom `Filter` | Check session/token |
| Rate Limiting | Custom `Filter` | ป้องกัน abuse |
| CORS | `CORSFilter` | Cross-origin requests |
| CSRF | `CSRFFilter` | Form security |
| Security Headers | `SecurityHeadersFilter` | XSS, Clickjacking protection |
| GZip | `GzipFilter` | Compress responses |
| Timeout | Custom `Filter` | ป้องกัน long-running requests |

---

## แบบฝึกหัด Part 49

1. **Audit Log Filter**: สร้าง filter ที่บันทึก audit log สำหรับ write operations (POST/PUT/PATCH/DELETE) ลงใน database พร้อม user identity

2. **IP Whitelist Filter**: สร้าง filter ที่อนุญาตเฉพาะ IP ที่อยู่ใน whitelist สำหรับ admin routes และ block ทุก request อื่น

3. **Request Dedup Filter**: สร้าง filter ที่ตรวจสอบ idempotency key (X-Idempotency-Key header) และ cache response เพื่อป้องกัน duplicate POST requests

4. **Content Negotiation Filter**: สร้าง filter ที่ตรวจสอบ Accept header และ return 406 Not Acceptable ถ้า client ขอ format ที่ไม่รองรับ

5. **Maintenance Mode Filter**: สร้าง filter ที่ check configuration flag และ return 503 Service Unavailable ด้วย HTML page เมื่อ application อยู่ใน maintenance mode

---

[→ ไปยัง Part 50: Play Testing](part-50-play-testing.md)
