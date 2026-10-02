# Part 38: Akka HTTP

## Steps 371-380: Routes, Directives, JSON Marshalling, WebSockets, Client

---

## Step 371: Akka HTTP Server พื้นฐาน

```scala
// build.sbt:
// libraryDependencies += "com.typesafe.akka" %% "akka-http" % "10.5.0"
// libraryDependencies += "com.typesafe.akka" %% "akka-http-spray-json" % "10.5.0"

import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.model._
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.server.Route
import scala.concurrent.duration._
import scala.concurrent.{Future, Await}

object AkkaHttpBasics extends App {
  implicit val system = ActorSystem("http-server")
  implicit val ec = system.dispatcher
  
  // Simple routes
  val routes: Route =
    pathPrefix("api") {
      concat(
        // GET /api/health
        path("health") {
          get {
            complete(StatusCodes.OK, """{"status":"healthy"}""")
          }
        },
        
        // GET /api/greet/name
        path("greet" / Segment) { name =>
          get {
            complete(s"""{"message":"Hello, $name!"}""")
          }
        },
        
        // POST /api/echo
        path("echo") {
          post {
            entity(as[String]) { body =>
              complete(StatusCodes.OK, s"""{"echo":$body}""")
            }
          }
        },
        
        // GET /api/items?page=1&limit=10
        path("items") {
          get {
            parameters("page".as[Int].withDefault(1), "limit".as[Int].withDefault(10)) { (page, limit) =>
              complete(s"""{"page":$page,"limit":$limit,"items":[]}""")
            }
          }
        }
      )
    }
  
  val binding = Http().newServerAt("localhost", 8080).bind(routes)
  println("Server started at http://localhost:8080")
  println("Press ENTER to stop...")
  scala.io.StdIn.readLine()
  
  binding
    .flatMap(_.unbind())
    .onComplete(_ => system.terminate())
}
```

---

## Step 372: JSON Marshalling

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.model._
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.marshallers.sprayjson.SprayJsonSupport._
import spray.json._
import DefaultJsonProtocol._
import scala.concurrent._

// Domain models
case class User(id: Option[Int], name: String, email: String, age: Int)
case class CreateUserRequest(name: String, email: String, age: Int)
case class PaginatedResponse[A](items: List[A], total: Int, page: Int, limit: Int)

// JSON protocols
object JsonProtocols extends DefaultJsonProtocol {
  implicit val userFormat: RootJsonFormat[User] = jsonFormat4(User)
  implicit val createUserFormat: RootJsonFormat[CreateUserRequest] = jsonFormat3(CreateUserRequest)
  
  implicit def paginatedFormat[A: JsonFormat]: RootJsonFormat[PaginatedResponse[A]] =
    jsonFormat4(PaginatedResponse[A])
}

object JsonApi extends App {
  implicit val system = ActorSystem("json-api")
  implicit val ec = system.dispatcher
  
  import JsonProtocols._
  
  // In-memory "database"
  var users = List(
    User(Some(1), "Alice", "alice@example.com", 30),
    User(Some(2), "Bob", "bob@example.com", 25),
    User(Some(3), "Carol", "carol@example.com", 35)
  )
  var nextId = 4
  
  // Error response
  case class ErrorResponse(error: String, code: Int)
  implicit val errorFormat: RootJsonFormat[ErrorResponse] = jsonFormat2(ErrorResponse)
  
  import akka.http.scaladsl.server.{ExceptionHandler, RejectionHandler}
  import akka.http.scaladsl.model.StatusCodes._
  
  // Exception handler
  implicit val exceptionHandler: ExceptionHandler = ExceptionHandler {
    case e: IllegalArgumentException =>
      complete(BadRequest -> ErrorResponse(e.getMessage, 400))
    case e: Exception =>
      complete(InternalServerError -> ErrorResponse(e.getMessage, 500))
  }
  
