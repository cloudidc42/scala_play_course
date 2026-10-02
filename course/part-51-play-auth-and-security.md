# Part 51: Play Auth and Security

## Steps 501-510: JWT Auth, Session Management, OAuth2, Role-Based Access, Security Headers

---

## Step 501: Authentication Strategies

```
Authentication Options ใน Play:
├── Session-based: เก็บ userId ใน encrypted cookie
├── JWT (JSON Web Token): stateless token ใน Authorization header
├── OAuth2: delegate auth ไปยัง Google, GitHub, etc.
├── API Key: simple key ใน header (สำหรับ API)
└── Basic Auth: username:password ใน header (ไม่แนะนำ)
```

### build.sbt

```scala
libraryDependencies ++= Seq(
  guice,
  "com.github.jwt-scala"   %% "jwt-play-json" % "10.0.1",
  "org.mindrot"             % "jbcrypt"        % "0.4",
  "com.auth0"               % "jwks-rsa"       % "0.22.1"
)
```

---

## Step 502: Password Hashing

```scala
// app/utils/PasswordHasher.scala
package utils

import org.mindrot.jbcrypt.BCrypt

object PasswordHasher {

  // Hash password ด้วย BCrypt
  def hash(password: String): String =
    BCrypt.hashpw(password, BCrypt.gensalt(12))  // cost factor 12

  // ตรวจสอบ password
  def verify(password: String, hash: String): Boolean =
    BCrypt.checkpw(password, hash)

  // ตรวจสอบว่า hash ต้องการ upgrade
  def needsRehash(hash: String): Boolean = {
    val currentCost = BCrypt.getRounds(hash)
    currentCost < 12
  }
}

// ใช้งาน
val hash = PasswordHasher.hash("mySecurePassword123!")
val isValid = PasswordHasher.verify("mySecurePassword123!", hash)
```

---

## Step 503: JWT Authentication

```scala
// app/utils/JwtUtils.scala
package utils

import pdi.jwt.*
import pdi.jwt.algorithms.JwtHmacAlgorithm
import play.api.libs.json.*
import scala.util.*

object JwtUtils {

  private val secret = sys.env.getOrElse("JWT_SECRET", "development-secret-change-in-production")
  private val algorithm = JwtAlgorithm.HS256
  private val expirationSeconds = 86400L  // 24 hours

  case class JwtClaims(
    userId: Long,
    username: String,
    role: String,
    iat: Long = System.currentTimeMillis() / 1000,
    exp: Long = System.currentTimeMillis() / 1000 + expirationSeconds
  )

  implicit val claimsFormat: Format[JwtClaims] = Json.format[JwtClaims]

  // สร้าง JWT token
  def createToken(userId: Long, username: String, role: String): String = {
    val claims = Json.obj(
      "userId"   -> userId,
      "username" -> username,
      "role"     -> role,
      "iat"      -> System.currentTimeMillis() / 1000,
      "exp"      -> (System.currentTimeMillis() / 1000 + expirationSeconds)
    )
    JwtJson.encode(claims, secret, algorithm)
  }

  // Validate และ decode token
  def validateToken(token: String): Either[String, JwtClaims] = {
    JwtJson.decode(token, secret, Seq(algorithm)) match {
      case Success(claim) =>
        Json.parse(claim.toJson).validate[JwtClaims] match {
          case JsSuccess(claims, _) =>
            if (claims.exp > System.currentTimeMillis() / 1000)
              Right(claims)
            else
              Left("Token expired")
          case JsError(errors) =>
            Left(s"Invalid token format: $errors")
        }
      case Failure(ex) =>
        Left(s"Invalid token: ${ex.getMessage}")
    }
  }

  // Refresh token (สร้าง token ใหม่จาก claims เดิม)
  def refreshToken(token: String): Either[String, String] =
    validateToken(token).map { claims =>
      createToken(claims.userId, claims.username, claims.role)
    }
}
```

---

## Step 504: JWT Authentication Action

