# Part 10 — SBT Build Tool และ Project Structure
## Steps 91–100: Mastering the Scala Build Ecosystem

> **เป้าหมาย**: เชี่ยวชาญ SBT, project structure, dependencies, testing, และ packaging

---

## Step 91 — SBT Deep Dive

```bash
# SBT version
sbt -version
sbt about

# SBT interactive shell
sbt

# Common commands
sbt compile          # compile sources
sbt run              # run main class
sbt test             # run all tests
sbt clean            # clean target directory
sbt package          # create JAR
sbt assembly         # fat JAR (needs plugin)
sbt "run arg1 arg2"  # run with args
sbt ~compile         # watch mode (recompile on change)
sbt ~test            # watch and test
```

### SBT Shell Commands

```
sbt> compile
sbt> run
sbt> test
sbt> testOnly com.example.MySuite
sbt> testQuick          # run only failed tests
sbt> clean
sbt> reload             # reload build definition
sbt> projects           # list projects
sbt> project <name>     # switch project
sbt> tasks              # list all tasks
sbt> settings           # list all settings
sbt> inspect compile    # inspect task
sbt> show scalaVersion  # show setting value
sbt> dependencies       # show dependency tree
sbt> update             # download dependencies
sbt> help               # show help
```

---

## Step 92 — build.sbt ครบถ้วน

```scala
// build.sbt — Complete project definition

// Global settings
ThisBuild / scalaVersion     := "3.3.1"
ThisBuild / organization     := "com.mycompany"
ThisBuild / organizationName := "My Company"

// Scala compiler options
lazy val compilerOptions = Seq(
  "-encoding", "UTF-8",
  "-feature",
  "-deprecation",
  "-unchecked",
  "-explain",
  "-Xfatal-warnings",     // treat warnings as errors
  "-Ycheck-init",         // check initialization
)

// Common settings
lazy val commonSettings = Seq(
  scalacOptions ++= compilerOptions,
  
  // Test framework
  libraryDependencies ++= Seq(
    "org.scalatest" %% "scalatest"         % "3.2.17" % Test,
    "org.scalameta" %% "munit"             % "0.7.29" % Test,
    "com.github.sbt" % "junit-interface"   % "0.13.3" % Test,
  ),
  
  // Parallel test execution
  Test / parallelExecution := false,
  
  // Show full stack traces
  Test / testOptions += Tests.Argument("-oF"),
  
  // Source encoding
  Compile / doc / scalacOptions ++= Seq(
    "-project", name.value,
    "-project-version", version.value,
  )
)

// Root project
lazy val root = project
  .in(file("."))
  .aggregate(core, api, data)
  .settings(
    name := "my-scala-app",
    version := "1.0.0"
  )

// Core module
lazy val core = project
  .in(file("modules/core"))
  .settings(
    commonSettings,
    name := "core",
    libraryDependencies ++= Seq(
      "org.typelevel" %% "cats-core"   % "2.10.0",
      "org.typelevel" %% "cats-effect" % "3.5.2",
      "io.circe"      %% "circe-core"  % "0.14.6",
      "io.circe"      %% "circe-generic" % "0.14.6",
      "io.circe"      %% "circe-parser"  % "0.14.6",
    )
  )

// API module
lazy val api = project
  .in(file("modules/api"))
  .dependsOn(core)
  .settings(
    commonSettings,
    name := "api",
    libraryDependencies ++= Seq(
      "com.typesafe.play" %% "play"         % "3.0.1",
      "com.typesafe.play" %% "play-json"    % "3.0.1",
      "com.typesafe.play" %% "play-slick"   % "6.0.0",
    )
  )

// Data module  
lazy val data = project
  .in(file("modules/data"))
  .dependsOn(core)
  .settings(
    commonSettings,
    name := "data",
    libraryDependencies ++= Seq(
      "org.apache.spark" %% "spark-core"    % "3.5.0",
      "org.apache.spark" %% "spark-sql"     % "3.5.0",
      "org.apache.kafka"  % "kafka-clients" % "3.6.0",
    )
  )
```

---

## Step 93 — Project Structure Multi-module

