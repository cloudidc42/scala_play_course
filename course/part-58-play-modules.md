# Part 58: Play Modules

## Steps 571-580: Creating Play Modules, Reusable Components, Module Lifecycle

---

## Step 571: Play Module คืออะไร

Play Module คือ package ของ code ที่สามารถ reuse ได้ระหว่าง Play applications

```
Play Module ประกอบด้วย:
├── Guice Module (dependency bindings)
├── Controllers
├── Services
├── Models
├── Configurations
├── Assets (optional)
└── Routes (optional)
```

### ประเภทของ Module

```
1. Internal Module    — organize code ภายใน app
2. Reusable Module    — package แยก ใช้ข้าม projects
3. Plugin Module      — extend Play framework
4. SBT Plugin         — extend sbt build
```

---

## Step 572: สร้าง Internal Module

```scala
// app/modules/NotificationModule.scala
package modules

import javax.inject.*
import com.google.inject.AbstractModule
import services.notification.*

class NotificationModule extends AbstractModule {
  override def configure(): Unit = {
    // bind interfaces to implementations
    bind(classOf[NotificationService]).to(classOf[EmailNotificationService])
    bind(classOf[PushNotificationService]).to(classOf[FirebasePushService])
    bind(classOf[SmsService]).to(classOf[TwilioSmsService])

    // Eager singletons (start on app startup)
    bind(classOf[NotificationScheduler]).asEagerSingleton()
  }
}
```

```hocon
# conf/application.conf — register module
play.modules.enabled += "modules.NotificationModule"
```

---

## Step 573: Module Lifecycle

```scala
// app/modules/ApplicationLifecycleModule.scala
package modules

import javax.inject.*
import com.google.inject.AbstractModule
import play.api.inject.ApplicationLifecycle
import play.api.Logger
import scala.concurrent.*

class ApplicationLifecycleModule extends AbstractModule {
  override def configure(): Unit = {
    bind(classOf[LifecycleHooks]).asEagerSingleton()
  }
}

@Singleton
class LifecycleHooks @Inject()(
  lifecycle: ApplicationLifecycle,
  scheduler: TaskScheduler,
  cacheWarmup: CacheWarmupService,
  implicit val ec: ExecutionContext
) {
  private val logger = Logger(getClass)

  // รันเมื่อ app start
  startup()

  private def startup(): Unit = {
    logger.info("Application starting up...")
    
    // เริ่ม background tasks
    scheduler.start()
    
    // warm up caches
    cacheWarmup.warmup().foreach { _ =>
      logger.info("Cache warmed up successfully")
    }

    // Register shutdown hook
    lifecycle.addStopHook { () =>
      logger.info("Application shutting down...")
      scheduler.stop()
      Future.successful(())
    }
  }
}
```

---

## Step 574: Reusable Module — Audit Log

```scala
// สร้าง audit-log module ที่ reuse ได้

// modules/audit/app/models/AuditEvent.scala
package audit.models

import java.time.Instant

case class AuditEvent(
  id: String = java.util.UUID.randomUUID().toString,
  userId: Option[String],
  action: String,
  resource: String,
  resourceId: Option[String],
  oldValue: Option[String],
  newValue: Option[String],
  ipAddress: Option[String],
  userAgent: Option[String],
  timestamp: Instant = Instant.now(),
  success: Boolean = true,
  errorMessage: Option[String] = None
)
```

```scala
// modules/audit/app/services/AuditService.scala
package audit.services

import audit.models.AuditEvent
import javax.inject.*
import play.api.db.slick.DatabaseConfigProvider
import scala.concurrent.*

trait AuditService {
  def log(event: AuditEvent): Future[Unit]
  def query(userId: Option[String], action: Option[String], from: Option[java.time.Instant], to: Option[java.time.Instant]): Future[Seq[AuditEvent]]
}

@Singleton
class DbAuditService @Inject()(
  dbConfigProvider: DatabaseConfigProvider
)(implicit ec: ExecutionContext) extends AuditService {

  override def log(event: AuditEvent): Future[Unit] = Future {
    // บันทึก audit event ลง database
    println(s"AUDIT: ${event.action} on ${event.resource} by ${event.userId}")
  }

  override def query(
    userId: Option[String],
    action: Option[String],
    from: Option[java.time.Instant],
    to: Option[java.time.Instant]
  ): Future[Seq[AuditEvent]] = Future.successful(Seq.empty)
}
```

