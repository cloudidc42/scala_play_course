# Part 39: Reactive Patterns

## Steps 381-390: CQRS, Event Sourcing, Saga, Outbox, Reactive Microservices

---

## Step 381: Reactive Manifesto และ Principles

```
Reactive System ตามหลัก Reactive Manifesto:
1. Responsive   — ตอบสนองเร็วและสม่ำเสมอ
2. Resilient    — ล้มเหลวได้โดยไม่กระทบส่วนอื่น
3. Elastic      — scale in/out ตาม load
4. Message-Driven — communicate ผ่าน async messages

ลักษณะ:
- Non-blocking I/O
- Backpressure
- Location transparency
- Failure isolation
```

```scala
import scala.concurrent._
import scala.concurrent.duration._
import ExecutionContext.Implicits.global
import java.util.concurrent.atomic._

// CQRS — Command Query Responsibility Segregation
// Write model (Commands) แยกจาก Read model (Queries)
object CQRSPattern extends App {
  
  // ===== Write Side (Command Model) =====
  
  sealed trait AccountCommand
  case class OpenAccount(id: String, ownerName: String, initialBalance: Double) extends AccountCommand
  case class Deposit(accountId: String, amount: Double) extends AccountCommand
  case class Withdraw(accountId: String, amount: Double) extends AccountCommand
  case class Transfer(fromId: String, toId: String, amount: Double) extends AccountCommand
  case class CloseAccount(accountId: String) extends AccountCommand
  
  sealed trait AccountEvent
  case class AccountOpened2(id: String, owner: String, balance: Double, timestamp: Long) extends AccountEvent
  case class MoneyDeposited2(accountId: String, amount: Double, newBalance: Double, timestamp: Long) extends AccountEvent
  case class MoneyWithdrawn2(accountId: String, amount: Double, newBalance: Double, timestamp: Long) extends AccountEvent
  case class AccountTransferred(fromId: String, toId: String, amount: Double, timestamp: Long) extends AccountEvent
  case class AccountClosed2(accountId: String, timestamp: Long) extends AccountEvent
  
  // Command handler — validate and produce events
  class AccountCommandHandler(
    eventStore: EventStore,
    readModel: ReadModel
  ) {
    def handle(command: AccountCommand): Either[String, List[AccountEvent]] = command match {
      case OpenAccount(id, owner, balance) =>
        if (balance < 0) Left("Initial balance cannot be negative")
        else if (readModel.accountExists(id)) Left(s"Account $id already exists")
        else Right(List(AccountOpened2(id, owner, balance, System.currentTimeMillis())))
        
      case Deposit(accountId, amount) =>
        if (amount <= 0) Left("Deposit amount must be positive")
        else readModel.getBalance(accountId) match {
          case None => Left(s"Account $accountId not found")
          case Some(balance) =>
            Right(List(MoneyDeposited2(accountId, amount, balance + amount, System.currentTimeMillis())))
        }
        
      case Withdraw(accountId, amount) =>
        if (amount <= 0) Left("Withdrawal amount must be positive")
        else readModel.getBalance(accountId) match {
          case None => Left(s"Account $accountId not found")
          case Some(balance) if balance < amount => Left("Insufficient funds")
          case Some(balance) =>
            Right(List(MoneyWithdrawn2(accountId, amount, balance - amount, System.currentTimeMillis())))
        }
        
      case Transfer(fromId, toId, amount) =>
        if (amount <= 0) Left("Transfer amount must be positive")
        else (readModel.getBalance(fromId), readModel.getBalance(toId)) match {
          case (None, _) => Left(s"Account $fromId not found")
          case (_, None) => Left(s"Account $toId not found")
          case (Some(fromBal), Some(_)) if fromBal < amount => Left("Insufficient funds")
          case (Some(_), Some(_)) =>
            Right(List(AccountTransferred(fromId, toId, amount, System.currentTimeMillis())))
        }
        
      case CloseAccount(accountId) =>
        if (!readModel.accountExists(accountId)) Left(s"Account $accountId not found")
        else Right(List(AccountClosed2(accountId, System.currentTimeMillis())))
    }
  }
  
  // Event store — append-only log
  class EventStore {
    private var events: List[AccountEvent] = List.empty
    
    def append(event: AccountEvent): Unit = synchronized {
      events = events :+ event
    }
    
    def appendAll(evts: List[AccountEvent]): Unit = synchronized {
      events = events ++ evts
    }
    
    def getEvents(accountId: String): List[AccountEvent] = events.filter {
      case AccountOpened2(id, _, _, _) => id == accountId
      case MoneyDeposited2(id, _, _, _) => id == accountId
      case MoneyWithdrawn2(id, _, _, _) => id == accountId
      case AccountTransferred(from, to, _, _) => from == accountId || to == accountId
      case AccountClosed2(id, _) => id == accountId
    }
    
    def allEvents: List[AccountEvent] = events
  }
  
  // ===== Read Side (Query Model) =====
  
  case class AccountView(
    id: String,
    owner: String,
    balance: Double,
    isOpen: Boolean,
    transactionCount: Int
  )
  
  class ReadModel {
    private var accounts = Map.empty[String, AccountView]
    
    def accountExists(id: String): Boolean = accounts.contains(id)
    def getBalance(id: String): Option[Double] = accounts.get(id).filter(_.isOpen).map(_.balance)
    def getAccount(id: String): Option[AccountView] = accounts.get(id)
    def getAllAccounts: List[AccountView] = accounts.values.toList
    
    // Update read model from events
    def applyEvent(event: AccountEvent): Unit = event match {
      case AccountOpened2(id, owner, balance, _) =>
        accounts = accounts + (id -> AccountView(id, owner, balance, isOpen = true, 0))
        
      case MoneyDeposited2(accountId, _, newBalance, _) =>
        accounts.get(accountId).foreach { acc =>
          accounts = accounts + (accountId -> acc.copy(balance = newBalance, transactionCount = acc.transactionCount + 1))
        }
        
      case MoneyWithdrawn2(accountId, _, newBalance, _) =>
        accounts.get(accountId).foreach { acc =>
          accounts = accounts + (accountId -> acc.copy(balance = newBalance, transactionCount = acc.transactionCount + 1))
        }
        
      case AccountTransferred(fromId, toId, amount, _) =>
        accounts.get(fromId).foreach { acc =>
          accounts = accounts + (fromId -> acc.copy(balance = acc.balance - amount, transactionCount = acc.transactionCount + 1))
        }
        accounts.get(toId).foreach { acc =>
          accounts = accounts + (toId -> acc.copy(balance = acc.balance + amount, transactionCount = acc.transactionCount + 1))
        }
        
      case AccountClosed2(accountId, _) =>
        accounts.get(accountId).foreach { acc =>
          accounts = accounts + (accountId -> acc.copy(isOpen = false))
        }
    }
  }
  
  // Wire everything together
  val eventStore = new EventStore
  val readModel = new ReadModel
  val handler = new AccountCommandHandler(eventStore, readModel)
  
  def execute(cmd: AccountCommand): Either[String, Unit] = {
    handler.handle(cmd) match {
      case Right(events) =>
        eventStore.appendAll(events)
        events.foreach(readModel.applyEvent)
        Right(())
      case Left(err) =>
        Left(err)
    }
  }
  
  println("=== CQRS Demo ===")
  
  // Execute commands
  val commands = List(
    OpenAccount("ACC-001", "Alice", 1000.0),
    OpenAccount("ACC-002", "Bob", 500.0),
    Deposit("ACC-001", 5000.0),
    Withdraw("ACC-001", 1000.0),
    Transfer("ACC-001", "ACC-002", 2000.0),
    Withdraw("ACC-999", 100.0),   // Should fail — not found
    Withdraw("ACC-002", 5000.0)   // Should fail — insufficient
  )
  
  commands.foreach { cmd =>
    execute(cmd) match {
      case Right(_)  => println(s"  OK: $cmd")
      case Left(err) => println(s"  FAIL: $cmd -> $err")
    }
  }
  
  println("\n=== Read Model State ===")
  readModel.getAllAccounts.foreach { acc =>
    println(s"  ${acc.id}: ${acc.owner}, ฿${acc.balance}, txns=${acc.transactionCount}, open=${acc.isOpen}")
  }
  
  println("\n=== Event Log ===")
  eventStore.allEvents.foreach(e => println(s"  $e"))
}
```

