# Part 52: Play Dependency Injection

## Steps 511-520: Guice DI, @Inject, @Singleton, @Provides, Custom Modules, Testing with DI

---

## Step 511: Dependency Injection คืออะไร?

Dependency Injection (DI) คือ design pattern ที่แยก object creation ออกจาก business logic

```
Without DI:
class UserController {
  val service = new UserService(new UserRepository(new Database()))  // ❌ tight coupling
}

With DI:
class UserController @Inject()(service: UserService) {  // ✅ loose coupling
  // service injected โดย DI container
}
```

### Play ใช้ Guice (Google's DI Framework)

```scala
// Guice key concepts:
// @Inject    - บอกว่า class นี้ต้องการ inject dependencies
// @Singleton - สร้าง instance เดียวตลอด app lifetime
// @Provides  - factory method สำหรับสร้าง dependency
// Module     - กำหนดว่า interface ใช้ implementation ใด
```

---

## Step 512: @Inject และ @Singleton

```scala
// app/repositories/UserRepository.scala
package repositories

import javax.inject.*
import play.api.db.slick.DatabaseConfigProvider
import slick.jdbc.PostgresProfile
import scala.concurrent.*

// @Singleton: Play สร้าง instance เดียว ทุก class ที่ inject UserRepository ได้ instance เดียวกัน
@Singleton
class UserRepository @Inject()(
  dbConfigProvider: DatabaseConfigProvider,
  implicit val ec: ExecutionContext
) {
  // implementation...

  def findById(id: Long): Future[Option[models.User]] =
    Future.successful(None)  // placeholder

  def create(username: String, email: String, passwordHash: String): Future[models.User] = {
    val user = models.User(
      id = System.currentTimeMillis(),
      username = username,
      email = email,
      role = models.UserRole.Viewer,
      createdAt = java.time.LocalDateTime.now()
    )
    Future.successful(user)  // placeholder
  }
}
```

```scala
// app/services/UserService.scala
package services

import javax.inject.*
import scala.concurrent.*
import repositories.UserRepository
import utils.PasswordHasher

@Singleton
class UserService @Inject()(
  userRepo: UserRepository,              // inject repository
  emailService: EmailService,            // inject email service
  cacheService: CacheService,            // inject cache
  implicit val ec: ExecutionContext
) {

  def register(username: String, email: String, password: String): Future[models.User] = {
    // ตรวจสอบ email ซ้ำก่อน
    userRepo.findByEmail(email).flatMap {
      case Some(_) =>
        Future.failed(new IllegalArgumentException(s"Email $email already exists"))
      case None =>
        val hashedPassword = PasswordHasher.hash(password)
        userRepo.create(username, email, hashedPassword).flatMap { user =>
          emailService.sendWelcome(user).map(_ => user)
        }
    }
  }

  def findByEmail(email: String): Future[Option[models.User]] =
    userRepo.findByEmail(email)
}
```

---

## Step 513: Constructor Injection vs Field Injection

```scala
// ✅ Constructor Injection (recommended)
@Singleton
class ArticleController @Inject()(
  val controllerComponents: ControllerComponents,
  articleService: services.ArticleService,
  cacheService: services.CacheService
) extends BaseController {
  // dependencies ใช้ได้ใน constructor และ methods
}

// ❌ Field Injection (ไม่แนะนำ - testing ยากกว่า)
class BadController extends BaseController {
  @Inject var controllerComponents: ControllerComponents = _  // mutable!
  @Inject var articleService: services.ArticleService = _
}

// Method Injection (ใช้กรณีพิเศษ)
class ConditionalService {
  @Inject
  def setOptionalDep(dep: OptionalDependency): Unit = {
    // inject method
  }
}
```

---

## Step 514: Binding Interfaces to Implementations

```scala
// app/services/EmailService.scala - Interface
package services

import scala.concurrent.Future
import models.User

trait EmailService {
  def sendWelcome(user: User): Future[Unit]
  def sendPasswordReset(user: User, token: String): Future[Unit]
  def sendNotification(userId: Long, message: String): Future[Unit]
}
```

```scala
// app/services/SmtpEmailService.scala - Production implementation
package services

import javax.inject.*
import scala.concurrent.*

@Singleton
class SmtpEmailService @Inject()(
  mailer: play.api.libs.mailer.MailerClient,
  config: play.api.Configuration,
  implicit val ec: ExecutionContext
) extends EmailService {

  override def sendWelcome(user: models.User): Future[Unit] = Future {
    // ส่ง email จริงผ่าน SMTP
    val email = play.api.libs.mailer.Email(
      subject = "ยินดีต้อนรับ!",
      from    = "noreply@myapp.com",
      to      = Seq(user.email),
      bodyHtml = Some(s"<h1>สวัสดี ${user.username}!</h1>")
    )
    mailer.send(email)
    ()
  }

  override def sendPasswordReset(user: models.User, token: String): Future[Unit] = Future {
    // ส่ง password reset email
    ()
  }

  override def sendNotification(userId: Long, message: String): Future[Unit] = Future {
    // ส่ง notification email
    ()
  }
}
```

