# Part 31: Futures and Promises

## Steps 301-310: Future, Promise, ExecutionContext, Async Patterns, Error Handling

---

## Step 301: Future พื้นฐาน

```scala
import scala.concurrent._
import scala.concurrent.duration._
import ExecutionContext.Implicits.global

object FutureBasics extends App {
  
  // Future = computation that will complete later
  val f1: Future[Int] = Future {
    Thread.sleep(100)  // Simulate async work
    42
  }
  
  val f2: Future[String] = Future {
    Thread.sleep(50)
    "Hello from Future"
  }
  
  println("=== Future Basics ===")
  
  // Non-blocking: callback style
  f1.foreach(n => println(s"  f1 result: $n"))
  f2.foreach(s => println(s"  f2 result: $s"))
  
  // Blocking (avoid in production, OK for tests/demos)
  val result1 = Await.result(f1, 5.seconds)
  val result2 = Await.result(f2, 5.seconds)
  
  println(s"  f1 blocked: $result1")
  println(s"  f2 blocked: $result2")
  
  // Future.successful / Future.failed — already completed
  val done: Future[Int] = Future.successful(100)
  val failed: Future[Int] = Future.failed(new RuntimeException("Oops"))
  
  println(s"\n  done.value: ${done.value}")
  println(s"  failed.value: ${failed.value}")
  
  // Map and flatMap
  val doubled: Future[Int] = f1.map(_ * 2)
  val asString: Future[String] = f1.map(n => s"The answer is $n")
  
  println(s"\n  doubled: ${Await.result(doubled, 2.seconds)}")
  println(s"  asString: ${Await.result(asString, 2.seconds)}")
  
  // For comprehension
  val computation: Future[String] = for {
    n <- Future(10)
    m <- Future(n * 5)
    s <- Future(s"$n x 5 = $m")
  } yield s
  
  println(s"  for-comp: ${Await.result(computation, 2.seconds)}")
  
  Thread.sleep(500)  // Let callbacks complete
}
```

---

## Step 302: Future Chaining and Combinators

```scala
import scala.concurrent._
import scala.concurrent.duration._
import ExecutionContext.Implicits.global

// Simulated async services
object AsyncServices {
  def fetchUser(id: String): Future[Map[String, Any]] = Future {
    Thread.sleep(100)
    id match {
      case "u1" => Map("id" -> "u1", "name" -> "Alice", "tier" -> "premium")
      case "u2" => Map("id" -> "u2", "name" -> "Bob", "tier" -> "standard")
      case _    => throw new Exception(s"User not found: $id")
    }
  }
  
  def fetchOrders(userId: String): Future[List[Map[String, Any]]] = Future {
    Thread.sleep(150)
    userId match {
      case "u1" => List(
        Map("id" -> "o1", "amount" -> 1500.0),
        Map("id" -> "o2", "amount" -> 850.0)
      )
      case "u2" => List(Map("id" -> "o3", "amount" -> 300.0))
      case _    => List.empty
    }
  }
  
  def fetchDiscount(tier: String): Future[Double] = Future {
    Thread.sleep(50)
    tier match {
      case "premium"  => 0.15
      case "standard" => 0.05
      case _          => 0.0
    }
  }
  
  def sendEmail(to: String, message: String): Future[Boolean] = Future {
    Thread.sleep(80)
    println(s"  [EMAIL] To $to: $message")
    true
  }
}

object FutureCombinators extends App {
  import AsyncServices._
  
  println("=== Future Chaining ===")
  
  // Sequential chain — each waits for previous
  def processUserSeq(userId: String): Future[String] = for {
    user    <- fetchUser(userId)
    orders  <- fetchOrders(userId)
    discount <- fetchDiscount(user("tier").toString)
    total    = orders.map(_("amount").asInstanceOf[Double]).sum
    discounted = total * (1 - discount)
  } yield s"User ${user("name")}: ${orders.size} orders, total $$${discounted}"
  
  val seqStart = System.currentTimeMillis()
  val seqResult = Await.result(processUserSeq("u1"), 10.seconds)
  println(s"Sequential: $seqResult (${System.currentTimeMillis() - seqStart}ms)")
  
  println("\n=== Parallel Futures ===")
  
  // Parallel — fetch user and orders simultaneously
  def processUserParallel(userId: String): Future[String] = {
    // Start both futures in parallel (BEFORE for comprehension)
    val userF    = fetchUser(userId)
    val ordersF  = fetchOrders(userId)
    
    for {
      user     <- userF      // Wait for already-running future
      orders   <- ordersF    // Already running!
      discount <- fetchDiscount(user("tier").toString)  // Sequential here
      total     = orders.map(_("amount").asInstanceOf[Double]).sum
      discounted = total * (1 - discount)
    } yield s"User ${user("name")}: ${orders.size} orders, total $$${discounted}"
  }
  
  val parStart = System.currentTimeMillis()
  val parResult = Await.result(processUserParallel("u1"), 10.seconds)
  println(s"Parallel: $parResult (${System.currentTimeMillis() - parStart}ms)")
  
  println("\n=== Future.sequence ===")
  
  // Run multiple futures and wait for all
  val userIds = List("u1", "u2")
  val userFutures: List[Future[Map[String, Any]]] = userIds.map(fetchUser)
  val allUsers: Future[List[Map[String, Any]]] = Future.sequence(userFutures)
  
  val users = Await.result(allUsers, 10.seconds)
  users.foreach(u => println(s"  ${u("name")} (${u("tier")})"))
  
  println("\n=== Future.traverse ===")
  
  // Traverse — map + sequence
  val orderResults: Future[List[List[Map[String, Any]]]] = 
    Future.traverse(userIds)(fetchOrders)
  
  val allOrders = Await.result(orderResults, 10.seconds)
  allOrders.zipWithIndex.foreach { case (orders, i) =>
    println(s"  User ${userIds(i)}: ${orders.size} orders")
  }
  
  println("\n=== Future.firstCompletedOf ===")
  
  // Race — first to complete wins
  val fastest = Future.firstCompletedOf(List(
    Future { Thread.sleep(300); "slow" },
    Future { Thread.sleep(50); "fast" },
    Future { Thread.sleep(200); "medium" }
  ))
  println(s"First: ${Await.result(fastest, 2.seconds)}")
  
  println("\n=== zip ===")
  
  // Zip two futures
  val (u, o) = Await.result(fetchUser("u2").zip(fetchOrders("u2")), 5.seconds)
  println(s"Zipped: user=${u("name")}, orders=${o.size}")
  
  Thread.sleep(500)
}
```

