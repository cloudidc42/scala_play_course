# Part 53: Play Configuration

## Steps 521-530: application.conf, Environment Configs, Feature Flags, Config at Runtime

---

## Step 521: HOCON Format

Play ใช้ HOCON (Human-Optimized Config Object Notation) สำหรับ configuration

```hocon
# conf/application.conf

# String
app.name = "My Play App"

# Number
app.port = 9000

# Boolean
app.debug = false

# List
app.tags = ["scala", "play", "web"]

# Object
app.database {
  host = "localhost"
  port = 5432
  name = "mydb"
}

# Nested object (dot notation)
app.redis.host = "localhost"
app.redis.port = 6379

# String substitution
app.baseUrl = "http://localhost:"${app.port}

# Environment variable override
app.secret = "default-secret"
app.secret = ${?APP_SECRET}   # override ด้วย env var ถ้ามี

# Include another file
include "database.conf"
include "email.conf"
```

---

## Step 522: การอ่าน Configuration ใน Scala

```scala
// app/config/AppConfig.scala
package config

import javax.inject.*
import play.api.Configuration

@Singleton
class AppConfig @Inject()(config: Configuration) {

  // อ่านค่า required (throw exception ถ้าไม่มี)
  val appName: String  = config.get[String]("app.name")
  val port: Int        = config.get[Int]("app.port")
  val debug: Boolean   = config.get[Boolean]("app.debug")

  // อ่านค่า optional (None ถ้าไม่มี)
  val adminEmail: Option[String] = config.getOptional[String]("app.adminEmail")

  // อ่าน nested config
  val dbHost: String = config.get[String]("app.database.host")
  val dbPort: Int    = config.get[Int]("app.database.port")

  // อ่าน config object
  val dbConfig: Configuration = config.get[Configuration]("app.database")

  // อ่าน list
  val tags: Seq[String] = config.get[Seq[String]]("app.tags")

  // อ่านด้วย default value
  val timeout: Int = config.getOptional[Int]("app.timeout").getOrElse(30)
  val maxRetries: Int = config.getOptional[Int]("app.maxRetries").getOrElse(3)

  // อ่าน Duration
  val sessionTimeout: scala.concurrent.duration.Duration =
    config.getOptional[scala.concurrent.duration.Duration]("app.sessionTimeout")
          .getOrElse(scala.concurrent.duration.Duration("1 hour"))
}
```

---

## Step 523: Configuration Case Class

```scala
// app/config/DatabaseConfig.scala
package config

import javax.inject.*
import play.api.Configuration

case class DatabaseConfig(
  host: String,
  port: Int,
  name: String,
  username: String,
  password: String,
  maxConnections: Int,
  connectionTimeout: Int
) {
  def url: String = s"jdbc:postgresql://$host:$port/$name"
}

object DatabaseConfig {
  def fromConfig(config: Configuration): DatabaseConfig = {
    val db = config.get[Configuration]("db.default")
    DatabaseConfig(
      host              = db.getOptional[String]("host").getOrElse("localhost"),
      port              = db.getOptional[Int]("port").getOrElse(5432),
      name              = db.getOptional[String]("name").getOrElse("mydb"),
      username          = db.getOptional[String]("username").getOrElse("postgres"),
      password          = db.getOptional[String]("password").getOrElse(""),
      maxConnections    = db.getOptional[Int]("maxConnections").getOrElse(10),
      connectionTimeout = db.getOptional[Int]("connectionTimeout").getOrElse(5000)
    )
  }
}

@Singleton
class DatabaseConfigProvider @Inject()(config: Configuration) {
  val dbConfig: DatabaseConfig = DatabaseConfig.fromConfig(config)
}
```

---

## Step 524: Environment-Specific Configuration

```hocon
# conf/application.conf - base configuration
app.name = "My App"
app.debug = false

# Database defaults
db.default {
  driver = "org.postgresql.Driver"
  url = "jdbc:postgresql://localhost/myapp"
  username = "postgres"
  password = ""
}

# Override ด้วย environment variable
db.default.url = ${?DATABASE_URL}
db.default.username = ${?DB_USERNAME}
db.default.password = ${?DB_PASSWORD}
```

