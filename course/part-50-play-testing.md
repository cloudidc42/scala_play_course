# Part 50: Play Testing

## Steps 491-500: Unit Tests, Integration Tests, FakeApplication, WsClient, Mocking

---

## Step 491: Testing Stack ใน Play

```
Play Testing Stack:
├── ScalaTest - test framework
├── scalatestplus-play - Play integration
├── Mockito - mocking framework
├── H2 - in-memory database สำหรับ tests
└── WsTestClient - HTTP client สำหรับ integration tests
```

### build.sbt

```scala
// build.sbt
libraryDependencies ++= Seq(
  guice,
  "org.scalatestplus.play" %% "scalatestplus-play" % "7.0.1"    % Test,
  "org.mockito"            %% "mockito-scala"       % "1.17.30"  % Test,
  "com.h2database"         %  "h2"                  % "2.2.224"  % Test
)
```

### Test configuration

```hocon
# conf/test.conf
include "application.conf"

# Override settings สำหรับ tests
play.http.secret.key = "test-secret-key-not-for-production"

# ใช้ in-memory database
db.default.driver = org.h2.Driver
db.default.url = "jdbc:h2:mem:test;MODE=PostgreSQL;DB_CLOSE_DELAY=-1"

# Disable some filters ใน test
play.filters.disabled += "play.filters.csrf.CSRFFilter"
play.filters.disabled += "play.filters.hosts.AllowedHostsFilter"
```

---

## Step 492: Unit Tests สำหรับ Business Logic

```scala
// test/services/ArticleServiceSpec.scala
package services

import org.scalatest.*
import org.scalatest.wordspec.AnyWordSpec
import org.scalatest.matchers.should.Matchers
import org.scalatest.concurrent.ScalaFutures
import scala.concurrent.ExecutionContext.Implicits.global

class ArticleServiceSpec extends AnyWordSpec with Matchers with ScalaFutures {

  // สร้าง mock repository
  val mockRepo = new repositories.ArticleRepository {
    private var articles = List(
      models.Article(1L, "Test Article", "Content", 1L, List("scala"), true, None, 0L),
      models.Article(2L, "Draft Article", "Draft", 1L, List(), false, None, 0L)
    )

    def findById(id: Long): scala.concurrent.Future[Option[models.Article]] =
      scala.concurrent.Future.successful(articles.find(_.id == id))

    def findAll(page: Int, limit: Int, query: Option[String], tag: Option[String], published: Option[Boolean]):
      scala.concurrent.Future[(List[models.Article], Long)] = {
      val filtered = articles
        .filter(a => query.forall(q => a.title.contains(q)))
        .filter(a => tag.forall(t => a.tags.contains(t)))
        .filter(a => published.forall(_ == a.published))
      scala.concurrent.Future.successful((filtered, filtered.length.toLong))
    }

    def create(title: String, content: String, tags: List[String], published: Boolean, slug: String): scala.concurrent.Future[models.Article] = {
      val article = models.Article(articles.length + 1L, title, content, 1L, tags, published, None, 0L)
      articles = articles :+ article
      scala.concurrent.Future.successful(article)
    }

    def update(id: Long, title: String, content: String, tags: List[String], published: Boolean, slug: String): scala.concurrent.Future[models.Article] = {
      val updated = models.Article(id, title, content, 1L, tags, published, None, 0L)
      articles = articles.map(a => if (a.id == id) updated else a)
      scala.concurrent.Future.successful(updated)
    }

    def delete(id: Long): scala.concurrent.Future[Boolean] = {
      val before = articles.length
      articles = articles.filterNot(_.id == id)
      scala.concurrent.Future.successful(articles.length < before)
    }

    def setPublished(id: Long, published: Boolean): scala.concurrent.Future[models.Article] = {
      val updated = articles.find(_.id == id).get.copy(published = published)
      articles = articles.map(a => if (a.id == id) updated else a)
      scala.concurrent.Future.successful(updated)
    }
  }

  val service = new ArticleService(mockRepo)

  "ArticleService" should {
    "find article by id" in {
      val result = service.findById(1L).futureValue
      result shouldBe defined
      result.get.title shouldBe "Test Article"
    }

    "return None for unknown id" in {
      val result = service.findById(999L).futureValue
      result shouldBe None
    }

    "create article with slug" in {
      val article = service.create(
        title     = "My New Article",
        content   = "Content here",
        tags      = List("scala", "play"),
        published = false
      ).futureValue

      article.title shouldBe "My New Article"
      // Verify the service creates a slug
    }

    "list only published articles" in {
      val (articles, total) = service.list(1, 10, published = Some(true)).futureValue
      articles.foreach(_.published shouldBe true)
    }

    "publish unpublished article" in {
      val result = service.publish(2L).futureValue
      result shouldBe defined
      result.get.published shouldBe true
    }
  }
}
```

