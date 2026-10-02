# Part 94: Cats Effect & ZIO — Steps 931-940

## บทนำ: Cats Effect & ZIO

Cats Effect และ ZIO คือ effect systems ที่ให้ functional, composable, resource-safe concurrency สำหรับ Scala production services

```scala
// build.sbt
libraryDependencies ++= Seq(
  // Cats Effect
  "org.typelevel" %% "cats-effect"         % "3.5.4",
  "org.typelevel" %% "cats-effect-testing-scalatest" % "1.5.0" % Test,
  
  // ZIO
  "dev.zio" %% "zio"         % "2.0.21",
  "dev.zio" %% "zio-streams" % "2.0.21",
  "dev.zio" %% "zio-http"    % "3.0.0-RC4"
)
```

---

## Step 931: Cats Effect IO Basics

```scala
// CatsEffectBasics.scala
import cats.effect._
import cats.effect.unsafe.implicits.global
import scala.concurrent.duration._

object CatsEffectBasics {
  
  // ===== IO: description of a computation =====
  
  // Pure value (no side effect)
  val pure: IO[Int] = IO.pure(42)
  
  // Defer side effects
  val sideEffect: IO[Unit] = IO(println("Hello from IO!"))
  
  // Async/blocking
  val asyncIO: IO[String] = IO.async_[String] { callback =>
    // simulate async operation
    new Thread(() => {
      Thread.sleep(100)
      callback(Right("async result"))
    }).start()
  }
  
  // ===== Composition =====
  val program: IO[String] = for {
    n    <- IO.pure(10)
    str  <- IO(s"Number is: $n")
    _    <- IO.println(str)
  } yield str
  
  // ===== Error handling =====
  val risky: IO[Int] = IO(Integer.parseInt("not-a-number"))
  
  val safe: IO[Either[Throwable, Int]] = risky.attempt
  
  val recovered: IO[Int] = risky.handleError(_ => 0)
  
  val raisedError: IO[Nothing] = IO.raiseError(new RuntimeException("oops"))
  
  // ===== Running IO =====
  def main(args: Array[String]): Unit = {
    // Run synchronously (test/main only — never in production async code)
    program.unsafeRunSync()
    
    // Run async
    program.unsafeRunAsync {
      case Right(v) => println(s"Success: $v")
      case Left(e)  => println(s"Error: ${e.getMessage}")
    }
  }
}

// Production: extend IOApp
object OrderServiceApp extends IOApp {
  override def run(args: List[String]): IO[ExitCode] = {
    for {
      _ <- IO.println("Starting Order Service...")
      _ <- startServer()
    } yield ExitCode.Success
  }
  
  def startServer(): IO[Unit] = IO.println("Server started on :8080")
}
```

---

## Step 932: Resource Management

```scala
// ResourceManagement.scala
import cats.effect._
import cats.effect.syntax.resource._
import cats.syntax.all._

// ===== Resource: safe acquire/release =====

object ResourceExamples {
  
  // File resource
  def openFile(path: String): Resource[IO, java.io.BufferedReader] =
    Resource.make(
      IO(new java.io.BufferedReader(new java.io.FileReader(path)))
    )(reader => IO(reader.close()).handleError(_ => ()))
  
  // Database connection
  def dbConnection(url: String): Resource[IO, java.sql.Connection] =
    Resource.make(
      IO(java.sql.DriverManager.getConnection(url))
    )(conn => IO(conn.close()))
  
  // Combine resources (both released on exit)
  def withFileAndDB(path: String, url: String): IO[String] = {
    (openFile(path), dbConnection(url)).tupled.use { case (reader, conn) =>
      for {
        line  <- IO(reader.readLine())
        // use conn...
      } yield s"Read: $line"
    }
    // reader.close() and conn.close() always called
  }
  
  // ===== Finalizers =====
  val withFinalizer: IO[Int] = IO.pure(42)
    .onError(e => IO.println(s"Error: ${e.getMessage}"))
    .guarantee(IO.println("Always runs"))
    .guaranteeCase {
      case Outcome.Succeeded(_) => IO.println("Succeeded")
      case Outcome.Errored(e)   => IO.println(s"Failed: $e")
      case Outcome.Canceled()   => IO.println("Cancelled")
    }
}
```

---

## Step 933: Fibers & Concurrency