---

## Step 382: Saga Pattern

```scala
import scala.concurrent._
import scala.concurrent.duration._
import ExecutionContext.Implicits.global

// Saga = sequence of local transactions with compensating actions
object SagaPattern extends App {
  
  // ===== Saga for Order Processing =====
  
  case class SagaState(
    orderId: String,
    completedSteps: List[String],
    compensations: List[() => Unit]
  )
  
  sealed trait SagaResult[+A]
  case class SagaSuccess[A](value: A, steps: List[String]) extends SagaResult[A]
  case class SagaFailed(error: String, compensated: List[String]) extends SagaResult[Nothing]
  
  class SagaBuilder {
    private var steps: List[(() => Future[Unit], () => Future[Unit], String)] = List.empty
    
    def step(
      name: String,
      action: () => Future[Unit],
      compensation: () => Future[Unit]
    ): SagaBuilder = {
      steps = steps :+ (action, compensation, name)
      this
    }
    
    def execute(): Future[SagaResult[Unit]] = {
      def runSteps(
        remaining: List[(() => Future[Unit], () => Future[Unit], String)],
        completed: List[(String, () => Future[Unit])]
      ): Future[SagaResult[Unit]] = remaining match {
        case Nil => 
          Future.successful(SagaSuccess((), completed.map(_._1)))
          
        case (action, compensation, name) :: rest =>
          println(s"  → Executing step: $name")
          action().transformWith {
            case scala.util.Success(_) =>
              runSteps(rest, completed :+ (name, compensation))
              
            case scala.util.Failure(e) =>
              println(s"  ✗ Step failed: $name (${e.getMessage})")
              println(s"  ← Running compensations...")
              val compensationFutures = completed.reverse.map { case (n, comp) =>
                println(s"    ← Compensating: $n")
                comp()
              }
              Future.sequence(compensationFutures).map { _ =>
                SagaFailed(e.getMessage, completed.map(_._1))
              }
          }
      }
      
      runSteps(steps, List.empty)
    }
  }
  
  // Order processing saga
  case class OrderContext(
    orderId: String,
    userId: String,
    items: List[String],
    amount: Double
  )
  
  def createOrderSaga(ctx: OrderContext): SagaBuilder = {
    val saga = new SagaBuilder()
    
    saga
      .step(
        "Validate Order",
        () => Future {
          println(s"    Validating order ${ctx.orderId}")
          if (ctx.items.isEmpty) throw new Exception("Empty order")
          Thread.sleep(50)
        },
        () => Future.successful(())  // No compensation needed
      )
      .step(
        "Reserve Inventory",
        () => Future {
          println(s"    Reserving inventory for ${ctx.items.size} items")
          Thread.sleep(80)
          // Simulate occasional failure
          // if (ctx.orderId == "fail-inv") throw new Exception("Out of stock")
        },
        () => Future {
          println(s"    COMPENSATE: Releasing inventory reservation")
          Thread.sleep(30)
        }
      )
      .step(
        "Process Payment",
        () => Future {
          println(s"    Processing payment of ฿${ctx.amount}")
          Thread.sleep(100)
          if (ctx.orderId.contains("fail-pay")) 
            throw new Exception("Payment declined")
        },
        () => Future {
          println(s"    COMPENSATE: Issuing refund of ฿${ctx.amount}")
          Thread.sleep(50)
        }
      )
      .step(
        "Fulfill Order",
        () => Future {
          println(s"    Preparing fulfillment")
          Thread.sleep(60)
        },
        () => Future {
          println(s"    COMPENSATE: Canceling fulfillment")
          Thread.sleep(30)
        }
      )
      .step(
        "Send Confirmation",
        () => Future {
          println(s"    Sending confirmation email")
          Thread.sleep(40)
        },
        () => Future.successful(())
      )
  }
  
  println("=== Saga Pattern ===")
  
  val successOrder = OrderContext("ORD-001", "USER-1", List("item-1", "item-2"), 1500.0)
  val failOrder = OrderContext("fail-pay-002", "USER-2", List("item-3"), 500.0)
  
  println("--- Order 1 (Success) ---")
  val result1 = Await.result(createOrderSaga(successOrder).execute(), 30.seconds)
  println(s"Result: $result1\n")
  
  println("--- Order 2 (Payment Failure) ---")
  val result2 = Await.result(createOrderSaga(failOrder).execute(), 30.seconds)
  println(s"Result: $result2")
}
```