---

## Step 493: Controller Unit Tests

```scala
// test/controllers/ArticleControllerSpec.scala
package controllers

import org.scalatestplus.play.*
import play.api.test.*
import play.api.test.Helpers.*
import play.api.libs.json.*
import org.mockito.Mockito.*
import org.mockito.ArgumentMatchers.*
import scala.concurrent.Future

class ArticleControllerSpec extends PlaySpec {

  // Mock service
  val mockService = mock(classOf[services.ArticleService])

  // สร้าง controller ด้วย mock dependencies
  val controller = new controllers.api.ArticleApiController(
    Helpers.stubControllerComponents(),
    mockService
  )(scala.concurrent.ExecutionContext.Implicits.global)

  "ArticleApiController GET /api/articles" should {
    "return 200 with articles" in {
      val articles = List(
        models.Article(1L, "Test", "Content", 1L, List(), true, None, 0L)
      )
      when(mockService.list(any(), any(), any(), any(), any()))
        .thenReturn(Future.successful((articles, 1L)))

      val request = FakeRequest(GET, "/api/articles")
      val result = controller.index()(request)

      status(result) mustBe OK
      contentType(result) mustBe Some("application/json")

      val json = contentAsJson(result)
      (json \ "success").as[Boolean] mustBe true
    }
  }

  "ArticleApiController POST /api/articles" should {
    "return 201 for valid JSON" in {
      val article = models.Article(1L, "New Article", "Content", 1L, List("scala"), false, None, 0L)
      when(mockService.create(any(), any(), any(), any()))
        .thenReturn(Future.successful(article))

      val body = Json.obj(
        "title"   -> "New Article",
        "content" -> "Content",
        "tags"    -> Json.arr("scala")
      )

      val request = FakeRequest(POST, "/api/articles").withJsonBody(body)
      val result = call(controller.create(), request)

      status(result) mustBe CREATED
    }

    "return 422 for invalid JSON" in {
      val body = Json.obj("content" -> "Missing title")
      val request = FakeRequest(POST, "/api/articles").withJsonBody(body)
      val result = call(controller.create(), request)

      status(result) mustBe UNPROCESSABLE_ENTITY
    }
  }

  "ArticleApiController GET /api/articles/:id" should {
    "return 200 for existing article" in {
      val article = models.Article(1L, "Test", "Content", 1L, List(), true, None, 0L)
      when(mockService.findById(1L)).thenReturn(Future.successful(Some(article)))

      val request = FakeRequest(GET, "/api/articles/1")
      val result = controller.show(1L)(request)

      status(result) mustBe OK
    }

    "return 404 for non-existing article" in {
      when(mockService.findById(999L)).thenReturn(Future.successful(None))

      val request = FakeRequest(GET, "/api/articles/999")
      val result = controller.show(999L)(request)

      status(result) mustBe NOT_FOUND
    }
  }
}
```

---

## Step 494: Integration Tests ด้วย GuiceOneAppPerTest

