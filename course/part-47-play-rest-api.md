# Part 47: Play REST API

## Steps 461-470: RESTful API Design, JSON API Controller, Pagination, HATEOAS, API Versioning

---

## Step 461: RESTful API Design Principles

```
REST (Representational State Transfer) Principles:
├── Stateless: ทุก request ต้องมีข้อมูลครบในตัว
├── Uniform Interface: ใช้ HTTP methods อย่างถูกต้อง
├── Resource-based: URL แทน resource ไม่ใช่ action
├── HATEOAS: Response ต้องมี links ไปยัง related resources
└── Layered: Client ไม่รู้ว่า backend ทำงานยังไง
```

### HTTP Methods และความหมาย

| Method | Action | Idempotent | Body |
|--------|--------|-----------|------|
| GET | อ่านข้อมูล | ✓ | ไม่มี |
| POST | สร้างใหม่ | ✗ | มี |
| PUT | อัพเดตทั้งหมด | ✓ | มี |
| PATCH | อัพเดตบางส่วน | ✓ | มี |
| DELETE | ลบ | ✓ | ไม่มี |

### URL Design

```
# ❌ แบบไม่ดี (action-based)
GET  /getArticles
POST /createArticle
GET  /deleteArticle?id=1

# ✅ แบบดี (resource-based)
GET    /api/articles        # list
POST   /api/articles        # create
GET    /api/articles/1      # show
PUT    /api/articles/1      # update
DELETE /api/articles/1      # delete
```

---

## Step 462: API Controller Base Class

```scala
// app/controllers/api/ApiController.scala
package controllers.api

import play.api.mvc.*
import play.api.libs.json.*
import scala.concurrent.{ExecutionContext, Future}

abstract class ApiController(
  val controllerComponents: ControllerComponents
)(implicit ec: ExecutionContext) extends BaseController {

  // Standard API response format
  protected def apiOk[T: Writes](data: T): Result =
    Ok(Json.obj("data" -> Json.toJson(data), "success" -> true))

  protected def apiCreated[T: Writes](data: T): Result =
    Created(Json.obj("data" -> Json.toJson(data), "success" -> true))

  protected def apiNoContent: Result = NoContent

  protected def apiError(
    message: String,
    statusCode: Int = 400,
    code: String = "ERROR",
    details: Option[JsValue] = None
  ): Result = {
    val body = Json.obj(
      "success" -> false,
      "error"   -> Json.obj(
        "code"    -> code,
        "message" -> message
      ).deepMerge(
        details.fold(Json.obj())(d => Json.obj("details" -> d))
      )
    )
    Status(statusCode)(body)
  }

  protected def apiNotFound(resource: String = "Resource"): Result =
    apiError(s"$resource not found", 404, "NOT_FOUND")

  protected def apiUnauthorized(message: String = "Unauthorized"): Result =
    apiError(message, 401, "UNAUTHORIZED")

  protected def apiForbidden(message: String = "Forbidden"): Result =
    apiError(message, 403, "FORBIDDEN")

  // Parse and validate JSON body
  protected def withJsonBody[T: Reads](
    block: T => Future[Result]
  )(implicit request: Request[JsValue]): Future[Result] = {
    request.body.validate[T].fold(
      errors => Future.successful(
        apiError(
          "Validation failed",
          422,
          "VALIDATION_ERROR",
          Some(JsError.toJson(errors))
        )
      ),
      block
    )
  }

  // Paginated response
  protected def apiPaginated[T: Writes](
    items: List[T],
    total: Long,
    page: Int,
    limit: Int
  ): Result = Ok(Json.obj(
    "success" -> true,
    "data"    -> Json.toJson(items),
    "meta"    -> Json.obj(
      "total"      -> total,
      "page"       -> page,
      "limit"      -> limit,
      "totalPages" -> Math.ceil(total.toDouble / limit).toInt,
      "hasNext"    -> ((page.toLong * limit) < total),
      "hasPrev"    -> (page > 1)
    )
  ))
}
```