```
my-scala-app/
├── build.sbt
├── project/
│   ├── build.properties        ← SBT version
│   ├── plugins.sbt             ← SBT plugins
│   └── Dependencies.scala      ← Dependency management
├── modules/
│   ├── core/
│   │   └── src/
│   │       ├── main/scala/com/myapp/core/
│   │       │   ├── domain/
│   │       │   │   ├── User.scala
│   │       │   │   └── Order.scala
│   │       │   ├── service/
│   │       │   │   └── UserService.scala
│   │       │   └── repository/
│   │       │       └── UserRepository.scala
│   │       └── test/scala/com/myapp/core/
│   │           └── domain/UserSpec.scala
│   ├── api/
│   │   └── src/main/scala/com/myapp/api/
│   │       ├── controllers/
│   │       ├── models/
│   │       └── routes/
│   └── data/
│       └── src/main/scala/com/myapp/data/
│           ├── pipeline/
│           └── jobs/
└── conf/
    ├── application.conf
    └── routes
```

### project/Dependencies.scala

```scala
// project/Dependencies.scala — Centralized dependency management
import sbt._

object Dependencies {
  
  object Versions {
    val scala3    = "3.3.1"
    val cats      = "2.10.0"
    val catsEffect = "3.5.2"
    val circe     = "0.14.6"
    val play      = "3.0.1"
    val slick     = "3.4.1"
    val spark     = "3.5.0"
    val kafka     = "3.6.0"
    val scalatest = "3.2.17"
    val munit     = "0.7.29"
    val logback   = "1.4.11"
    val typesafeConfig = "1.4.3"
    val postgres  = "42.7.1"
    val hikari    = "5.1.0"
  }
  
  import Versions._
  
  // Functional
  val cats          = "org.typelevel" %% "cats-core"         % cats
  val catsEffect    = "org.typelevel" %% "cats-effect"       % catsEffect
  val catsEffectTest = "org.typelevel" %% "cats-effect-testing-scalatest" % "1.5.0" % Test
  
  // JSON
  val circeCore    = "io.circe" %% "circe-core"    % circe
  val circeGeneric = "io.circe" %% "circe-generic" % circe
  val circeParser  = "io.circe" %% "circe-parser"  % circe
  val circeAll     = Seq(circeCore, circeGeneric, circeParser)
  
  // Database
  val slickCore    = "com.typesafe.slick" %% "slick"          % slick
  val slickHikari  = "com.typesafe.slick" %% "slick-hikaricp" % slick
  val postgres     = "org.postgresql"      % "postgresql"      % postgres
  val hikari       = "com.zaxxer"          % "HikariCP"        % hikari
  
  // Logging
  val logback      = "ch.qos.logback"     % "logback-classic" % logback
  val scalaLogging = "com.typesafe.scala-logging" %% "scala-logging" % "3.9.5"
  
  // Config
  val typesafeConf = "com.typesafe"       % "config"          % typesafeConfig
  
  // Testing
  val scalatest    = "org.scalatest"      %% "scalatest"       % scalatest % Test
  val scalamock    = "org.scalamock"      %% "scalamock"       % "5.2.0"   % Test
  val munit        = "org.scalameta"      %% "munit"           % munit     % Test
  
  // Groups
  val coreLibs  = Seq(cats, catsEffect, typesafeConf, logback, scalaLogging)
  val dbLibs    = Seq(slickCore, slickHikari, postgres, hikari)
  val jsonLibs  = circeAll
  val testLibs  = Seq(scalatest, scalamock, catsEffectTest)
}
```

### project/plugins.sbt

```scala
// project/plugins.sbt

// sbt-assembly: create fat JARs
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "2.1.3")

// sbt-native-packager: Docker, RPM, deb packages
addSbtPlugin("com.github.sbt" % "sbt-native-packager" % "1.9.16")

// sbt-scalafmt: code formatting
addSbtPlugin("org.scalameta" % "sbt-scalafmt" % "2.5.2")

// sbt-scoverage: test coverage
addSbtPlugin("org.scoverage" % "sbt-scoverage" % "2.0.9")

// sbt-revolver: hot reload
addSbtPlugin("io.spray" % "sbt-revolver" % "0.10.0")

// sbt-wartremover: static analysis
addSbtPlugin("org.wartremover" % "sbt-wartremover" % "3.1.5")

// sbt-dependency-graph: visualize dependencies
addSbtPlugin("net.virtual-void" % "sbt-dependency-graph" % "0.10.0-RC1")
```