```scala
// app/actions/JwtAction.scala
package actions

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*
import utils.JwtUtils
import scala.concurrent.*

// Custom Request ที่มี user info จาก JWT
class JwtRequest[A](
  val userId: Long,
  val username: String,
  val role: String,
  request: Request[A]
) extends WrappedRequest[A](request)

// Action Builder สำหรับ JWT authentication
class JwtAction @Inject()(
  parser: BodyParsers.Default,
  implicit val ec: ExecutionContext
) extends ActionBuilder[JwtRequest, AnyContent] {

  override def parser: BodyParser[AnyContent] = parser

  override def invokeBlock[A](
    request: Request[A],
    block: JwtRequest[A] => Future[Result]
  ): Future[Result] = {
    // อ่าน token จาก Authorization header
    extractToken(request) match {
      case Some(token) =>
        JwtUtils.validateToken(token) match {
          case Right(claims) =>
            val jwtRequest = new JwtRequest(claims.userId, claims.username, claims.role, request)
            block(jwtRequest)
          case Left(error) =>
            Future.successful(
              Unauthorized(Json.obj(
                "error"   -> "Invalid token",
                "message" -> error
              ))
            )
        }
      case None =>
        Future.successful(
          Unauthorized(Json.obj(
            "error"   -> "Missing token",
            "message" -> "Authorization header required"
          ))
        )
    }
  }

  private def extractToken[A](request: Request[A]): Option[String] =
    request.headers.get("Authorization").flatMap { header =>
      if (header.startsWith("Bearer "))
        Some(header.drop(7).trim)
      else
        None
    }

  override protected def executionContext: ExecutionContext = ec
}
```

---

## Step 505: Auth Controller

```scala
// app/controllers/AuthController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*
import utils.{JwtUtils, PasswordHasher}
import scala.concurrent.*

@Singleton
class AuthController @Inject()(
  val controllerComponents: ControllerComponents,
  userService: services.UserService,
  implicit val ec: ExecutionContext
) extends BaseController {

  case class LoginRequest(email: String, password: String)
  case class RegisterRequest(username: String, email: String, password: String)

  implicit val loginReads: Reads[LoginRequest] = Json.reads[LoginRequest]
  implicit val registerReads: Reads[RegisterRequest] = Json.reads[RegisterRequest]

  // POST /api/auth/login
  def login(): Action[JsValue] = Action.async(parse.json) { implicit request =>
    request.body.validate[LoginRequest].fold(
      errors => Future.successful(BadRequest(Json.obj("error" -> "Invalid request"))),
      loginReq => {
        userService.findByEmail(loginReq.email).flatMap {
          case None =>
            // ใช้เวลา hash ปกติ เพื่อป้องกัน timing attack
            PasswordHasher.hash("dummy")
            Future.successful(Unauthorized(Json.obj("error" -> "Invalid credentials")))

          case Some(user) =>
            if (PasswordHasher.verify(loginReq.password, user.passwordHash)) {
              val token = JwtUtils.createToken(user.id, user.username, user.role.toString)
              val refreshToken = JwtUtils.createToken(user.id, user.username, user.role.toString)

              Future.successful(
                Ok(Json.obj(
                  "accessToken"  -> token,
                  "refreshToken" -> refreshToken,
                  "expiresIn"    -> 86400,
                  "tokenType"    -> "Bearer",
                  "user"         -> Json.obj(
                    "id"       -> user.id,
                    "username" -> user.username,
                    "email"    -> user.email,
                    "role"     -> user.role.toString
                  )
                ))
              )
            } else {
              Future.successful(Unauthorized(Json.obj("error" -> "Invalid credentials")))
            }
        }
      }
    )
  }

  // POST /api/auth/register
  def register(): Action[JsValue] = Action.async(parse.json) { implicit request =>
    request.body.validate[RegisterRequest].fold(
      errors => Future.successful(BadRequest(Json.obj("error" -> "Validation failed"))),
      registerReq => {
        userService.findByEmail(registerReq.email).flatMap {
          case Some(_) =>
            Future.successful(Conflict(Json.obj("error" -> "Email already exists")))
          case None =>
            val hashedPassword = PasswordHasher.hash(registerReq.password)
            userService.create(registerReq.username, registerReq.email, hashedPassword).map { user =>
              val token = JwtUtils.createToken(user.id, user.username, user.role.toString)
              Created(Json.obj(
                "message"     -> "Registration successful",
                "accessToken" -> token,
                "user"        -> Json.obj("id" -> user.id, "username" -> user.username)
              ))
            }
        }
      }
    )
  }

  // POST /api/auth/refresh
  def refresh(): Action[JsValue] = Action.async(parse.json) { implicit request =>
    val refreshToken = (request.body \ "refreshToken").asOpt[String]
    refreshToken match {
      case None =>
        Future.successful(BadRequest(Json.obj("error" -> "Refresh token required")))
      case Some(token) =>
        JwtUtils.refreshToken(token) match {
          case Right(newToken) =>
            Future.successful(Ok(Json.obj("accessToken" -> newToken, "expiresIn" -> 86400)))
          case Left(error) =>
            Future.successful(Unauthorized(Json.obj("error" -> error)))
        }
    }
  }

  // POST /api/auth/logout (stateless JWT ไม่ต้องทำอะไร แต่ log ได้)
  def logout(): Action[AnyContent] = Action { implicit request =>
    Ok(Json.obj("message" -> "Logged out successfully"))
  }
}
```