---

## Step 463: Full CRUD API Controller

```scala
// app/controllers/api/ArticleApiController.scala
package controllers.api

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*
import scala.concurrent.*
import models.*

@Singleton
class ArticleApiController @Inject()(
  controllerComponents: ControllerComponents,
  articleService: ArticleService,
  implicit val ec: ExecutionContext
) extends ApiController(controllerComponents) {

  // Request/Response models
  case class CreateArticleRequest(
    title: String,
    content: String,
    tags: List[String],
    published: Boolean = false
  )

  case class UpdateArticleRequest(
    title: Option[String],
    content: Option[String],
    tags: Option[List[String]],
    published: Option[Boolean]
  )

  implicit val createReads: Reads[CreateArticleRequest] = Json.reads[CreateArticleRequest]
  implicit val updateReads: Reads[UpdateArticleRequest] = Json.reads[UpdateArticleRequest]
  implicit val articleWrites: Writes[Article] = Json.writes[Article]

  // GET /api/articles?page=1&limit=20&q=search&tag=scala
  def index(
    page: Int = 1,
    limit: Int = 20,
    q: Option[String] = None,
    tag: Option[String] = None,
    published: Option[Boolean] = None
  ): Action[AnyContent] = Action.async { implicit request =>
    val safeLimit = Math.min(100, Math.max(1, limit))
    val safePage = Math.max(1, page)

    articleService.list(
      page = safePage,
      limit = safeLimit,
      query = q,
      tag = tag,
      published = published
    ).map { case (articles, total) =>
      apiPaginated(articles, total, safePage, safeLimit)
    }.recover {
      case ex => apiError(s"Failed to fetch articles: ${ex.getMessage}", 500, "SERVER_ERROR")
    }
  }

  // GET /api/articles/:id
  def show(id: Long): Action[AnyContent] = Action.async { implicit request =>
    articleService.findById(id).map {
      case Some(article) => apiOk(article)
      case None          => apiNotFound("Article")
    }
  }

  // POST /api/articles
  def create(): Action[JsValue] = Action.async(parse.json) { implicit request =>
    withJsonBody[CreateArticleRequest] { req =>
      articleService.create(
        title     = req.title,
        content   = req.content,
        tags      = req.tags,
        published = req.published
      ).map { article =>
        apiCreated(article)
          .withHeaders("Location" -> routes.ArticleApiController.show(article.id).url)
      }
    }
  }

  // PUT /api/articles/:id (full update)
  def update(id: Long): Action[JsValue] = Action.async(parse.json) { implicit request =>
    implicit val fullUpdateReads: Reads[CreateArticleRequest] = Json.reads[CreateArticleRequest]
    withJsonBody[CreateArticleRequest] { req =>
      articleService.findById(id).flatMap {
        case None => Future.successful(apiNotFound("Article"))
        case Some(_) =>
          articleService.update(id, req.title, req.content, req.tags, req.published)
            .map(article => apiOk(article))
      }
    }
  }

  // PATCH /api/articles/:id (partial update)
  def patch(id: Long): Action[JsValue] = Action.async(parse.json) { implicit request =>
    withJsonBody[UpdateArticleRequest] { req =>
      articleService.findById(id).flatMap {
        case None => Future.successful(apiNotFound("Article"))
        case Some(existing) =>
          articleService.patch(id,
            title     = req.title.getOrElse(existing.title),
            content   = req.content.getOrElse(existing.content),
            tags      = req.tags.getOrElse(existing.tags),
            published = req.published.getOrElse(existing.published)
          ).map(article => apiOk(article))
      }
    }
  }

  // DELETE /api/articles/:id
  def delete(id: Long): Action[AnyContent] = Action.async { implicit request =>
    articleService.delete(id).map {
      case true  => apiNoContent
      case false => apiNotFound("Article")
    }
  }

  // POST /api/articles/:id/publish (custom action)
  def publish(id: Long): Action[AnyContent] = Action.async { implicit request =>
    articleService.publish(id).map {
      case Some(article) =>
        apiOk(article).withHeaders("X-Published-At" -> java.time.Instant.now().toString)
      case None =>
        apiNotFound("Article")
    }
  }
}
```