```scala
// modules/audit/app/filters/AuditFilter.scala
package audit.filters

import audit.models.AuditEvent
import audit.services.AuditService
import javax.inject.*
import org.apache.pekko.stream.Materializer
import play.api.mvc.*
import scala.concurrent.*

@Singleton
class AuditFilter @Inject()(
  auditService: AuditService
)(implicit val mat: Materializer, ec: ExecutionContext) extends Filter {

  private val auditedMethods = Set("POST", "PUT", "PATCH", "DELETE")

  override def apply(next: RequestHeader => Future[Result])(request: RequestHeader): Future[Result] = {
    if (!auditedMethods.contains(request.method)) {
      next(request)
    } else {
      val startTime = System.currentTimeMillis()
      next(request).flatMap { result =>
        val event = AuditEvent(
          userId    = request.session.get("userId"),
          action    = request.method,
          resource  = request.path,
          ipAddress = Some(request.remoteAddress),
          userAgent = request.headers.get("User-Agent"),
          success   = result.header.status < 400
        )
        auditService.log(event).map(_ => result)
      }
    }
  }
}
```

```scala
// modules/audit/app/modules/AuditModule.scala
package audit.modules

import audit.filters.AuditFilter
import audit.services.{AuditService, DbAuditService}
import com.google.inject.AbstractModule
import play.api.mvc.EssentialFilter
import com.google.inject.multibindings.Multibinder

class AuditModule extends AbstractModule {
  override def configure(): Unit = {
    bind(classOf[AuditService]).to(classOf[DbAuditService])

    // Add filter to Play's filter chain
    val filterBinder = Multibinder.newSetBinder(binder(), classOf[EssentialFilter])
    filterBinder.addBinding().to(classOf[AuditFilter])
  }
}
```

---

## Step 575: Module Configuration

```scala
// modules/audit/app/config/AuditConfig.scala
package audit.config

import javax.inject.*
import play.api.Configuration

@Singleton
class AuditConfig @Inject()(config: Configuration) {
  val enabled: Boolean = config.get[Boolean]("audit.enabled")
  val retentionDays: Int = config.get[Int]("audit.retentionDays")
  val excludePaths: Seq[String] = config.get[Seq[String]]("audit.excludePaths")
  val asyncLogging: Boolean = config.getOrElse[Boolean]("audit.asyncLogging", true)
}
```

```hocon
# conf/reference.conf — default config ใน module
audit {
  enabled = true
  retentionDays = 90
  excludePaths = ["/health", "/metrics", "/assets"]
  asyncLogging = true
}
```

---

## Step 576: Module Routes

```
# modules/audit/conf/audit.routes

GET  /audit/events        audit.controllers.AuditController.list(userId: Option[String], action: Option[String])
GET  /audit/events/:id    audit.controllers.AuditController.get(id: String)
```

```scala
// modules/audit/app/controllers/AuditController.scala
package audit.controllers

import audit.services.AuditService
import javax.inject.*
import play.api.libs.json.*
import play.api.mvc.*
import scala.concurrent.*

@Singleton
class AuditController @Inject()(
  val controllerComponents: ControllerComponents,
  auditService: AuditService,
  implicit val ec: ExecutionContext
) extends BaseController {

  def list(userId: Option[String], action: Option[String]): Action[AnyContent] = Action.async {
    auditService.query(userId, action, None, None).map { events =>
      Ok(Json.toJson(events.map(e => Json.obj(
        "id"        -> e.id,
        "userId"    -> e.userId,
        "action"    -> e.action,
        "resource"  -> e.resource,
        "timestamp" -> e.timestamp.toString,
        "success"   -> e.success
      ))))
    }
  }

  def get(id: String): Action[AnyContent] = Action.async {
    // ดึง event by id
    Future.successful(NotFound(Json.obj("error" -> "Not found")))
  }
}
```

```hocon
# conf/application.conf — เพิ่ม module routes
play.http.router = router.Routes

# Include module routes
# หรือใน conf/routes:
# ->  /audit   audit.Routes
```

---

## Step 577: Feature Toggle Module

```scala
// app/modules/FeatureToggleModule.scala
package modules

import com.google.inject.AbstractModule
import services.FeatureToggleService

class FeatureToggleModule extends AbstractModule {
  override def configure(): Unit = {
    bind(classOf[FeatureToggleService]).asEagerSingleton()
  }
}
```