---

## Step 506: Role-Based Access Control (RBAC)

```scala
// app/actions/RoleAction.scala
package actions

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*
import scala.concurrent.*

// Action ที่ตรวจสอบ role
class RoleAction(
  requiredRoles: Set[String],
  jwtAction: JwtAction,
  parser: BodyParsers.Default,
  implicit val ec: ExecutionContext
) extends ActionBuilder[JwtRequest, AnyContent] {

  override def parser: BodyParser[AnyContent] = parser

  override def invokeBlock[A](
    request: Request[A],
    block: JwtRequest[A] => Future[Result]
  ): Future[Result] = {
    jwtAction.invokeBlock(request, { jwtRequest: JwtRequest[A] =>
      if (requiredRoles.contains(jwtRequest.role) ||
          requiredRoles.contains("*")) {
        block(jwtRequest)
      } else {
        Future.successful(
          Forbidden(Json.obj(
            "error"         -> "Forbidden",
            "message"       -> s"Role '${jwtRequest.role}' is not authorized",
            "requiredRoles" -> requiredRoles
          ))
        )
      }
    })
  }

  override protected def executionContext: ExecutionContext = ec
}

// Factory สำหรับสร้าง RoleAction
class RoleActionFactory @Inject()(
  jwtAction: JwtAction,
  parser: BodyParsers.Default,
  implicit val ec: ExecutionContext
) {
  def apply(roles: String*): RoleAction =
    new RoleAction(roles.toSet, jwtAction, parser, ec)

  def admin: RoleAction = apply("admin")
  def editor: RoleAction = apply("admin", "editor")
  def authenticated: RoleAction = apply("admin", "editor", "viewer")
}
```

### ใช้ RBAC ใน Controller

```scala
// app/controllers/api/AdminController.scala
package controllers.api

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*
import actions.*
import scala.concurrent.*

@Singleton
class AdminController @Inject()(
  val controllerComponents: ControllerComponents,
  roleActions: RoleActionFactory,
  implicit val ec: ExecutionContext
) extends BaseController {

  // เฉพาะ admin เท่านั้น
  def listUsers(): Action[AnyContent] = roleActions.admin { implicit request =>
    Ok(Json.obj("users" -> Json.arr()))
  }

  // admin และ editor
  def createArticle(): Action[JsValue] =
    roleActions.editor(parse.json) { implicit request =>
      Created(Json.obj("message" -> "Article created"))
    }

  // ทุก authenticated user
  def profile(): Action[AnyContent] = roleActions.authenticated { implicit request =>
    Ok(Json.obj(
      "userId"   -> request.userId,
      "username" -> request.username,
      "role"     -> request.role
    ))
  }
}
```

---

## Step 507: OAuth2 Integration

