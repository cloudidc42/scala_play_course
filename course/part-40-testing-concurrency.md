# Part 40: Testing Concurrency

## Steps 391-400: ScalaTest, Property-based Testing, Actor Testing, Async Tests, Load Testing

---

## Step 391: ScalaTest พื้นฐาน

```scala
// build.sbt:
// libraryDependencies += "org.scalatest" %% "scalatest" % "3.2.15" % Test
// libraryDependencies += "org.scalatestplus" %% "scalacheck-1-17" % "3.2.15.0" % Test

import org.scalatest.flatspec.AnyFlatSpec
import org.scalatest.matchers.should.Matchers
import org.scalatest.funsuite.AnyFunSuite
import org.scalatest.concurrent.{ScalaFutures, Eventually}
import org.scalatest.time.{Millis, Seconds, Span}
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

// ===== FlatSpec style — BDD =====
class CalculatorSpec extends AnyFlatSpec with Matchers {
  
  class Calculator {
    def add(a: Int, b: Int): Int = a + b
    def divide(a: Double, b: Double): Double = {
      require(b != 0, "Cannot divide by zero")
      a / b
    }
    def factorial(n: Int): Long = {
      require(n >= 0, "Negative input")
      if (n == 0) 1L else n * factorial(n - 1)
    }
  }
  
  val calc = new Calculator
  
  "A Calculator" should "add two numbers" in {
    calc.add(2, 3) shouldBe 5
    calc.add(-1, 1) shouldBe 0
    calc.add(0, 0) shouldBe 0
  }
  
  it should "divide two numbers" in {
    calc.divide(10.0, 2.0) shouldBe 5.0
    calc.divide(1.0, 3.0) shouldBe 0.333 +- 0.001  // Tolerance
  }
  
  it should "throw on division by zero" in {
    an [IllegalArgumentException] should be thrownBy {
      calc.divide(10.0, 0.0)
    }
    
    the [IllegalArgumentException] thrownBy {
      calc.divide(5.0, 0.0)
    } should have message "requirement failed: Cannot divide by zero"
  }
  
  it should "compute factorial" in {
    calc.factorial(0) shouldBe 1L
    calc.factorial(5) shouldBe 120L
    calc.factorial(10) shouldBe 3628800L
  }
  
  "A Calculator factorial" should "work for large inputs" in {
    calc.factorial(20) should be > 0L
  }
}

// ===== FunSuite style =====
class StringOpsSpec extends AnyFunSuite with Matchers {
  
  test("reverse a string") {
    "hello".reverse shouldBe "olleh"
    "".reverse shouldBe ""
    "a".reverse shouldBe "a"
  }
  
  test("count characters") {
    "hello".count(_ == 'l') shouldBe 2
    "hello world".count(_.isSpace) shouldBe 1
  }
  
  test("split and join") {
    val words = "hello world scala".split(" ").toList
    words.length shouldBe 3
    words.mkString(", ") shouldBe "hello, world, scala"
  }
  
  test("options") {
    val opt: Option[Int] = Some(42)
    opt shouldBe defined
    opt.get shouldBe 42
    
    val none: Option[Int] = None
    none shouldBe empty
    none.getOrElse(0) shouldBe 0
  }
  
  test("either") {
    val right: Either[String, Int] = Right(42)
    right.isRight shouldBe true
    right.getOrElse(0) shouldBe 42
    
    val left: Either[String, Int] = Left("error")
    left.isLeft shouldBe true
    left.left.get shouldBe "error"
  }
}
```

---

## Step 392: Async Testing

