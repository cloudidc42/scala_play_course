# Part 33: Async Programming Patterns

## Steps 321-330: Reactive Streams, Backpressure, Pipelines, Async State Machines

---

## Step 321: Reactive Patterns ด้วย Future

```scala
import scala.concurrent._
import scala.concurrent.duration._
import ExecutionContext.Implicits.global
import java.util.concurrent.atomic._

object ReactivePatternsDemo extends App {
  
  // Async Event Stream simulation
  class EventBus[A] {
    private val subscribers = new java.util.concurrent.CopyOnWriteArrayList[A => Unit]()
    
    def subscribe(handler: A => Unit): Unit = subscribers.add(handler)
    
    def publish(event: A): Unit = {
      import scala.jdk.CollectionConverters._
      subscribers.asScala.foreach(h => Future(h(event)))
    }
    
    def unsubscribe(handler: A => Unit): Unit = subscribers.remove(handler)
  }
  
  sealed trait DomainEvent
  case class UserCreated(id: String, name: String) extends DomainEvent
  case class OrderPlaced(orderId: String, userId: String, amount: Double) extends DomainEvent
  case class PaymentProcessed(orderId: String, status: String) extends DomainEvent
  
  val bus = new EventBus[DomainEvent]()
  
  // Subscribers
  bus.subscribe {
    case UserCreated(id, name) => println(s"  [Email] Welcome $name!")
    case OrderPlaced(oid, uid, amount) => println(s"  [Email] Order $oid for $$${amount}")
    case _ => ()
  }
  
  bus.subscribe {
    case UserCreated(id, _) => println(s"  [Metrics] New user: $id")
    case OrderPlaced(oid, _, amount) => println(s"  [Metrics] Order: $oid, $$${amount}")
    case PaymentProcessed(oid, status) => println(s"  [Metrics] Payment $oid: $status")
  }
  
  bus.subscribe {
    case OrderPlaced(oid, uid, _) => println(s"  [Inventory] Reserve for order $oid, user $uid")
    case _ => ()
  }
  
  println("=== Event Bus ===")
  bus.publish(UserCreated("u1", "Alice"))
  Thread.sleep(50)
  bus.publish(OrderPlaced("o1", "u1", 1500.0))
  Thread.sleep(50)
  bus.publish(PaymentProcessed("o1", "approved"))
  Thread.sleep(100)
  
  // Async accumulator
  class AsyncAccumulator[A, B](
    initial: B,
    f: (B, A) => B
  ) {
    private val state = new AtomicReference[B](initial)
    private val queue = new java.util.concurrent.LinkedBlockingQueue[A]()
    
    def add(item: A): Unit = queue.put(item)
    
    def processAll(): Future[B] = Future {
      var item = queue.poll()
      while (item != null) {
        state.updateAndGet(b => f(b, item))
        item = queue.poll()
      }
      state.get()
    }
    
    def current: B = state.get()
  }
  
  val counter = new AsyncAccumulator[Int, Int](0, _ + _)
  
  println("\n=== Async Accumulator ===")
  (1 to 10).foreach(counter.add)
  val total = Await.result(counter.processAll(), 2.seconds)
  println(s"Total: $total")
  
  Thread.sleep(200)
}
```

---

## Step 322: Pipeline Pattern

