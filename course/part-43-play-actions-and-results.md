# Part 43: Play Actions and Results

## Steps 421-430: Action, Result, HTTP Status Codes, Headers, Cookies, Session, Flash

---

## Step 421: Action คืออะไร?

`Action` คือ function ที่รับ `Request` และ return `Result`

```scala
// Type signature ของ Action
type Action[A] = Request[A] => Result

// หรือแบบ async
type Action[A] = Request[A] => Future[Result]
```

### Action พื้นฐาน

```scala
// app/controllers/ExamplesController.scala
package controllers

import javax.inject.*
import play.api.mvc.*

@Singleton
class ExamplesController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  // Action ที่ return plain text
  def hello(): Action[AnyContent] = Action { implicit request =>
    Ok("Hello, World!")
  }

  // Action ที่ return HTML
  def htmlPage(): Action[AnyContent] = Action { implicit request =>
    Ok("<h1>Hello HTML</h1>").as(HTML)
  }

  // Action ที่ return JSON
  def jsonResponse(): Action[AnyContent] = Action { implicit request =>
    import play.api.libs.json.*
    Ok(Json.obj("message" -> "Hello JSON", "status" -> "ok"))
  }

  // Action แบบ explicit request type
  def withRequest(): Action[AnyContent] = Action { (request: Request[AnyContent]) =>
    val ip = request.remoteAddress
    Ok(s"Your IP: $ip")
  }
}
```

---

## Step 422: Result Types และ HTTP Status Codes

### Standard HTTP Results

```scala
// app/controllers/StatusController.scala
package controllers

import javax.inject.*
import play.api.mvc.*

@Singleton
class StatusController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  // 2xx Success
  def ok(): Action[AnyContent] = Action { _ => Ok("200 OK") }
  def created(): Action[AnyContent] = Action { _ => Created("201 Created") }
  def noContent(): Action[AnyContent] = Action { _ => NoContent }  // 204
  def accepted(): Action[AnyContent] = Action { _ => Accepted }    // 202

  // 3xx Redirect
  def redirectTemp(): Action[AnyContent] = Action { _ =>
    Redirect("/new-location")  // 303 See Other (default สำหรับ Redirect)
  }
  def redirectPerm(): Action[AnyContent] = Action { _ =>
    Redirect("/new-location", MOVED_PERMANENTLY)  // 301
  }
  def found(): Action[AnyContent] = Action { _ =>
    Redirect("/new-location", FOUND)  // 302
  }

  // 4xx Client Errors
  def badRequest(): Action[AnyContent] = Action { _ =>
    BadRequest("400 - Bad Request")
  }
  def unauthorized(): Action[AnyContent] = Action { _ =>
    Unauthorized("401 - Unauthorized")
      .withHeaders("WWW-Authenticate" -> "Bearer")
  }
  def forbidden(): Action[AnyContent] = Action { _ =>
    Forbidden("403 - Forbidden")
  }
  def notFound(): Action[AnyContent] = Action { _ =>
    NotFound("404 - Not Found")
  }
  def conflict(): Action[AnyContent] = Action { _ =>
    Conflict("409 - Conflict")
  }
  def unprocessable(): Action[AnyContent] = Action { _ =>
    UnprocessableEntity("422 - Unprocessable Entity")
  }
  def tooManyRequests(): Action[AnyContent] = Action { _ =>
    TooManyRequests("429 - Too Many Requests")
  }

  // 5xx Server Errors
  def serverError(): Action[AnyContent] = Action { _ =>
    InternalServerError("500 - Internal Server Error")
  }
  def notImplemented(): Action[AnyContent] = Action { _ =>
    NotImplemented("501 - Not Implemented")
  }
  def serviceUnavailable(): Action[AnyContent] = Action { _ =>
    ServiceUnavailable("503 - Service Unavailable")
  }

  // Custom status code
  def customStatus(): Action[AnyContent] = Action { _ =>
    Status(418)("I'm a teapot")  // RFC 2324 ;)
  }
}
```

### Result ด้วย Content Type ต่างๆ

```scala
// Content Types
Ok("plain text").as("text/plain")
Ok("<html>...</html>").as(HTML)           // text/html
Ok("""{"key": "value"}""").as(JSON)      // application/json
Ok(bytes).as("application/octet-stream")

// ใช้ play-json
import play.api.libs.json.*
Ok(Json.obj("key" -> "value"))  // Content-Type: application/json อัตโนมัติ

// XML
import scala.xml.*
Ok(<response><message>Hello</message></response>)  // text/xml อัตโนมัติ
```

---

## Step 423: Request Object