---

## Step 464: Service Layer

```scala
// app/services/ArticleService.scala
package services

import javax.inject.*
import scala.concurrent.*
import models.*

@Singleton
class ArticleService @Inject()(
  articleRepo: repositories.ArticleRepository,
  implicit val ec: ExecutionContext
) {

  def list(
    page: Int,
    limit: Int,
    query: Option[String] = None,
    tag: Option[String] = None,
    published: Option[Boolean] = None
  ): Future[(List[Article], Long)] = {
    articleRepo.findAll(page, limit, query, tag, published)
  }

  def findById(id: Long): Future[Option[Article]] =
    articleRepo.findById(id)

  def create(
    title: String,
    content: String,
    tags: List[String],
    published: Boolean
  ): Future[Article] = {
    // business logic ก่อน save
    val slug = generateSlug(title)
    articleRepo.create(title, content, tags, published, slug)
  }

  def update(id: Long, title: String, content: String, tags: List[String], published: Boolean): Future[Article] = {
    val slug = generateSlug(title)
    articleRepo.update(id, title, content, tags, published, slug)
  }

  def patch(id: Long, title: String, content: String, tags: List[String], published: Boolean): Future[Article] =
    update(id, title, content, tags, published)

  def delete(id: Long): Future[Boolean] =
    articleRepo.delete(id)

  def publish(id: Long): Future[Option[Article]] =
    articleRepo.findById(id).flatMap {
      case None          => Future.successful(None)
      case Some(article) =>
        if (article.published) Future.successful(Some(article))
        else articleRepo.setPublished(id, published = true).map(Some(_))
    }

  private def generateSlug(title: String): String =
    title.toLowerCase
         .replaceAll("[^a-z0-9ก-๙\\s-]", "")
         .replaceAll("\\s+", "-")
         .take(100)
}
```

---

## Step 465: Pagination

```scala
// app/models/Pagination.scala
package models

import play.api.libs.json.*

case class PaginationMeta(
  total: Long,
  page: Int,
  limit: Int,
  totalPages: Int,
  hasNext: Boolean,
  hasPrev: Boolean,
  nextPage: Option[Int],
  prevPage: Option[Int]
)

object PaginationMeta {
  def apply(total: Long, page: Int, limit: Int): PaginationMeta = {
    val totalPages = Math.ceil(total.toDouble / limit).toInt
    PaginationMeta(
      total      = total,
      page       = page,
      limit      = limit,
      totalPages = totalPages,
      hasNext    = page < totalPages,
      hasPrev    = page > 1,
      nextPage   = if (page < totalPages) Some(page + 1) else None,
      prevPage   = if (page > 1) Some(page - 1) else None
    )
  }

  implicit val writes: Writes[PaginationMeta] = Json.writes[PaginationMeta]
}

case class PaginatedResult[T](
  items: List[T],
  meta: PaginationMeta
)

object PaginatedResult {
  def writes[T: Writes]: Writes[PaginatedResult[T]] = Writes { result =>
    Json.obj(
      "data" -> Json.toJson(result.items),
      "meta" -> Json.toJson(result.meta)
    )
  }
}
```

### Cursor-based Pagination