```scala
import scala.concurrent._
import scala.concurrent.duration._
import ExecutionContext.Implicits.global

object PipelinePattern extends App {
  
  // Async pipeline: data flows through stages
  trait Stage[A, B] {
    def process(input: A): Future[B]
    
    def andThen[C](next: Stage[B, C]): Stage[A, C] = new Stage[A, C] {
      def process(input: A): Future[C] = Stage.this.process(input).flatMap(next.process)
    }
    
    def map[C](f: B => C): Stage[A, C] = new Stage[A, C] {
      def process(input: A): Future[C] = Stage.this.process(input).map(f)
    }
  }
  
  object Stage {
    def apply[A, B](f: A => Future[B]): Stage[A, B] = input => f(input)
    def sync[A, B](f: A => B): Stage[A, B] = input => Future.successful(f(input))
    def parallel[A, B](stages: List[Stage[A, B]]): Stage[A, List[B]] = input =>
      Future.traverse(stages)(_.process(input))
  }
  
  // Define stages
  val parseStage: Stage[String, List[String]] = Stage.sync { line =>
    println(s"  [Parse] '$line'")
    line.split(",").map(_.trim).toList
  }
  
  val validateStage: Stage[List[String], List[String]] = Stage { fields =>
    Future {
      println(s"  [Validate] $fields")
      if (fields.length < 2) throw new Exception("Too few fields")
      fields
    }
  }
  
  val enrichStage: Stage[List[String], Map[String, String]] = Stage { fields =>
    Future {
      Thread.sleep(50)  // Simulate DB lookup
      println(s"  [Enrich] $fields")
      Map("name" -> fields(0), "value" -> fields(1), "source" -> "enriched")
    }
  }
  
  val saveStage: Stage[Map[String, String], String] = Stage { data =>
    Future {
      Thread.sleep(30)  // Simulate save
      println(s"  [Save] $data")
      s"saved-${data("name")}"
    }
  }
  
  // Compose pipeline
  val pipeline = parseStage
    .andThen(validateStage)
    .andThen(enrichStage)
    .andThen(saveStage)
  
  println("=== Async Pipeline ===")
  val lines = List("Alice,100", "Bob,200", "Carol,150", "x")
  
  val results = Future.traverse(lines) { line =>
    pipeline.process(line).map(r => Right(r): Either[String, String]).recover {
      case e => Left(s"Failed '$line': ${e.getMessage}")
    }
  }
  
  val outcomes = Await.result(results, 10.seconds)
  println("\nResults:")
  outcomes.foreach {
    case Right(r) => println(s"  OK: $r")
    case Left(e)  => println(s"  ERR: $e")
  }
  
  // Fan-out pipeline (parallel stages)
  val analyticsStage: Stage[String, String] = Stage.sync { s => s"analytics:$s" }
  val loggingStage: Stage[String, String] = Stage.sync { s => s"log:$s" }
  val cacheStage: Stage[String, String] = Stage.sync { s => s"cache:$s" }
  
  val fanOut = Stage.parallel(List(analyticsStage, loggingStage, cacheStage))
  
  println("\n=== Fan-out Pipeline ===")
  val fanResult = Await.result(fanOut.process("order-123"), 2.seconds)
  fanResult.foreach(r => println(s"  $r"))
}
```

---

## Step 323: Async State Machine