```scala
// FibersConcurrency.scala
import cats.effect._
import cats.effect.syntax.all._
import cats.syntax.all._
import scala.concurrent.duration._

object FibersConcurrency {
  
  // ===== Fiber: lightweight green thread =====
  
  def concurrentTasks: IO[Unit] = for {
    fiber1 <- IO.sleep(2.seconds).as("task1").start  // non-blocking start
    fiber2 <- IO.sleep(1.second).as("task2").start
    
    r2 <- fiber2.joinWithNever  // wait for fiber2
    r1 <- fiber1.joinWithNever  // wait for fiber1
    
    _ <- IO.println(s"Results: $r1, $r2")
  } yield ()
  
  // ===== parMapN: run concurrently, combine results =====
  
  def fetchUserAndOrders(userId: Long): IO[(String, List[String])] = (
    IO.sleep(500.millis).as(s"User-$userId"),    // simulate DB call
    IO.sleep(800.millis).as(List("Order-1", "Order-2"))
  ).parMapN((user, orders) => (user, orders))
  // Both run in parallel — total time ~800ms, not 1300ms
  
  // ===== parTraverse: parallel collection operations =====
  
  def fetchAllUsers(ids: List[Long]): IO[List[String]] =
    ids.parTraverse(id => IO.sleep(100.millis).as(s"User-$id"))
  // All fetched in parallel — time = max(individual times) not sum
  
  // ===== race: first wins =====
  
  def withTimeout[A](io: IO[A], timeout: FiniteDuration): IO[Either[Unit, A]] =
    IO.race(IO.sleep(timeout), io)
  
  // ===== Semaphore: limit concurrency =====
  
  def rateLimited: IO[Unit] = for {
    semaphore <- Semaphore[IO](10)  // max 10 concurrent
    _ <- List.fill(100)(
      semaphore.permit.use(_ => IO.sleep(100.millis))
    ).parSequence
    _ <- IO.println("All done")
  } yield ()
  
  // ===== Ref: concurrent mutable state =====
  
  def counter: IO[Unit] = for {
    ref    <- Ref.of[IO, Int](0)
    _      <- List.fill(1000)(ref.update(_ + 1)).parSequence
    count  <- ref.get
    _      <- IO.println(s"Count: $count")  // always 1000
  } yield ()
  
  // ===== Deferred: one-shot promise =====
  
  def producerConsumer: IO[Unit] = for {
    deferred <- Deferred[IO, String]
    _        <- IO.println("Consumer waiting...").start
    consumer <- deferred.get.flatMap(IO.println).start  // blocks until set
    _        <- IO.sleep(500.millis)
    _        <- deferred.complete("Hello from producer!")
    _        <- consumer.join
  } yield ()
}
```

---

## Step 934: Cats Effect HTTP Service

```scala
// CatsEffectHTTP.scala
// Using http4s with Cats Effect

/*
# build.sbt
"org.http4s" %% "http4s-ember-server"  % "0.23.26",
"org.http4s" %% "http4s-ember-client"  % "0.23.26",
"org.http4s" %% "http4s-circe"         % "0.23.26",
"org.http4s" %% "http4s-dsl"           % "0.23.26",
"io.circe"   %% "circe-generic"        % "0.14.7",
*/

import cats.effect._
import org.http4s._
import org.http4s.dsl.io._
import org.http4s.circe._
import io.circe.generic.auto._
import io.circe.syntax._
import org.http4s.implicits._
import org.http4s.ember.server.EmberServerBuilder
import com.comcast.ip4s._

case class OrderResponse(id: Long, status: String, total: Double)
case class CreateOrderReq(customerId: Long, total: Double)

object Http4sServer extends IOApp {
  
  implicit val orderEncoder = jsonEncoderOf[IO, OrderResponse]
  implicit val createDecoder = jsonOf[IO, CreateOrderReq]
  
  val orderRoutes: HttpRoutes[IO] = {
    val dsl = new Http4sDsl[IO] {}; import dsl._
    
    HttpRoutes.of[IO] {
      case GET -> Root / "orders" / LongVar(id) =>
        Ok(OrderResponse(id, "pending", 99.99).asJson)
      
      case req @ POST -> Root / "orders" =>
        for {
          createReq <- req.as[CreateOrderReq]
          orderId   = System.currentTimeMillis()
          resp      <- Created(OrderResponse(orderId, "created", createReq.total).asJson)
        } yield resp
      
      case GET -> Root / "health" =>
        Ok("""{"status":"healthy"}""")
    }
  }
  
  override def run(args: List[String]): IO[ExitCode] = {
    EmberServerBuilder
      .default[IO]
      .withHost(ipv4"0.0.0.0")
      .withPort(port"8080")
      .withHttpApp(orderRoutes.orNotFound)
      .build
      .use(_ => IO.never)
      .as(ExitCode.Success)
  }
}
```