```scala
// app/controllers/api/CursorPaginationController.scala
package controllers.api

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*
import java.util.Base64

@Singleton
class CursorPaginationController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  // Cursor-based pagination (ดีกว่า offset สำหรับ real-time data)
  def list(
    cursor: Option[String] = None,
    limit: Int = 20
  ): Action[AnyContent] = Action { implicit request =>
    val decodedCursor = cursor.map(c => new String(Base64.getDecoder.decode(c)).toLong)
    val safeLimit = Math.min(100, limit)

    // Query items after cursor
    val items = fetchItemsAfterCursor(decodedCursor, safeLimit + 1)
    val hasMore = items.length > safeLimit
    val resultItems = items.take(safeLimit)

    val nextCursor = if (hasMore) {
      val lastId = resultItems.last._1
      Some(Base64.getEncoder.encodeToString(lastId.toString.getBytes))
    } else None

    Ok(Json.obj(
      "data"       -> resultItems.map { case (id, name) => Json.obj("id" -> id, "name" -> name) },
      "meta"       -> Json.obj(
        "limit"     -> safeLimit,
        "hasMore"   -> hasMore,
        "nextCursor"-> nextCursor
      )
    ))
  }

  // Dummy data fetcher
  private def fetchItemsAfterCursor(afterId: Option[Long], limit: Int): List[(Long, String)] = {
    val startId = afterId.getOrElse(0L) + 1
    (startId until (startId + limit)).map(i => (i, s"Item $i")).toList
  }
}
```

---

## Step 466: HATEOAS

HATEOAS = Hypermedia As The Engine Of Application State

```scala
// app/models/HateoasLinks.scala
package models

import play.api.libs.json.*
import play.api.mvc.Call

case class Link(
  href: String,
  rel: String,
  method: String = "GET"
)

object Link {
  implicit val writes: Writes[Link] = Json.writes[Link]

  def self(call: Call): Link = Link(call.url, "self", call.method)
  def next(call: Call): Link = Link(call.url, "next")
  def prev(call: Call): Link = Link(call.url, "prev")
  def collection(call: Call): Link = Link(call.url, "collection")
}

case class HateoasResource[T](
  data: T,
  links: List[Link]
)

object HateoasResource {
  def writes[T: Writes]: Writes[HateoasResource[T]] = Writes { resource =>
    Json.obj(
      "data"  -> Json.toJson(resource.data),
      "_links" -> Json.toJson(resource.links)
    )
  }
}
```

```scala
// app/controllers/api/HateoasArticleController.scala
package controllers.api

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*
import models.*

@Singleton
class HateoasArticleController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  implicit val articleWrites: Writes[Article] = Json.writes[Article]

  def show(id: Long): Action[AnyContent] = Action { implicit request =>
    // สมมติ fetch article
    val article = Article(id, "Test Article", "Content", 1L, List("scala"), true, None, 0L)

    val links = List(
      Link(routes.HateoasArticleController.show(id).url, "self"),
      Link(routes.HateoasArticleController.index().url, "collection"),
      Link(routes.HateoasArticleController.update(id).url, "update", "PUT"),
      Link(routes.HateoasArticleController.delete(id).url, "delete", "DELETE")
    )

    implicit val hw: Writes[HateoasResource[Article]] = HateoasResource.writes[Article]
    Ok(Json.toJson(HateoasResource(article, links)))
  }

  def index(): Action[AnyContent] = Action { _ => Ok("list") }
  def update(id: Long): Action[AnyContent] = Action { _ => Ok("update") }
  def delete(id: Long): Action[AnyContent] = Action { _ => Ok("delete") }
}
```

---

## Step 467: API Versioning

```
# conf/routes - API versioning strategies

# Strategy 1: URL versioning (most common)
->  /api/v1    api.v1.Routes
->  /api/v2    api.v2.Routes

# Strategy 2: Header versioning
# GET /api/articles  Accept: application/vnd.myapp.v2+json

# Strategy 3: Query param versioning
# GET /api/articles?version=2
```