---

## Step 303: Future Error Handling

```scala
import scala.concurrent._
import scala.concurrent.duration._
import scala.util.{Success, Failure}
import ExecutionContext.Implicits.global

object FutureErrorHandling extends App {
  
  def riskyOperation(n: Int): Future[Int] = Future {
    if (n < 0) throw new IllegalArgumentException(s"Negative: $n")
    if (n == 0) throw new ArithmeticException("Division by zero")
    100 / n
  }
  
  println("=== Future Error Handling ===")
  
  // recover — handle specific errors
  val recovered = riskyOperation(0).recover {
    case _: ArithmeticException => -1
    case _: IllegalArgumentException => -2
  }
  println(s"recovered: ${Await.result(recovered, 2.seconds)}")
  
  // recoverWith — return new Future from error
  val recovered2 = riskyOperation(-5).recoverWith {
    case e: IllegalArgumentException =>
      Future.successful(0)  // Fallback
  }
  println(s"recoverWith: ${Await.result(recovered2, 2.seconds)}")
  
  // fallbackTo — try second future if first fails
  val withFallback = riskyOperation(-1).fallbackTo(Future.successful(999))
  println(s"fallbackTo: ${Await.result(withFallback, 2.seconds)}")
  
  // transform — handle both cases
  val transformed = riskyOperation(5).transform {
    case Success(n)  => Success(s"OK: $n")
    case Failure(e)  => Success(s"Error: ${e.getMessage}")
  }
  println(s"transform: ${Await.result(transformed, 2.seconds)}")
  
  // transformWith
  val transformed2 = riskyOperation(0).transformWith {
    case Success(n)  => Future.successful(n * 2)
    case Failure(_)  => Future.successful(-99)
  }
  println(s"transformWith: ${Await.result(transformed2, 2.seconds)}")
  
  // onComplete callback
  riskyOperation(10).onComplete {
    case Success(n)  => println(s"  onComplete Success: $n")
    case Failure(e)  => println(s"  onComplete Failure: ${e.getMessage}")
  }
  
  riskyOperation(-3).onComplete {
    case Success(n)  => println(s"  onComplete Success: $n")
    case Failure(e)  => println(s"  onComplete Failure: ${e.getMessage}")
  }
  
  // Retry pattern
  def retry[A](n: Int, delay: Duration = 100.milliseconds)(f: => Future[A])
              (implicit ec: ExecutionContext): Future[A] = {
    f.recoverWith {
      case e if n > 1 =>
        println(s"  Retrying... ($n attempts left)")
        akka.pattern.after(delay, ???)(retry(n - 1, delay)(f))
    }
  }
  
  // Simple retry without akka
  def retrySimple[A](n: Int)(f: => Future[A])(implicit ec: ExecutionContext): Future[A] = {
    var attempt = 0
    def tryOnce(): Future[A] = {
      attempt += 1
      f.recoverWith {
        case e if attempt < n =>
          println(s"  Attempt $attempt failed: ${e.getMessage}, retrying...")
          tryOnce()
      }
    }
    tryOnce()
  }
  
  var callCount = 0
  def flakyService(): Future[String] = Future {
    callCount += 1
    if (callCount < 3) throw new Exception("Service unavailable")
    "Success!"
  }
  
  println("\n=== Retry Pattern ===")
  val retryResult = retrySimple(5)(flakyService())
  println(s"Result: ${Await.result(retryResult, 5.seconds)}")
  
  Thread.sleep(200)
}
```

