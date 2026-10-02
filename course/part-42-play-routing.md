# Part 42: Play Routing

## Steps 411-420: Routes File, URL Patterns, Path Params, Query Params, HTTP Methods, Reverse Routing

---

## Step 411: Routes File คืออะไร?

Routes file (`conf/routes`) คือ heart ของ Play application - กำหนดการ mapping ระหว่าง HTTP requests กับ Controller actions

```
# syntax พื้นฐาน
# HTTP_METHOD   URL_PATTERN   Controller.action(params)

GET     /               controllers.HomeController.index()
POST    /users          controllers.UserController.create()
GET     /users/:id      controllers.UserController.show(id: Long)
```

### Play Router ทำงานอย่างไร?

```
HTTP Request
  GET /users/42
      │
      ▼
  Routes File
  GET /users/:id  → UserController.show(id: Long)
      │
      ▼
  Type Conversion
  id: Long = 42
      │
      ▼
  Controller.show(42L)
      │
      ▼
  HTTP Response
```

---

## Step 412: HTTP Methods

```
# conf/routes - HTTP Methods ทั้งหมด

# GET - ดึงข้อมูล
GET     /articles               controllers.ArticleController.index()

# POST - สร้างข้อมูลใหม่
POST    /articles               controllers.ArticleController.create()

# PUT - อัพเดตข้อมูลทั้งหมด (replace)
PUT     /articles/:id           controllers.ArticleController.update(id: Long)

# PATCH - อัพเดตข้อมูลบางส่วน
PATCH   /articles/:id           controllers.ArticleController.patch(id: Long)

# DELETE - ลบข้อมูล
DELETE  /articles/:id           controllers.ArticleController.delete(id: Long)

# HEAD - เหมือน GET แต่ไม่มี response body
HEAD    /articles               controllers.ArticleController.index()

# OPTIONS - ดู allowed methods (ใช้สำหรับ CORS preflight)
OPTIONS /articles               controllers.ArticleController.options()
```

---

## Step 413: Path Parameters

### Basic Path Parameters

```
# conf/routes

# String parameter (default type)
GET   /users/:username        controllers.UserController.profile(username: String)

# Long parameter
GET   /articles/:id           controllers.ArticleController.show(id: Long)

# Int parameter  
GET   /pages/:page            controllers.PageController.show(page: Int)

# UUID parameter
GET   /orders/:uuid           controllers.OrderController.show(uuid: java.util.UUID)

# Multiple path parameters
GET   /users/:userId/posts/:postId    controllers.PostController.showUserPost(userId: Long, postId: Long)
```

### Wildcard Path Parameter

```
# conf/routes

# *file จะ match path ที่เหลือทั้งหมด รวมถึง /
GET   /files/*path            controllers.FileController.show(path)

# ตัวอย่าง:
# GET /files/images/photo.jpg → path = "images/photo.jpg"
# GET /files/docs/2024/report.pdf → path = "docs/2024/report.pdf"

# Versioned assets
GET   /assets/*file           controllers.Assets.versioned(path="/public", file: Asset)
```

### Regular Expression in Routes

```
# conf/routes

# $param<regex> syntax
GET   /articles/$id<[0-9]+>   controllers.ArticleController.show(id: Long)
GET   /users/$name<[a-z]+>    controllers.UserController.byName(name: String)

# ตัวอย่างที่ซับซ้อน
GET   /years/$year<20[0-9]{2}>     controllers.YearController.show(year: Int)
```

### Controller รับ Path Parameters