```scala
// app/services/FeatureToggleService.scala
package services

import javax.inject.*
import play.api.Configuration
import play.api.Logger
import scala.collection.concurrent.TrieMap

@Singleton
class FeatureToggleService @Inject()(config: Configuration) {
  private val logger = Logger(getClass)

  // In-memory feature flags (could be loaded from DB/Redis)
  private val flags = TrieMap[String, Boolean](
    "new-ui"          -> config.getOrElse("features.newUi", false),
    "beta-api"        -> config.getOrElse("features.betaApi", false),
    "dark-mode"       -> config.getOrElse("features.darkMode", true),
    "payment-v2"      -> config.getOrElse("features.paymentV2", false),
    "recommendation"  -> config.getOrElse("features.recommendation", true)
  )

  def isEnabled(feature: String): Boolean =
    flags.getOrElse(feature, false)

  def isEnabled(feature: String, userId: String): Boolean = {
    // A/B testing: enable for % of users
    val baseEnabled = isEnabled(feature)
    if (!baseEnabled) false
    else {
      // Hash userId to determine if user is in test group
      val hash = Math.abs(userId.hashCode) % 100
      hash < rolloutPercentage(feature)
    }
  }

  def enable(feature: String): Unit = {
    flags.put(feature, true)
    logger.info(s"Feature '$feature' enabled")
  }

  def disable(feature: String): Unit = {
    flags.put(feature, false)
    logger.info(s"Feature '$feature' disabled")
  }

  def all(): Map[String, Boolean] = flags.toMap

  private def rolloutPercentage(feature: String): Int =
    config.getOrElse(s"features.rollout.$feature", 100)
}
```

```scala
// ใช้ feature toggle ใน controller
@Singleton
class HomeController @Inject()(
  val controllerComponents: ControllerComponents,
  featureToggle: FeatureToggleService
) extends BaseController {

  def index(): Action[AnyContent] = Action { implicit request =>
    val showNewUi = featureToggle.isEnabled("new-ui")
    val userId = request.session.get("userId").getOrElse("")
    val showBetaApi = featureToggle.isEnabled("beta-api", userId)

    Ok(views.html.index(showNewUi, showBetaApi))
  }
}
```

---

## Step 578: Health Check Module

```scala
// app/modules/HealthCheckModule.scala
package modules

import com.google.inject.AbstractModule
import services.health.*

class HealthCheckModule extends AbstractModule {
  override def configure(): Unit = {
    bind(classOf[DatabaseHealthCheck]).asEagerSingleton()
    bind(classOf[CacheHealthCheck]).asEagerSingleton()
    bind(classOf[ExternalServiceHealthCheck]).asEagerSingleton()
  }
}
```

```scala
// app/services/health/HealthCheck.scala
package services.health

import javax.inject.*
import play.api.db.*
import scala.concurrent.*
import scala.util.*

trait HealthCheck {
  def name: String
  def check()(implicit ec: ExecutionContext): Future[HealthStatus]
}

case class HealthStatus(
  name: String,
  status: String,  // "UP", "DOWN", "DEGRADED"
  message: Option[String] = None,
  responseTime: Long = 0
)

@Singleton
class DatabaseHealthCheck @Inject()(db: Database) extends HealthCheck {
  val name = "database"

  def check()(implicit ec: ExecutionContext): Future[HealthStatus] = Future {
    val start = System.currentTimeMillis()
    Try {
      db.withConnection { conn =>
        val stmt = conn.createStatement()
        stmt.executeQuery("SELECT 1")
        stmt.close()
      }
    } match {
      case Success(_) =>
        HealthStatus(name, "UP", responseTime = System.currentTimeMillis() - start)
      case Failure(ex) =>
        HealthStatus(name, "DOWN", Some(ex.getMessage))
    }
  }
}

// Health Controller
@Singleton
class HealthController @Inject()(
  val controllerComponents: ControllerComponents,
  checks: Seq[HealthCheck],
  implicit val ec: ExecutionContext
) extends BaseController {

  def health(): Action[AnyContent] = Action.async {
    Future.sequence(checks.map(_.check())).map { statuses =>
      val allUp = statuses.forall(_.status == "UP")
      val statusCode = if (allUp) 200 else 503

      val body = play.api.libs.json.Json.obj(
        "status"  -> (if (allUp) "UP" else "DOWN"),
        "checks"  -> play.api.libs.json.Json.toJson(statuses.map(s =>
          play.api.libs.json.Json.obj(
            "name"         -> s.name,
            "status"       -> s.status,
            "responseTime" -> s.responseTime
          )
        ))
      )

      Status(statusCode)(body)
    }
  }
}
```