---

## Step 304: Promise — Completing Futures Manually

```scala
import scala.concurrent._
import scala.concurrent.duration._
import ExecutionContext.Implicits.global

object PromiseDemo extends App {
  
  // Promise = writable Future
  // Use when: bridging callback APIs, coordinating between threads
  
  // Basic Promise
  val p1 = Promise[Int]()
  val f1: Future[Int] = p1.future
  
  // Complete in another thread
  Future {
    Thread.sleep(100)
    println("  Completing promise...")
    p1.success(42)
  }
  
  println(s"Promise result: ${Await.result(f1, 2.seconds)}")
  
  // Promise.trySuccess — safe, won't throw if already completed
  val p2 = Promise[String]()
  p2.trySuccess("first")
  p2.trySuccess("second")  // Ignored, already completed
  println(s"trySuccess: ${Await.result(p2.future, 1.second)}")
  
  // Converting callback API to Future
  def callbackApi(input: String, callback: Either[Exception, String] => Unit): Unit = {
    new Thread(() => {
      Thread.sleep(100)
      if (input.isEmpty) callback(Left(new Exception("Empty input")))
      else callback(Right(s"Processed: $input"))
    }).start()
  }
  
  def asyncVersion(input: String): Future[String] = {
    val promise = Promise[String]()
    callbackApi(input, {
      case Right(result) => promise.success(result)
      case Left(error)   => promise.failure(error)
    })
    promise.future
  }
  
  println(s"\nCallback to Future: ${Await.result(asyncVersion("hello"), 2.seconds)}")
  
  // Promise for coordination
  def producerConsumer(): Future[List[Int]] = {
    val ready = Promise[Unit]()
    var buffer: List[Int] = List.empty
    
    // Producer
    Future {
      (1 to 5).foreach { i =>
        Thread.sleep(30)
        buffer = buffer :+ i
        println(s"  Produced: $i")
      }
      ready.success(())
    }
    
    // Consumer — waits for signal
    ready.future.map { _ =>
      println(s"  Consumer received: $buffer")
      buffer
    }
  }
  
  println("\n=== Producer/Consumer ===")
  println(s"Buffer: ${Await.result(producerConsumer(), 5.seconds)}")
  
  // Implement timeout with Promise
  def withTimeout[A](future: Future[A], timeout: Duration): Future[A] = {
    val p = Promise[A]()
    
    future.onComplete(p.tryComplete)
    
    val timer = new java.util.Timer()
    timer.schedule(new java.util.TimerTask {
      def run(): Unit = {
        p.tryFailure(new TimeoutException(s"Timed out after $timeout"))
        timer.cancel()
      }
    }, timeout.toMillis)
    
    p.future
  }
  
  println("\n=== Timeout with Promise ===")
  
  val slowFuture = Future { Thread.sleep(2000); "done" }
  val fastFuture = Future { Thread.sleep(100); "fast" }
  
  withTimeout(fastFuture, 500.millis).onComplete {
    case scala.util.Success(v) => println(s"  fast: $v")
    case scala.util.Failure(e) => println(s"  fast error: ${e.getMessage}")
  }
  
  withTimeout(slowFuture, 500.millis).onComplete {
    case scala.util.Success(v) => println(s"  slow: $v")
    case scala.util.Failure(e) => println(s"  slow error: ${e.getMessage}")
  }
  
  Thread.sleep(1000)
}
```

---

## Step 305-310: Real-World Async Patterns