```scala
// app/controllers/ArticleController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import models.ArticleRepository

@Singleton
class ArticleController @Inject()(
  val controllerComponents: ControllerComponents,
  articleRepo: ArticleRepository
) extends BaseController {

  private val logger = play.api.Logger(this.getClass)

  // รับ Long parameter - Play แปลงให้อัตโนมัติ
  def show(id: Long): Action[AnyContent] = Action { implicit request =>
    logger.info(s"Showing article with id: $id")

    articleRepo.findById(id) match {
      case Some(article) => Ok(views.html.articles.show(article))
      case None          => NotFound(views.html.notFound(s"Article $id not found"))
    }
  }

  // รับ multiple path params
  def showUserPost(userId: Long, postId: Long): Action[AnyContent] = Action { implicit request =>
    articleRepo.findByUserAndId(userId, postId) match {
      case Some(post) => Ok(views.html.posts.show(post))
      case None       => NotFound("Post not found")
    }
  }
}
```

---

## Step 414: Query Parameters

### Query Parameters ใน Routes

```
# conf/routes

# Optional query parameter
GET   /articles         controllers.ArticleController.list(page: Int ?= 1, limit: Int ?= 10)

# Required query parameter (ถ้าไม่ส่งมาจะ error)
GET   /search           controllers.SearchController.search(q: String)

# Optional String parameter
GET   /users            controllers.UserController.list(role: Option[String] ?= None)

# Multiple query params
GET   /products         controllers.ProductController.list(
                          category: Option[String] ?= None,
                          minPrice: Option[Double] ?= None,
                          maxPrice: Option[Double] ?= None,
                          page: Int ?= 1)
```

### Controller รับ Query Parameters

```scala
// app/controllers/ArticleController.scala
package controllers

import javax.inject.*
import play.api.mvc.*

@Singleton
class ArticleController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  // รับ query params ที่มี default values
  def list(page: Int = 1, limit: Int = 10): Action[AnyContent] = Action { implicit request =>
    // validation
    val safePage = Math.max(1, page)
    val safeLimit = Math.min(100, Math.max(1, limit))

    val articles = ArticleRepository.findPaginated(safePage, safeLimit)
    val total = ArticleRepository.count()

    Ok(views.html.articles.list(articles, safePage, safeLimit, total))
  }

  // รับ optional query params
  def search(
    q: String,
    category: Option[String] = None,
    page: Int = 1
  ): Action[AnyContent] = Action { implicit request =>
    val results = ArticleRepository.search(q, category, page)
    Ok(views.html.articles.search(q, results, category, page))
  }

  // อ่าน query params จาก request โดยตรง (flexible approach)
  def filter(): Action[AnyContent] = Action { implicit request =>
    // อ่านค่าจาก query string
    val tags = request.queryString.getOrElse("tag", Seq.empty)
    val fromDate = request.getQueryString("from")
    val toDate = request.getQueryString("to")
    val sortBy = request.getQueryString("sort").getOrElse("date")

    val filtered = ArticleRepository.filter(tags, fromDate, toDate, sortBy)
    Ok(views.html.articles.list(filtered, 1, 20, filtered.length))
  }
}
```

---

## Step 415: Reverse Routing

Reverse Routing คือการสร้าง URL จาก Controller action (ไปทิศทางตรงข้าม)

### ทำไมต้องใช้ Reverse Routing?

```scala
// ❌ แบบ hardcode URL - ไม่ดี
<a href="/articles/42">อ่านบทความ</a>

// ✅ ใช้ Reverse Routing - ดีกว่า
<a href="@routes.ArticleController.show(42)">อ่านบทความ</a>

// ข้อดีของ Reverse Routing:
// 1. Type-safe - compile error ถ้า route ไม่มีอยู่
// 2. ถ้าเปลี่ยน URL pattern ไม่ต้อง update ทุกที่
// 3. Play generate URL ที่ถูกต้องให้อัตโนมัติ
```

### ใช้ Reverse Routing ใน Controller