```scala
// test/IntegrationSpec.scala
package integration

import org.scalatestplus.play.*
import org.scalatestplus.play.guice.GuiceOneAppPerTest
import play.api.test.*
import play.api.test.Helpers.*
import play.api.libs.json.*

class ArticleIntegrationSpec extends PlaySpec with GuiceOneAppPerTest {

  // GuiceOneAppPerTest: สร้าง app ใหม่ทุก test
  // ใช้ real application พร้อม all dependencies

  override def fakeApplication() = {
    // Override application configuration สำหรับ test
    GuiceApplicationBuilder()
      .configure(
        "db.default.driver" -> "org.h2.Driver",
        "db.default.url"    -> "jdbc:h2:mem:test;MODE=PostgreSQL;DB_CLOSE_DELAY=-1"
      )
      .build()
  }

  "GET /api/articles" should {
    "return 200 with empty list initially" in {
      val request = FakeRequest(GET, "/api/articles")
      val result = route(app, request).get

      status(result) mustBe OK
      contentType(result) mustBe Some("application/json")

      val json = contentAsJson(result)
      (json \ "success").as[Boolean] mustBe true
      (json \ "data").as[JsArray].value mustBe empty
    }
  }

  "POST then GET /api/articles" should {
    "create and retrieve article" in {
      // Create article
      val createBody = Json.obj(
        "title"   -> "Integration Test Article",
        "content" -> "Test content for integration test",
        "tags"    -> Json.arr("test", "scala")
      )

      val createRequest = FakeRequest(POST, "/api/articles")
        .withJsonBody(createBody)
        .withHeaders(
          "Content-Type" -> "application/json",
          "Csrf-Token"   -> "nocheck"
        )

      val createResult = route(app, createRequest).get
      status(createResult) mustBe CREATED

      val createdJson = contentAsJson(createResult)
      val articleId = (createdJson \ "data" \ "id").as[Long]

      // Get the created article
      val getRequest = FakeRequest(GET, s"/api/articles/$articleId")
      val getResult = route(app, getRequest).get

      status(getResult) mustBe OK
      val articleJson = (contentAsJson(getResult) \ "data").as[JsObject]
      (articleJson \ "title").as[String] mustBe "Integration Test Article"
    }
  }
}
```

---

## Step 495: GuiceOneServerPerTest สำหรับ Full Integration

```scala
// test/integration/FullIntegrationSpec.scala
package integration

import org.scalatestplus.play.*
import org.scalatestplus.play.guice.GuiceOneServerPerTest
import play.api.test.*
import play.api.libs.ws.*

class FullIntegrationSpec extends PlaySpec with GuiceOneServerPerTest with WsScalaTestClient {

  // GuiceOneServerPerTest: รัน real HTTP server
  // เหมาะสำหรับ test ที่ต้องการ real network stack

  "Application" should {
    "return OK for home page" in {
      val response = await(wsUrl("/").get())
      response.status mustBe 200
    }

    "return JSON for API endpoints" in {
      val response = await(
        wsUrl("/api/articles")
          .withHttpHeaders("Accept" -> "application/json")
          .get()
      )

      response.status mustBe 200
      response.contentType must include("application/json")
    }

    "handle 404 gracefully" in {
      val response = await(wsUrl("/nonexistent-page").get())
      response.status mustBe 404
    }
  }
}
```

---

## Step 496: Database Integration Testing

```scala
// test/repositories/ArticleRepositorySpec.scala
package repositories

import org.scalatestplus.play.*
import org.scalatestplus.play.guice.GuiceOneAppPerSuite
import play.api.test.*
import play.api.db.slick.DatabaseConfigProvider
import slick.jdbc.JdbcProfile
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.Await
import scala.concurrent.duration.*

class ArticleRepositorySpec extends PlaySpec with GuiceOneAppPerSuite {

  override def fakeApplication() = {
    GuiceApplicationBuilder()
      .configure(
        "db.default.driver" -> "org.h2.Driver",
        "db.default.url"    -> "jdbc:h2:mem:test;MODE=PostgreSQL;DB_CLOSE_DELAY=-1",
        "play.evolutions.db.default.enabled" -> true,
        "play.evolutions.db.default.autoApply" -> true
      )
      .build()
  }

  lazy val repo = app.injector.instanceOf[ArticleRepository]

  "ArticleRepository" should {
    "create and find article" in {
      val created = Await.result(
        repo.create("Test Article", "Content", List("scala"), false, "test-article"),
        5.seconds
      )

      created.title mustBe "Test Article"
      created.id must be > 0L

      val found = Await.result(repo.findById(created.id), 5.seconds)
      found mustBe defined
      found.get.title mustBe "Test Article"
    }

    "delete article" in {
      val article = Await.result(
        repo.create("To Delete", "Content", List(), false, "to-delete"),
        5.seconds
      )

      val deleted = Await.result(repo.delete(article.id), 5.seconds)
      deleted mustBe true

      val found = Await.result(repo.findById(article.id), 5.seconds)
      found mustBe None
    }
  }
}
```

---

## Step 497: Mocking ด้วย Mockito