```scala
import scala.concurrent._
import scala.concurrent.duration._
import ExecutionContext.Implicits.global

object AsyncStateMachine extends App {
  
  // Async state machine for order processing
  sealed trait OrderState
  case object New extends OrderState
  case object Validated extends OrderState
  case object Reserved extends OrderState
  case object PaymentPending extends OrderState
  case object Paid extends OrderState
  case object Fulfilled extends OrderState
  case object Cancelled extends OrderState
  case class Failed(reason: String) extends OrderState
  
  sealed trait OrderCommand
  case object Validate extends OrderCommand
  case object Reserve extends OrderCommand
  case object ProcessPayment extends OrderCommand
  case object Fulfill extends OrderCommand
  case object Cancel extends OrderCommand
  
  case class Order2(id: String, items: List[String], total: Double, state: OrderState)
  
  // State transitions
  def transition(order: Order2, command: OrderCommand): Future[Order2] = 
    (order.state, command) match {
      case (New, Validate) => Future {
        Thread.sleep(50)  // Validate items, check stock, etc.
        println(s"  [Order ${order.id}] Validated")
        order.copy(state = Validated)
      }
      case (Validated, Reserve) => Future {
        Thread.sleep(80)  // Reserve inventory
        println(s"  [Order ${order.id}] Inventory reserved")
        order.copy(state = Reserved)
      }
      case (Reserved, ProcessPayment) => Future {
        Thread.sleep(200)  // Call payment gateway
        println(s"  [Order ${order.id}] Payment processing...")
        // Simulate 80% success
        if (math.random() > 0.2) {
          println(s"  [Order ${order.id}] Payment approved")
          order.copy(state = Paid)
        } else {
          println(s"  [Order ${order.id}] Payment declined")
          order.copy(state = Failed("Payment declined"))
        }
      }
      case (Paid, Fulfill) => Future {
        Thread.sleep(100)  // Ship order
        println(s"  [Order ${order.id}] Fulfilling order")
        order.copy(state = Fulfilled)
      }
      case (_, Cancel) => Future {
        println(s"  [Order ${order.id}] Cancelling from ${order.state}")
        order.copy(state = Cancelled)
      }
      case (state, cmd) => 
        Future.failed(new Exception(s"Invalid: $cmd in state $state"))
    }
  
  // Process a sequence of commands
  def processCommands(order: Order2, commands: List[OrderCommand]): Future[Order2] = {
    commands.foldLeft(Future.successful(order)) { (orderF, command) =>
      orderF.flatMap { o =>
        if (o.state.isInstanceOf[Failed] || o.state == Cancelled)
          Future.successful(o)  // Short circuit
        else
          transition(o, command).recover { case e =>
            o.copy(state = Failed(e.getMessage))
          }
      }
    }
  }
  
  println("=== Async State Machine ===")
  
  // Process multiple orders
  val orders = List(
    Order2("O-001", List("item-1", "item-2"), 1500.0, New),
    Order2("O-002", List("item-3"), 250.0, New),
    Order2("O-003", List("item-4", "item-5"), 3000.0, New)
  )
  
  val commands = List(Validate, Reserve, ProcessPayment, Fulfill)
  
  val allFutures = orders.map { order =>
    processCommands(order, commands).map { finalOrder =>
      println(s"\nFinal state for ${finalOrder.id}: ${finalOrder.state}")
      finalOrder
    }
  }
  
  val finalOrders = Await.result(Future.sequence(allFutures), 30.seconds)
  
  println(s"\nSummary:")
  println(s"  Fulfilled: ${finalOrders.count(_.state == Fulfilled)}")
  println(s"  Failed: ${finalOrders.count(_.state.isInstanceOf[Failed])}")
  println(s"  Cancelled: ${finalOrders.count(_.state == Cancelled)}")
}
```

---

## Step 324-330: Advanced Async Patterns