```scala
// app/controllers/api/v2/ArticleController.scala
package controllers.api.v2

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*

// V2 ของ Article มี fields เพิ่มเติม
case class ArticleV2(
  id: Long,
  title: String,
  content: String,
  excerpt: String,        // ใหม่ใน v2
  readTimeMinutes: Int,   // ใหม่ใน v2
  author: AuthorV2        // nested object (ต่างจาก v1 ที่เป็น authorId)
)

case class AuthorV2(id: Long, name: String, avatar: String)

@Singleton
class ArticleController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  implicit val authorWrites: Writes[AuthorV2] = Json.writes[AuthorV2]
  implicit val articleWrites: Writes[ArticleV2] = Json.writes[ArticleV2]

  def index(): Action[AnyContent] = Action { _ =>
    // v2 response format
    val articles = List(
      ArticleV2(1L, "Article 1", "Content...", "Summary...", 5,
                AuthorV2(1L, "John", "https://example.com/avatar.jpg"))
    )
    Ok(Json.toJson(articles))
  }
}
```

### Header-based Versioning

```scala
// app/controllers/api/VersionedController.scala
package controllers.api

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*

@Singleton
class VersionedController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  def articles(): Action[AnyContent] = Action { implicit request =>
    // อ่าน API version จาก Accept header
    val version = request.headers.get("API-Version")
                    .orElse(request.getQueryString("version"))
                    .getOrElse("1")

    version match {
      case "1" => Ok(Json.obj("version" -> 1, "format" -> "basic"))
      case "2" => Ok(Json.obj("version" -> 2, "format" -> "enhanced"))
      case _   => BadRequest(Json.obj("error" -> s"Unknown API version: $version"))
    }
  }
}
```

---

## Step 468: API Rate Limiting

```scala
// app/filters/RateLimitFilter.scala
package filters

import javax.inject.*
import org.apache.pekko.stream.Materializer
import play.api.mvc.*
import play.api.libs.json.*
import scala.concurrent.*
import java.util.concurrent.atomic.AtomicInteger
import java.util.concurrent.ConcurrentHashMap

@Singleton
class RateLimitFilter @Inject()(implicit
  val mat: Materializer,
  ec: ExecutionContext
) extends Filter {

  // Simple in-memory rate limiter (ใช้ Redis ใน production)
  private val requestCounts = new ConcurrentHashMap[String, AtomicInteger]()
  private val windowStart = new ConcurrentHashMap[String, Long]()

  private val maxRequests = 100  // requests per window
  private val windowMs = 60000L  // 1 minute window

  override def apply(
    nextFilter: RequestHeader => Future[Result]
  )(rh: RequestHeader): Future[Result] = {
    // ใช้ IP address เป็น rate limit key
    val key = rh.remoteAddress

    val now = System.currentTimeMillis()
    val start = windowStart.computeIfAbsent(key, _ => now)

    // Reset window ถ้าหมดเวลา
    if (now - start > windowMs) {
      windowStart.put(key, now)
      requestCounts.put(key, new AtomicInteger(0))
    }

    val count = requestCounts.computeIfAbsent(key, _ => new AtomicInteger(0))
    val requestCount = count.incrementAndGet()

    if (requestCount > maxRequests) {
      val retryAfter = ((start + windowMs - now) / 1000).toInt
      Future.successful(
        TooManyRequests(Json.obj(
          "error"       -> "Rate limit exceeded",
          "retryAfter"  -> retryAfter
        )).withHeaders(
          "X-RateLimit-Limit"     -> maxRequests.toString,
          "X-RateLimit-Remaining" -> "0",
          "Retry-After"           -> retryAfter.toString
        )
      )
    } else {
      nextFilter(rh).map { result =>
        result.withHeaders(
          "X-RateLimit-Limit"     -> maxRequests.toString,
          "X-RateLimit-Remaining" -> (maxRequests - requestCount).toString
        )
      }
    }
  }
}
```

---

## Step 469: API Documentation (OpenAPI)

```yaml
# public/api-docs/openapi.yaml
openapi: 3.0.3
info:
  title: My Play API
  version: 1.0.0
  description: REST API สำหรับ My Play Application

servers:
  - url: http://localhost:9000/api/v1
    description: Development

paths:
  /articles:
    get:
      summary: List articles
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ArticleList'

    post:
      summary: Create article
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateArticleRequest'
      responses:
        '201':
          description: Created

components:
  schemas:
    Article:
      type: object
      properties:
        id:
          type: integer
        title:
          type: string
        content:
          type: string
        tags:
          type: array
          items:
            type: string
```