```scala
// app/controllers/ArticleController.scala
package controllers

import javax.inject.*
import play.api.mvc.*

@Singleton
class ArticleController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  def create(): Action[AnyContent] = Action { implicit request =>
    // หลังจาก create สำเร็จ ให้ redirect ไปหน้า list
    // ใช้ routes.ArticleController.list() แทนการ hardcode URL
    Redirect(routes.ArticleController.list())
      .flashing("success" -> "สร้างบทความสำเร็จ!")
  }

  def createWithId(): Action[AnyContent] = Action { implicit request =>
    val newId = 42L  // สมมติว่า create แล้วได้ id
    // Redirect ไปหน้าแสดงบทความที่เพิ่งสร้าง
    Redirect(routes.ArticleController.show(newId))
      .flashing("success" -> "สร้างบทความสำเร็จ!")
  }

  def list(page: Int = 1, limit: Int = 10): Action[AnyContent] = Action { implicit request =>
    Ok("article list")
  }

  def show(id: Long): Action[AnyContent] = Action { implicit request =>
    Ok(s"article $id")
  }
}
```

### ใช้ Reverse Routing ใน View

```html
@* app/views/articles/list.scala.html *@
@(articles: List[models.Article])

<ul>
  @for(article <- articles) {
    <li>
      @* สร้าง URL สำหรับแต่ละ article *@
      <a href="@routes.ArticleController.show(article.id)">
        @article.title
      </a>
    </li>
  }
</ul>

@* Link สำหรับสร้างบทความใหม่ *@
<a href="@routes.ArticleController.newArticle()" class="btn btn-primary">
  เขียนบทความใหม่
</a>

@* Pagination links *@
<a href="@routes.ArticleController.list(page = 2, limit = 10)">
  หน้าถัดไป
</a>
```

### Reverse Routing กับ Query Parameters

```scala
// สร้าง URL พร้อม query parameters
val url = routes.ArticleController.list(page = 2, limit = 20).url
// ผลลัพธ์: "/articles?page=2&limit=20"

// สร้าง URL สำหรับ search
val searchUrl = routes.SearchController.search(q = "scala play").url
// ผลลัพธ์: "/search?q=scala+play"

// สร้าง absolute URL (พร้อม protocol และ host)
val absoluteUrl = routes.ArticleController.show(42).absoluteURL()
// ผลลัพธ์: "http://localhost:9000/articles/42"
```

---

## Step 416: Route Prefix และ Sub-routes

```
# conf/routes

# API v1 prefix
->  /api/v1          api.v1.Routes

# Admin prefix
->  /admin           admin.Routes
```

```
# conf/api/v1/routes (app/api/v1/routes หรือ separate routes file)
GET   /articles       controllers.api.v1.ArticleController.list()
POST  /articles       controllers.api.v1.ArticleController.create()
GET   /articles/:id   controllers.api.v1.ArticleController.show(id: Long)
```

### สร้าง Sub-routes File

```scala
// project structure สำหรับ API versioning
conf/
├── routes               # main routes file
├── api.routes           # API routes
└── admin.routes         # Admin routes
```

```
# conf/routes (main)
# Delegate ไปยัง sub-routes files
->  /api                api.Routes
->  /admin              admin.Routes

GET   /                 controllers.HomeController.index()
GET   /assets/*file     controllers.Assets.versioned(path="/public", file: Asset)
```

```
# conf/api.routes
GET   /articles         controllers.api.ArticleApiController.list()
GET   /articles/:id     controllers.api.ArticleApiController.show(id: Long)
POST  /articles         controllers.api.ArticleApiController.create()
```

---

## Step 417: Advanced Routing Patterns

### Fixed Values และ Default Parameters

```
# conf/routes

# Route ที่ส่ง fixed value ให้ controller
GET   /                   controllers.HomeController.index()
GET   /about              controllers.HomeController.about()

# Route ที่มี default parameter
GET   /articles           controllers.ArticleController.list(page: Int ?= 1, limit: Int ?= 10)

# หลาย routes ไปหา action เดียวกัน
GET   /en/home            controllers.HomeController.index()
GET   /th/home            controllers.HomeController.index()
```

### Boolean Parameters

```
# conf/routes
GET   /articles           controllers.ArticleController.list(published: Boolean ?= true)
```