---

## Step 94 — Testing กับ ScalaTest

```scala
// src/test/scala/com/myapp/UserSpec.scala

import org.scalatest.funsuite.AnyFunSuite
import org.scalatest.matchers.should.Matchers
import org.scalatest.BeforeAndAfterEach

class UserSpec extends AnyFunSuite with Matchers with BeforeAndAfterEach {
  
  // Setup
  var testUsers: List[User] = _
  
  override def beforeEach(): Unit = {
    testUsers = List(
      User(1, "Alice", "alice@example.com"),
      User(2, "Bob",   "bob@example.com"),
      User(3, "Charlie", "charlie@example.com")
    )
  }
  
  test("User can be created") {
    val user = User(1, "Test", "test@test.com")
    user.name shouldBe "Test"
    user.email shouldBe "test@test.com"
  }
  
  test("Users can be filtered by name") {
    val filtered = testUsers.filter(_.name.startsWith("A"))
    filtered should have length 1
    filtered.head.name shouldBe "Alice"
  }
  
  test("User email validation") {
    val valid   = User.validateEmail("test@example.com")
    val invalid = User.validateEmail("not-an-email")
    
    valid shouldBe Right("test@example.com")
    invalid shouldBe a[Left[?, ?]]
  }
  
  test("User sorting by name") {
    val sorted = testUsers.sortBy(_.name)
    sorted.map(_.name) shouldBe List("Alice", "Bob", "Charlie")
  }
}

// FlatSpec style
import org.scalatest.flatspec.AnyFlatSpec

class CalculatorSpec extends AnyFlatSpec with Matchers {
  
  "A Calculator" should "add two numbers correctly" in {
    Calculator.add(2, 3) shouldBe 5
  }
  
  it should "handle negative numbers" in {
    Calculator.add(-1, 1) shouldBe 0
  }
  
  "Division" should "throw ArithmeticException for zero divisor" in {
    an[ArithmeticException] should be thrownBy {
      Calculator.divide(10, 0)
    }
  }
}

// WordSpec style
import org.scalatest.wordspec.AnyWordSpec

class StringProcessorSpec extends AnyWordSpec with Matchers {
  
  "StringProcessor" when {
    "processing valid input" should {
      "convert to uppercase" in {
        StringProcessor.process("hello") shouldBe "HELLO"
      }
      
      "trim whitespace" in {
        StringProcessor.process("  hello  ") shouldBe "HELLO"
      }
    }
    
    "processing empty input" should {
      "return empty string" in {
        StringProcessor.process("") shouldBe ""
      }
    }
  }
}
```

---

## Step 95 — Testing กับ MUnit

```scala
// src/test/scala/com/myapp/OrderSuite.scala

import munit._

class OrderSuite extends FunSuite {
  
  // Basic test
  test("empty order has zero total") {
    val order = Order.empty
    assertEquals(order.total, 0.0)
  }
  
  test("add item increases total") {
    val order = Order.empty
      .addItem("widget", 2, 9.99)
    assertEqualsDouble(order.total, 19.98, delta = 0.001)
  }
  
  // Fixtures
  val testOrder = FunFixture[Order](
    setup = _ => Order.create(List(
      OrderItem("a", 1, 10.0),
      OrderItem("b", 2, 5.0)
    )),
    teardown = order => {
      // cleanup if needed
    }
  )
  
  testOrder.test("order has correct item count") { order =>
    assertEquals(order.items.length, 2)
  }
  
  testOrder.test("order total is correct") { order =>
    assertEqualsDouble(order.total, 20.0, delta = 0.001)
  }
  
  // Property-based testing with MUnit
  test("sum of n numbers equals arithmetic sum") {
    for (n <- 1 to 100) {
      val nums = (1 to n).toList
      assertEquals(nums.sum, n * (n + 1) / 2)
    }
  }
}
```

### Property-Based Testing กับ ScalaCheck