```scala
// app/controllers/OAuth2Controller.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.libs.ws.*
import play.api.libs.json.*
import play.api.Configuration
import scala.concurrent.*

@Singleton
class OAuth2Controller @Inject()(
  val controllerComponents: ControllerComponents,
  wsClient: WSClient,
  config: Configuration,
  implicit val ec: ExecutionContext
) extends BaseController {

  private val githubClientId     = config.get[String]("oauth2.github.clientId")
  private val githubClientSecret = config.get[String]("oauth2.github.clientSecret")
  private val callbackUrl        = config.get[String]("oauth2.github.callbackUrl")

  // Step 1: Redirect to GitHub
  def githubLogin(): Action[AnyContent] = Action { implicit request =>
    val state = java.util.UUID.randomUUID().toString
    val authUrl = s"https://github.com/login/oauth/authorize" +
      s"?client_id=$githubClientId" +
      s"&redirect_uri=${java.net.URLEncoder.encode(callbackUrl, "UTF-8")}" +
      s"&scope=user:email" +
      s"&state=$state"

    Redirect(authUrl).withSession("oauth2State" -> state)
  }

  // Step 2: Handle callback
  def githubCallback(code: String, state: String): Action[AnyContent] = Action.async { implicit request =>
    // ตรวจสอบ state เพื่อป้องกัน CSRF
    val savedState = request.session.get("oauth2State")
    if (!savedState.contains(state)) {
      return Future.successful(BadRequest("Invalid state parameter"))
    }

    // Exchange code for access token
    wsClient
      .url("https://github.com/login/oauth/access_token")
      .addHttpHeaders("Accept" -> "application/json")
      .post(Json.obj(
        "client_id"     -> githubClientId,
        "client_secret" -> githubClientSecret,
        "code"          -> code,
        "redirect_uri"  -> callbackUrl
      ))
      .flatMap { tokenResponse =>
        val accessToken = (tokenResponse.json \ "access_token").as[String]

        // Get user info
        wsClient
          .url("https://api.github.com/user")
          .addHttpHeaders(
            "Authorization" -> s"Bearer $accessToken",
            "Accept"        -> "application/vnd.github.v3+json"
          )
          .get()
          .map { userResponse =>
            val githubUser = userResponse.json
            val githubId = (githubUser \ "id").as[Long]
            val username = (githubUser \ "login").as[String]
            val email = (githubUser \ "email").asOpt[String]
              .getOrElse(s"$username@users.noreply.github.com")

            // Create or update user ใน DB
            val jwtToken = utils.JwtUtils.createToken(githubId, username, "viewer")

            Redirect(routes.HomeController.index())
              .withSession("userId" -> githubId.toString, "username" -> username)
              .flashing("success" -> s"Welcome, $username!")
          }
      }
  }
}
```

---

## Step 508: Security Headers

```hocon
# conf/application.conf - Security configuration

# Security Headers
play.filters.headers {
  # Prevent XSS attacks
  contentSecurityPolicy = "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com; img-src 'self' data: https:"

  # Prevent clickjacking
  xFrameOptions = "DENY"

  # Enable browser XSS protection
  xssProtection = "1; mode=block"

  # Prevent MIME type sniffing
  contentTypeOptions = "nosniff"

  # Referrer Policy
  referrerPolicy = "strict-origin-when-cross-origin"

  # HSTS (เปิดใน production ที่มี HTTPS เท่านั้น)
  # strictTransportSecurity = "max-age=31536000; includeSubDomains"
}

# Secret key สำหรับ session encryption
play.http.secret.key = "change-me-in-production-use-long-random-string"
play.http.secret.key = ${?APPLICATION_SECRET}
```

---

## Step 509: Input Validation และ SQL Injection Prevention