```scala
// app/controllers/RequestController.scala
package controllers

import javax.inject.*
import play.api.mvc.*

@Singleton
class RequestController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  def requestInfo(): Action[AnyContent] = Action { implicit request =>
    // ข้อมูลจาก Request object
    val method = request.method           // "GET", "POST", etc.
    val path = request.path              // "/users/42"
    val uri = request.uri                // "/users/42?page=1"
    val remoteAddress = request.remoteAddress  // "127.0.0.1"
    val host = request.host              // "localhost:9000"
    val domain = request.domain          // "localhost"
    val port = request.port              // Some(9000)

    // Query string
    val page = request.getQueryString("page")          // Option[String]
    val tags = request.queryString.getOrElse("tag", Seq.empty)  // Seq[String]

    // Headers
    val contentType = request.contentType            // Option[String]
    val accept = request.headers.get("Accept")       // Option[String]
    val userAgent = request.headers.get("User-Agent") // Option[String]
    val allHeaders = request.headers.toMap           // Map[String, Seq[String]]

    // Is it AJAX?
    val isAjax = request.headers.get("X-Requested-With")
                          .contains("XMLHttpRequest")

    // Is it secure (HTTPS)?
    val isSecure = request.secure  // Boolean

    Ok(s"""
      Method: $method
      Path: $path
      URI: $uri
      Remote: $remoteAddress
      Host: $host
      Secure: $isSecure
      User-Agent: ${userAgent.getOrElse("unknown")}
    """)
  }

  // รับ JSON body
  def receiveJson(): Action[play.api.libs.json.JsValue] =
    Action(parse.json) { implicit request =>
      import play.api.libs.json.*
      val body = request.body
      val name = (body \ "name").asOpt[String].getOrElse("unknown")
      Ok(Json.obj("received" -> name))
    }

  // รับ Form data
  def receiveForm(): Action[AnyContent] = Action { implicit request =>
    val formData = request.body.asFormUrlEncoded
    val name = formData.flatMap(_.get("name")).flatMap(_.headOption).getOrElse("")
    Ok(s"Hello, $name!")
  }
}
```

---

## Step 424: Headers

### Response Headers

```scala
// app/controllers/HeaderController.scala
package controllers

import javax.inject.*
import play.api.mvc.*

@Singleton
class HeaderController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  // เพิ่ม headers ใน response
  def withHeaders(): Action[AnyContent] = Action { implicit request =>
    Ok("Response with headers")
      .withHeaders(
        "X-Custom-Header" -> "my-value",
        "X-Request-Id"    -> java.util.UUID.randomUUID().toString,
        "Cache-Control"   -> "max-age=3600",
        "Vary"            -> "Accept-Encoding"
      )
  }

  // CORS headers (ถ้าไม่ใช้ CORSFilter)
  def withCors(): Action[AnyContent] = Action { implicit request =>
    Ok("CORS Response")
      .withHeaders(
        "Access-Control-Allow-Origin"  -> "*",
        "Access-Control-Allow-Methods" -> "GET, POST, PUT, DELETE, OPTIONS",
        "Access-Control-Allow-Headers" -> "Content-Type, Authorization"
      )
  }

  // Security headers
  def withSecurityHeaders(): Action[AnyContent] = Action { implicit request =>
    Ok("Secure Response")
      .withHeaders(
        "Strict-Transport-Security" -> "max-age=31536000; includeSubDomains",
        "X-Content-Type-Options"    -> "nosniff",
        "X-Frame-Options"           -> "DENY",
        "X-XSS-Protection"          -> "1; mode=block",
        "Content-Security-Policy"   -> "default-src 'self'"
      )
  }

  // Cache headers
  def cached(): Action[AnyContent] = Action { implicit request =>
    val etag = "abc123"
    val lastModified = "Wed, 01 Jan 2025 00:00:00 GMT"

    // ตรวจสอบ conditional request
    if (request.headers.get("If-None-Match").contains(etag)) {
      NotModified  // 304
    } else {
      Ok("cached content")
        .withHeaders(
          "ETag"          -> etag,
          "Last-Modified" -> lastModified,
          "Cache-Control" -> "public, max-age=3600"
        )
    }
  }
}
```

---

## Step 425: Cookies