```scala
import org.scalatest.flatspec.AsyncFlatSpec
import org.scalatest.matchers.should.Matchers
import org.scalatest.concurrent.{ScalaFutures, Eventually, IntegrationPatience}
import scala.concurrent.Future
import scala.concurrent.duration._

// AsyncFlatSpec — native async test support
class AsyncServiceSpec extends AsyncFlatSpec with Matchers {
  
  class UserService {
    def findUser(id: String): Future[Option[String]] = Future {
      Thread.sleep(50)  // Simulate DB call
      id match {
        case "u1" => Some("Alice")
        case "u2" => Some("Bob")
        case _    => None
      }
    }
    
    def createUser(name: String): Future[String] = Future {
      Thread.sleep(30)
      s"user-${name.toLowerCase}"
    }
    
    def deleteUser(id: String): Future[Boolean] = Future {
      Thread.sleep(20)
      id.startsWith("u")
    }
  }
  
  val service = new UserService
  
  "UserService" should "find existing user" in {
    service.findUser("u1").map { result =>
      result shouldBe defined
      result.get shouldBe "Alice"
    }
  }
  
  it should "return None for missing user" in {
    service.findUser("nonexistent").map { result =>
      result shouldBe empty
    }
  }
  
  it should "create and retrieve user" in {
    for {
      created <- service.createUser("Charlie")
      found   <- service.findUser(created)
    } yield {
      created shouldBe "user-charlie"
      // found would depend on actual implementation
      assert(true)  // Placeholder
    }
  }
  
  it should "handle multiple parallel requests" in {
    val futures = List("u1", "u2", "u3").map(service.findUser)
    Future.sequence(futures).map { results =>
      results.length shouldBe 3
      results.take(2).forall(_.isDefined) shouldBe true
      results(2) shouldBe empty
    }
  }
}

// ScalaFutures — synchronous future assertions
class SyncFutureSpec extends AnyFlatSpec with Matchers with ScalaFutures {
  
  // Configure timeout
  implicit val defaultPatience = PatienceConfig(
    timeout = scaled(Span(5, Seconds)),
    interval = scaled(Span(100, Millis))
  )
  
  "Future results" should "be checkable synchronously" in {
    val f = Future { Thread.sleep(100); 42 }
    f.futureValue shouldBe 42  // Blocks until done
  }
  
  it should "fail properly" in {
    val f = Future.failed[Int](new RuntimeException("oops"))
    a [RuntimeException] should be thrownBy f.futureValue
  }
  
  it should "handle sequence" in {
    val futures = (1 to 5).map(n => Future(n * n)).toList
    val results = Future.sequence(futures).futureValue
    results shouldBe List(1, 4, 9, 16, 25)
  }
}
```

---

## Step 393: Property-Based Testing