```scala
// app/utils/InputValidator.scala
package utils

object InputValidator {

  // Sanitize input ป้องกัน XSS
  def sanitizeHtml(input: String): String = {
    import org.jsoup.Jsoup
    import org.jsoup.safety.Safelist

    // อนุญาตเฉพาะ HTML tags ที่ปลอดภัย
    Jsoup.clean(input, Safelist.basicWithImages())
  }

  // Validate email format
  def isValidEmail(email: String): Boolean =
    email.matches("^[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}$")

  // ป้องกัน path traversal
  def sanitizePath(path: String): String =
    path.replaceAll("\\.\\./", "")
        .replaceAll("\\./", "")
        .replaceAll("^/", "")

  // Validate file extension
  def isAllowedFileType(filename: String, allowed: Set[String]): Boolean = {
    val extension = filename.split("\\.").lastOption.getOrElse("").toLowerCase
    allowed.contains(extension)
  }

  // ป้องกัน SSRF (Server-Side Request Forgery)
  def isAllowedUrl(url: String): Boolean = {
    try {
      val uri = new java.net.URI(url)
      val host = uri.getHost

      // Block internal addresses
      !host.matches("^(localhost|127\\.0\\.0\\.1|10\\..+|172\\.(1[6-9]|2[0-9]|3[01])\\..+|192\\.168\\..+)$") &&
      // Allow only HTTPS
      uri.getScheme == "https"
    } catch {
      case _: Exception => false
    }
  }
}
```

---

## Step 510: Security Audit Logging

```scala
// app/services/AuditService.scala
package services

import javax.inject.*
import play.api.Logger
import play.api.libs.json.*

case class AuditEvent(
  userId: Option[Long],
  action: String,
  resource: String,
  resourceId: Option[Long],
  success: Boolean,
  ipAddress: String,
  userAgent: String,
  timestamp: java.time.Instant = java.time.Instant.now(),
  metadata: Map[String, String] = Map.empty
)

@Singleton
class AuditService @Inject()() {

  private val auditLogger = Logger("audit")

  def log(event: AuditEvent): Unit = {
    val json = Json.obj(
      "timestamp"  -> event.timestamp.toString,
      "userId"     -> event.userId,
      "action"     -> event.action,
      "resource"   -> event.resource,
      "resourceId" -> event.resourceId,
      "success"    -> event.success,
      "ip"         -> event.ipAddress,
      "userAgent"  -> event.userAgent,
      "metadata"   -> event.metadata
    )

    if (event.success) auditLogger.info(Json.stringify(json))
    else auditLogger.warn(Json.stringify(json))
  }

  // Convenience methods
  def logLogin(userId: Long, success: Boolean, ip: String, userAgent: String): Unit =
    log(AuditEvent(Some(userId), "LOGIN", "auth", None, success, ip, userAgent))

  def logAccess(userId: Long, resource: String, resourceId: Long, ip: String): Unit =
    log(AuditEvent(Some(userId), "ACCESS", resource, Some(resourceId), true, ip, ""))

  def logDataChange(
    userId: Long,
    action: String,
    resource: String,
    resourceId: Long,
    ip: String
  ): Unit =
    log(AuditEvent(Some(userId), action, resource, Some(resourceId), true, ip, ""))
}
```

---

## สรุป Part 51

| Security Topic | Implementation | Best Practice |
|----------------|----------------|--------------|
| Password | BCrypt cost 12 | Never store plaintext |
| JWT | HS256 + expiry | Short-lived access tokens |
| OAuth2 | State param | ป้องกัน CSRF |
| RBAC | Role-based actions | Least privilege |
| Security Headers | CSP, HSTS, X-Frame | เปิดใน production |
| Input Validation | Sanitize HTML | ป้องกัน XSS, injection |
| Audit Logging | Structured logs | ทุก auth event |

---

## แบบฝึกหัด Part 51

1. **JWT Refresh Token**: Implement refresh token flow ที่ใช้ short-lived access tokens (15 min) และ long-lived refresh tokens (7 days) เก็บใน database

2. **Two-Factor Auth**: เพิ่ม TOTP-based 2FA โดยใช้ Google Authenticator compatible algorithm (RFC 6238)

3. **Permission System**: สร้าง permission-based access control ที่ granular กว่า RBAC เช่น `articles:read`, `articles:write`, `users:admin`

4. **Brute Force Protection**: สร้าง login attempt tracker ที่ lock account หลัง 5 failed attempts ใน 10 นาที

5. **Security Audit**: ทำ security review ของ application และ identify vulnerabilities เช่น missing CSRF, insecure direct object reference, missing rate limiting

---

[→ ไปยัง Part 52: Play Dependency Injection](part-52-play-dependency-injection.md)