---

## Step 383-390: Outbox Pattern and Reactive Microservices

```scala
import scala.concurrent._
import scala.concurrent.duration._
import ExecutionContext.Implicits.global
import java.util.UUID

// Outbox Pattern — reliable event publishing
object OutboxPattern extends App {
  
  case class OutboxMessage(
    id: String,
    aggregateId: String,
    eventType: String,
    payload: String,
    createdAt: Long,
    published: Boolean = false
  )
  
  // Outbox table (would be in same DB transaction as domain operation)
  class OutboxRepository {
    private var messages: List[OutboxMessage] = List.empty
    
    def save(message: OutboxMessage): Unit = synchronized {
      messages = messages :+ message
    }
    
    def getUnpublished: List[OutboxMessage] = synchronized {
      messages.filter(!_.published)
    }
    
    def markPublished(id: String): Unit = synchronized {
      messages = messages.map(m => if (m.id == id) m.copy(published = true) else m)
    }
  }
  
  // Message publisher
  class MessagePublisher(outbox: OutboxRepository) {
    def publish(topic: String, message: String): Unit = {
      // In real: publish to Kafka/RabbitMQ/etc
      println(s"  [MQ] Publishing to $topic: $message")
    }
    
    // Outbox publisher — poll and publish
    def publishPending(): Unit = {
      val pending = outbox.getUnpublished
      pending.foreach { msg =>
        try {
          publish(msg.eventType, msg.payload)
          outbox.markPublished(msg.id)
          println(s"  [Outbox] Published: ${msg.eventType} for ${msg.aggregateId}")
        } catch {
          case e: Exception =>
            println(s"  [Outbox] Failed to publish ${msg.id}: ${e.getMessage}")
        }
      }
    }
  }
  
  // Domain service using outbox
  class OrderDomainService(outbox: OutboxRepository) {
    
    def placeOrder(customerId: String, items: List[String], amount: Double): String = {
      val orderId = UUID.randomUUID().toString.take(8)
      
      // In real: this entire block would be in a DB transaction
      println(s"  [DB] Creating order $orderId")
      
      // Add to outbox (same transaction)
      outbox.save(OutboxMessage(
        id = UUID.randomUUID().toString,
        aggregateId = orderId,
        eventType = "order.created",
        payload = s"""{"orderId":"$orderId","customerId":"$customerId","amount":$amount}""",
        createdAt = System.currentTimeMillis()
      ))
      
      outbox.save(OutboxMessage(
        id = UUID.randomUUID().toString,
        aggregateId = customerId,
        eventType = "customer.order_placed",
        payload = s"""{"customerId":"$customerId","orderId":"$orderId"}""",
        createdAt = System.currentTimeMillis()
      ))
      
      orderId
    }
  }
  
  println("=== Outbox Pattern ===")
  
  val outbox = new OutboxRepository
  val publisher = new MessagePublisher(outbox)
  val orderService = new OrderDomainService(outbox)
  
  // Place orders
  val order1 = orderService.placeOrder("CUST-001", List("item-1"), 500.0)
  val order2 = orderService.placeOrder("CUST-002", List("item-2", "item-3"), 1200.0)
  
  println(s"\nPlaced orders: $order1, $order2")
  println(s"Pending messages: ${outbox.getUnpublished.size}")
  
  // Publisher runs periodically
  println("\n--- Publishing outbox messages ---")
  publisher.publishPending()
  
  println(s"\nPending after publish: ${outbox.getUnpublished.size}")
  
  // ===== Reactive Microservices Pattern =====
  
  // Service discovery (simplified)
  class ServiceRegistry {
    private var services = Map.empty[String, List[String]]
    
    def register(name: String, endpoint: String): Unit = {
      services = services + (name -> (services.getOrElse(name, List.empty) :+ endpoint))
      println(s"  [Registry] Registered $name at $endpoint")
    }
    
    def deregister(name: String, endpoint: String): Unit = {
      services = services + (name -> services.getOrElse(name, List.empty).filterNot(_ == endpoint))
    }
    
    def discover(name: String): Option[String] = {
      services.get(name).filter(_.nonEmpty).map { endpoints =>
        endpoints(scala.util.Random.nextInt(endpoints.size))
      }
    }
    
    def discoverAll(name: String): List[String] = services.getOrElse(name, List.empty)
  }
  
  // Health check aggregator
  class HealthAggregator(registry: ServiceRegistry) {
    def checkAll(): Future[Map[String, Boolean]] = {
      val services = List("user-service", "order-service", "payment-service")
      
      Future.traverse(services) { serviceName =>
        registry.discover(serviceName) match {
          case None => Future.successful(serviceName -> false)
          case Some(endpoint) =>
            Future {
              // Simulate health check
              Thread.sleep(50)
              serviceName -> true
            }
        }
      }.map(_.toMap)
    }
  }
  
  println("\n=== Service Registry ===")
  val registry = new ServiceRegistry
  registry.register("user-service", "http://user-1:8080")
  registry.register("user-service", "http://user-2:8080")
  registry.register("order-service", "http://order-1:8080")
  registry.register("payment-service", "http://payment-1:8080")
  
  println(s"\nDiscover user-service: ${registry.discover("user-service")}")
  println(s"Discover user-service: ${registry.discover("user-service")}")  // Round robin
  
  val healthChecker = new HealthAggregator(registry)
  println("\n=== Health Check ===")
  val health = Await.result(healthChecker.checkAll(), 5.seconds)
  health.foreach { case (service, ok) =>
    println(s"  $service: ${if (ok) "OK" else "DOWN"}")
  }
}
```