---

## Step 935: ZIO Basics

```scala
// ZIOBasics.scala
import zio._
import zio.console._

object ZIOBasics {
  
  // ZIO[R, E, A]:
  //   R = environment (dependencies)
  //   E = error type
  //   A = success value
  
  // ===== ZIO variants =====
  val effect: ZIO[Any, Nothing, Int] = ZIO.succeed(42)  // can't fail
  val failing: ZIO[Any, String, Nothing] = ZIO.fail("oops")
  val task: Task[Int] = ZIO.attempt(Integer.parseInt("123"))  // = ZIO[Any, Throwable, Int]
  val uio: UIO[Int] = ZIO.succeed(1)                          // = ZIO[Any, Nothing, Int]
  
  // ===== Composition =====
  val program: ZIO[Any, Throwable, String] = for {
    n   <- ZIO.succeed(10)
    str <- ZIO.attempt(s"Number: $n")
    _   <- ZIO.debug(str)
  } yield str
  
  // ===== Error handling =====
  val safe: ZIO[Any, Nothing, Int] = ZIO.attempt(Integer.parseInt("bad"))
    .catchAll(_ => ZIO.succeed(0))
  
  val mapped: ZIO[Any, String, Int] = ZIO.attempt(Integer.parseInt("bad"))
    .mapError(_.getMessage)
  
  val either: ZIO[Any, Nothing, Either[Throwable, Int]] = task.either
  
  // ===== Running ZIO =====
  object Main extends ZIOAppDefault {
    def run = program.debug("Result")
  }
}
```

---

## Step 936: ZIO Environment

```scala
// ZIOEnvironment.scala
import zio._

// ===== ZIO Environment: type-safe DI =====

// Define service interfaces as traits
trait OrderRepo {
  def findById(id: Long): Task[Option[OrderZIO]]
  def save(order: OrderZIO): Task[OrderZIO]
}

trait EmailSvc {
  def send(to: String, subject: String, body: String): Task[Unit]
}

case class OrderZIO(id: Long, customerId: Long, total: Double, status: String)

// ===== Live implementations =====
object OrderRepo {
  val live: ZLayer[Any, Nothing, OrderRepo] = ZLayer.succeed(
    new OrderRepo {
      private val store = scala.collection.concurrent.TrieMap.empty[Long, OrderZIO]
      
      def findById(id: Long): Task[Option[OrderZIO]] = ZIO.succeed(store.get(id))
      def save(order: OrderZIO): Task[OrderZIO] = ZIO.succeed { store(order.id) = order; order }
    }
  )
  
  // Test implementation
  def test(orders: Map[Long, OrderZIO]): ZLayer[Any, Nothing, OrderRepo] = ZLayer.succeed(
    new OrderRepo {
      def findById(id: Long): Task[Option[OrderZIO]] = ZIO.succeed(orders.get(id))
      def save(order: OrderZIO): Task[OrderZIO] = ZIO.succeed(order)
    }
  )
}

object EmailSvc {
  val live: ZLayer[Any, Nothing, EmailSvc] = ZLayer.succeed(
    new EmailSvc {
      def send(to: String, subject: String, body: String): Task[Unit] =
        ZIO.attempt(println(s"EMAIL: to=$to, subject=$subject"))
    }
  )
}

// ===== Business logic using environment =====
object OrderService {
  
  def cancelOrder(id: Long): ZIO[OrderRepo & EmailSvc, Throwable, Either[String, OrderZIO]] = for {
    repo  <- ZIO.service[OrderRepo]
    email <- ZIO.service[EmailSvc]
    result <- repo.findById(id).flatMap {
      case None => ZIO.succeed(Left(s"Order $id not found"))
      case Some(order) =>
        val cancelled = order.copy(status = "cancelled")
        for {
          saved <- repo.save(cancelled)
          _     <- email.send(s"user@example.com", "Order Cancelled", s"Order ${id} cancelled")
        } yield Right(saved)
    }
  } yield result
}

// ===== Composition root =====
object ZIOApp extends ZIOAppDefault {
  
  val appLayer = OrderRepo.live ++ EmailSvc.live
  
  def run = OrderService.cancelOrder(123L)
    .provideLayer(appLayer)
    .debug("Result")
}
```

---

## Step 937: ZIO Streams