---

## Step 579: Module Testing

```scala
// test/modules/AuditModuleSpec.scala
package modules

import org.scalatestplus.play.*
import org.scalatestplus.play.guice.GuiceOneAppPerTest
import play.api.inject.guice.GuiceApplicationBuilder
import play.api.test.*
import audit.services.AuditService
import audit.models.AuditEvent

class AuditModuleSpec extends PlaySpec with GuiceOneAppPerTest {

  override def fakeApplication() =
    GuiceApplicationBuilder()
      .configure("audit.enabled" -> true)
      .build()

  "AuditModule" should {
    "provide AuditService" in {
      val service = app.injector.instanceOf[AuditService]
      service must not be null
    }

    "log audit events" in {
      val service = app.injector.instanceOf[AuditService]
      import scala.concurrent.ExecutionContext.Implicits.global
      import scala.concurrent.Await
      import scala.concurrent.duration.*

      val event = AuditEvent(
        userId   = Some("user-1"),
        action   = "DELETE",
        resource = "/api/articles/42"
      )

      val result = Await.result(service.log(event), 5.seconds)
      result mustBe (())
    }
  }
}
```

---

## Step 580: Module Best Practices

```scala
// ✅ Good: Module ที่ configurable
class EmailModule extends AbstractModule {
  override def configure(): Unit = {
    // ตรวจสอบ config ก่อน bind
  }

  @Provides
  @Singleton
  def emailService(config: Configuration): EmailService = {
    val provider = config.getOrElse("email.provider", "smtp")
    provider match {
      case "smtp"     => new SmtpEmailService(config)
      case "sendgrid" => new SendgridEmailService(config)
      case "ses"      => new SesEmailService(config)
      case other      => throw new IllegalArgumentException(s"Unknown email provider: $other")
    }
  }
}

// ✅ Good: Module ที่มี reference.conf สำหรับ defaults
// modules/email/conf/reference.conf
// email {
//   provider = smtp
//   smtp.host = localhost
//   smtp.port = 25
//   from = noreply@example.com
// }

// ✅ Good: Module ที่ test ได้ง่าย
class TestEmailModule extends AbstractModule {
  override def configure(): Unit = {
    bind(classOf[EmailService]).to(classOf[MockEmailService])
  }
}

// ใน test
val app = GuiceApplicationBuilder()
  .overrides(new TestEmailModule)
  .build()
```

---

## สรุป Part 58

| Concept | Implementation | ตัวอย่าง |
|---------|---------------|---------|
| Module registration | `play.modules.enabled` | `+= "modules.MyModule"` |
| AbstractModule | `configure()` | `bind(...).to(...)` |
| Eager singleton | `asEagerSingleton()` | Start on app boot |
| Lifecycle hooks | `ApplicationLifecycle` | `addStopHook` |
| Module routes | Separate routes file | `-> /path module.Routes` |
| Reference config | `conf/reference.conf` | Module defaults |
| Feature toggles | `FeatureToggleService` | A/B testing |
| Health checks | `HealthCheck` trait | `/health` endpoint |

---

## แบบฝึกหัด Part 58

1. **Audit Module**: สร้าง reusable audit module ที่ log ทุก state-changing request พร้อม user, timestamp, IP, และ before/after values

2. **Feature Toggle**: Implement feature toggle system ที่ support per-user rollout percentages และ admin UI สำหรับ enable/disable features

3. **Metrics Module**: สร้าง metrics module ที่เก็บ request counts, response times, error rates และ expose ผ่าน `/metrics` endpoint ในรูปแบบ Prometheus

4. **Module Packaging**: Pack notification module เป็น separate sbt subproject และ publish เป็น local Maven artifact ที่ projects อื่นใช้ได้

5. **Health Check System**: Implement comprehensive health check system ที่ตรวจสอบ database, cache, external APIs และส่ง alert เมื่อ service DOWN

---

[→ ไปยัง Part 59: Play Deployment](part-59-play-deployment.md)