  val routes: Route = handleExceptions(exceptionHandler) {
    pathPrefix("api" / "v1") {
      
      // ===== Users CRUD =====
      pathPrefix("users") {
        concat(
          
          // GET /api/v1/users?page=1&limit=10
          (get & pathEnd & parameters("page".as[Int].withDefault(1), "limit".as[Int].withDefault(10))) { (page, limit) =>
            val offset = (page - 1) * limit
            val paged = users.drop(offset).take(limit)
            complete(PaginatedResponse(paged, users.size, page, limit))
          },
          
          // GET /api/v1/users/:id
          (get & path(IntNumber)) { id =>
            users.find(_.id.contains(id)) match {
              case Some(user) => complete(user)
              case None       => complete(NotFound -> ErrorResponse(s"User $id not found", 404))
            }
          },
          
          // POST /api/v1/users
          (post & pathEnd & entity(as[CreateUserRequest])) { req =>
            if (req.name.isEmpty) throw new IllegalArgumentException("Name cannot be empty")
            if (!req.email.contains("@")) throw new IllegalArgumentException("Invalid email")
            if (req.age < 0 || req.age > 150) throw new IllegalArgumentException("Invalid age")
            
            if (users.exists(_.email == req.email)) {
              complete(Conflict -> ErrorResponse(s"Email ${req.email} already exists", 409))
            } else {
              val user = User(Some(nextId), req.name, req.email, req.age)
              users = users :+ user
              nextId += 1
              complete(Created -> user)
            }
          },
          
          // PUT /api/v1/users/:id
          (put & path(IntNumber) & entity(as[CreateUserRequest])) { (id, req) =>
            users.indexWhere(_.id.contains(id)) match {
              case -1 => complete(NotFound -> ErrorResponse(s"User $id not found", 404))
              case idx =>
                val updated = User(Some(id), req.name, req.email, req.age)
                users = users.updated(idx, updated)
                complete(updated)
            }
          },
          
          // DELETE /api/v1/users/:id
          (delete & path(IntNumber)) { id =>
            users.find(_.id.contains(id)) match {
              case None => complete(NotFound -> ErrorResponse(s"User $id not found", 404))
              case Some(_) =>
                users = users.filterNot(_.id.contains(id))
                complete(NoContent)
            }
          }
        )
      }
    )
  }
  
  val binding = Http().newServerAt("localhost", 8081).bind(routes)
  println("JSON API running at http://localhost:8081")
  println("Press ENTER to stop...")
  scala.io.StdIn.readLine()
  Await.ready(binding.flatMap(_.unbind()), 5.seconds)
  system.terminate()
}
```

---

## Step 373: Directives ลึก

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.model._
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.server.{Directive0, Directive1, Route}

object DirectivesDeep extends App {
  implicit val system = ActorSystem("directives")
  implicit val ec = system.dispatcher
  
  // ===== Custom Directives =====
  
  // Authentication directive
  def authenticate: Directive1[String] = {
    optionalHeaderValueByName("Authorization").flatMap {
      case Some(token) if token.startsWith("Bearer ") =>
        val userId = token.drop(7)  // Simplified — would verify JWT
        provide(userId)
      case _ =>
        complete(StatusCodes.Unauthorized -> """{"error":"Unauthorized"}""")
    }
  }
  
  // Rate limiting directive (simple)
  val requestCounts = new java.util.concurrent.ConcurrentHashMap[String, Int]()
  
  def rateLimited(maxRequests: Int): Directive0 = {
    extractClientIP.flatMap { ip =>
      val count = requestCounts.merge(ip.toString, 1, (a, b) => a + b)
      if (count > maxRequests) {
        complete(StatusCodes.TooManyRequests -> """{"error":"Rate limit exceeded"}""")
      } else {
        pass
      }
    }
  }
  
  // Logging directive
  def logRequest: Directive0 = {
    extractRequest.flatMap { req =>
      println(s"  [${java.time.LocalTime.now()}] ${req.method.value} ${req.uri}")
      pass
    }
  }
  
  // Validate request directive
  def validateContentType(expectedType: ContentType): Directive0 = {
    extractRequest.flatMap { req =>
      if (req.method == HttpMethods.GET || req.entity.contentType == expectedType) pass
      else complete(StatusCodes.UnsupportedMediaType -> """{"error":"Wrong content type"}""")
    }
  }
  
  // Timing directive
  def timed: Directive0 = {
    val start = System.currentTimeMillis()
    mapResponse { response =>
      println(s"  Request took ${System.currentTimeMillis() - start}ms")
      response
    }
  }
  
  // Routes using custom directives
  val routes: Route = logRequest {
    timed {
      pathPrefix("api") {
        concat(
          path("public") {
            get { complete("Public endpoint") }
          },
          
          path("protected") {
            authenticate { userId =>
              get {
                complete(s"""{"message":"Hello, $userId"}""")
              }
            }
          },
          
          path("admin") {
            authenticate { userId =>
              authorize(userId.startsWith("admin")) {
                get { complete("""{"admin":"data"}""") }
              }
            }
          },
          
          path("file") {
            // File upload
            (post & fileUpload("document")) { case (metadata, byteSource) =>
              val filename = metadata.fileName
              import akka.stream.scaladsl.Sink
              val content = byteSource.runWith(Sink.fold("")(_ + _.utf8String))
              onSuccess(content) { data =>
                complete(s"""{"filename":"$filename","size":${data.length}}""")
              }
            }
          }
        )
      }
    }
  }
  
  val binding = Http().newServerAt("localhost", 8082).bind(routes)
  println("Directives demo at http://localhost:8082")
  scala.io.StdIn.readLine()
  binding.flatMap(_.unbind()).onComplete(_ => system.terminate())
}
```

