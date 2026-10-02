# Part 41: Play Framework Introduction

## Steps 401-410: Play Framework Overview, Installation, Creating First App, MVC Architecture, Hot Reload

---

## Step 401: Play Framework คืออะไร?

Play Framework คือ web framework สำหรับ Scala (และ Java) ที่ออกแบบมาเพื่อสร้าง modern web applications ด้วย approach แบบ reactive และ stateless

### ลักษณะเด่นของ Play Framework

```
Play Framework 3.0
├── Reactive by default (non-blocking I/O)
├── Stateless architecture
├── Convention over configuration
├── Hot reload (ไม่ต้อง restart server ระหว่าง development)
├── Type-safe routing
├── Built on Pekko (Apache Pekko - successor of Akka)
└── Built-in testing support
```

### ทำไมต้องใช้ Play?

| Feature | Play Framework | Spring Boot | Node.js Express |
|---------|---------------|-------------|-----------------|
| Language | Scala/Java | Java/Kotlin | JavaScript |
| Async Model | Reactive (Pekko) | Reactive/Blocking | Event Loop |
| Type Safety | Strong (Scala) | Moderate | Weak |
| Hot Reload | Built-in | DevTools | Nodemon |
| Performance | High | Moderate | High |
| Learning Curve | Moderate | Low | Low |

---

## Step 402: ติดตั้ง Play Framework

### Prerequisites

```bash
# ต้องมี Java 17+ และ sbt 1.9+
java -version
# openjdk version "17.0.x"

sbt -version
# sbt version 1.9.x

# ติดตั้ง sbt (ถ้ายังไม่มี)
# macOS
brew install sbt

# Linux (Ubuntu/Debian)
echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" | sudo tee /etc/apt/sources.list.d/sbt.list
sudo apt-get update
sudo apt-get install sbt

# Windows
# ดาวน์โหลดจาก https://www.scala-sbt.org/download.html
```

### สร้าง Play Project ด้วย sbt new

```bash
# สร้าง project ใหม่
sbt new playframework/play-scala-seed.g8

# กรอกข้อมูล:
# name: my-play-app
# organization: com.example
# scala_version: 3.3.x

cd my-play-app
```

### โครงสร้าง Project ที่ได้

```
my-play-app/
├── app/
│   ├── controllers/
│   │   └── HomeController.scala
│   ├── views/
│   │   ├── index.scala.html
│   │   └── main.scala.html
│   └── Module.scala
├── conf/
│   ├── application.conf
│   ├── logback.xml
│   └── routes
├── public/
│   ├── images/
│   ├── javascripts/
│   └── stylesheets/
├── test/
│   └── controllers/
│       └── HomeControllerSpec.scala
├── build.sbt
└── project/
    ├── build.properties
    └── plugins.sbt
```

---

## Step 403: build.sbt สำหรับ Play 3.0

```scala
// build.sbt - Play Framework 3.0 with Scala 3
ThisBuild / scalaVersion := "3.3.3"
ThisBuild / version      := "1.0-SNAPSHOT"

lazy val root = (project in file("."))
  .enablePlugins(PlayScala)
  .settings(
    name := "my-play-app",
    libraryDependencies ++= Seq(
      guice,                          // Dependency Injection
      "org.scalatestplus.play" %% "scalatestplus-play" % "7.0.1" % Test
    ),
    // Disable PlayLayoutPlugin for custom structure
    // Scala 3 settings
    scalacOptions ++= Seq(
      "-Xfatal-warnings",
      "-deprecation",
      "-feature"
    )
  )
```

### project/plugins.sbt

```scala
// project/plugins.sbt
addSbtPlugin("com.typesafe.play" % "sbt-plugin" % "3.0.3")
addSbtPlugin("org.scalameta" % "sbt-scalafmt" % "2.5.2")
```

### project/build.properties

```properties
sbt.version=1.9.8
```

---

## Step 404: รัน Play Application

```bash
# รัน development server
sbt run

# หรือเข้า sbt shell ก่อน แล้วค่อย run
sbt
> run

# Play จะ start ที่ http://localhost:9000
# ครั้งแรกจะ compile นาน รอสักครู่

# รัน port อื่น
sbt "run 9001"

# รัน with hot reload และ live reload
sbt ~run
```

### เปิดดูผลลัพธ์

```bash
# เปิด browser ไปที่
# http://localhost:9000

# หรือ test ด้วย curl
curl http://localhost:9000
```