```scala
// ZIOStreams.scala
import zio._
import zio.stream._

object ZIOStreams {
  
  // ===== ZStream: lazy, composable stream =====
  
  val numbers: ZStream[Any, Nothing, Int] = ZStream.fromIterable(1 to 1000000)
  
  val pipeline: ZStream[Any, Nothing, Int] = numbers
    .filter(_ % 2 == 0)
    .map(_ * 3)
    .take(100)
  
  // ===== Stream from various sources =====
  
  // From Queue
  def queueStream: ZIO[Any, Nothing, ZStream[Any, Nothing, String]] = for {
    queue <- Queue.unbounded[String]
    stream = ZStream.fromQueue(queue)
  } yield stream
  
  // From file
  def fileStream(path: String): ZStream[Any, Throwable, String] =
    ZStream.fromFileName(path)
      .via(ZPipeline.utf8Decode)
      .via(ZPipeline.splitLines)
  
  // ===== Kafka-like stream processing =====
  val orderEvents: ZStream[Any, Nothing, String] = ZStream.iterate(0)(_ + 1)
    .map(n => s"""{"orderId":$n,"event":"created"}""")
    .throttleShape(100, 1.second)(_.length.toLong)  // rate limit
  
  val processedOrders = orderEvents
    .mapZIOParUnordered(10) { event =>   // parallel processing
      ZIO.succeed(s"Processed: $event").delay(100.millis)
    }
    .grouped(50)                          // batch
    .mapZIO { batch =>
      ZIO.succeed(s"Batch of ${batch.size} processed")
    }
  
  // ===== Sink =====
  val sink: ZSink[Any, Nothing, Int, Nothing, Long] = ZSink.sum[Int].map(_.toLong)
  
  def runStream: ZIO[Any, Nothing, Long] = numbers.run(sink)
  
  // ===== Tap for side effects =====
  val withLogging: ZStream[Any, Nothing, Int] = numbers
    .tap(n => ZIO.when(n % 100000 == 0)(ZIO.debug(s"Processed: $n")))
}
```

---

## Step 938: ZIO HTTP Service

```scala
// ZIOHTTPService.scala
import zio._
import zio.http._
import zio.json._

case class Product(id: Long, name: String, price: Double)
case class CreateProductReq(name: String, price: Double)

object Product {
  implicit val encoder: JsonEncoder[Product] = DeriveJsonEncoder.gen[Product]
  implicit val decoder: JsonDecoder[Product] = DeriveJsonDecoder.gen[Product]
}

object CreateProductReq {
  implicit val decoder: JsonDecoder[CreateProductReq] = DeriveJsonDecoder.gen[CreateProductReq]
}

object ProductRoutes {
  
  val routes: Routes[Any, Response] = Routes(
    Method.GET / "api" / "v1" / "products" / long("id") ->
      handler { (id: Long, _: Request) =>
        ZIO.succeed(Response.json(Product(id, s"Product-$id", 29.99).toJson))
      },
    
    Method.POST / "api" / "v1" / "products" ->
      handler { (req: Request) =>
        for {
          body      <- req.body.asString
          createReq <- ZIO.fromEither(body.fromJson[CreateProductReq])
            .mapError(e => Response.badRequest(e))
          product   = Product(System.currentTimeMillis(), createReq.name, createReq.price)
          resp      = Response.json(product.toJson).status(Status.Created)
        } yield resp
      },
    
    Method.GET / "health" ->
      handler(Response.json("""{"status":"ok"}"""))
  )
}

object ZIOHttpApp extends ZIOAppDefault {
  override def run: ZIO[Any, Throwable, Unit] =
    Server.serve(ProductRoutes.routes)
      .provide(Server.defaultWithPort(8080))
}
```

---

## Step 939: Cats Effect vs ZIO Comparison