```scala
// app/controllers/CookieController.scala
package controllers

import javax.inject.*
import play.api.mvc.*

@Singleton
class CookieController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  // Set cookie
  def setCookie(): Action[AnyContent] = Action { implicit request =>
    Ok("Cookie set!")
      .withCookies(
        Cookie(
          name     = "user_preference",
          value    = "dark_mode",
          maxAge   = Some(30 * 24 * 60 * 60),  // 30 วัน (วินาที)
          path     = "/",
          domain   = None,
          secure   = false,   // true ใน production (HTTPS only)
          httpOnly = true,    // ป้องกัน XSS
          sameSite = Some(Cookie.SameSite.Lax)
        )
      )
  }

  // อ่าน cookie
  def readCookie(): Action[AnyContent] = Action { implicit request =>
    val theme = request.cookies.get("user_preference")
                              .map(_.value)
                              .getOrElse("light_mode")

    Ok(s"Your theme: $theme")
  }

  // ลบ cookie (by setting maxAge = 0)
  def deleteCookie(): Action[AnyContent] = Action { implicit request =>
    Ok("Cookie deleted!")
      .discardingCookies(DiscardingCookie("user_preference"))
  }

  // Multiple cookies
  def multiCookies(): Action[AnyContent] = Action { implicit request =>
    Ok("Multiple cookies!")
      .withCookies(
        Cookie("lang", "th"),
        Cookie("timezone", "Asia/Bangkok"),
        Cookie("currency", "THB")
      )
  }

  // อ่าน cookies ทั้งหมด
  def allCookies(): Action[AnyContent] = Action { implicit request =>
    val cookieInfo = request.cookies.toList
      .map(c => s"${c.name}=${c.value}")
      .mkString(", ")

    Ok(s"Cookies: $cookieInfo")
  }
}
```

---

## Step 426: Session Management

Session ใน Play เก็บใน encrypted cookie ที่ client side (ไม่ใช่ server side)

```scala
// app/controllers/SessionController.scala
package controllers

import javax.inject.*
import play.api.mvc.*

@Singleton
class SessionController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  // Set session values
  def login(): Action[AnyContent] = Action { implicit request =>
    // สมมติว่า validate credentials แล้ว
    val userId = "user123"
    val username = "john_doe"
    val role = "admin"

    Redirect(routes.HomeController.index())
      .withSession(
        "userId"   -> userId,
        "username" -> username,
        "role"     -> role
      )
  }

  // เพิ่ม value ใน session ที่มีอยู่
  def addToSession(): Action[AnyContent] = Action { implicit request =>
    val currentSession = request.session

    Ok("Added to session")
      .withSession(
        currentSession +
        ("lastActivity" -> System.currentTimeMillis().toString) +
        ("loginTime"    -> System.currentTimeMillis().toString)
      )
  }

  // อ่าน session
  def profile(): Action[AnyContent] = Action { implicit request =>
    // ดู session ว่ามี userId หรือไม่
    request.session.get("userId") match {
      case Some(userId) =>
        val username = request.session.get("username").getOrElse("Unknown")
        val role = request.session.get("role").getOrElse("user")
        Ok(s"User: $username, Role: $role, ID: $userId")

      case None =>
        Redirect(routes.SessionController.login())
          .flashing("error" -> "กรุณา login ก่อน")
    }
  }

  // ลบ session ทั้งหมด (logout)
  def logout(): Action[AnyContent] = Action { implicit request =>
    Redirect(routes.HomeController.index())
      .withNewSession
      .flashing("success" -> "Logout สำเร็จ")
  }

  // ลบ key เฉพาะจาก session
  def removeFromSession(): Action[AnyContent] = Action { implicit request =>
    val newSession = request.session - "temporaryValue"
    Ok("Removed from session").withSession(newSession)
  }
}
```

### Session ใน View

```html
@* app/views/header.scala.html *@
@()(implicit request: Request[?])

<nav>
  @request.session.get("username").map { username =>
    <span>Welcome, @username!</span>
    <a href="@routes.SessionController.logout()">Logout</a>
  }.getOrElse {
    <a href="@routes.SessionController.login()">Login</a>
  }
</nav>
```

---

## Step 427: Flash Scope

Flash scope เป็น session ที่มีชีวิตแค่ request เดียว (one-time message)

```scala
// app/controllers/FlashController.scala
package controllers

import javax.inject.*
import play.api.mvc.*

@Singleton
class FlashController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  def createItem(): Action[AnyContent] = Action { implicit request =>
    // หลังจาก create สำเร็จ
    Redirect(routes.FlashController.list())
      .flashing(
        "success" -> "สร้างรายการสำเร็จแล้ว!",
        "newId"   -> "42"
      )
  }

  def failedOperation(): Action[AnyContent] = Action { implicit request =>
    Redirect(routes.FlashController.list())
      .flashing("error" -> "เกิดข้อผิดพลาด กรุณาลองใหม่")
  }

  def list(): Action[AnyContent] = Action { implicit request =>
    // อ่าน flash message
    val successMsg = request.flash.get("success")
    val errorMsg = request.flash.get("error")
    val newId = request.flash.get("newId")

    Ok(views.html.items.list(successMsg, errorMsg))
  }
}
```