---

## Step 405: MVC Architecture ใน Play

Play ใช้ MVC (Model-View-Controller) pattern:

```
HTTP Request
     │
     ▼
┌─────────────┐
│   Routes    │ conf/routes - จับคู่ URL กับ Controller Action
└─────────────┘
     │
     ▼
┌─────────────┐
│ Controller  │ app/controllers/ - logic การจัดการ request
└─────────────┘
     │
     ▼
┌─────────────┐
│    Model    │ app/models/ - business logic และ data
└─────────────┘
     │
     ▼
┌─────────────┐
│    View     │ app/views/ - Twirl templates (.scala.html)
└─────────────┘
     │
     ▼
HTTP Response
```

### ตัวอย่าง: Controller

```scala
// app/controllers/HomeController.scala
package controllers

import javax.inject.*
import play.api.*
import play.api.mvc.*

/**
 * Controller ที่ extend BaseController
 * @Inject() ใช้ inject dependencies ผ่าน Guice
 */
@Singleton
class HomeController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  // Action.apply รับ Request และ return Result
  def index(): Action[AnyContent] = Action { implicit request: Request[AnyContent] =>
    Ok(views.html.index())
  }

  // Action ที่ส่ง plain text
  def hello(name: String): Action[AnyContent] = Action {
    Ok(s"Hello, $name!")
  }
}
```

### ตัวอย่าง: Routes

```
# conf/routes
# Method   URL Pattern              Controller#Action

GET     /                           controllers.HomeController.index()
GET     /hello/:name                controllers.HomeController.hello(name: String)

# Static assets
GET     /assets/*file               controllers.Assets.versioned(path="/public", file: Asset)
```

### ตัวอย่าง: View (Twirl Template)

```html
@* app/views/index.scala.html *@
@()

@main("Welcome to Play") {
  <h1>Welcome to Play Framework 3.0!</h1>
  <p>You are now using Scala 3 with Play Framework.</p>
}
```

```html
@* app/views/main.scala.html *@
@(title: String)(content: Html)

<!DOCTYPE html>
<html lang="en">
<head>
  <title>@title</title>
  <link rel="stylesheet" media="screen" href="@routes.Assets.versioned("stylesheets/main.css")">
</head>
<body>
  @content
</body>
</html>
```

---

## Step 406: สร้าง Simple CRUD Application

มาสร้าง Todo application อย่างง่ายเพื่อเรียนรู้ MVC:

### Model

```scala
// app/models/Todo.scala
package models

import java.time.LocalDateTime

// Case class สำหรับ Todo item
case class Todo(
  id: Long,
  title: String,
  description: String,
  completed: Boolean = false,
  createdAt: LocalDateTime = LocalDateTime.now()
)

// In-memory repository (ใช้แทน database สำหรับตัวอย่าง)
object TodoRepository {
  // ใช้ var เพราะเป็น mutable in-memory store
  private var todos: List[Todo] = List(
    Todo(1, "เรียน Play Framework", "เรียน Step 401-410", false),
    Todo(2, "สร้าง REST API", "สร้าง API สำหรับ Todo app", false),
    Todo(3, "Deploy to production", "Deploy บน cloud", false)
  )
  private var nextId: Long = 4

  def findAll(): List[Todo] = todos

  def findById(id: Long): Option[Todo] = todos.find(_.id == id)

  def create(title: String, description: String): Todo = {
    val todo = Todo(nextId, title, description)
    todos = todos :+ todo
    nextId += 1
    todo
  }

  def update(id: Long, title: String, description: String, completed: Boolean): Option[Todo] = {
    todos.find(_.id == id).map { existing =>
      val updated = existing.copy(title = title, description = description, completed = completed)
      todos = todos.map(t => if (t.id == id) updated else t)
      updated
    }
  }

  def delete(id: Long): Boolean = {
    val before = todos.length
    todos = todos.filterNot(_.id == id)
    todos.length < before
  }
}
```

### Controller