```scala
// app/services/LogEmailService.scala - Development/test implementation
package services

import javax.inject.*
import play.api.Logger
import scala.concurrent.*

@Singleton
class LogEmailService @Inject()(
  implicit val ec: ExecutionContext
) extends EmailService {

  private val logger = Logger("email")

  override def sendWelcome(user: models.User): Future[Unit] = {
    logger.info(s"[FAKE EMAIL] Welcome email to: ${user.email}")
    Future.successful(())
  }

  override def sendPasswordReset(user: models.User, token: String): Future[Unit] = {
    logger.info(s"[FAKE EMAIL] Password reset to: ${user.email}, token: $token")
    Future.successful(())
  }

  override def sendNotification(userId: Long, message: String): Future[Unit] = {
    logger.info(s"[FAKE EMAIL] Notification to user $userId: $message")
    Future.successful(())
  }
}
```

---

## Step 515: Custom Module

```scala
// app/Module.scala
package app

import com.google.inject.AbstractModule
import play.api.{Configuration, Environment, Mode}
import services.*
import repositories.*

class Module(
  environment: Environment,
  configuration: Configuration
) extends AbstractModule {

  override def configure(): Unit = {

    // Bind interface to implementation ตาม environment
    environment.mode match {
      case Mode.Prod =>
        // Production: ใช้ real email service
        bind(classOf[EmailService]).to(classOf[SmtpEmailService])
      case _ =>
        // Dev/Test: ใช้ log email service
        bind(classOf[EmailService]).to(classOf[LogEmailService])
    }

    // Always use these implementations
    bind(classOf[CacheService]).to(classOf[EhCacheService])
    bind(classOf[StorageService]).to(classOf[S3StorageService])

    // เป็น Singleton
    bind(classOf[UserRepository]).asEagerSingleton()
    bind(classOf[ArticleRepository]).asEagerSingleton()

    // Bind named configuration
    // bind(classOf[String]).annotatedWith(Names.named("apiKey")).toInstance("my-api-key")
  }
}
```

```hocon
# conf/application.conf
# Register custom module
play.modules.enabled += "app.Module"
```

---

## Step 516: @Provides - Factory Methods

```scala
// app/modules/DatabaseModule.scala
package modules

import com.google.inject.{AbstractModule, Provides, Singleton}
import play.api.Configuration
import slick.jdbc.JdbcBackend.Database

class DatabaseModule extends AbstractModule {

  // @Provides: Guice เรียก method นี้เมื่อต้องการ Database instance
  @Provides
  @Singleton
  def provideDatabase(config: Configuration): Database = {
    val url      = config.get[String]("db.default.url")
    val user     = config.get[String]("db.default.username")
    val password = config.get[String]("db.default.password")

    Database.forURL(url, user, password)
  }

  // @Provides สำหรับ external libraries
  @Provides
  @Singleton
  def provideRedisClient(config: Configuration): redis.clients.jedis.JedisPool = {
    val host = config.get[String]("redis.host")
    val port = config.get[Int]("redis.port")
    new redis.clients.jedis.JedisPool(host, port)
  }

  override def configure(): Unit = {}
}
```

---

## Step 517: Named Bindings

```scala
// app/modules/ConfigModule.scala
package modules

import com.google.inject.{AbstractModule, Provides, Singleton, Named}
import play.api.Configuration

class ConfigModule extends AbstractModule {

  @Provides
  @Named("apiKey")
  def provideApiKey(config: Configuration): String =
    config.get[String]("myapp.apiKey")

  @Provides
  @Named("baseUrl")
  def provideBaseUrl(config: Configuration): String =
    config.get[String]("myapp.baseUrl")

  override def configure(): Unit = {}
}
```

```scala
// ใช้ @Named ใน class
@Singleton
class ExternalApiClient @Inject()(
  @Named("apiKey") apiKey: String,
  @Named("baseUrl") baseUrl: String,
  wsClient: play.api.libs.ws.WSClient
) {
  def fetchData(path: String) =
    wsClient.url(s"$baseUrl/$path")
            .addHttpHeaders("X-API-Key" -> apiKey)
            .get()
}
```

---

## Step 518: Eager Singletons

