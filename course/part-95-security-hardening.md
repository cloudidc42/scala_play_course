# Part 95: Security Hardening — Steps 941-950

## บทนำ: Security Hardening

Security hardening ครอบคลุม OWASP Top 10, input validation, SQL injection prevention, authentication, authorization, secure headers และ secrets management

---

## Step 941: OWASP Top 10 for Scala

```scala
// OWASPTop10.scala

/*
===== OWASP Top 10 (2021) for Scala Web Apps =====

1. Broken Access Control     — ต้องตรวจสอบ authorization ทุก request
2. Cryptographic Failures    — ใช้ encryption ที่แข็งแกร่ง, no plaintext secrets
3. Injection                 — SQL, LDAP, Command injection
4. Insecure Design           — Security by design, threat modeling
5. Security Misconfiguration — Defaults, error messages, unnecessary features
6. Vulnerable Components     — Update dependencies, SBOM
7. Auth Failures             — Weak passwords, no MFA, session management
8. Software Integrity        — Supply chain, unsigned dependencies
9. Logging Failures          — Log security events, no sensitive data in logs
10. SSRF                     — Server-side request forgery

Each addressed in following steps.
*/

// ===== A01: Access Control =====
case class AuthContext(userId: Long, roles: Set[String], permissions: Set[String])

object AccessControl {
  
  def hasPermission(ctx: AuthContext, permission: String): Boolean =
    ctx.permissions.contains(permission) || ctx.roles.contains("admin")
  
  def requirePermission(ctx: AuthContext, permission: String): Either[String, Unit] =
    if (hasPermission(ctx, permission)) Right(())
    else Left(s"Access denied: missing permission '$permission'")
  
  // Resource-level authorization
  def canAccessOrder(ctx: AuthContext, order: Order): Either[String, Unit] = {
    if (ctx.roles.contains("admin")) Right(())  // admin can see all
    else if (order.customerId == ctx.userId) Right(())  // owner can see own
    else Left("Access denied: not owner or admin")
  }
}

case class Order(id: Long, customerId: Long, total: Double, status: String)
```

---

## Step 942: Input Validation & Sanitization

```scala
// InputValidation.scala

object InputValidation {
  
  // ===== SQL Injection Prevention =====
  
  // ❌ VULNERABLE: string concatenation
  def unsafeQuery(name: String): String =
    s"SELECT * FROM users WHERE name = '$name'"
  // Input: "'; DROP TABLE users; --"  → DISASTER
  
  // ✅ SAFE: parameterized queries (Doobie example)
  import doobie._
  import doobie.implicits._
  
  def safeQuery(name: String): Query0[(Long, String)] =
    sql"SELECT id, name FROM users WHERE name = $name".query[(Long, String)]
  // Always use interpolation — never string concat for SQL
  
  // ===== XSS Prevention =====
  
  def escapeHtml(input: String): String = input
    .replace("&",  "&amp;")
    .replace("<",  "&lt;")
    .replace(">",  "&gt;")
    .replace("\"", "&quot;")
    .replace("'",  "&#x27;")
  
  // Use Scalatags or similar — safe by default
  // scalatags.Text.tags.p("User input: " + userInput)
  // → auto-escaped: <p>User input: &lt;script&gt;</p>
  
  // ===== Path Traversal Prevention =====
  
  def safePath(baseDir: String, userInput: String): Either[String, java.nio.file.Path] = {
    val base = java.nio.file.Paths.get(baseDir).normalize().toAbsolutePath
    val path = base.resolve(userInput).normalize().toAbsolutePath
    
    if (path.startsWith(base)) Right(path)
    else Left(s"Path traversal detected: $userInput")
  }
  
  // ===== Command Injection Prevention =====
  
  // ❌ VULNERABLE: shell=true with user input
  def unsafeExec(filename: String): String = {
    import scala.sys.process._
    s"ls -la $filename".!!  // NEVER DO THIS with user input
  }
  
  // ✅ SAFE: use args array, no shell
  def safeExec(filename: String): Either[String, String] = {
    // Whitelist allowed characters
    if (!filename.matches("[a-zA-Z0-9._-]+")) {
      Left("Invalid filename")
    } else {
      import scala.sys.process._
      val result = Seq("ls", "-la", filename).!!
      Right(result)
    }
  }
  
  // ===== URL Validation =====
  
  def validateUrl(url: String): Either[String, java.net.URL] = {
    try {
      val parsed = new java.net.URL(url)
      if (Set("http", "https").contains(parsed.getProtocol)) Right(parsed)
      else Left(s"Invalid protocol: ${parsed.getProtocol}")
    } catch {
      case _: java.net.MalformedURLException => Left(s"Malformed URL: $url")
    }
  }
  
  // ===== JSON Input Validation =====
  
  case class CreateOrderRequest(
    customerId: Long,
    items: List[OrderItem],
    note: Option[String]
  )
  
  case class OrderItem(productId: Long, quantity: Int)
  
  def validateCreateOrder(req: CreateOrderRequest): Either[List[String], CreateOrderRequest] = {
    val errors = scala.collection.mutable.ListBuffer.empty[String]
    
    if (req.customerId <= 0) errors += "customerId must be positive"
    if (req.items.isEmpty)   errors += "items must not be empty"
    if (req.items.size > 100) errors += "max 100 items per order"
    
    req.items.foreach { item =>
      if (item.productId <= 0) errors += s"productId must be positive"
      if (item.quantity <= 0 || item.quantity > 10000) errors += s"quantity must be 1-10000"
    }
    
    req.note.foreach { note =>
      if (note.length > 500) errors += "note max 500 characters"
      if (note.contains("<script")) errors += "note contains invalid content"
    }
    
    if (errors.isEmpty) Right(req) else Left(errors.toList)
  }
}
```