```scala
import scala.concurrent._
import scala.concurrent.duration._
import ExecutionContext.Implicits.global

object AsyncPatterns extends App {
  
  // Fan-out / Fan-in pattern
  def batchFetch[A, B](items: List[A], maxConcurrency: Int)
                      (fetch: A => Future[B]): Future[List[B]] = {
    items.grouped(maxConcurrency)
      .foldLeft(Future.successful(List.empty[B])) { (acc, batch) =>
        for {
          existing <- acc
          results  <- Future.sequence(batch.map(fetch))
        } yield existing ++ results
      }
  }
  
  // Simulate slow fetch
  def fetchItem(id: Int): Future[String] = Future {
    Thread.sleep(100)
    s"item-$id"
  }
  
  println("=== Batch Fetch ===")
  val startTime = System.currentTimeMillis()
  val results = Await.result(
    batchFetch((1 to 10).toList, 3)(fetchItem),
    30.seconds
  )
  println(s"Fetched ${results.size} items in ${System.currentTimeMillis() - startTime}ms")
  
  // Async cache
  class AsyncCache[K, V](ttl: Duration) {
    private val cache = new java.util.concurrent.ConcurrentHashMap[K, (V, Long)]()
    
    def get(key: K)(fetch: K => Future[V]): Future[V] = {
      val now = System.currentTimeMillis()
      Option(cache.get(key)) match {
        case Some((value, expiresAt)) if expiresAt > now =>
          Future.successful(value)
        case _ =>
          fetch(key).map { value =>
            cache.put(key, (value, now + ttl.toMillis))
            value
          }
      }
    }
  }
  
  val cache = new AsyncCache[String, String](10.seconds)
  
  var fetchCount = 0
  def expensiveFetch(key: String): Future[String] = Future {
    fetchCount += 1
    Thread.sleep(100)
    s"value-for-$key (fetch #$fetchCount)"
  }
  
  println("\n=== Async Cache ===")
  val r1 = Await.result(cache.get("key1")(expensiveFetch), 2.seconds)
  val r2 = Await.result(cache.get("key1")(expensiveFetch), 2.seconds)  // From cache
  val r3 = Await.result(cache.get("key2")(expensiveFetch), 2.seconds)  // New fetch
  println(s"r1: $r1")
  println(s"r2 (cached): $r2")
  println(s"r3: $r3")
  println(s"Total fetches: $fetchCount")
  
  // Circuit Breaker pattern
  class CircuitBreaker(maxFailures: Int, resetTimeout: Duration) {
    private var failures = 0
    private var lastFailureTime = 0L
    private var state: String = "closed"
    
    def execute[A](f: => Future[A]): Future[A] = {
      val now = System.currentTimeMillis()
      
      state match {
        case "open" if now - lastFailureTime > resetTimeout.toMillis =>
          state = "half-open"
          println(s"  [CB] Trying half-open...")
          attemptExecution(f)
        case "open" =>
          Future.failed(new Exception("Circuit breaker is OPEN"))
        case _ =>
          attemptExecution(f)
      }
    }
    
    private def attemptExecution[A](f: => Future[A]): Future[A] = {
      f.andThen {
        case scala.util.Success(_) =>
          failures = 0
          if (state == "half-open") {
            println("  [CB] Circuit CLOSED")
            state = "closed"
          }
        case scala.util.Failure(_) =>
          failures += 1
          lastFailureTime = System.currentTimeMillis()
          if (failures >= maxFailures) {
            println(s"  [CB] Circuit OPENED after $failures failures!")
            state = "open"
          }
      }
    }
  }
  
  println("\n=== Circuit Breaker ===")
  val cb = new CircuitBreaker(3, 2.seconds)
  
  var serviceFailures = 0
  def unreliableService(): Future[String] = Future {
    serviceFailures += 1
    if (serviceFailures <= 4) throw new Exception(s"Service error #$serviceFailures")
    "Service recovered"
  }
  
  (1 to 6).foreach { i =>
    cb.execute(unreliableService()).onComplete {
      case scala.util.Success(v) => println(s"  Call $i: SUCCESS - $v")
      case scala.util.Failure(e) => println(s"  Call $i: FAILED - ${e.getMessage}")
    }
    Thread.sleep(100)
  }
  
  Thread.sleep(1000)
}
```

---

## สรุป Part 31

| Concept | API | ใช้เมื่อ |
|---------|-----|---------|
| Future.apply | `Future { ... }` | Async computation |
| Future.successful | `Future.successful(v)` | Wrap value |
| Future.failed | `Future.failed(e)` | Wrap error |
| Future.sequence | `Future.sequence(list)` | Wait for all |
| Future.traverse | `Future.traverse(list)(f)` | map + sequence |
| Future.firstCompletedOf | `Future.firstCompletedOf(list)` | Race futures |
| map/flatMap | `.map(f)` | Transform result |
| recover | `.recover { case e => }` | Handle errors |
| recoverWith | `.recoverWith { case e => Future }` | Async recovery |
| zip | `.zip(other)` | Combine two |
| Promise | `Promise[A]()` | Manual completion |

---

## แบบฝึกหัด Part 31

**ข้อ 1:** Implement `retry` function with exponential backoff และ max delay

**ข้อ 2:** สร้าง `BulkheadExecutor` ที่ limit concurrent executions ด้วย Semaphore

**ข้อ 3:** Implement saga pattern สำหรับ distributed transactions ด้วย Future

**ข้อ 4:** สร้าง `RateLimiter` ที่ limit requests ต่อ second ด้วย Token Bucket algorithm

**ข้อ 5:** Implement `AsyncPipeline[A, B, C]` ที่ chain async transformations แบบ typesafe

---

➡️ ต่อไป: [Part 32 — Execution Contexts](part-32-execution-contexts.md)