---

## สรุป Part 39

| Pattern | ปัญหาที่แก้ | ใช้เมื่อ |
|---------|------------|---------|
| CQRS | Read/write scaling | High read vs write load |
| Event Sourcing | Audit trail, time travel | Financial, compliance |
| Saga | Distributed transactions | Multi-service operations |
| Outbox | Reliable event publishing | Dual-write problem |
| Circuit Breaker | Cascading failures | External service calls |
| Bulkhead | Failure isolation | Mixed criticality workloads |
| Service Mesh | Cross-cutting concerns | Microservices infrastructure |
| Sidecar | Transparent middleware | Logging, tracing, auth |
| Strangler Fig | Legacy migration | Gradual system replacement |

---

## แบบฝึกหัด Part 39

**ข้อ 1:** Implement full CQRS system สำหรับ inventory management ด้วย event sourcing

**ข้อ 2:** สร้าง saga orchestrator ที่ handle compensation correctly สำหรับ booking system

**ข้อ 3:** Implement outbox pattern ด้วย simulated database transactions

**ข้อ 4:** สร้าง reactive microservice ที่ communicate ผ่าน event bus

**ข้อ 5:** Implement two-phase commit protocol ด้วย actors

---

➡️ ต่อไป: [Part 40 — Testing Concurrency](part-40-testing-concurrency.md)