---

## Step 943: Authentication

```scala
// Authentication.scala

import javax.crypto.spec.SecretKeySpec
import javax.crypto.Mac
import java.util.Base64
import java.time.Instant

object JWTAuth {
  
  // ===== JWT Generation =====
  def generateToken(
    userId: Long,
    roles: List[String],
    secret: String,
    expirySeconds: Long = 3600
  ): String = {
    val header  = Base64.getUrlEncoder.withoutPadding().encodeToString(
      """{"alg":"HS256","typ":"JWT"}""".getBytes
    )
    val expiry  = Instant.now().getEpochSecond + expirySeconds
    val payload = Base64.getUrlEncoder.withoutPadding().encodeToString(
      s"""{"sub":"$userId","roles":${roles.mkString("[\"","\",\"","\"]")},"exp":$expiry,"iat":${Instant.now().getEpochSecond}}""".getBytes
    )
    val sig = hmacSHA256(s"$header.$payload", secret)
    s"$header.$payload.$sig"
  }
  
  // ===== JWT Validation =====
  case class JWTClaims(userId: Long, roles: List[String], exp: Long, iat: Long)
  
  def validateToken(token: String, secret: String): Either[String, JWTClaims] = {
    val parts = token.split("\\.")
    if (parts.length != 3) return Left("Invalid token format")
    
    val (header, payload, signature) = (parts(0), parts(1), parts(2))
    
    // Verify signature
    val expected = hmacSHA256(s"$header.$payload", secret)
    if (!constantTimeEquals(expected, signature)) return Left("Invalid signature")
    
    // Decode payload
    try {
      val json = new String(Base64.getUrlDecoder.decode(payload))
      import spray.json._
      import DefaultJsonProtocol._
      val obj = json.parseJson.asJsObject
      
      val exp = obj.fields("exp").convertTo[Long]
      if (Instant.now().getEpochSecond > exp) return Left("Token expired")
      
      Right(JWTClaims(
        userId = obj.fields("sub").convertTo[String].toLong,
        roles  = obj.fields("roles").convertTo[List[String]],
        exp    = exp,
        iat    = obj.fields("iat").convertTo[Long]
      ))
    } catch {
      case ex: Exception => Left(s"Token parse error: ${ex.getMessage}")
    }
  }
  
  // ===== Password Hashing (BCrypt) =====
  import org.mindrot.jbcrypt.BCrypt
  
  def hashPassword(plaintext: String): String =
    BCrypt.hashpw(plaintext, BCrypt.gensalt(12))  // cost factor 12
  
  def verifyPassword(plaintext: String, hashed: String): Boolean =
    BCrypt.checkpw(plaintext, hashed)
  
  // ===== CSRF Token =====
  def generateCSRFToken(): String = {
    val bytes = new Array[Byte](32)
    new java.security.SecureRandom().nextBytes(bytes)
    Base64.getUrlEncoder.withoutPadding().encodeToString(bytes)
  }
  
  // Helpers
  private def hmacSHA256(data: String, secret: String): String = {
    val mac = Mac.getInstance("HmacSHA256")
    mac.init(new SecretKeySpec(secret.getBytes("UTF-8"), "HmacSHA256"))
    Base64.getUrlEncoder.withoutPadding().encodeToString(mac.doFinal(data.getBytes("UTF-8")))
  }
  
  // Timing-safe comparison (prevents timing attacks)
  private def constantTimeEquals(a: String, b: String): Boolean = {
    if (a.length != b.length) return false
    var result = 0
    for (i <- a.indices) result |= a(i) ^ b(i)
    result == 0
  }
}
```