```scala
import org.scalatest.flatspec.AnyFlatSpec
import org.scalatest.matchers.should.Matchers
import org.scalatestplus.scalacheck.ScalaCheckPropertyChecks
import org.scalacheck.{Gen, Arbitrary, Prop}
import org.scalacheck.Prop.forAll

class PropertyBasedTests extends AnyFlatSpec 
  with Matchers 
  with ScalaCheckPropertyChecks {
  
  // ===== Custom Generators =====
  
  val positiveInts: Gen[Int] = Gen.posNum[Int]
  val nonEmptyStrings: Gen[String] = Gen.nonEmptyListOf(Gen.alphaChar).map(_.mkString)
  
  val emailGen: Gen[String] = for {
    user   <- Gen.nonEmptyListOf(Gen.alphaNumChar).map(_.mkString)
    domain <- Gen.nonEmptyListOf(Gen.alphaNumChar).map(_.mkString)
    tld    <- Gen.oneOf("com", "org", "net", "co.th")
  } yield s"$user@$domain.$tld"
  
  case class Money(amount: Double, currency: String)
  
  val moneyGen: Gen[Money] = for {
    amount   <- Gen.choose(0.01, 1000000.0)
    currency <- Gen.oneOf("THB", "USD", "EUR", "GBP")
  } yield Money(amount, currency)
  
  // ===== Properties =====
  
  // Reverse of reverse is identity
  "Reverse" should "be its own inverse" in {
    forAll { (s: String) =>
      s.reverse.reverse shouldBe s
    }
  }
  
  "Sort" should "produce correct results" in {
    forAll { (list: List[Int]) =>
      val sorted = list.sorted
      // Property: sorted list has same elements
      sorted.toSet shouldBe list.toSet
      // Property: sorted list is non-decreasing
      sorted.zip(sorted.tail).forall { case (a, b) => a <= b } shouldBe true
    }
  }
  
  "String split-join" should "round-trip" in {
    forAll(Gen.nonEmptyListOf(nonEmptyStrings)) { words =>
      val joined = words.mkString(",")
      val split = joined.split(",").toList
      split shouldBe words
    }
  }
  
  "Map operations" should "maintain size" in {
    forAll { (map: Map[String, Int], key: String, value: Int) =>
      val newMap = map + (key -> value)
      newMap.size should be >= map.size
      newMap.size should be <= map.size + 1
    }
  }
  
  // Money arithmetic properties
  "Money addition" should "be commutative" in {
    forAll(moneyGen, moneyGen) { (m1, m2) =>
      if (m1.currency == m2.currency) {
        val sum1 = Money(m1.amount + m2.amount, m1.currency)
        val sum2 = Money(m2.amount + m1.amount, m2.currency)
        sum1.amount shouldBe sum2.amount +- 0.0001
      } else {
        // Cross-currency addition should fail
        assert(true)
      }
    }
  }
  
  "List operations" should "obey length laws" in {
    forAll { (list: List[Int], n: Int) =>
      whenever(n >= 0) {
        list.take(n).length shouldBe Math.min(n, list.length)
        list.drop(n).length shouldBe Math.max(0, list.length - n)
        (list.take(n) ++ list.drop(n)) shouldBe list
      }
    }
  }
  
  // Stateful property testing
  "Stack" should "follow LIFO order" in {
    forAll(Gen.listOf(Gen.posNum[Int])) { items =>
      val stack = scala.collection.mutable.Stack[Int]()
      items.foreach(stack.push)
      val popped = (1 to items.length).map(_ => stack.pop()).toList
      popped shouldBe items.reverse
    }
  }
}
```

---

## Step 394: Actor Testing