### Flash ใน View

```html
@* app/views/items/list.scala.html *@
@(success: Option[String], error: Option[String])(implicit request: Request[?])

@* Flash messages *@
@success.map { msg =>
  <div class="alert alert-success">@msg</div>
}
@error.map { msg =>
  <div class="alert alert-danger">@msg</div>
}

@* หรือใช้ implicit flash scope *@
@request.flash.get("warning").map { msg =>
  <div class="alert alert-warning">@msg</div>
}
```

---

## Step 428: Action Composition

### Action Builder

```scala
// app/controllers/ActionBuilderController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import scala.concurrent.*

// Custom Action ที่ตรวจสอบ authentication
class AuthenticatedAction @Inject()(
  parser: BodyParsers.Default,
  implicit val ec: ExecutionContext
) extends ActionBuilderImpl(parser) {

  override def invokeBlock[A](
    request: Request[A],
    block: Request[A] => Future[Result]
  ): Future[Result] = {
    // ตรวจสอบ session
    request.session.get("userId") match {
      case Some(userId) =>
        block(request)  // ผ่าน authentication
      case None =>
        Future.successful(
          Redirect(routes.SessionController.login())
            .flashing("error" -> "กรุณา login ก่อน")
        )
    }
  }
}

// Custom Request ที่มี user info
class UserRequest[A](val userId: String, request: Request[A])
  extends WrappedRequest[A](request)

// Action ที่ inject user info ใน request
class UserAction @Inject()(
  parser: BodyParsers.Default,
  implicit val ec: ExecutionContext
) extends ActionBuilder[UserRequest, AnyContent] {

  override def parser: BodyParser[AnyContent] = parser

  override def invokeBlock[A](
    request: Request[A],
    block: UserRequest[A] => Future[Result]
  ): Future[Result] = {
    request.session.get("userId") match {
      case Some(userId) =>
        block(new UserRequest(userId, request))
      case None =>
        Future.successful(Unauthorized("กรุณา login"))
    }
  }

  override protected def executionContext: ExecutionContext = ec
}
```

### ใช้ Action Composition

```scala
@Singleton
class ProfileController @Inject()(
  val controllerComponents: ControllerComponents,
  authenticatedAction: AuthenticatedAction,
  userAction: UserAction
) extends BaseController {

  // ใช้ authenticated action
  def settings(): Action[AnyContent] = authenticatedAction { implicit request =>
    val userId = request.session.get("userId").get
    Ok(s"Settings for user: $userId")
  }

  // ใช้ user action (มี userId ใน request)
  def dashboard(): Action[AnyContent] = userAction { implicit request =>
    // request เป็น UserRequest ที่มี userId
    Ok(s"Dashboard for: ${request.userId}")
  }
}
```

---

## Step 429: Body Parsers

```scala
// app/controllers/BodyParserController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*

@Singleton
class BodyParserController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  // Parse JSON body
  def receiveJson(): Action[JsValue] = Action(parse.json) { implicit request =>
    val body: JsValue = request.body
    val name = (body \ "name").asOpt[String].getOrElse("")
    Ok(Json.obj("received" -> name, "status" -> "ok"))
  }

  // Parse JSON และ validate เป็น case class
  case class CreateUser(name: String, email: String, age: Int)
  implicit val userReads: Reads[CreateUser] = Json.reads[CreateUser]

  def createUser(): Action[JsValue] = Action(parse.json) { implicit request =>
    request.body.validate[CreateUser] match {
      case JsSuccess(user, _) =>
        // บันทึก user...
        Created(Json.obj(
          "message" -> s"สร้าง user ${user.name} สำเร็จ",
          "email"   -> user.email
        ))
      case JsError(errors) =>
        BadRequest(Json.obj(
          "message" -> "ข้อมูลไม่ถูกต้อง",
          "errors"  -> JsError.toJson(errors)
        ))
    }
  }

  // Parse form URL encoded
  def receiveForm(): Action[Map[String, Seq[String]]] =
    Action(parse.formUrlEncoded) { implicit request =>
      val name = request.body.get("name").flatMap(_.headOption).getOrElse("")
      Ok(s"Form received: $name")
    }

  // Parse multipart form (สำหรับ file uploads)
  def uploadFile(): Action[MultipartFormData[play.api.libs.Files.TemporaryFile]] =
    Action(parse.multipartFormData) { implicit request =>
      request.body.file("file").map { file =>
        val filename = file.filename
        val contentType = file.contentType
        val fileSize = file.fileSize

        // บันทึกไฟล์...
        file.ref.moveTo(java.nio.file.Paths.get(s"/tmp/$filename"))

        Ok(s"File uploaded: $filename (${fileSize} bytes)")
      }.getOrElse {
        BadRequest("ไม่พบไฟล์ที่อัพโหลด")
      }
    }

  // Limit body size
  def limitedBody(): Action[JsValue] =
    Action(parse.json(maxLength = 1024 * 1024)) { implicit request =>  // 1MB limit
      Ok("Body received within size limit")
    }

  // Tolerate body type (ยอมรับทุก Content-Type)
  def anyBody(): Action[AnyContent] = Action { implicit request =>
    val bodyText = request.body match {
      case AnyContentAsJson(json)     => s"JSON: ${json}"
      case AnyContentAsFormUrlEncoded(form) => s"Form: $form"
      case AnyContentAsText(text)     => s"Text: $text"
      case AnyContentAsRaw(raw)       => s"Raw: ${raw.size} bytes"
      case _                          => "Unknown body type"
    }
    Ok(bodyText)
  }
}
```