---

## Step 944: Secure Headers

```scala
// SecureHeaders.scala
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.server.{Directive0, Route}
import akka.http.scaladsl.model.headers._

object SecureHeaders {
  
  val securityHeaders: Directive0 = {
    respondWithHeaders(
      // Prevent clickjacking
      RawHeader("X-Frame-Options", "DENY"),
      
      // Prevent MIME sniffing
      `X-Content-Type-Options`(Seq()),  // nosniff
      
      // XSS protection (legacy)
      RawHeader("X-XSS-Protection", "1; mode=block"),
      
      // Force HTTPS
      `Strict-Transport-Security`(31536000L, includeSubDomains = true),
      
      // Content Security Policy
      RawHeader("Content-Security-Policy",
        "default-src 'self'; " +
        "script-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; " +
        "style-src 'self' 'unsafe-inline'; " +
        "img-src 'self' data: https:; " +
        "connect-src 'self'; " +
        "frame-ancestors 'none'; " +
        "base-uri 'self';"
      ),
      
      // Referrer policy
      RawHeader("Referrer-Policy", "strict-origin-when-cross-origin"),
      
      // Permissions policy (restrict browser features)
      RawHeader("Permissions-Policy",
        "geolocation=(), microphone=(), camera=(), payment=(), usb=()"
      ),
      
      // Remove server info
      RawHeader("Server", ""),
      
      // Cache control for sensitive pages
      RawHeader("Cache-Control", "no-store, no-cache, must-revalidate"),
      RawHeader("Pragma", "no-cache")
    )
  }
  
  def secureRoute(inner: Route): Route = securityHeaders(inner)
}
```

---

## Step 945: Rate Limiting & DDoS Protection

```scala
// RateLimitingDDoS.scala
import scala.collection.concurrent.TrieMap
import java.util.concurrent.atomic.{AtomicLong, AtomicInteger}
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.model.StatusCodes

class SlidingWindowRateLimiter(windowSeconds: Int, maxRequests: Int) {
  
  // Track request timestamps per key
  private val windows = TrieMap.empty[String, scala.collection.mutable.Queue[Long]]
  
  def isAllowed(key: String): Boolean = {
    val now = System.currentTimeMillis()
    val windowMs = windowSeconds * 1000L
    val windowStart = now - windowMs
    
    val queue = windows.getOrElseUpdate(key, scala.collection.mutable.Queue.empty[Long])
    
    // Remove old entries
    while (queue.headOption.exists(_ < windowStart)) queue.dequeue()
    
    if (queue.size < maxRequests) {
      queue.enqueue(now)
      true
    } else {
      false
    }
  }
}

// ===== IP-based blocking =====
class IPBlocker {
  private val blockedIPs = TrieMap.empty[String, Long]  // IP → expiry timestamp
  private val failedAttempts = TrieMap.empty[String, AtomicInteger]
  
  def recordFailedAttempt(ip: String, maxAttempts: Int = 10, blockSeconds: Int = 300): Unit = {
    val attempts = failedAttempts.getOrElseUpdate(ip, new AtomicInteger(0))
    val count = attempts.incrementAndGet()
    
    if (count >= maxAttempts) {
      val expiry = System.currentTimeMillis() + blockSeconds * 1000L
      blockedIPs.put(ip, expiry)
      attempts.set(0)
      println(s"Blocked IP: $ip for ${blockSeconds}s")
    }
  }
  
  def isBlocked(ip: String): Boolean = {
    blockedIPs.get(ip) match {
      case None => false
      case Some(expiry) =>
        if (System.currentTimeMillis() > expiry) {
          blockedIPs.remove(ip)
          false
        } else true
    }
  }
}
```