```scala
import akka.actor.testkit.typed.scaladsl.{ActorTestKit, TestProbe}
import akka.actor.typed._
import akka.actor.typed.scaladsl._
import org.scalatest.BeforeAndAfterAll
import org.scalatest.flatspec.AnyFlatSpec
import org.scalatest.matchers.should.Matchers
import scala.concurrent.duration._

class ActorTestSpec extends AnyFlatSpec 
  with Matchers 
  with BeforeAndAfterAll {
  
  val testKit = ActorTestKit()
  
  override def afterAll(): Unit = testKit.shutdownTestKit()
  
  // ===== Counter Actor Test =====
  object CounterActorForTest {
    sealed trait Command
    case object Increment extends Command
    case object Decrement extends Command
    case class GetValue(replyTo: ActorRef[Int]) extends Command
    case class SetValue(value: Int) extends Command
    
    def apply(initial: Int = 0): Behavior[Command] = counter(initial)
    
    private def counter(n: Int): Behavior[Command] = Behaviors.receiveMessage {
      case Increment          => counter(n + 1)
      case Decrement          => counter(n - 1)
      case SetValue(v)        => counter(v)
      case GetValue(replyTo)  => replyTo ! n; Behaviors.same
    }
  }
  
  import CounterActorForTest._
  
  "CounterActor" should "start at 0" in {
    val counter = testKit.spawn(CounterActorForTest(), "counter-1")
    val probe = testKit.createTestProbe[Int]()
    
    counter ! GetValue(probe.ref)
    probe.expectMessage(0)
  }
  
  it should "increment correctly" in {
    val counter = testKit.spawn(CounterActorForTest(), "counter-2")
    val probe = testKit.createTestProbe[Int]()
    
    counter ! Increment
    counter ! Increment
    counter ! Increment
    counter ! GetValue(probe.ref)
    probe.expectMessage(3)
  }
  
  it should "handle set and get" in {
    val counter = testKit.spawn(CounterActorForTest(), "counter-3")
    val probe = testKit.createTestProbe[Int]()
    
    counter ! SetValue(100)
    counter ! Increment
    counter ! GetValue(probe.ref)
    probe.expectMessage(101)
  }
  
  it should "not receive unexpected messages" in {
    val probe = testKit.createTestProbe[Int]()
    probe.expectNoMessage(100.millis)
  }
  
  // ===== Ping-Pong Actor Test =====
  object PingPong {
    case class Ping(replyTo: ActorRef[Pong.type])
    case object Pong
    
    def pingBehavior: Behavior[Ping] = Behaviors.receiveMessage {
      case Ping(replyTo) =>
        replyTo ! Pong
        Behaviors.same
    }
  }
  
  "PingPong" should "respond to ping with pong" in {
    val ping = testKit.spawn(PingPong.pingBehavior, "ping")
    val probe = testKit.createTestProbe[PingPong.Pong.type]()
    
    ping ! PingPong.Ping(probe.ref)
    probe.expectMessage(PingPong.Pong)
    
    // Multiple rounds
    (1 to 5).foreach { _ =>
      ping ! PingPong.Ping(probe.ref)
      probe.expectMessage(PingPong.Pong)
    }
  }
  
  // ===== Stateful Actor Test =====
  object InventoryActor {
    sealed trait Command
    case class Reserve(itemId: String, qty: Int, replyTo: ActorRef[ReserveResult]) extends Command
    case class Release(itemId: String, qty: Int) extends Command
    case class GetStock(itemId: String, replyTo: ActorRef[Int]) extends Command
    
    sealed trait ReserveResult
    case class Reserved(itemId: String, qty: Int) extends ReserveResult
    case class InsufficientStock(itemId: String, available: Int, requested: Int) extends ReserveResult
    
    def apply(stock: Map[String, Int]): Behavior[Command] = managing(stock)
    
    private def managing(stock: Map[String, Int]): Behavior[Command] = 
      Behaviors.receiveMessage {
        case Reserve(itemId, qty, replyTo) =>
          val available = stock.getOrElse(itemId, 0)
          if (available >= qty) {
            replyTo ! Reserved(itemId, qty)
            managing(stock + (itemId -> (available - qty)))
          } else {
            replyTo ! InsufficientStock(itemId, available, qty)
            Behaviors.same
          }
        case Release(itemId, qty) =>
          val current = stock.getOrElse(itemId, 0)
          managing(stock + (itemId -> (current + qty)))
        case GetStock(itemId, replyTo) =>
          replyTo ! stock.getOrElse(itemId, 0)
          Behaviors.same
      }
  }
  
  "InventoryActor" should "track stock correctly" in {
    val inventory = testKit.spawn(
      InventoryActor(Map("ITEM-A" -> 100, "ITEM-B" -> 50)),
      "inventory"
    )
    
    val resultProbe = testKit.createTestProbe[InventoryActor.ReserveResult]()
    val stockProbe = testKit.createTestProbe[Int]()
    
    inventory ! InventoryActor.Reserve("ITEM-A", 30, resultProbe.ref)
    resultProbe.expectMessage(InventoryActor.Reserved("ITEM-A", 30))
    
    inventory ! InventoryActor.GetStock("ITEM-A", stockProbe.ref)
    stockProbe.expectMessage(70)
    
    inventory ! InventoryActor.Reserve("ITEM-A", 80, resultProbe.ref)
    resultProbe.expectMessage(InventoryActor.InsufficientStock("ITEM-A", 70, 80))
    
    inventory ! InventoryActor.Release("ITEM-A", 30)
    inventory ! InventoryActor.GetStock("ITEM-A", stockProbe.ref)
    stockProbe.expectMessage(100)
  }
}
```

---