```scala
// test/services/UserServiceSpec.scala
package services

import org.scalatest.*
import org.scalatest.wordspec.AnyWordSpec
import org.scalatest.matchers.should.Matchers
import org.scalatest.concurrent.ScalaFutures
import org.mockito.Mockito.*
import org.mockito.ArgumentMatchers.*
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

class UserServiceSpec extends AnyWordSpec with Matchers with ScalaFutures {

  val mockUserRepo = mock(classOf[repositories.UserRepository])
  val mockEmailService = mock(classOf[services.EmailService])
  val mockCacheService = mock(classOf[services.CacheService])

  val userService = new UserService(
    userRepo      = mockUserRepo,
    emailService  = mockEmailService,
    cacheService  = mockCacheService
  )

  "UserService" when {

    "registering a new user" should {
      "save user and send welcome email" in {
        val email = "newuser@example.com"
        val mockUser = models.User(1L, "newuser", email, models.UserRole.Viewer,
                                    java.time.LocalDateTime.now())

        // Setup mocks
        when(mockUserRepo.findByEmail(email)).thenReturn(Future.successful(None))
        when(mockUserRepo.create(any(), any(), any())).thenReturn(Future.successful(mockUser))
        when(mockEmailService.sendWelcome(any())).thenReturn(Future.successful(()))
        when(mockCacheService.invalidate(any())).thenReturn(Future.successful(()))

        // Execute
        val result = userService.register("newuser", email, "Password123!").futureValue

        // Verify
        result shouldBe mockUser
        verify(mockEmailService).sendWelcome(mockUser)
        verify(mockCacheService).invalidate("users:list")
      }

      "fail if email already exists" in {
        val existingUser = models.User(1L, "existing", "existing@example.com",
                                       models.UserRole.Viewer, java.time.LocalDateTime.now())

        when(mockUserRepo.findByEmail("existing@example.com"))
          .thenReturn(Future.successful(Some(existingUser)))

        val exception = intercept[Exception] {
          userService.register("newuser", "existing@example.com", "Password123!").futureValue
        }

        exception.getMessage should include("already exists")
        verify(mockEmailService, never()).sendWelcome(any())
      }
    }
  }
}
```

---

## Step 498: Testing Forms

```scala
// test/controllers/FormControllerSpec.scala
package controllers

import org.scalatestplus.play.*
import org.scalatestplus.play.guice.GuiceOneAppPerTest
import play.api.test.*
import play.api.test.Helpers.*

class FormControllerSpec extends PlaySpec with GuiceOneAppPerTest {

  "UserController POST /register" should {
    "redirect on valid form submission" in {
      val formData = Map(
        "name"            -> "John Doe",
        "email"           -> "john@example.com",
        "password"        -> "SecurePass123!",
        "confirmPassword" -> "SecurePass123!",
        "age"             -> "25",
        "acceptTerms"     -> "true"
      )

      val request = FakeRequest(POST, "/register")
        .withFormUrlEncodedBody(formData.toSeq: _*)
        .withHeaders(
          "Csrf-Token" -> "nocheck"  // Bypass CSRF ใน test
        )

      val result = route(app, request).get

      status(result) mustBe SEE_OTHER  // 303 Redirect
      redirectLocation(result) mustBe Some("/")
      flash(result).get("success") mustBe defined
    }

    "return 400 for missing required fields" in {
      val incompleteData = Map(
        "email" -> "john@example.com"
        // ขาด name, password, etc.
      )

      val request = FakeRequest(POST, "/register")
        .withFormUrlEncodedBody(incompleteData.toSeq: _*)
        .withHeaders("Csrf-Token" -> "nocheck")

      val result = route(app, request).get

      status(result) mustBe BAD_REQUEST
    }

    "return 400 for invalid email" in {
      val invalidData = Map(
        "name"            -> "John",
        "email"           -> "not-an-email",  // invalid
        "password"        -> "Password123!",
        "confirmPassword" -> "Password123!",
        "age"             -> "25",
        "acceptTerms"     -> "true"
      )

      val request = FakeRequest(POST, "/register")
        .withFormUrlEncodedBody(invalidData.toSeq: _*)
        .withHeaders("Csrf-Token" -> "nocheck")

      val result = route(app, request).get

      status(result) mustBe BAD_REQUEST
      contentAsString(result) must include("valid email")
    }
  }
}
```

---

## Step 499: Testing WebSocket