```hocon
# conf/application.dev.conf - Development overrides
include "application.conf"

app.debug = true

db.default {
  url = "jdbc:postgresql://localhost/myapp_dev"
  username = "dev_user"
  password = "dev_password"
}

# Development: ใช้ log email
services.email.provider = "log"

# Development: disable some filters
play.filters.disabled += "play.filters.hosts.AllowedHostsFilter"
```

```hocon
# conf/application.test.conf - Test overrides
include "application.conf"

db.default {
  driver = "org.h2.Driver"
  url = "jdbc:h2:mem:test;MODE=PostgreSQL;DB_CLOSE_DELAY=-1"
}

# Test: disable real external services
services.email.provider = "log"
services.payment.provider = "mock"
services.storage.provider = "memory"

# Test: disable rate limiting
play.filters.disabled += "filters.RateLimitFilter"
```

```hocon
# conf/application.prod.conf - Production overrides
include "application.conf"

# Production: strict security
play.http.secret.key = ${APPLICATION_SECRET}

play.filters.headers {
  strictTransportSecurity = "max-age=31536000; includeSubDomains"
}

# Production: real services
services.email.provider = "smtp"
services.storage.provider = "s3"
```

### เลือก config ตาม environment

```bash
# รัน development
sbt run

# รัน test
sbt -Dconfig.file=conf/application.test.conf test

# รัน production
sbt -Dconfig.file=conf/application.prod.conf run

# หรือ environment variable
CONFIG_FILE=conf/application.prod.conf sbt run
```

```hocon
# conf/application.conf - auto-detect environment
# ใช้ play.mode ในการเลือก config
play.mode = ${?PLAY_MODE}  # dev, test, prod

# Include environment-specific config
include "environments/"${?PLAY_MODE}".conf"
```

---

## Step 525: Feature Flags

```scala
// app/config/FeatureFlags.scala
package config

import javax.inject.*
import play.api.Configuration

@Singleton
class FeatureFlags @Inject()(config: Configuration) {

  private def flag(key: String, default: Boolean = false): Boolean =
    config.getOptional[Boolean](s"features.$key").getOrElse(default)

  // Feature flags
  val newCheckoutFlow: Boolean     = flag("newCheckoutFlow")
  val darkModeEnabled: Boolean     = flag("darkModeEnabled", default = true)
  val betaFeatures: Boolean        = flag("betaFeatures")
  val maintenanceMode: Boolean     = flag("maintenanceMode")
  val analyticsEnabled: Boolean    = flag("analyticsEnabled", default = true)
  val aiRecommendations: Boolean   = flag("aiRecommendations")

  // Feature flag ที่ขึ้นกับ environment
  val debugTools: Boolean          = flag("debugTools") &&
                                     !config.get[String]("play.mode").equalsIgnoreCase("prod")
}
```

```hocon
# conf/application.conf
features {
  newCheckoutFlow = false
  darkModeEnabled = true
  betaFeatures = false
  maintenanceMode = false
  analyticsEnabled = true
  aiRecommendations = false
}
```

```scala
// Controller ที่ใช้ feature flags
@Singleton
class CheckoutController @Inject()(
  val controllerComponents: ControllerComponents,
  features: FeatureFlags
) extends BaseController {

  def checkout(): Action[AnyContent] = Action { implicit request =>
    if (features.maintenanceMode) {
      ServiceUnavailable(views.html.maintenance())
    } else if (features.newCheckoutFlow) {
      Ok(views.html.checkout.new_flow())
    } else {
      Ok(views.html.checkout.classic())
    }
  }
}
```

---

## Step 526: Dynamic Configuration (Config Reload)

```scala
// app/services/DynamicConfigService.scala
package services

import javax.inject.*
import play.api.Configuration
import java.util.concurrent.atomic.AtomicReference

// Config ที่ update ได้ runtime (ตัวอย่าง: อ่านจาก DB หรือ Redis)
@Singleton
class DynamicConfigService @Inject()(
  initialConfig: Configuration,
  implicit val ec: scala.concurrent.ExecutionContext
) {

  // AtomicReference สำหรับ thread-safe updates
  private val configRef = new AtomicReference[Map[String, Any]](Map(
    "maxUploadSize"  -> 5242880,   // 5MB
    "sessionTimeout" -> 3600,      // 1 hour
    "maintenanceMode"-> false,
    "rateLimit"      -> 100
  ))

  def get(key: String): Option[Any] = configRef.get().get(key)

  def getString(key: String, default: String = ""): String =
    get(key).map(_.toString).getOrElse(default)

  def getInt(key: String, default: Int = 0): Int =
    get(key).map(_.toString.toInt).getOrElse(default)

  def getBoolean(key: String, default: Boolean = false): Boolean =
    get(key).map(_.toString.toBoolean).getOrElse(default)

  // Update config values
  def update(key: String, value: Any): Unit = {
    val current = configRef.get()
    configRef.set(current + (key -> value))
  }

  // Reload from external source
  def reload(): scala.concurrent.Future[Unit] = scala.concurrent.Future {
    // อ่านค่าจาก database หรือ Redis
    // val newConfig = database.fetchConfig()
    // configRef.set(newConfig)
    ()
  }
}
```