## Step 395-400: Concurrent Testing and Load Testing

```scala
import org.scalatest.flatspec.AnyFlatSpec
import org.scalatest.matchers.should.Matchers
import org.scalatest.concurrent.Eventually
import org.scalatest.time.{Millis, Seconds, Span}
import scala.concurrent._
import scala.concurrent.duration._
import ExecutionContext.Implicits.global
import java.util.concurrent.atomic._

class ConcurrencyTests extends AnyFlatSpec with Matchers with Eventually {
  
  implicit val patience = PatienceConfig(
    timeout = scaled(Span(10, Seconds)),
    interval = scaled(Span(100, Millis))
  )
  
  // Thread-safe counter tests
  "AtomicCounter" should "be thread-safe" in {
    val counter = new AtomicLong(0)
    val numThreads = 100
    val incrementsPerThread = 1000
    
    val futures = (1 to numThreads).map { _ =>
      Future {
        (1 to incrementsPerThread).foreach { _ =>
          counter.incrementAndGet()
        }
      }
    }
    
    Await.ready(Future.sequence(futures), 10.seconds)
    counter.get() shouldBe (numThreads * incrementsPerThread).toLong
  }
  
  // Race condition detector
  "ConcurrentMap" should "handle concurrent writes" in {
    val map = new java.util.concurrent.ConcurrentHashMap[String, Int]()
    val operations = 10000
    
    val writes = Future {
      (1 to operations).foreach { n =>
        map.put(s"key-${n % 100}", n)
      }
    }
    
    val reads = Future {
      (1 to operations).foreach { n =>
        map.get(s"key-${n % 100}")  // Concurrent reads
      }
    }
    
    Await.ready(Future.sequence(List(writes, reads)), 10.seconds)
    map.size() should be <= 100
  }
  
  // Eventually — wait for async condition
  "Eventually" should "wait for condition to become true" in {
    val flag = new AtomicBoolean(false)
    
    Future {
      Thread.sleep(300)
      flag.set(true)
    }
    
    eventually {
      flag.get() shouldBe true
    }
  }
  
  // Futures with timeout
  "Futures" should "complete within time limit" in {
    val f = Future { Thread.sleep(200); 42 }
    val result = Await.result(f, 2.seconds)
    result shouldBe 42
  }
  
  // Test deadlock-free execution
  "Thread pool" should "not deadlock with nested futures" in {
    def nested(depth: Int): Future[Int] = {
      if (depth == 0) Future.successful(0)
      else nested(depth - 1).map(_ + 1)
    }
    
    val result = Await.result(nested(100), 10.seconds)
    result shouldBe 100
  }
  
  // Load test
  "System" should "handle concurrent load" in {
    val requestCount = 1000
    val startTime = System.currentTimeMillis()
    
    // Simulate concurrent API calls
    val futures = (1 to requestCount).map { i =>
      Future {
        Thread.sleep(1)  // Simulate fast operation
        i * 2
      }
    }
    
    val results = Await.result(Future.sequence(futures.toList), 30.seconds)
    val elapsed = System.currentTimeMillis() - startTime
    
    results.length shouldBe requestCount
    results.sum shouldBe (1 to requestCount).map(_ * 2).sum
    
    println(s"  Processed $requestCount requests in ${elapsed}ms")
    println(s"  Throughput: ${requestCount * 1000 / elapsed} req/s")
    
    elapsed should be < 10000L  // Should complete in 10 seconds
  }
  
  // Test for data race
  "Shared state" should "be protected from concurrent access" in {
    // Using synchronized
    var unsafeCount = 0
    val safeCount = new AtomicInteger(0)
    
    val futures = (1 to 1000).map { _ =>
      Future {
        safeCount.incrementAndGet()
        // unsafeCount += 1  // This would race
      }
    }
    
    Await.ready(Future.sequence(futures.toList), 5.seconds)
    safeCount.get() shouldBe 1000
  }
}

// Integration test helpers
class TestHelpers {
  
  // Wait for condition with timeout
  def waitFor(
    condition: => Boolean,
    timeout: Duration = 5.seconds,
    interval: Duration = 100.millis
  ): Boolean = {
    val deadline = System.currentTimeMillis() + timeout.toMillis
    while (!condition && System.currentTimeMillis() < deadline) {
      Thread.sleep(interval.toMillis)
    }
    condition
  }
  
  // Measure execution time
  def measure[A](block: => A): (A, Long) = {
    val start = System.nanoTime()
    val result = block
    val elapsed = (System.nanoTime() - start) / 1000000  // ms
    (result, elapsed)
  }
  
  // Stress test helper
  def stressTest[A](
    name: String,
    threads: Int,
    iterations: Int,
    action: () => A
  ): Map[String, Long] = {
    val counter = new AtomicLong(0)
    val errors = new AtomicLong(0)
    val totalTime = new AtomicLong(0)
    
    val futures = (1 to threads).map { _ =>
      Future {
        (1 to iterations).foreach { _ =>
          val start = System.nanoTime()
          try {
            action()
            counter.incrementAndGet()
          } catch {
            case _: Exception => errors.incrementAndGet()
          }
          totalTime.addAndGet(System.nanoTime() - start)
        }
      }
    }
    
    Await.ready(Future.sequence(futures.toList), 60.seconds)
    
    val totalOps = counter.get() + errors.get()
    Map(
      "total" -> totalOps,
      "success" -> counter.get(),
      "errors" -> errors.get(),
      "avgLatencyUs" -> (if (totalOps > 0) totalTime.get() / totalOps / 1000 else 0)
    )
  }
}
```