---

## Step 374-380: HTTP Client and WebSocket

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.model._
import akka.http.scaladsl.model.ws._
import akka.stream.scaladsl._
import akka.http.scaladsl.server.Directives._
import scala.concurrent._
import scala.concurrent.duration._

object HttpClientDemo extends App {
  implicit val system = ActorSystem("http-client")
  implicit val ec = system.dispatcher
  
  // ===== HTTP Client =====
  
  // Simple GET
  def getRequest(url: String): Future[String] = {
    Http().singleRequest(HttpRequest(uri = url))
      .flatMap(_.entity.toStrict(5.seconds))
      .map(_.data.utf8String)
  }
  
  // POST with JSON body
  def postJson(url: String, body: String): Future[String] = {
    Http().singleRequest(
      HttpRequest(
        method = HttpMethods.POST,
        uri = url,
        entity = HttpEntity(ContentTypes.`application/json`, body)
      )
    ).flatMap(_.entity.toStrict(5.seconds))
     .map(_.data.utf8String)
  }
  
  // Client with connection pool (efficient for many requests)
  def batchRequests(urls: List[String]): Future[List[String]] = {
    Future.traverse(urls) { url =>
      Http().singleRequest(HttpRequest(uri = url))
        .flatMap(_.entity.toStrict(5.seconds))
        .map(_.data.utf8String)
        .recover { case e => s"Error: ${e.getMessage}" }
    }
  }
  
  // Retry client
  def getWithRetry(url: String, maxRetries: Int = 3): Future[String] = {
    def attempt(retries: Int): Future[String] = {
      Http().singleRequest(HttpRequest(uri = url))
        .flatMap { resp =>
          if (resp.status.isSuccess())
            resp.entity.toStrict(5.seconds).map(_.data.utf8String)
          else
            resp.entity.discardBytes()
            Future.failed(new Exception(s"HTTP ${resp.status}"))
        }
        .recoverWith {
          case e if retries > 0 =>
            println(s"  Retry ${maxRetries - retries + 1}/${maxRetries}: ${e.getMessage}")
            akka.pattern.after(500.millis, system.scheduler)(attempt(retries - 1))
        }
    }
    attempt(maxRetries)
  }
  
  println("HTTP Client demo (not running actual requests in this demo)")
  
  // ===== WebSocket Server =====
  
  object ChatServer {
    
    case class Client(name: String, sink: Sink[String, _])
    
    def chatRoomFlow(chatRoom: akka.actor.ActorRef): Flow[Message, Message, Any] = {
      // Simplified WebSocket flow
      Flow[Message].map {
        case TextMessage.Strict(text) => TextMessage(s"Echo: $text")
        case _ => TextMessage("Binary not supported")
      }
    }
  }
  
  val wsRoute =
    path("ws") {
      handleWebSocketMessages(
        Flow[Message].map {
          case TextMessage.Strict(text) =>
            println(s"  WebSocket received: $text")
            TextMessage(s"Server echo: $text")
          case TextMessage.Streamed(stream) =>
            val collapsed: Future[TextMessage] = stream.runFold("")(_ + _).map(TextMessage(_))
            TextMessage(Source.futureSource(collapsed.map(_.textStream)))
          case _ =>
            TextMessage("Unsupported message type")
        }
      )
    } ~
    path("ws-status") {
      get {
        complete("""{"status":"WebSocket server running"}""")
      }
    }
  
  // Server-Sent Events
  val sseRoute = path("events") {
    get {
      complete(
        HttpEntity.Chunked(
          ContentType(MediaTypes.`text/event-stream`, HttpCharsets.`UTF-8`),
          Source.tick(0.seconds, 1.second, ())
            .scan(0)((acc, _) => acc + 1)
            .take(5)
            .map { n =>
              val data = s"data: {\"count\":$n,\"time\":${System.currentTimeMillis()}}\n\n"
              akka.util.ByteString(data)
            }
        )
      )
    }
  }
  