---

## Step 527: Custom ConfigLoader

```scala
// app/config/CustomLoaders.scala
package config

import play.api.ConfigLoader
import com.typesafe.config.Config

// Custom ConfigLoader สำหรับ custom types
case class EmailConfig(
  host: String,
  port: Int,
  username: String,
  password: String,
  tls: Boolean
)

object EmailConfig {
  // สร้าง ConfigLoader สำหรับ EmailConfig
  implicit val configLoader: ConfigLoader[EmailConfig] = (rootConfig: Config, path: String) => {
    val config = rootConfig.getConfig(path)
    EmailConfig(
      host     = config.getString("host"),
      port     = config.getInt("port"),
      username = config.getString("username"),
      password = config.getString("password"),
      tls      = config.getBoolean("tls")
    )
  }
}

// ใช้ custom loader
@javax.inject.Singleton
class EmailService @javax.inject.Inject()(config: play.api.Configuration) {
  val emailConfig: EmailConfig = config.get[EmailConfig]("email.smtp")
}
```

```hocon
# conf/application.conf
email.smtp {
  host = "smtp.gmail.com"
  port = 587
  username = ${?SMTP_USERNAME}
  password = ${?SMTP_PASSWORD}
  tls = true
}
```

---

## Step 528: Configuration Validation

```scala
// app/config/ConfigValidator.scala
package config

import javax.inject.*
import play.api.{Configuration, Logger}

@Singleton
class ConfigValidator @Inject()(config: Configuration) {

  private val logger = Logger(this.getClass)

  // Validate ทันทีที่ start
  validate()

  def validate(): Unit = {
    val errors = scala.collection.mutable.ListBuffer[String]()

    // Required configs
    requiredString("play.http.secret.key", errors)
    requiredString("db.default.url", errors)

    // Validate secret key ไม่ใช่ default value ใน production
    val secretKey = config.getOptional[String]("play.http.secret.key").getOrElse("")
    val isProduction = config.getOptional[String]("play.mode")
                              .exists(_ == "prod")
    if (isProduction && secretKey == "changeme") {
      errors += "play.http.secret.key ต้องไม่ใช่ 'changeme' ใน production"
    }

    // Validate URL format
    config.getOptional[String]("db.default.url").foreach { url =>
      if (!url.startsWith("jdbc:")) {
        errors += "db.default.url ต้องขึ้นต้นด้วย 'jdbc:'"
      }
    }

    if (errors.nonEmpty) {
      val message = s"Configuration errors:\n${errors.mkString("\n- ", "\n- ", "")}"
      if (isProduction) {
        throw new RuntimeException(message)
      } else {
        errors.foreach(e => logger.warn(s"Config warning: $e"))
      }
    } else {
      logger.info("Configuration validated successfully")
    }
  }

  private def requiredString(key: String, errors: scala.collection.mutable.ListBuffer[String]): Unit = {
    config.getOptional[String](key) match {
      case None | Some("") => errors += s"Required config '$key' is missing or empty"
      case _               => // OK
    }
  }
}
```

---

## Step 529: Multi-environment Config ด้วย Variables