```scala
// app/controllers/TodoController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.data.*
import play.api.data.Forms.*
import models.{Todo, TodoRepository}

@Singleton
class TodoController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  // Form definition สำหรับ Todo
  val todoForm: Form[(String, String)] = Form(
    tuple(
      "title"       -> nonEmptyText,
      "description" -> text
    )
  )

  // แสดง list ของ todos ทั้งหมด
  def index(): Action[AnyContent] = Action { implicit request =>
    val todos = TodoRepository.findAll()
    Ok(views.html.todos.index(todos))
  }

  // แสดง form สร้าง todo ใหม่
  def newTodo(): Action[AnyContent] = Action { implicit request =>
    Ok(views.html.todos.create(todoForm))
  }

  // รับ POST request และสร้าง todo ใหม่
  def createTodo(): Action[AnyContent] = Action { implicit request =>
    todoForm.bindFromRequest().fold(
      formWithErrors => BadRequest(views.html.todos.create(formWithErrors)),
      { case (title, description) =>
        val todo = TodoRepository.create(title, description)
        Redirect(routes.TodoController.index())
          .flashing("success" -> s"สร้าง Todo '$title' เรียบร้อยแล้ว!")
      }
    )
  }

  // ลบ todo
  def deleteTodo(id: Long): Action[AnyContent] = Action { implicit request =>
    if (TodoRepository.delete(id)) {
      Redirect(routes.TodoController.index())
        .flashing("success" -> "ลบ Todo เรียบร้อยแล้ว!")
    } else {
      NotFound("ไม่พบ Todo ที่ต้องการลบ")
    }
  }
}
```

---

## Step 407: Routes Configuration

```
# conf/routes

# Home
GET     /                           controllers.HomeController.index()

# Todo routes
GET     /todos                      controllers.TodoController.index()
GET     /todos/new                  controllers.TodoController.newTodo()
POST    /todos                      controllers.TodoController.createTodo()
DELETE  /todos/:id                  controllers.TodoController.deleteTodo(id: Long)

# RESTful pattern
GET     /api/todos                  controllers.api.TodoApiController.list()
POST    /api/todos                  controllers.api.TodoApiController.create()
GET     /api/todos/:id              controllers.api.TodoApiController.show(id: Long)
PUT     /api/todos/:id              controllers.api.TodoApiController.update(id: Long)
DELETE  /api/todos/:id              controllers.api.TodoApiController.delete(id: Long)

# Assets
GET     /assets/*file               controllers.Assets.versioned(path="/public", file: Asset)
```

---

## Step 408: Play Application Configuration

```hocon
# conf/application.conf
# Play Framework ใช้ HOCON (Human-Optimized Config Object Notation) format

# Application secret key (MUST change in production!)
play.http.secret.key = "changeme"
play.http.secret.key = ${?APPLICATION_SECRET}  # override ด้วย env var

# Server configuration
play.server {
  http.port = 9000
  http.port = ${?PORT}  # override ด้วย env var

  # HTTPS configuration (optional)
  # https.port = 9443
  # https.keyStore.path = "server.jks"
}

# Database (จะเรียนใน Part 61+)
# db.default.driver = org.postgresql.Driver
# db.default.url = "jdbc:postgresql://localhost/mydb"

# Logging
play.filters.enabled += "play.filters.headers.SecurityHeadersFilter"
play.filters.enabled += "play.filters.cors.CORSFilter"

# Allowed hosts filter
play.filters.hosts {
  allowed = ["localhost", "127.0.0.1"]
}

# Session configuration
play.http.session {
  cookieName = "PLAY_SESSION"
  secure = false  # true ใน production (HTTPS only)
  maxAge = null   # null = expire เมื่อ browser ปิด
  httpOnly = true
  sameSite = "lax"
}

# i18n
play.i18n.langs = ["th", "en"]
```

---

## Step 409: Hot Reload และ Development Workflow

### Hot Reload คืออะไร?

Hot Reload คือ feature ที่ทำให้ Play compile และ reload code อัตโนมัติเมื่อมีการเปลี่ยนแปลงไฟล์ โดยไม่ต้อง restart server

```bash
# รัน play ด้วย hot reload
sbt run
# หรือ
sbt ~run  # tilde (~) ทำให้ watch file changes และ recompile อัตโนมัติ
```

### Development workflow

```
1. sbt run (หรือ sbt ~run)
2. แก้ไขไฟล์ .scala หรือ .html
3. รีเฟรช browser
4. Play จะ compile อัตโนมัติและ reload
5. ถ้ามี compile error จะแสดงหน้า error ใน browser
```

### Play Error Page

เมื่อมี compile error Play จะแสดง:
- ชื่อไฟล์ที่มี error
- บรรทัดที่มี error พร้อม highlight
- Error message ที่ชัดเจน
- Stack trace (ถ้ามี runtime error)