```scala
// build.sbt: libraryDependencies += "org.scalacheck" %% "scalacheck" % "1.17.0" % Test

import org.scalacheck._
import org.scalacheck.Prop._
import org.scalatest.propspec.AnyPropSpec
import org.scalatestplus.scalacheck.ScalaCheckPropertyChecks

class ListPropertySpec extends AnyPropSpec with ScalaCheckPropertyChecks {
  
  property("reverse twice is identity") {
    forAll { (list: List[Int]) =>
      list.reverse.reverse == list
    }
  }
  
  property("sort is idempotent") {
    forAll { (list: List[Int]) =>
      list.sorted.sorted == list.sorted
    }
  }
  
  property("concat length equals sum of lengths") {
    forAll { (a: List[Int], b: List[Int]) =>
      (a ++ b).length == a.length + b.length
    }
  }
  
  property("filter removes elements correctly") {
    forAll { (list: List[Int]) =>
      val evens = list.filter(_ % 2 == 0)
      evens.forall(_ % 2 == 0)
    }
  }
}
```

---

## Step 96 — Packaging และ Deployment

```scala
// build.sbt — packaging settings

import com.typesafe.sbt.packager.archetypes.JavaAppPackaging

lazy val app = project
  .in(file("."))
  .enablePlugins(JavaAppPackaging, DockerPlugin)
  .settings(
    name    := "my-scala-app",
    version := "1.0.0",
    
    // Main class
    Compile / mainClass := Some("com.myapp.Main"),
    assembly / mainClass := Some("com.myapp.Main"),
    
    // Assembly (fat JAR)
    assembly / assemblyMergeStrategy := {
      case PathList("META-INF", xs @ _*) => xs match {
        case "MANIFEST.MF" :: Nil => MergeStrategy.discard
        case "services" :: _      => MergeStrategy.concat
        case _                    => MergeStrategy.discard
      }
      case "reference.conf" => MergeStrategy.concat
      case x                => MergeStrategy.first
    },
    
    // Docker settings
    Docker / packageName := "my-scala-app",
    Docker / version     := version.value,
    dockerBaseImage      := "eclipse-temurin:17-jre-alpine",
    dockerExposedPorts   := Seq(8080),
    dockerExposedVolumes := Seq("/data"),
  )
```

```bash
# Create fat JAR
sbt assembly
java -jar target/scala-3.3.1/my-scala-app-assembly-1.0.0.jar

# Create Docker image
sbt Docker/publishLocal
docker run -p 8080:8080 my-scala-app:1.0.0

# Create distribution
sbt Universal/packageBin     # .zip
sbt Universal/packageZipTarball  # .tgz
sbt Debian/packageBin        # .deb
sbt Rpm/packageBin           # .rpm
```

---

## Step 97 — Code Quality Tools

```scala
// .scalafmt.conf — code formatting

version = "3.7.17"
runner.dialect = scala3

maxColumn = 120

align.preset = more
align.openParenDefnSite = false
align.openParenCallSite = false

newlines.topLevelBodyIfMinStatements = []
newlines.beforeCurlyLambdaParams = never
newlines.afterCurlyLambdaParams = squash

indent.defnSite = 2
indent.callSite = 2

rewrite.rules = [
  PreferCurlyFors
  AvoidInfix
  SortImports
]

docstrings.style = SpaceAsterisk
```

```scala
// project/plugins.sbt — WartRemover (static analysis)
addSbtPlugin("org.wartremover" % "sbt-wartremover" % "3.1.5")

// build.sbt
wartremoverWarnings ++= Warts.allBut(
  Wart.Any,
  Wart.Nothing,
  Wart.DefaultArguments,
  Wart.ImplicitParameter,
)
```

```bash
# Format code
sbt scalafmtAll           # format all files
sbt scalafmtCheck         # check formatting (CI)
sbt scalafmtSbt           # format build files

# Coverage
sbt coverage test         # run with coverage
sbt coverageReport        # generate report
```

---

## Step 98 — Continuous Integration

```yaml
# .github/workflows/ci.yml

name: CI

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: password
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: --health-cmd pg_isready
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
          cache: sbt
      
      - name: Format check
        run: sbt scalafmtCheck
      
      - name: Compile
        run: sbt compile
      
      - name: Test with coverage
        run: sbt coverage test coverageReport
        env:
          DB_URL: jdbc:postgresql://localhost:5432/testdb
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
      
      - name: Build assembly
        run: sbt assembly
      
      - name: Build Docker image
        run: sbt Docker/publishLocal
```