---

## Step 470: Testing REST API

```scala
// test/controllers/api/ArticleApiControllerSpec.scala
package controllers.api

import org.scalatestplus.play.*
import org.scalatestplus.play.guice.GuiceOneAppPerTest
import play.api.test.*
import play.api.test.Helpers.*
import play.api.libs.json.*

class ArticleApiControllerSpec extends PlaySpec with GuiceOneAppPerTest {

  "ArticleApiController" should {

    "return 200 with articles list for GET /api/articles" in {
      val request = FakeRequest(GET, "/api/articles")
      val result = route(app, request).get

      status(result) mustBe OK
      contentType(result) mustBe Some("application/json")

      val json = contentAsJson(result)
      (json \ "success").as[Boolean] mustBe true
      (json \ "data").as[JsArray].value must not be empty
    }

    "return 201 for POST /api/articles with valid data" in {
      val body = Json.obj(
        "title"   -> "Test Article",
        "content" -> "Article content here",
        "tags"    -> Json.arr("scala", "test")
      )

      val request = FakeRequest(POST, "/api/articles")
        .withJsonBody(body)
        .withHeaders("Content-Type" -> "application/json")

      val result = route(app, request).get

      status(result) mustBe CREATED
      val json = contentAsJson(result)
      (json \ "data" \ "title").as[String] mustBe "Test Article"
    }

    "return 422 for POST /api/articles with missing title" in {
      val body = Json.obj("content" -> "Content without title")

      val request = FakeRequest(POST, "/api/articles")
        .withJsonBody(body)

      val result = route(app, request).get

      status(result) mustBe UNPROCESSABLE_ENTITY
      val json = contentAsJson(result)
      (json \ "error" \ "code").as[String] mustBe "VALIDATION_ERROR"
    }

    "return 404 for GET /api/articles/99999" in {
      val request = FakeRequest(GET, "/api/articles/99999")
      val result = route(app, request).get

      status(result) mustBe NOT_FOUND
    }
  }
}
```

---

## สรุป Part 47

| Pattern | Description | Best Practice |
|---------|-------------|--------------|
| Resource URLs | `/api/articles/:id` | noun ไม่ใช่ verb |
| HTTP Methods | GET/POST/PUT/PATCH/DELETE | ใช้ให้ถูกต้องตาม semantics |
| Status Codes | 200/201/204/400/404/422 | ตรงกับ outcome |
| Pagination | offset หรือ cursor-based | cursor-based สำหรับ real-time |
| HATEOAS | `_links` ใน response | เพิ่ม discoverability |
| Versioning | URL/Header/Query | URL versioning ชัดเจนที่สุด |
| Rate Limiting | `X-RateLimit-*` headers | ป้องกัน abuse |
| Error Format | `{ error: { code, message } }` | consistent format |

---

## แบบฝึกหัด Part 47

1. **Complete CRUD API**: สร้าง REST API สำหรับ `Product` resource ครบทุก operations พร้อม proper status codes และ error handling

2. **Pagination**: Implement cursor-based pagination สำหรับ large dataset ที่มีการ sort ด้วย multiple fields

3. **HATEOAS**: เพิ่ม HATEOAS links ใน Article API response ที่ include: self, collection, edit, delete, related articles

4. **Rate Limiting**: สร้าง rate limiter ที่มี different limits สำหรับ: unauthenticated users (10 req/min), authenticated users (100 req/min), premium users (1000 req/min)

5. **API Versioning**: Migrate existing API จาก v1 ไป v2 โดยเพิ่ม fields ใหม่ใน v2 และยังคง backward compatible กับ v1

---

[→ ไปยัง Part 48: Play WebSockets](part-48-play-websockets.md)