---

## Step 430: Async Actions

```scala
// app/controllers/AsyncController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import scala.concurrent.*
import scala.concurrent.duration.*

@Singleton
class AsyncController @Inject()(
  val controllerComponents: ControllerComponents,
  implicit val ec: ExecutionContext
) extends BaseController {

  // Async action พื้นฐาน
  def asyncHello(): Action[AnyContent] = Action.async { implicit request =>
    // Future ที่ return ใน background
    Future {
      Thread.sleep(100)  // simulate delay
      "Hello from async!"
    }.map { message =>
      Ok(message)
    }
  }

  // Async ที่เรียก service
  def asyncData(): Action[AnyContent] = Action.async { implicit request =>
    // เรียก async service
    val futureData = fetchDataFromDatabase()

    futureData.map { data =>
      Ok(data)
    }.recover {
      case ex: Exception =>
        InternalServerError(s"Error: ${ex.getMessage}")
    }
  }

  // รวม multiple futures
  def combinedAsync(): Action[AnyContent] = Action.async { implicit request =>
    val future1 = fetchUser(1L)
    val future2 = fetchArticles(1L)

    for {
      user     <- future1
      articles <- future2
    } yield Ok(s"User: $user, Articles: $articles")
  }

  // Helper methods
  private def fetchDataFromDatabase(): Future[String] =
    Future.successful("Data from DB")

  private def fetchUser(id: Long): Future[String] =
    Future.successful(s"User $id")

  private def fetchArticles(userId: Long): Future[String] =
    Future.successful(s"Articles for user $userId")
}
```

---

## สรุป Part 43

| Concept | คำอธิบาย | ตัวอย่าง |
|---------|---------|---------|
| Action | Function รับ Request return Result | `Action { implicit request => Ok("hi") }` |
| Result | HTTP Response | `Ok`, `NotFound`, `Redirect` |
| Status codes | 2xx, 3xx, 4xx, 5xx | `Ok`, `Created`, `BadRequest`, `NotFound` |
| Headers | HTTP Response headers | `.withHeaders("X-Custom" -> "value")` |
| Cookies | Client-side storage | `.withCookies(Cookie("name", "value"))` |
| Session | Encrypted cookie | `.withSession("key" -> "value")` |
| Flash | One-time message | `.flashing("success" -> "Done!")` |
| Body Parser | Parse request body | `Action(parse.json)` |
| Async Action | Non-blocking | `Action.async { Future { ... } }` |

---

## แบบฝึกหัด Part 43

1. **Status Codes**: สร้าง controller ที่ return status code ที่แตกต่างกันตาม query parameter `code` เช่น `GET /status?code=404` → NotFound

2. **Session Auth**: สร้าง authentication flow ที่: login → set session → protected page (ตรวจ session) → logout → clear session

3. **Cookies**: สร้าง preference system ที่ใช้ cookies เก็บ theme (dark/light), language (th/en), timezone พร้อม UI สำหรับ update

4. **Action Composition**: สร้าง `RateLimitAction` ที่จำกัด request ไม่เกิน 10 ครั้งต่อนาที โดยใช้ IP address

5. **Body Parsers**: สร้าง API endpoint ที่รับ JSON, validate ด้วย case class, และ return structured error response เมื่อ validation fail

---

[→ ไปยัง Part 44: Play Templates](part-44-play-templates.md)