  val fullRoutes = wsRoute ~ sseRoute
  
  val binding = Http().newServerAt("localhost", 8083).bind(fullRoutes)
  println("WebSocket server at ws://localhost:8083/ws")
  println("SSE at http://localhost:8083/events")
  scala.io.StdIn.readLine()
  binding.flatMap(_.unbind()).onComplete(_ => system.terminate())
}

// ===== Complete REST API with Actor backend =====
object CompleteRestApi extends App {
  import akka.actor.typed._
  import akka.actor.typed.scaladsl._
  import akka.http.scaladsl.server.Directives._
  import akka.http.scaladsl.marshallers.sprayjson.SprayJsonSupport._
  import spray.json._
  import DefaultJsonProtocol._
  
  // Actor-backed storage
  object ProductCatalog {
    case class Product(id: String, name: String, price: Double, stock: Int)
    
    sealed trait Command
    case class GetProducts(replyTo: ActorRef[List[Product]]) extends Command
    case class GetProduct(id: String, replyTo: ActorRef[Option[Product]]) extends Command
    case class AddProduct(product: Product, replyTo: ActorRef[Either[String, Product]]) extends Command
    case class UpdateStock(id: String, delta: Int, replyTo: ActorRef[Either[String, Product]]) extends Command
    
    implicit val productFormat: RootJsonFormat[Product] = jsonFormat4(Product)
    
    def apply(): Behavior[Command] = catalog(Map(
      "p1" -> Product("p1", "Laptop", 45000.0, 10),
      "p2" -> Product("p2", "Mouse", 850.0, 50),
      "p3" -> Product("p3", "Keyboard", 1200.0, 30)
    ))
    
    private def catalog(products: Map[String, Product]): Behavior[Command] =
      Behaviors.receiveMessage {
        case GetProducts(replyTo) =>
          replyTo ! products.values.toList
          Behaviors.same
        case GetProduct(id, replyTo) =>
          replyTo ! products.get(id)
          Behaviors.same
        case AddProduct(product, replyTo) =>
          if (products.contains(product.id)) {
            replyTo ! Left(s"Product ${product.id} already exists")
            Behaviors.same
          } else {
            replyTo ! Right(product)
            catalog(products + (product.id -> product))
          }
        case UpdateStock(id, delta, replyTo) =>
          products.get(id) match {
            case None => replyTo ! Left(s"Product $id not found"); Behaviors.same
            case Some(p) =>
              val newStock = p.stock + delta
              if (newStock < 0) {
                replyTo ! Left("Insufficient stock")
                Behaviors.same
              } else {
                val updated = p.copy(stock = newStock)
                replyTo ! Right(updated)
                catalog(products + (id -> updated))
              }
          }
      }
  }
  
  println("Complete REST API patterns shown above")
  println("See full example in the exercises")
}
```

---

## สรุป Part 38

| Feature | Directive / Method | ตัวอย่าง |
|---------|-------------------|---------|
| Path matching | `path("users")` | Match exact path |
| Path segment | `path(Segment)` | Capture `:id` |
| Method | `get`, `post`, `put`, `delete` | HTTP method |
| Parameters | `parameters("p".as[Int])` | Query params |
| Body | `entity(as[T])` | Request body |
| Headers | `headerValueByName("Auth")` | Extract header |
| Authentication | `authenticate` | Custom auth |
| JSON | SprayJson, Circe | Marshal/Unmarshal |
| WebSocket | `handleWebSocketMessages` | Upgrade connection |
| Client | `Http().singleRequest` | Make HTTP call |
| SSE | `HttpEntity.Chunked` | Server-sent events |

---

## แบบฝึกหัด Part 38

**ข้อ 1:** สร้าง REST API สำหรับ Todo app ด้วย CRUD endpoints พร้อม validation

**ข้อ 2:** Implement JWT authentication middleware สำหรับ protected routes

**ข้อ 3:** สร้าง WebSocket chat room ด้วย actor-based broadcast

**ข้อ 4:** Implement streaming endpoint ที่ serve large CSV file ด้วย Akka Streams

**ข้อ 5:** สร้าง API Gateway ที่ proxy requests ไปยัง multiple backend services

---

➡️ ต่อไป: [Part 39 — Reactive Patterns](part-39-reactive-patterns.md)