---

## Step 99 — Configuration Management

```hocon
# conf/application.conf — Typesafe Config

app {
  name = "My Scala App"
  version = "1.0.0"
  
  server {
    host = "0.0.0.0"
    host = ${?SERVER_HOST}  # override from env var
    port = 8080
    port = ${?SERVER_PORT}
  }
  
  database {
    url      = "jdbc:postgresql://localhost:5432/myapp"
    url      = ${?DB_URL}
    user     = "postgres"
    user     = ${?DB_USER}
    password = "password"
    password = ${?DB_PASSWORD}
    
    pool {
      min-size = 5
      max-size = 20
      timeout  = 30000
    }
  }
  
  cache {
    ttl     = 3600s
    max-size = 10000
  }
}
```

```scala
// Config loading
import com.typesafe.config.{Config, ConfigFactory}

object AppConfig {
  private val config = ConfigFactory.load()
  
  object Server {
    val host = config.getString("app.server.host")
    val port = config.getInt("app.server.port")
  }
  
  object Database {
    val url      = config.getString("app.database.url")
    val user     = config.getString("app.database.user")
    val password = config.getString("app.database.password")
    val poolMin  = config.getInt("app.database.pool.min-size")
    val poolMax  = config.getInt("app.database.pool.max-size")
  }
  
  // Type-safe config with case classes
  case class DatabaseConfig(url: String, user: String, password: String)
  
  def dbConfig: DatabaseConfig = DatabaseConfig(
    url      = Database.url,
    user     = Database.user,
    password = Database.password
  )
}
```

---

## Step 100 — โปรแกรม SBT & Build Complete

สร้าง multi-module project ที่ production-ready:

```bash
# Structure
my-app/
├── build.sbt
├── project/
│   ├── build.properties
│   ├── plugins.sbt
│   └── Dependencies.scala
├── modules/
│   ├── core/src/main/scala/com/myapp/
│   │   ├── domain/
│   │   ├── repository/
│   │   └── service/
│   └── api/src/main/scala/com/myapp/
│       └── controllers/
├── conf/
│   └── application.conf
└── .github/
    └── workflows/ci.yml
```

```scala
// modules/core/src/main/scala/com/myapp/domain/User.scala
package com.myapp.domain

import java.time.Instant

case class UserId(value: Long) extends AnyVal

case class User(
  id:        UserId,
  name:      String,
  email:     String,
  createdAt: Instant = Instant.now()
)

object User {
  def create(name: String, email: String): Either[String, User] =
    for {
      validName  <- validateName(name)
      validEmail <- validateEmail(email)
    } yield User(UserId(0), validName, validEmail)
  
  private def validateName(name: String): Either[String, String] =
    if (name.trim.length >= 2) Right(name.trim)
    else Left("Name must be at least 2 characters")
  
  private def validateEmail(email: String): Either[String, String] =
    if (email.contains("@") && email.contains(".")) Right(email.trim.toLowerCase)
    else Left("Invalid email format")
}
```

```scala
// modules/core/src/main/scala/com/myapp/repository/UserRepository.scala
package com.myapp.repository

import com.myapp.domain.{User, UserId}
import scala.concurrent.Future

trait UserRepository {
  def findById(id: UserId): Future[Option[User]]
  def findAll: Future[List[User]]
  def save(user: User): Future[User]
  def delete(id: UserId): Future[Boolean]
}

// In-memory implementation for testing
class InMemoryUserRepository extends UserRepository {
  import scala.collection.mutable
  import scala.concurrent.ExecutionContext.Implicits.global
  
  private val store = mutable.Map[UserId, User]()
  private var nextId = 1L
  
  def findById(id: UserId): Future[Option[User]] =
    Future.successful(store.get(id))
  
  def findAll: Future[List[User]] =
    Future.successful(store.values.toList)
  
  def save(user: User): Future[User] = Future.successful {
    val id = if (user.id.value == 0) UserId(nextId++) else user.id
    val saved = user.copy(id = id)
    store(id) = saved
    saved
  }
  
  def delete(id: UserId): Future[Boolean] = Future.successful {
    val existed = store.contains(id)
    store.remove(id)
    existed
  }
}
```