---

## Step 946: Secrets Management

```scala
// SecretsManagement.scala

/*
===== Secrets Anti-patterns =====
❌ Hardcode in source code
❌ Environment variables (visible in process list)
❌ Log secrets accidentally
❌ Store in Git
❌ Plaintext in config files

===== Secrets Best Practices =====
✅ AWS Secrets Manager / HashiCorp Vault
✅ Kubernetes Secrets (encrypted at rest)
✅ Environment-specific secrets
✅ Rotation
✅ Audit trail
*/

import software.amazon.awssdk.services.secretsmanager.SecretsManagerClient
import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueRequest
import scala.concurrent.duration._
import java.util.concurrent.{Executors, ScheduledExecutorService, TimeUnit}

class SecretsCache(client: SecretsManagerClient) {
  
  private case class CachedSecret(value: String, cachedAt: Long, ttlMs: Long) {
    def isExpired: Boolean = System.currentTimeMillis() > cachedAt + ttlMs
  }
  
  private val cache = scala.collection.concurrent.TrieMap.empty[String, CachedSecret]
  
  def getSecret(
    secretArn: String,
    ttl: Duration = 5.minutes
  ): Either[Throwable, String] = {
    cache.get(secretArn) match {
      case Some(cached) if !cached.isExpired => Right(cached.value)
      case _ =>
        try {
          val req = GetSecretValueRequest.builder().secretId(secretArn).build()
          val value = client.getSecretValue(req).secretString()
          cache.put(secretArn, CachedSecret(value, System.currentTimeMillis(), ttl.toMillis))
          Right(value)
        } catch {
          case ex: Exception => Left(ex)
        }
    }
  }
  
  // Proactive refresh (before TTL expires)
  private val scheduler: ScheduledExecutorService = Executors.newScheduledThreadPool(1)
  
  def startAutoRefresh(): Unit = {
    scheduler.scheduleAtFixedRate(
      () => {
        val expiringSoon = cache.filter { case (_, v) =>
          v.cachedAt + v.ttlMs - System.currentTimeMillis() < 60000  // 1 minute before expiry
        }
        expiringSoon.keys.foreach(arn => getSecret(arn))
      },
      30, 30, TimeUnit.SECONDS
    )
  }
}

// ===== No secrets in logs =====
object SecureLogging {
  
  case class SensitiveData(value: String) {
    override def toString: String = "***REDACTED***"
  }
  
  // Redact patterns in log messages
  def redactSensitive(message: String): String = message
    .replaceAll("""password["\s:=]+[^,}\s"]+""", "password=***")
    .replaceAll("""token["\s:=]+[^,}\s"]+""", "token=***")
    .replaceAll("""(\d{4})[- ]?(\d{4})[- ]?(\d{4})[- ]?(\d{4})""", "$1-****-****-$4")  // CC
    .replaceAll("""Authorization:\s*(Bearer|Basic)\s+\S+""", "Authorization: ***")
}
```

---

## Step 947: HTTPS & TLS Configuration

```scala
// TLSConfiguration.scala
import akka.actor.typed.ActorSystem
import akka.actor.typed.scaladsl.Behaviors
import akka.http.scaladsl.{Http, HttpsConnectionContext}
import akka.http.scaladsl.server.Directives._
import javax.net.ssl._
import java.security.KeyStore

object TLSConfig {
  
  def createSSLContext(
    keystorePath: String,
    keystorePassword: String,
    truststorePath: Option[String] = None
  ): SSLContext = {
    // Load keystore
    val ks = KeyStore.getInstance("PKCS12")
    val ksStream = new java.io.FileInputStream(keystorePath)
    ks.load(ksStream, keystorePassword.toCharArray)
    ksStream.close()
    
    // Key manager
    val kmf = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm)
    kmf.init(ks, keystorePassword.toCharArray)
    
    // Trust manager (optional for mutual TLS)
    val tmf = truststorePath.map { path =>
      val ts = KeyStore.getInstance("JKS")
      val tsStream = new java.io.FileInputStream(path)
      ts.load(tsStream, "changeit".toCharArray)
      tsStream.close()
      val factory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm)
      factory.init(ts)
      factory
    }
    
    val ctx = SSLContext.getInstance("TLS")
    ctx.init(
      kmf.getKeyManagers,
      tmf.map(_.getTrustManagers).orNull,
      new java.security.SecureRandom()
    )
    ctx
  }
  
  def httpsServer()(implicit system: ActorSystem[_]): Unit = {
    val sslContext = createSSLContext(
      "/etc/ssl/keystore.p12",
      sys.env.getOrElse("KEYSTORE_PASSWORD", "changeit")
    )
    
    val httpsContext = ConnectionContext.httpsServer(sslContext)
    
    Http().newServerAt("0.0.0.0", 443)
      .enableHttps(httpsContext)
      .bind(get { complete("Secure!") })
  }
}
```