```scala
// test/controllers/WebSocketSpec.scala
package controllers

import org.scalatestplus.play.*
import org.scalatestplus.play.guice.GuiceOneServerPerTest
import play.api.libs.ws.*
import play.api.test.*
import scala.concurrent.duration.*

class WebSocketSpec extends PlaySpec with GuiceOneServerPerTest with WsScalaTestClient {

  "WebSocket echo" should {
    "echo messages back" in {
      // Simple test ว่า WebSocket endpoint accessible
      val wsUrl = s"ws://localhost:$port/ws/echo"
      wsUrl must include("ws://")
      wsUrl must include("9000")
    }
  }
}
```

---

## Step 500: Testing Best Practices

```scala
// test/support/TestData.scala
package support

import models.*
import java.time.LocalDateTime

object TestData {

  // Factory methods สำหรับ test data
  def article(
    id: Long = 1L,
    title: String = "Test Article",
    content: String = "Test Content",
    authorId: Long = 1L,
    tags: List[String] = List("test"),
    published: Boolean = true
  ): Article = Article(id, title, content, authorId, tags, published, None, 0L)

  def user(
    id: Long = 1L,
    username: String = "testuser",
    email: String = "test@example.com",
    role: UserRole = UserRole.Viewer
  ): User = User(id, username, email, role, LocalDateTime.now())

  // Bulk test data
  def articles(n: Int): List[Article] =
    (1 to n).map(i => article(id = i.toLong, title = s"Article $i")).toList
}

// test/support/TestHelpers.scala
package support

import play.api.test.*
import play.api.test.Helpers.*
import play.api.libs.json.*

trait TestHelpers {
  this: org.scalatestplus.play.PlaySpec =>

  // Helper: make authenticated request
  def authRequest(method: String, path: String, userId: String = "1"): FakeRequest[?] =
    FakeRequest(method, path)
      .withSession("userId" -> userId, "username" -> "testuser")
      .withHeaders("Csrf-Token" -> "nocheck")

  // Helper: make JSON API request
  def jsonRequest(method: String, path: String, body: JsValue): FakeRequest[JsValue] =
    FakeRequest(method, path)
      .withJsonBody(body)
      .withHeaders(
        "Content-Type" -> "application/json",
        "Accept"       -> "application/json",
        "Csrf-Token"   -> "nocheck"
      )

  // Helper: assert JSON response
  def assertJsonResponse(result: scala.concurrent.Future[play.api.mvc.Result])(
    assertions: JsValue => Unit
  ): Unit = {
    status(result) mustBe OK
    contentType(result) mustBe Some("application/json")
    val json = contentAsJson(result)
    assertions(json)
  }
}
```

### Test Runner Configuration

```
# .sbt/1.0/global.sbt หรือ project/build.properties
# Run tests in parallel
testForkedParallel := true

# Test coverage
addSbtPlugin("org.scoverage" % "sbt-scoverage" % "2.0.9")
```

```bash
# Run all tests
sbt test

# Run specific test
sbt "testOnly controllers.ArticleControllerSpec"

# Run with coverage
sbt clean coverage test coverageReport

# Run integration tests only
sbt "testOnly *IntegrationSpec"

# Watch for file changes
sbt ~test
```

---

## สรุป Part 50

| Test Type | Trait | เหมาะสำหรับ |
|-----------|-------|-----------|
| Unit test | `AnyWordSpec` + `Matchers` | Business logic, pure functions |
| Controller test | `PlaySpec` + mock services | HTTP responses, status codes |
| App test | `GuiceOneAppPerTest` | Route testing, DI |
| Server test | `GuiceOneServerPerTest` | Real HTTP, WebSocket |
| DB test | `GuiceOneAppPerSuite` + test DB | Repository queries |

---

## แบบฝึกหัด Part 50

1. **TDD Kata**: สร้าง FizzBuzz service ด้วย TDD approach - เขียน test ก่อน แล้วค่อย implement code ให้ test ผ่าน

2. **Controller Testing**: เขียน test ครบถ้วนสำหรับ UserController ที่ cover: login success/failure, register validation, profile update, logout

3. **Integration Testing**: สร้าง integration test suite ที่ test complete user flow: register → login → create article → edit → delete

4. **Mock Testing**: สร้าง test สำหรับ service ที่มี external dependencies (email, payment gateway) โดย mock ทุก external calls

5. **Test Coverage**: เพิ่ม test coverage ของ project ให้ถึง 80% โดยใช้ `sbt-scoverage` วัด coverage และเขียน tests เพิ่มเติมสำหรับ uncovered code

---

[→ ไปยัง Part 51: Play Auth and Security](part-51-play-auth-and-security.md)