```hocon
# conf/application.conf - Production-ready configuration

# App settings
app {
  name = "My Play App"
  version = "1.0.0"
  environment = "development"
  environment = ${?PLAY_ENV}  # override: development, staging, production
}

# Database
db.default {
  driver = "org.postgresql.Driver"
  url = "jdbc:postgresql://localhost/myapp_dev"
  url = ${?DATABASE_URL}
  username = "postgres"
  username = ${?DB_USER}
  password = ""
  password = ${?DB_PASSWORD}
  hikaricp {
    maximumPoolSize = 10
    minimumIdle = 2
    connectionTimeout = 5000
    idleTimeout = 600000
    maxLifetime = 1800000
  }
}

# Redis
redis {
  host = "localhost"
  host = ${?REDIS_HOST}
  port = 6379
  port = ${?REDIS_PORT}
  password = ${?REDIS_PASSWORD}
  database = 0
}

# JWT
jwt {
  secret = "dev-secret-change-in-production"
  secret = ${?JWT_SECRET}
  expirationMinutes = 1440  # 24 hours
}

# S3
s3 {
  bucket = ${?S3_BUCKET}
  region = "ap-southeast-1"
  region = ${?AWS_REGION}
}

# Email
email {
  provider = "log"  # log, smtp, sendgrid
  provider = ${?EMAIL_PROVIDER}
  smtp {
    host = "smtp.gmail.com"
    host = ${?SMTP_HOST}
    port = 587
    port = ${?SMTP_PORT}
    username = ${?SMTP_USER}
    password = ${?SMTP_PASSWORD}
  }
}
```

---

## Step 530: Config ใน Controller/Service

```scala
// app/controllers/ConfigDemoController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.Configuration
import play.api.libs.json.*
import config.{AppConfig, FeatureFlags}

@Singleton
class ConfigDemoController @Inject()(
  val controllerComponents: ControllerComponents,
  config: Configuration,
  appConfig: AppConfig,
  features: FeatureFlags
) extends BaseController {

  // แสดง config (ระวัง - ไม่ควรแสดงใน production!)
  def showConfig(): Action[AnyContent] = Action { implicit request =>
    if (!config.getOptional[Boolean]("app.debug").getOrElse(false)) {
      return Forbidden("Config endpoint disabled in non-debug mode")
    }

    Ok(Json.obj(
      "appName"     -> appConfig.appName,
      "environment" -> config.getOptional[String]("app.environment").getOrElse("unknown"),
      "features"    -> Json.obj(
        "newCheckout"   -> features.newCheckoutFlow,
        "darkMode"      -> features.darkModeEnabled,
        "maintenance"   -> features.maintenanceMode
      )
    ))
  }

  // Toggle feature flag (admin only)
  def toggleFeature(feature: String): Action[AnyContent] = Action { implicit request =>
    // ใน production อาจต้องการ admin auth
    feature match {
      case "maintenance" =>
        // Update config dynamic
        Ok(Json.obj("feature" -> feature, "toggled" -> true))
      case _ =>
        NotFound(Json.obj("error" -> s"Unknown feature: $feature"))
    }
  }
}
```

---

## สรุป Part 53

| Config Pattern | HOCON Syntax | Scala API |
|----------------|-------------|-----------|
| String | `key = "value"` | `config.get[String]("key")` |
| Number | `key = 42` | `config.get[Int]("key")` |
| Boolean | `key = true` | `config.get[Boolean]("key")` |
| Optional | `key = ${?ENV_VAR}` | `config.getOptional[String]("key")` |
| Nested | `key { sub = value }` | `config.get[Configuration]("key")` |
| List | `key = [a, b, c]` | `config.get[Seq[String]]("key")` |
| Override | `include "other.conf"` | env-specific overrides |
| Env var | `${?ENV_VAR}` | read from environment |

---

## แบบฝึกหัด Part 53

1. **Config Hierarchy**: สร้าง config hierarchy สำหรับ 3 environments: dev, staging, prod โดยแต่ละ environment override เฉพาะค่าที่แตกต่าง

2. **Feature Toggle API**: สร้าง admin API endpoint สำหรับ toggle feature flags แบบ runtime โดยไม่ต้อง restart server (เก็บใน Redis)

3. **Config Validation**: สร้าง startup validator ที่ตรวจสอบว่า config ครบถ้วนและถูกต้องก่อน application start - fail fast ถ้า config ไม่ valid

4. **Custom ConfigLoader**: สร้าง `ConfigLoader[RateLimitConfig]` ที่ parse complex rate limit configuration จาก HOCON

5. **Secret Management**: Design configuration สำหรับ production ที่ secrets ทั้งหมด (DB password, JWT secret, API keys) ถูก inject ผ่าน environment variables และไม่มีใน codebase

---

[→ ไปยัง Part 54: Play Async and Streaming](part-54-play-async-and-streaming.md)