```scala
// Controller รับ Boolean
def list(published: Boolean = true): Action[AnyContent] = Action { implicit request =>
  val articles = if (published) ArticleRepository.findPublished()
                 else ArticleRepository.findAll()
  Ok(views.html.articles.list(articles))
}
```

### Custom Parameter Binders (PathBindable)

```scala
// app/models/ArticleStatus.scala
package models

import play.api.mvc.PathBindable

enum ArticleStatus:
  case Draft, Published, Archived

object ArticleStatus:
  // Custom PathBindable สำหรับ enum
  given PathBindable[ArticleStatus] with
    def bind(key: String, value: String): Either[String, ArticleStatus] =
      value.toLowerCase match
        case "draft"     => Right(ArticleStatus.Draft)
        case "published" => Right(ArticleStatus.Published)
        case "archived"  => Right(ArticleStatus.Archived)
        case _           => Left(s"Unknown status: $value")

    def unbind(key: String, value: ArticleStatus): String =
      value.toString.toLowerCase
```

```
# conf/routes
GET   /articles/:status   controllers.ArticleController.byStatus(status: models.ArticleStatus)
```

---

## Step 418: Routes ที่ดีสำหรับ RESTful API

### RESTful Convention

```
# conf/routes - RESTful API design

# Resources: articles
GET     /api/articles              controllers.api.ArticleController.index()      # List
POST    /api/articles              controllers.api.ArticleController.create()     # Create
GET     /api/articles/:id          controllers.api.ArticleController.show(id: Long)    # Show
PUT     /api/articles/:id          controllers.api.ArticleController.update(id: Long)  # Update (replace)
PATCH   /api/articles/:id          controllers.api.ArticleController.patch(id: Long)   # Update (partial)
DELETE  /api/articles/:id          controllers.api.ArticleController.destroy(id: Long) # Delete

# Nested resources: comments under articles
GET     /api/articles/:articleId/comments          controllers.api.CommentController.index(articleId: Long)
POST    /api/articles/:articleId/comments          controllers.api.CommentController.create(articleId: Long)
GET     /api/articles/:articleId/comments/:id      controllers.api.CommentController.show(articleId: Long, id: Long)
DELETE  /api/articles/:articleId/comments/:id      controllers.api.CommentController.destroy(articleId: Long, id: Long)

# Special actions
POST    /api/articles/:id/publish  controllers.api.ArticleController.publish(id: Long)
POST    /api/articles/:id/archive  controllers.api.ArticleController.archive(id: Long)
```

---

## Step 419: Routing Errors และ Error Handling

### Custom Error Handling

```scala
// app/ErrorHandler.scala
package app

import javax.inject.*
import play.api.*
import play.api.http.DefaultHttpErrorHandler
import play.api.mvc.*
import play.api.mvc.Results.*
import play.api.routing.Router
import scala.concurrent.*

@Singleton
class ErrorHandler @Inject()(
  env: Environment,
  config: Configuration,
  sourceMapper: OptionalSourceMapper,
  router: Provider[Router]
) extends DefaultHttpErrorHandler(env, config, sourceMapper, router) {

  // Handle 404 Not Found
  override def onNotFound(request: RequestHeader, message: String): Future[Result] = {
    Future.successful(
      NotFound(views.html.errors.notFound(request, message))
    )
  }

  // Handle 500 Internal Server Error
  override def onServerError(request: RequestHeader, exception: Throwable): Future[Result] = {
    Future.successful(
      InternalServerError(views.html.errors.serverError(request, exception))
    )
  }

  // Handle Bad Request (400)
  override def onBadRequest(request: RequestHeader, message: String): Future[Result] = {
    Future.successful(
      BadRequest(views.html.errors.badRequest(request, message))
    )
  }
}
```

```hocon
# conf/application.conf - register custom error handler
play.http.errorHandler = "app.ErrorHandler"
```

### Route Parameter Type Mismatch