```scala
// app/Module.scala
class Module extends AbstractModule {
  override def configure(): Unit = {
    // asEagerSingleton(): สร้าง instance ทันทีที่ app start
    // (ไม่รอให้มีการ request ครั้งแรก)
    bind(classOf[services.DatabaseMigrationService]).asEagerSingleton()
    bind(classOf[services.SchedulerService]).asEagerSingleton()
    bind(classOf[services.CacheWarmupService]).asEagerSingleton()

    // ใช้เมื่อต้องการ run initialization code ตอน startup
  }
}
```

```scala
// app/services/DatabaseMigrationService.scala
package services

import javax.inject.*
import play.api.Logger

// Service นี้จะ start ทันทีเมื่อ app start
@Singleton
class DatabaseMigrationService @Inject()() {

  private val logger = Logger(this.getClass)

  // Constructor code runs on startup
  logger.info("Running database migrations...")
  runMigrations()

  private def runMigrations(): Unit = {
    // run flyway migrations หรือ play evolutions
    logger.info("Database migrations completed")
  }
}
```

---

## Step 519: Testing with DI

```scala
// test/controllers/ArticleControllerWithDISpec.scala
package controllers

import org.scalatestplus.play.*
import org.scalatestplus.play.guice.*
import play.api.inject.*
import play.api.inject.guice.GuiceApplicationBuilder
import play.api.test.*
import play.api.test.Helpers.*
import org.mockito.Mockito.*
import scala.concurrent.Future

class ArticleControllerWithDISpec extends PlaySpec with GuiceOneAppPerTest {

  // Override bindings สำหรับ test
  override def fakeApplication() = {
    val mockArticleService = mock(classOf[services.ArticleService])
    val mockEmailService   = mock(classOf[services.EmailService])

    // Setup mock behavior
    when(mockArticleService.list(any(), any(), any(), any(), any()))
      .thenReturn(Future.successful((List.empty, 0L)))

    GuiceApplicationBuilder()
      .overrides(
        // Override real service ด้วย mock
        bind[services.ArticleService].toInstance(mockArticleService),
        bind[services.EmailService].toInstance(mockEmailService)
      )
      .build()
  }

  "ArticleController" should {
    "return empty list initially" in {
      val result = route(app, FakeRequest(GET, "/api/articles")).get
      status(result) mustBe OK
    }
  }
}
```

---

## Step 520: Module Lifecycle

```scala
// app/services/ApplicationLifecycle.scala
package services

import javax.inject.*
import play.api.inject.ApplicationLifecycle
import scala.concurrent.Future

@Singleton
class CleanupService @Inject()(
  lifecycle: ApplicationLifecycle
) {

  // Register cleanup code ที่จะ run เมื่อ app shutdown
  lifecycle.addStopHook { () =>
    Future.successful {
      // cleanup resources
      println("Cleaning up resources...")
      // close connections, flush caches, etc.
    }
  }

  // Startup code
  initialize()

  private def initialize(): Unit = {
    println("Application started, initializing services...")
  }
}
```

---

## สรุป Part 52

| Concept | Annotation/API | Use Case |
|---------|---------------|---------|
| Constructor Injection | `@Inject()` ใน constructor | Primary injection method |
| Singleton | `@Singleton` | Shared instances (DB connections) |
| Interface Binding | `bind(classOf[Interface]).to(classOf[Impl])` | Swappable implementations |
| Named Binding | `@Named("key")` | Multiple bindings ของ type เดียว |
| Provides | `@Provides` | Factory methods สำหรับ complex objects |
| Test Override | `bind[T].toInstance(mock)` | Mock dependencies ใน tests |
| Eager Singleton | `asEagerSingleton()` | Initialize ตอน startup |
| Lifecycle | `ApplicationLifecycle.addStopHook` | Cleanup ตอน shutdown |

---

## แบบฝึกหัด Part 52

1. **Repository Pattern**: สร้าง interface `ArticleRepository` พร้อม 2 implementations: `InMemoryArticleRepository` สำหรับ development และ `SlickArticleRepository` สำหรับ production

2. **Configuration Module**: สร้าง Module ที่ bind configuration values เป็น named dependencies เช่น S3 bucket name, JWT secret, external API keys

3. **Conditional Binding**: สร้าง Module ที่ bind ต่างกัน ตาม environment variable `FEATURE_PAYMENT_GATEWAY` เพื่อ switch ระหว่าง Stripe และ mock implementation

4. **Actor Binding**: สร้าง Module ที่ bind Pekko actors ด้วย `bindActor` พร้อม lifecycle management

5. **Test Setup**: สร้าง test ที่ใช้ GuiceApplicationBuilder.overrides เพื่อ inject mock services และ verify behavior โดยไม่ต้องใช้ real database

---

[→ ไปยัง Part 53: Play Configuration](part-53-play-configuration.md)