```scala
import scala.concurrent._
import scala.concurrent.duration._
import ExecutionContext.Implicits.global

object AdvancedAsyncPatterns extends App {
  
  // Debounce — coalesce rapid calls into one
  class Debouncer[A](delay: FiniteDuration) {
    private var lastScheduled: Option[java.util.concurrent.ScheduledFuture[_]] = None
    private val scheduler = java.util.concurrent.Executors.newSingleThreadScheduledExecutor()
    @volatile private var pendingPromise: Option[Promise[A]] = None
    
    def debounce(f: => Future[A]): Future[A] = synchronized {
      lastScheduled.foreach(_.cancel(false))
      val promise = Promise[A]()
      pendingPromise = Some(promise)
      
      val task = scheduler.schedule(
        new Runnable { def run(): Unit = {
          val p = synchronized { pendingPromise }.getOrElse(promise)
          f.onComplete(p.tryComplete)
        }},
        delay.toMillis,
        java.util.concurrent.TimeUnit.MILLISECONDS
      )
      lastScheduled = Some(task)
      promise.future
    }
    
    def shutdown(): Unit = scheduler.shutdown()
  }
  
  println("=== Debouncer ===")
  val debouncer = new Debouncer[String](200.millis)
  var callCount = 0
  
  // Rapid calls — only last should execute
  val futures = (1 to 5).map { i =>
    callCount += 1
    val n = i
    Thread.sleep(50)
    debouncer.debounce {
      Future {
        println(s"  Executing call $n (after debounce)")
        s"result-$n"
      }
    }
  }
  
  Thread.sleep(500)
  println(s"Initiated $callCount calls")
  
  // Throttle — limit rate of execution
  class Throttler(maxPerSecond: Int) {
    private val minInterval = (1000.0 / maxPerSecond).toLong
    private var lastCall = 0L
    
    def throttle[A](f: => Future[A]): Future[A] = synchronized {
      val now = System.currentTimeMillis()
      val elapsed = now - lastCall
      if (elapsed < minInterval) {
        val waitTime = minInterval - elapsed
        val p = Promise[A]()
        Future {
          Thread.sleep(waitTime)
          f.onComplete(p.tryComplete)
        }
        lastCall = now + waitTime
        p.future
      } else {
        lastCall = now
        f
      }
    }
  }
  
  println("\n=== Throttler ===")
  val throttler = new Throttler(5)  // max 5 per second
  
  val throttled = (1 to 10).map { i =>
    throttler.throttle(Future {
      s"throttled-$i"
    })
  }
  
  val start = System.currentTimeMillis()
  val results = Await.result(Future.sequence(throttled.toList), 30.seconds)
  println(s"${results.size} results in ${System.currentTimeMillis() - start}ms")
  
  // Semaphore for controlled concurrency
  class AsyncSemaphore(permits: Int) {
    private val sem = new java.util.concurrent.Semaphore(permits)
    
    def withPermit[A](f: => Future[A]): Future[A] = {
      Future(sem.acquire()).flatMap { _ =>
        f.andThen { case _ => sem.release() }
      }
    }
    
    def availablePermits: Int = sem.availablePermits()
  }
  
  println("\n=== Async Semaphore ===")
  val semaphore = new AsyncSemaphore(3)
  
  val semTasks = (1 to 8).map { i =>
    semaphore.withPermit {
      Future {
        println(s"  Task $i running (${3 - semaphore.availablePermits()} slots used)")
        Thread.sleep(150)
        s"task-$i-done"
      }
    }
  }
  
  val semStart = System.currentTimeMillis()
  val semResults = Await.result(Future.sequence(semTasks.toList), 10.seconds)
  println(s"All ${semResults.size} tasks done in ${System.currentTimeMillis() - semStart}ms")
  
  debouncer.shutdown()
  Thread.sleep(500)
}
```

---

## สรุป Part 33

| Pattern | ประโยชน์ | Use Case |
|---------|---------|---------|
| Event Bus | Decouple producer/consumer | Domain events, notifications |
| Pipeline | Chain transformations | ETL, request processing |
| Fan-out/Fan-in | Parallel + aggregate | Multi-service aggregation |
| Circuit Breaker | Fail fast on errors | External service calls |
| Retry | Handle transient failures | Network calls, DB ops |
| Debounce | Rate-limit rapid calls | Search input, scroll events |
| Throttle | Fixed-rate limiting | API rate limits |
| Semaphore | Limit concurrency | DB connections, API quota |
| Bulkhead | Isolate failures | Service isolation |

---

## แบบฝึกหัด Part 33

**ข้อ 1:** Implement async `ReadWriteLock` ที่ allow multiple concurrent readers, exclusive writer

**ข้อ 2:** สร้าง async pub/sub system ที่ support topics, wildcards, persistent subscriptions

**ข้อ 3:** Implement `LoadBalancer` ที่ round-robin requests across multiple backend Futures

**ข้อ 4:** สร้าง `AsyncQueue[A]` ที่ support `offer(item)` และ `take(): Future[A]`

**ข้อ 5:** Implement saga pattern ด้วย compensating transactions สำหรับ distributed operations

---

➡️ ต่อไป: [Part 34 — Akka Actors Basics](part-34-akka-actors-basics.md)