```scala
// CatsEffectVsZIO.scala

/*
===== Cats Effect IO vs ZIO =====

                Cats Effect IO         ZIO[R,E,A]
─────────────────────────────────────────────────────
Error type    Throwable only          Typed errors
Dependencies  Constructor injection   Environment R
Fiber         Fiber[IO, A]            Fiber[E, A]
Streams       fs2                     ZStream
HTTP          http4s                  zio-http
DB            doobie                  zio-quill
Testing       IOSpec + cats-effect    ZIOSpec
Performance   Excellent               Excellent
Ecosystem     Very large (typelevel)  Growing (zio ecosystem)

When to use Cats Effect:
  - Already using typelevel stack (doobie, http4s, fs2)
  - Need broad library compatibility
  - Team familiar with FP / Haskell-style

When to use ZIO:
  - Need typed errors throughout
  - ZLayer for complex DI
  - Prefer OOP-friendly API
  - Real-time, high-concurrency workloads

Both are production-ready. Many companies use either successfully.
*/

// ===== Feature parity examples =====

// Cats Effect retry
import retry._
import cats.effect.IO

val withCatsRetry: IO[String] = retryingOnAllErrors[IO, String](
  policy  = RetryPolicies.limitRetries[IO](3) |+| RetryPolicies.exponentialBackoff[IO](100.millis),
  onError = (err, retryDetails) => IO.println(s"Retrying: $err, attempt ${retryDetails.retriesSoFar}")
)(IO.raiseError(new RuntimeException("temp error")))

// ZIO retry
import zio._
val withZIORetry: ZIO[Any, Throwable, String] = ZIO.fail(new RuntimeException("temp error"))
  .retry(Schedule.recurs(3) && Schedule.exponential(100.millis))
  .tapError(e => ZIO.debug(s"Error: $e"))
```

---

## Step 940: Production Patterns

```scala
// ProductionPatterns.scala
import cats.effect._
import cats.effect.std.Queue
import cats.effect.syntax.all._

// ===== Graceful shutdown =====
object GracefulShutdown extends IOApp {
  
  def serverResource(signal: Deferred[IO, Unit]): Resource[IO, Unit] =
    Resource.make(
      IO.println("Server started")
    )(_ =>
      IO.println("Server stopping...") *>
      signal.complete(()).void *>
      IO.sleep(5.seconds) *>
      IO.println("Server stopped")
    )
  
  override def run(args: List[String]): IO[ExitCode] = {
    for {
      signal <- Deferred[IO, Unit]
      _ <- serverResource(signal).use { _ =>
        // Install signal handler
        signal.get  // wait for shutdown signal
      }
    } yield ExitCode.Success
  }
}

// ===== Event-driven architecture =====
class EventBus[F[_]: Async] {
  private val subscribers = scala.collection.concurrent.TrieMap.empty[String, Queue[F, String]]
  
  def subscribe(topic: String): F[Queue[F, String]] = for {
    queue <- Queue.bounded[F, String](1000)
    _     = subscribers.put(topic, queue)
  } yield queue
  
  def publish(topic: String, message: String): F[Unit] =
    subscribers.get(topic) match {
      case Some(queue) => queue.offer(message)
      case None        => Async[F].unit
    }
}

// ===== Circuit breaker with Cats Effect =====
class IOCircuitBreaker[F[_]: Async](maxFailures: Int, resetDuration: FiniteDuration) {
  import cats.effect.std.AtomicCell
  
  sealed trait CBState
  case object Closed   extends CBState
  case object Open     extends CBState
  case object HalfOpen extends CBState
  
  case class CBData(state: CBState, failures: Int, openedAt: Long)
  
  def protect[A](fa: F[A])(fallback: F[A])(implicit clock: Clock[F]): F[A] = {
    // Simplified — use resilience4j or cats-retry in production
    fa.handleErrorWith(_ => fallback)
  }
}
```

---

## สรุป Part 94: Cats Effect & ZIO

| Feature | Cats Effect | ZIO |
|---------|-------------|-----|
| Core type | `IO[A]` | `ZIO[R,E,A]` |
| Error | `Throwable` | Typed `E` |
| DI | Constructor | `ZLayer` |
| Concurrency | `Fiber`, `Ref`, `Deferred` | Same |
| Streams | fs2 | ZStream |
| HTTP | http4s | zio-http |

---

## แบบฝึกหัด Part 94

1. **Cats Effect service**: implement HTTP service ด้วย http4s + Cats Effect ที่ handle concurrent requests, ใช้ `Ref` เก็บ in-memory state, proper error handling

2. **ZIO layers**: implement `OrderService` ด้วย ZIO ที่ require `ZLayer[OrderRepo & EmailSvc & Logger]`, test ด้วย test layers

3. **Concurrent counter**: implement ด้วยทั้ง `Ref[IO, Int]` และ `ZIO Ref` ที่ safe under 1000 concurrent increments

4. **Resource management**: implement database connection pool ด้วย `cats.effect.Resource` ที่ guarantee connections ถูก returned เสมอ แม้จะ error

5. **ZStream pipeline**: implement Kafka consumer simulation ด้วย ZStream ที่ read events, process in parallel (10 fibers), batch write results to "database"

---

## ไปต่อ: Part 95 — Security Hardening
[→ Part 95: Security Hardening](./part-95-security-hardening.md)