---

## Step 948: Dependency Security Scanning

```scala
// DependencyScanning.scala

/*
===== Dependency Security =====

1. OWASP Dependency Check:
   sbt-dependency-check plugin

2. Snyk:
   snyk test --file=build.sbt

3. GitHub Dependabot (auto PRs)

4. SBOM (Software Bill of Materials)
*/

// build.sbt
/*
addSbtPlugin("net.vonbuchholtz" % "sbt-dependency-check" % "5.1.0")

dependencyCheckSettings := Seq(
  dependencyCheckFormat := "ALL",
  dependencyCheckFailBuildOnCVSS := 7.0,  // Fail on HIGH+ severity
  dependencyCheckOutputDirectory := target.value / "dependency-check"
)
*/

// ===== Library audit =====
object LibraryAudit {
  
  // Known vulnerable versions (simplified example)
  val knownVulnerabilities = Map(
    "log4j-core" -> Set("2.0.0", "2.14.1"),      // Log4Shell CVE-2021-44228
    "jackson-databind" -> Set("2.9.0", "2.9.10"), // Various CVEs
    "commons-text" -> Set("1.9.0")                // CVE-2022-42889
  )
  
  def checkDependency(name: String, version: String): Either[String, Unit] = {
    knownVulnerabilities.get(name) match {
      case Some(vulnVersions) if vulnVersions.contains(version) =>
        Left(s"VULNERABLE: $name:$version — update immediately!")
      case _ =>
        Right(())
    }
  }
}
```

---

## Step 949: Security Audit Logging

```scala
// SecurityAuditLogging.scala
import org.slf4j.{Logger, LoggerFactory, MDC}

object SecurityAuditLog {
  
  private val auditLogger: Logger = LoggerFactory.getLogger("SECURITY_AUDIT")
  
  sealed trait SecurityEvent
  case class LoginAttempt(userId: String, success: Boolean, ip: String)     extends SecurityEvent
  case class AccessDenied(userId: String, resource: String, action: String) extends SecurityEvent
  case class DataExport(userId: String, dataType: String, recordCount: Int) extends SecurityEvent
  case class PasswordChanged(userId: String)                                 extends SecurityEvent
  case class SuspiciousActivity(userId: String, description: String)        extends SecurityEvent
  
  def log(event: SecurityEvent, requestId: String = java.util.UUID.randomUUID().toString): Unit = {
    MDC.put("requestId", requestId)
    MDC.put("timestamp", java.time.Instant.now().toString)
    
    event match {
      case LoginAttempt(userId, success, ip) =>
        val msg = s"""{"event":"LOGIN","userId":"$userId","success":$success,"ip":"$ip"}"""
        if (success) auditLogger.info(msg)
        else         auditLogger.warn(msg)
      
      case AccessDenied(userId, resource, action) =>
        auditLogger.warn(s"""{"event":"ACCESS_DENIED","userId":"$userId","resource":"$resource","action":"$action"}""")
      
      case DataExport(userId, dataType, count) =>
        auditLogger.info(s"""{"event":"DATA_EXPORT","userId":"$userId","dataType":"$dataType","recordCount":$count}""")
      
      case PasswordChanged(userId) =>
        auditLogger.info(s"""{"event":"PASSWORD_CHANGED","userId":"$userId"}""")
      
      case SuspiciousActivity(userId, desc) =>
        auditLogger.error(s"""{"event":"SUSPICIOUS","userId":"$userId","description":"$desc"}""")
    }
    
    MDC.clear()
  }
}
```

---

## Step 950: Security Testing