---

## สรุป Part 40 — จบ Parts 11-40

| Testing Type | Library | ใช้เมื่อ |
|-------------|---------|---------|
| Unit tests | ScalaTest FlatSpec | Pure functions, classes |
| BDD tests | ScalaTest FunSuite | Business behavior |
| Async tests | AsyncFlatSpec, ScalaFutures | Futures, promises |
| Property tests | ScalaCheck | Invariants, laws |
| Actor tests | ActorTestKit | Akka typed actors |
| Integration | Eventually, custom helpers | Async state changes |
| Load tests | Concurrent futures | Throughput, latency |
| Mutation | Stryker4s | Test quality |

---

## แบบฝึกหัด Part 40

**ข้อ 1:** เขียน comprehensive test suite สำหรับ custom `Either` implementation จาก Part 25

**ข้อ 2:** สร้าง property-based tests สำหรับ `Sorted Merge` algorithm

**ข้อ 3:** Test actor supervision: verify restart behavior และ state recovery

**ข้อ 4:** Implement load test framework ที่ measure p50, p95, p99 latency percentiles

**ข้อ 5:** สร้าง chaos testing utility ที่ inject random failures ใน Future pipelines

---

## 🎓 ยินดีด้วย! คุณได้เรียนจบ Parts 11-40

### สรุป Topics ที่ครอบคลุม:

| Parts | หัวข้อ |
|-------|--------|
| 11-13 | OOP: Inheritance, Abstract Classes, Companion Objects |
| 14-17 | FP: HOF, Lambdas, Map/Filter/Reduce, Immutability |
| 18-19 | Recursion, For Comprehensions |
| 20-22 | Type Classes, List/Vector, Map/Set |
| 23-24 | Pattern Matching, Extractors |
| 25-27 | Generics, Implicits, Type Bounds |
| 28-30 | ADTs, Error Handling, Standard Library |
| 31-33 | Futures, Execution Contexts, Async Patterns |
| 34-36 | Akka Actors, Communication, Supervision |
| 37-38 | Akka Streams, Akka HTTP |
| 39-40 | Reactive Patterns, Testing Concurrency |

➡️ ต่อไป: [Part 41 — Advanced Functional Programming](part-41-advanced-fp.md)