```
// ตัวอย่าง error ที่ Play แสดง
[error] app/controllers/HomeController.scala:15:5: value toUpperCase is not a member of Int
[error]   15:     val x: Int = "hello".toUpperCase
[error]                              ^
```

---

## Step 410: Logging Configuration

```xml
<!-- conf/logback.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration>
<configuration>

  <!-- Console appender - แสดง log บน terminal -->
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%coloredLevel %logger{15} - %message%n%xException{10}</pattern>
    </encoder>
  </appender>

  <!-- File appender - บันทึก log ลงไฟล์ -->
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>logs/application.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <fileNamePattern>logs/application.%d{yyyy-MM-dd}.log</fileNamePattern>
      <maxHistory>30</maxHistory>
    </rollingPolicy>
    <encoder>
      <pattern>%date [%level] from %logger in %thread - %message%n%xException{10}</pattern>
    </encoder>
  </appender>

  <!-- Application logs -->
  <logger name="controllers" level="DEBUG" />
  <logger name="models" level="DEBUG" />

  <!-- Play framework logs -->
  <logger name="play" level="INFO" />
  <logger name="application" level="DEBUG" />

  <!-- Root logger -->
  <root level="WARN">
    <appender-ref ref="STDOUT" />
    <appender-ref ref="FILE" />
  </root>

</configuration>
```

### การใช้ Logger ใน Controller

```scala
// app/controllers/HomeController.scala
package controllers

import javax.inject.*
import play.api.*
import play.api.mvc.*

@Singleton
class HomeController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  // Logger ของ Play - สร้างได้ง่าย
  private val logger = Logger(this.getClass)

  def index(): Action[AnyContent] = Action { implicit request =>
    logger.info("HomeController.index called")
    logger.debug(s"Request headers: ${request.headers}")

    // ทำงาน...
    val result = "Hello Play!"

    logger.info(s"Returning result: $result")
    Ok(result)
  }

  def riskyOperation(): Action[AnyContent] = Action { implicit request =>
    try {
      // some risky operation
      val data = someOperation()
      Ok(data)
    } catch {
      case ex: Exception =>
        logger.error("Error in riskyOperation", ex)
        InternalServerError("เกิดข้อผิดพลาด กรุณาลองใหม่")
    }
  }

  private def someOperation(): String = "some result"
}
```

---

## สรุป Part 41

| หัวข้อ | รายละเอียด |
|--------|-----------|
| Play Framework | Web framework แบบ reactive สำหรับ Scala/Java |
| Play 3.0 | ใช้ Apache Pekko แทน Akka |
| MVC | Model-View-Controller pattern |
| Routes | conf/routes file สำหรับ URL mapping |
| Controller | class ที่ extend BaseController |
| View | Twirl templates (.scala.html) |
| Hot Reload | Auto-compile เมื่อแก้ไขไฟล์ |
| Configuration | HOCON format ใน application.conf |

### build.sbt dependencies ที่ใช้ใน Part นี้

```scala
libraryDependencies ++= Seq(
  guice,
  "org.scalatestplus.play" %% "scalatestplus-play" % "7.0.1" % Test
)
```

---

## แบบฝึกหัด Part 41

1. **สร้าง Play Project ใหม่**: ใช้ `sbt new playframework/play-scala-seed.g8` สร้าง project ชื่อ "book-store" และรัน development server

2. **เพิ่ม Controller**: สร้าง `BookController` ที่มี action `index()`, `show(id: Long)`, และ `about()` พร้อม routes ที่สอดคล้อง

3. **MVC Practice**: สร้าง `Book` case class ใน models, `BookRepository` ที่เก็บ data ใน memory, และ controller ที่แสดง list ของ books

4. **Logging**: เพิ่ม Logger ใน controller ทุกตัวและ log การเรียก action ทุกครั้ง ทั้ง INFO สำหรับปกติ และ ERROR สำหรับ exception

5. **Configuration**: เพิ่ม custom configuration ใน application.conf เช่น `myapp.maxBooks = 100` และอ่านค่านี้ใน controller โดยใช้ `Configuration` injection

---

## ต่อไป: Part 42 - Play Routing

ใน Part 42 เราจะเรียนรู้เกี่ยวกับ:
- Routes file syntax อย่างละเอียด
- URL patterns และ path parameters
- Query parameters
- HTTP methods (GET, POST, PUT, DELETE, PATCH)
- Reverse routing
- Prefix routes
- Default values

[→ ไปยัง Part 42: Play Routing](part-42-play-routing.md)