```scala
// SecurityTesting.scala

/*
===== Security Testing Checklist =====

1. SAST (Static Analysis):
   - Scala WartRemover: catch bad practices
   - SpotBugs: bytecode analysis
   - SonarQube: code quality + security

2. DAST (Dynamic Analysis):
   - OWASP ZAP: automated web app scanning
   - Burp Suite: manual + automated

3. Dependency Scanning:
   - sbt-dependency-check
   - Snyk

4. Infrastructure:
   - Trivy: container image scanning
   - Checkov: IaC scanning

5. Pen Testing:
   - Manual testing of auth flows
   - Input fuzzing
*/

// build.sbt security plugins
/*
addSbtPlugin("org.wartremover" % "sbt-wartremover" % "3.1.6")
wartremoverErrors ++= Warts.unsafe  // Catch unsafe operations
wartremoverErrors ++= Seq(
  Wart.StringPlusAny,     // Prevent accidental toString
  Wart.EitherProjectionPartial,  // Unsafe .get on Either
  Wart.OptionPartial,     // Unsafe .get on Option
  Wart.TraversableOps,    // Unsafe head/tail
  Wart.Throw,             // Force typed errors
  Wart.Return             // No return statements
)
*/

// Security-focused unit tests
class AuthSecuritySpec extends org.scalatest.flatspec.AnyFlatSpec {
  
  "JWTAuth" should "reject expired tokens" in {
    val token = JWTAuth.generateToken(1L, List("user"), "secret", expirySeconds = -1)
    JWTAuth.validateToken(token, "secret").isLeft shouldBe true
  }
  
  it should "reject tokens with wrong secret" in {
    val token = JWTAuth.generateToken(1L, List("user"), "correct-secret")
    JWTAuth.validateToken(token, "wrong-secret").isLeft shouldBe true
  }
  
  it should "reject malformed tokens" in {
    JWTAuth.validateToken("not.a.jwt", "secret").isLeft shouldBe true
    JWTAuth.validateToken("", "secret").isLeft shouldBe true
  }
  
  "InputValidation" should "detect path traversal" in {
    InputValidation.safePath("/app/files", "../etc/passwd").isLeft shouldBe true
    InputValidation.safePath("/app/files", "documents/file.pdf").isRight shouldBe true
  }
  
  it should "reject SQL injection attempts" in {
    // With parameterized queries, injection attempts are harmless
    // Test that user input is never directly concatenated
    val malicious = "'; DROP TABLE users; --"
    InputValidation.safeQuery(malicious) // should produce safe parameterized query
  }
}

import org.scalatest.matchers.should.Matchers
class AuthSecuritySpec2 extends AuthSecuritySpec with Matchers
```

---

## สรุป Part 95: Security Hardening

| Category | Tool/Technique | Priority |
|----------|----------------|---------|
| Input Validation | Whitelist, Regex | Critical |
| SQL Injection | Parameterized queries | Critical |
| Authentication | BCrypt + JWT | Critical |
| Secrets | AWS Secrets Manager | High |
| Headers | CSP, HSTS, X-Frame | High |
| Dependencies | sbt-dependency-check | High |
| Audit Logging | Structured logs | Medium |
| TLS | TLS 1.2+, strong ciphers | High |

---

## แบบฝึกหัด Part 95

1. **Input Validation**: implement comprehensive validator สำหรับ user registration: email format, password strength (min 12 chars, uppercase, digit, special), username whitelist `[a-zA-Z0-9_]`

2. **JWT Middleware**: implement Akka HTTP directive ที่ extract และ validate JWT, attach `AuthContext` ไปยัง request, return 401 เมื่อ invalid

3. **SQL Injection Test**: เขียน test ที่ demonstrate ว่า Doobie parameterized queries ป้องกัน SQL injection ได้ โดยใช้ malicious input `'; DROP TABLE orders; --`

4. **Secrets Rotation**: implement `SecretsCache` ที่มี auto-refresh mechanism สำหรับ database credentials และ handle rotation gracefully

5. **Security Headers**: เพิ่ม security middleware ใน Akka HTTP ที่เพิ่ม CSP, HSTS, X-Frame-Options ทุก response และ เขียน test verify headers

---

## ไปต่อ: Part 96 — Scalability Patterns
[→ Part 96: Scalability Patterns](./part-96-scalability-patterns.md)