```
# ถ้า URL ไม่ตรง type ที่กำหนด Play จะ return 400 Bad Request

GET   /articles/:id   controllers.ArticleController.show(id: Long)

# GET /articles/abc → 400 Bad Request (abc ไม่ใช่ Long)
# GET /articles/42  → 200 OK ปกติ
```

---

## Step 420: Routes Testing และ Debugging

### ดู Routes ที่ generate ขึ้น

```bash
# ใน sbt shell
sbt routes

# หรือ
sbt "run" แล้วเปิด http://localhost:9000/@documentation ใน development mode
```

### Test Routes ใน Unit Test

```scala
// test/controllers/RoutesSpec.scala
package controllers

import org.scalatestplus.play.*
import play.api.test.*
import play.api.test.Helpers.*

class RoutesSpec extends PlaySpec with GuiceOneAppPerTest {

  "Routes" should {
    "route GET / to HomeController.index" in {
      val request = FakeRequest(GET, "/")
      val home = route(app, request).get

      status(home) mustBe OK
      contentType(home) mustBe Some("text/html")
    }

    "route GET /articles/1 to ArticleController.show" in {
      val request = FakeRequest(GET, "/articles/1")
      val result = route(app, request).get

      status(result) mustBe OK
    }

    "return 400 for invalid article id" in {
      val request = FakeRequest(GET, "/articles/not-a-number")
      val result = route(app, request).get

      status(result) mustBe BAD_REQUEST
    }

    "return 404 for unknown route" in {
      val request = FakeRequest(GET, "/unknown-path")
      val result = route(app, request).get

      status(result) mustBe NOT_FOUND
    }
  }
}
```

### Production Considerations

```
# best practices สำหรับ routes ใน production

# 1. ใช้ versioning สำหรับ API
GET   /api/v1/articles    controllers.api.v1.ArticleController.list()
GET   /api/v2/articles    controllers.api.v2.ArticleController.list()

# 2. ระวัง route ordering - Play match route แรกที่ตรง
# ดังนั้น specific routes ต้องมาก่อน general routes
GET   /users/admin        controllers.AdminController.index()    # ต้องมาก่อน
GET   /users/:id          controllers.UserController.show(id: Long)  # general

# 3. ใช้ HTTPS redirect ใน production
# (จัดการใน nginx/load balancer แทน Play)
```

---

## สรุป Part 42

| Concept | Syntax | ตัวอย่าง |
|---------|--------|---------|
| Basic route | `METHOD URL Controller.action` | `GET / HomeController.index()` |
| Path param | `:paramName` | `/articles/:id` |
| Wildcard | `*paramName` | `/files/*path` |
| Regex param | `$name<regex>` | `$id<[0-9]+>` |
| Query param | `?= defaultValue` | `page: Int ?= 1` |
| Optional param | `Option[T] ?= None` | `role: Option[String] ?= None` |
| Sub-routes | `-> /prefix file.Routes` | `-> /api api.Routes` |
| Reverse routing | `routes.Controller.action(params)` | `routes.ArticleController.show(42)` |

---

## แบบฝึกหัด Part 42

1. **Basic Routes**: สร้าง routes สำหรับ `BlogController` ที่มี list, show (ด้วย slug), create, update, delete และ search (ด้วย query params)

2. **Nested Routes**: สร้าง routes สำหรับ `users/:userId/posts` ที่รองรับ CRUD operations ทั้งหมด

3. **Path Binder**: สร้าง custom `QueryStringBindable` สำหรับ `DateRange` ที่ parse `from=2024-01-01&to=2024-12-31` จาก query string

4. **Reverse Routing**: สร้าง navigation menu ใน Twirl template ที่ใช้ reverse routing ทั้งหมด ไม่มี hardcoded URL แม้แต่ตัวเดียว

5. **Error Pages**: สร้าง custom ErrorHandler ที่ return JSON สำหรับ routes ที่ขึ้นด้วย `/api/` และ HTML สำหรับ routes อื่นๆ

---

[→ ไปยัง Part 43: Play Actions and Results](part-43-play-actions-and-results.md)