```scala
// modules/core/src/main/scala/com/myapp/service/UserService.scala
package com.myapp.service

import com.myapp.domain.{User, UserId}
import com.myapp.repository.UserRepository
import scala.concurrent.{ExecutionContext, Future}

class UserService(repository: UserRepository)(implicit ec: ExecutionContext) {
  
  def createUser(name: String, email: String): Future[Either[String, User]] =
    User.create(name, email) match {
      case Left(error) => Future.successful(Left(error))
      case Right(user) => repository.save(user).map(Right(_))
    }
  
  def getUser(id: UserId): Future[Option[User]] =
    repository.findById(id)
  
  def getAllUsers: Future[List[User]] =
    repository.findAll
  
  def updateUser(id: UserId, name: String): Future[Option[User]] =
    for {
      maybeUser <- repository.findById(id)
      result    <- maybeUser match {
        case None       => Future.successful(None)
        case Some(user) => repository.save(user.copy(name = name)).map(Some(_))
      }
    } yield result
  
  def deleteUser(id: UserId): Future[Boolean] =
    repository.delete(id)
}
```

```scala
// modules/core/src/test/scala/com/myapp/service/UserServiceSpec.scala
package com.myapp.service

import com.myapp.domain.UserId
import com.myapp.repository.InMemoryUserRepository
import org.scalatest.funsuite.AsyncFunSuite
import org.scalatest.matchers.should.Matchers

class UserServiceSpec extends AsyncFunSuite with Matchers {
  
  def makeService(): UserService =
    new UserService(new InMemoryUserRepository())
  
  test("create valid user") {
    val service = makeService()
    service.createUser("Alice", "alice@example.com").map {
      case Right(user) =>
        user.name shouldBe "Alice"
        user.email shouldBe "alice@example.com"
      case Left(err) => fail(s"Expected Right but got Left($err)")
    }
  }
  
  test("reject invalid email") {
    val service = makeService()
    service.createUser("Bob", "not-an-email").map {
      case Left(_)     => succeed
      case Right(user) => fail(s"Expected Left but got Right($user)")
    }
  }
  
  test("get all users") {
    val service = makeService()
    for {
      _ <- service.createUser("Alice", "alice@example.com")
      _ <- service.createUser("Bob", "bob@example.com")
      users <- service.getAllUsers
    } yield {
      users should have length 2
    }
  }
  
  test("delete user") {
    val service = makeService()
    for {
      Right(user) <- service.createUser("Alice", "alice@example.com")
      deleted <- service.deleteUser(user.id)
      found <- service.getUser(user.id)
    } yield {
      deleted shouldBe true
      found shouldBe None
    }
  }
}
```

```bash
# Run everything
sbt "project core" test   # test core only
sbt test                  # test all modules
sbt "testOnly com.myapp.service.*"  # test pattern
sbt coverage test coverageReport    # with coverage
```

---

## สรุป Part 10 (และ Phase 1!)

| Step | สิ่งที่เรียน |
|------|-------------|
| 91 | SBT commands และ shell |
| 92 | build.sbt ครบถ้วน |
| 93 | Multi-module project structure |
| 94 | Testing กับ ScalaTest |
| 95 | Testing กับ MUnit + ScalaCheck |
| 96 | Packaging (JAR, Docker) |
| 97 | Code quality tools |
| 98 | CI/CD |
| 99 | Configuration management |
| 100 | Production-ready project |

## แบบฝึกหัด Phase 1 Final

1. สร้าง multi-module SBT project สำหรับ TODO app
2. เพิ่ม ScalaFmt และ WartRemover
3. เขียน unit tests ครอบคลุม > 80%
4. สร้าง Docker image
5. ตั้ง GitHub Actions CI

---

## 🎉 Phase 1 Complete!

คุณได้เรียนรู้:
- ✅ Scala syntax ทั้งหมด
- ✅ Type system
- ✅ Functions (HOF, recursion, closures)
- ✅ Collections
- ✅ OOP (classes, traits, ADTs)
- ✅ Error handling (Option, Either, Try)
- ✅ SBT, testing, packaging

---

## ต่อไป Phase 2

**[Part 11 →](part-11-oop-inheritance.md)** — Inheritance และ Polymorphism (Steps 101–110)
