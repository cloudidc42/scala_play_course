# Part 93: Functional Architecture — Steps 921-930

## บทนำ: Functional Architecture

Functional Architecture นำหลักการ FP มาใช้ใน system design — Hexagonal Architecture, Effect Systems, Dependency Injection แบบ functional

---

## Step 921: Hexagonal Architecture

```scala
// HexagonalArchitecture.scala

/*
===== Hexagonal (Ports & Adapters) Architecture =====

         ┌─────────────────────────────────────┐
         │          Domain Core                │
         │   Business Logic (pure functions)   │
         └────────────┬────────────────────────┘
                      │ Ports (interfaces)
         ┌────────────┼────────────────────────┐
    Driving           │               Driven
    (Inbound)         │               (Outbound)
    ┌────────┐        │        ┌───────────────┐
    │  HTTP  │──────→ │ ──────→│  PostgreSQL   │
    │  gRPC  │        │        │  Redis        │
    │  Kafka │        │        │  S3           │
    └────────┘        │        └───────────────┘
    Adapters          │        Adapters
                      │
         (Application Service orchestrates)

Key rule: Domain Core has NO dependencies on infrastructure
*/

// ===== Domain (pure) =====
package domain {
  
  case class OrderId(value: Long)    extends AnyVal
  case class CustomerId(value: Long) extends AnyVal
  case class Money(cents: Long)      extends AnyVal {
    def +(other: Money): Money = Money(cents + other.cents)
    def >(other: Money): Boolean = cents > other.cents
  }
  
  sealed trait OrderStatus
  case object Pending   extends OrderStatus
  case object Confirmed extends OrderStatus
  case object Shipped   extends OrderStatus
  case object Cancelled extends OrderStatus
  
  case class Order(
    id: OrderId,
    customerId: CustomerId,
    total: Money,
    status: OrderStatus
  )
  
  // Domain error
  sealed trait DomainError
  case class OrderNotFound(id: OrderId) extends DomainError
  case class InsufficientInventory(productId: Long, requested: Int, available: Int) extends DomainError
  case class InvalidOrderState(current: OrderStatus, attempted: String) extends DomainError
  
  // Pure domain logic
  object OrderDomain {
    def canCancel(order: Order): Either[DomainError, Order] =
      order.status match {
        case Pending | Confirmed =>
          Right(order.copy(status = Cancelled))
        case _ =>
          Left(InvalidOrderState(order.status, "cancel"))
      }
    
    def canShip(order: Order): Either[DomainError, Order] =
      order.status match {
        case Confirmed => Right(order.copy(status = Shipped))
        case _ =>        Left(InvalidOrderState(order.status, "ship"))
      }
  }
}

// ===== Ports (interfaces) =====
package ports {
  import domain._
  import scala.concurrent.Future
  
  // Outbound port (driven)
  trait OrderRepository {
    def findById(id: OrderId): Future[Option[Order]]
    def save(order: Order): Future[Order]
    def findByCustomer(customerId: CustomerId): Future[List[Order]]
  }
  
  // Another outbound port
  trait PaymentGateway {
    def charge(customerId: CustomerId, amount: Money): Future[Either[String, String]]
    def refund(chargeId: String): Future[Either[String, Unit]]
  }
  
  // Event publisher port
  trait EventPublisher {
    def publish(event: OrderEvent): Future[Unit]
  }
  
  sealed trait OrderEvent
  case class OrderCreated(order: Order) extends OrderEvent
  case class OrderCancelled(order: Order, reason: String) extends OrderEvent
}

// ===== Application Service =====
package application {
  import domain._
  import ports._
  import scala.concurrent.{ExecutionContext, Future}
  
  class OrderService(
    repo: OrderRepository,
    payment: PaymentGateway,
    events: EventPublisher
  )(implicit ec: ExecutionContext) {
    
    def cancelOrder(id: OrderId, reason: String): Future[Either[DomainError, Order]] = {
      repo.findById(id).flatMap {
        case None => Future.successful(Left(OrderNotFound(id)))
        case Some(order) =>
          OrderDomain.canCancel(order) match {
            case Left(err) => Future.successful(Left(err))
            case Right(cancelled) =>
              for {
                saved <- repo.save(cancelled)
                _     <- events.publish(OrderCancelled(saved, reason))
              } yield Right(saved)
          }
      }
    }
  }
}
```

---

## Step 922: Onion Architecture

```scala
// OnionArchitecture.scala

/*
===== Onion Architecture =====

Layers (inner → outer, dependencies point inward):

  ┌─────────────────────────────────────┐
  │  Infrastructure (HTTP, DB, Kafka)   │
  │  ┌───────────────────────────────┐  │
  │  │  Application (Use Cases)      │  │
  │  │  ┌─────────────────────────┐  │  │
  │  │  │  Domain Services        │  │  │
  │  │  │  ┌─────────────────┐    │  │  │
  │  │  │  │  Domain Entities │    │  │  │
  │  │  │  └─────────────────┘    │  │  │
  │  │  └─────────────────────────┘  │  │
  │  └───────────────────────────────┘  │
  └─────────────────────────────────────┘

- Entities: Order, Customer, Product (pure data + business rules)
- Domain Services: OrderPricingService, InventoryService (stateless domain logic)
- Application: OrderApplicationService (orchestrates use cases)
- Infrastructure: PostgresOrderRepo, KafkaEventBus, AkkaHttpRoutes
*/

// Each layer defined as package object
package object entities {
  case class Product(id: Long, name: String, price: BigDecimal, stock: Int)
  case class OrderItem(product: Product, quantity: Int) {
    def subtotal: BigDecimal = product.price * quantity
  }
}

package object domain_services {
  import entities._
  
  object PricingService {
    def calculateTotal(items: List[OrderItem], discount: BigDecimal = 0): BigDecimal = {
      val subtotal = items.map(_.subtotal).sum
      subtotal * (1 - discount)
    }
    
    def applyBulkDiscount(quantity: Int): BigDecimal = quantity match {
      case q if q >= 100 => 0.20
      case q if q >= 50  => 0.10
      case q if q >= 10  => 0.05
      case _             => 0.00
    }
  }
  
  object InventoryService {
    def checkAvailability(product: Product, requested: Int): Either[String, Unit] =
      if (product.stock >= requested) Right(())
      else Left(s"Insufficient stock: requested=$requested, available=${product.stock}")
  }
}
```

---

## Step 923: Effect System Design

```scala
// EffectSystemDesign.scala

/*
===== Effect Systems =====

An effect is a description of a computation (not the computation itself).
It enables:
- Referential transparency
- Composability
- Testability (mock effects)
- Error handling (typed errors)
- Resource safety

Options:
1. scala.concurrent.Future  — simplest, eager, impure
2. cats.effect.IO           — lazy, pure, resource-safe
3. ZIO[R,E,A]               — environment + typed error + value
4. Monix Task               — eager/lazy hybrid
*/

// ===== Custom simple effect (educational) =====
sealed trait Effect[+A] {
  def map[B](f: A => B): Effect[B]
  def flatMap[B](f: A => Effect[B]): Effect[B]
}

case class Pure[A](value: A) extends Effect[A] {
  def map[B](f: A => B): Effect[B] = Pure(f(value))
  def flatMap[B](f: A => Effect[B]): Effect[B] = f(value)
}

case class Defer[A](thunk: () => Effect[A]) extends Effect[A] {
  def map[B](f: A => B): Effect[B] = Defer(() => thunk().map(f))
  def flatMap[B](f: A => Effect[B]): Effect[B] = Defer(() => thunk().flatMap(f))
}

case class Fail[A](error: Throwable) extends Effect[A] {
  def map[B](f: A => B): Effect[B] = Fail(error)
  def flatMap[B](f: A => Effect[B]): Effect[B] = Fail(error)
}

object Effect {
  def pure[A](a: A): Effect[A] = Pure(a)
  def delay[A](thunk: => A): Effect[A] = Defer(() => try Pure(thunk) catch { case e: Throwable => Fail(e) })
  def fail[A](e: Throwable): Effect[A] = Fail(e)
  
  def run[A](effect: Effect[A]): Either[Throwable, A] = effect match {
    case Pure(v)  => Right(v)
    case Fail(e)  => Left(e)
    case Defer(t) => run(t())
  }
}

// Usage
object EffectExample {
  def readConfig(key: String): Effect[String] =
    Effect.delay(sys.env.getOrElse(key, throw new RuntimeException(s"Missing: $key")))
  
  def program: Effect[String] = for {
    host <- readConfig("DB_HOST")
    port <- readConfig("DB_PORT")
  } yield s"Connected to $host:$port"
}
```

---

## Step 924: Functional Dependency Injection

```scala
// FunctionalDI.scala

// ===== Reader Monad (environment passing) =====
case class Reader[Env, A](run: Env => A) {
  def map[B](f: A => B): Reader[Env, B] = Reader(env => f(run(env)))
  def flatMap[B](f: A => Reader[Env, B]): Reader[Env, B] = Reader(env => f(run(env)).run(env))
}

object Reader {
  def ask[Env]: Reader[Env, Env] = Reader(identity)
  def pure[Env, A](a: A): Reader[Env, A] = Reader(_ => a)
}

// Dependencies
case class AppDependencies(
  orderRepo: OrderRepository,
  emailService: EmailService,
  logger: Logger
)

trait OrderRepository {
  def findById(id: Long): Option[Order]
  def save(order: Order): Order
}

trait EmailService { def sendEmail(to: String, subject: String, body: String): Unit }
trait Logger       { def info(msg: String): Unit }

case class Order(id: Long, customerId: Long, total: Double, status: String)

// Business logic as Reader
object OrderLogic {
  type AppReader[A] = Reader[AppDependencies, A]
  
  def findOrder(id: Long): AppReader[Option[Order]] =
    Reader(deps => {
      deps.logger.info(s"Finding order $id")
      deps.orderRepo.findById(id)
    })
  
  def cancelOrder(id: Long): AppReader[Either[String, Order]] =
    Reader(deps => {
      deps.orderRepo.findById(id) match {
        case None => Left(s"Order $id not found")
        case Some(order) if order.status == "cancelled" =>
          Left("Already cancelled")
        case Some(order) =>
          val cancelled = order.copy(status = "cancelled")
          deps.orderRepo.save(cancelled)
          deps.emailService.sendEmail(
            s"customer-${order.customerId}@example.com",
            "Order Cancelled",
            s"Your order #${order.id} has been cancelled."
          )
          Right(cancelled)
      }
    })
}

// Compose and run
object OrderApp {
  def main(args: Array[String]): Unit = {
    val deps = AppDependencies(
      orderRepo    = new InMemoryOrderRepo(),
      emailService = new ConsoleEmailService(),
      logger       = new ConsoleLogger()
    )
    
    val result = OrderLogic.cancelOrder(123L).run(deps)
    println(result)
  }
  
  class InMemoryOrderRepo extends OrderRepository {
    private val store = scala.collection.mutable.Map[Long, Order](
      123L -> Order(123L, 1L, 99.99, "pending")
    )
    def findById(id: Long): Option[Order]  = store.get(id)
    def save(order: Order): Order          = { store(order.id) = order; order }
  }
  
  class ConsoleEmailService extends EmailService {
    def sendEmail(to: String, subject: String, body: String): Unit =
      println(s"EMAIL to=$to subject=$subject")
  }
  
  class ConsoleLogger extends Logger {
    def info(msg: String): Unit = println(s"[INFO] $msg")
  }
}
```

---

## Step 925: Tagless Final Pattern

```scala
// TaglessFinal.scala

// Tagless Final: abstract over effect type F[_]
// Allows switching between IO, Future, ZIO, or even Id (for testing)

import scala.concurrent.{ExecutionContext, Future}

// ===== Algebra (interface with F[_]) =====
trait OrderAlgebra[F[_]] {
  def findById(id: Long): F[Option[Order]]
  def save(order: Order): F[Order]
  def findByCustomer(customerId: Long): F[List[Order]]
}

trait EventAlgebra[F[_]] {
  def publish(event: String, payload: String): F[Unit]
}

// ===== Business logic (abstract over F) =====
class OrderService[F[_]](
  orders: OrderAlgebra[F],
  events: EventAlgebra[F]
)(implicit F: cats.Monad[F]) {
  import cats.syntax.flatMap._
  import cats.syntax.functor._
  
  def createOrder(customerId: Long, total: Double): F[Order] = {
    val newOrder = Order(System.currentTimeMillis(), customerId, total, "pending")
    for {
      saved <- orders.save(newOrder)
      _     <- events.publish("order.created", s"""{"id":${saved.id}}""")
    } yield saved
  }
  
  def cancelOrder(id: Long): F[Either[String, Order]] = {
    orders.findById(id).flatMap {
      case None => F.pure(Left(s"Order $id not found"))
      case Some(order) =>
        val cancelled = order.copy(status = "cancelled")
        orders.save(cancelled).flatMap { saved =>
          events.publish("order.cancelled", s"""{"id":${saved.id}}""")
            .map(_ => Right(saved))
        }
    }
  }
}

// ===== Production implementation (Future) =====
class FutureOrderRepo(implicit ec: ExecutionContext) extends OrderAlgebra[Future] {
  private val store = scala.collection.concurrent.TrieMap.empty[Long, Order]
  
  def findById(id: Long): Future[Option[Order]] = Future.successful(store.get(id))
  
  def save(order: Order): Future[Order] = Future.successful {
    store.put(order.id, order); order
  }
  
  def findByCustomer(customerId: Long): Future[List[Order]] = Future.successful {
    store.values.filter(_.customerId == customerId).toList
  }
}

// ===== Test implementation (Id = no effect) =====
type Id[A] = A  // Id is just A, no wrapping

class InMemoryOrderRepo extends OrderAlgebra[Id] {
  private val store = scala.collection.mutable.Map.empty[Long, Order]
  
  def findById(id: Long): Id[Option[Order]]       = store.get(id)
  def save(order: Order): Id[Order]               = { store(order.id) = order; order }
  def findByCustomer(cid: Long): Id[List[Order]]  = store.values.filter(_.customerId == cid).toList
}

// ===== Switch at composition root =====
object ProductionApp {
  import scala.concurrent.ExecutionContext.Implicits.global
  
  val orderRepo: OrderAlgebra[Future] = new FutureOrderRepo()
  val eventPub: EventAlgebra[Future] = new EventAlgebra[Future] {
    def publish(event: String, payload: String): Future[Unit] =
      Future.successful(println(s"EVENT: $event $payload"))
  }
  
  // Use cats.instances.future._ for Monad[Future]
  import cats.instances.future._
  val service = new OrderService[Future](orderRepo, eventPub)
}
```

---

## Step 926: Functional Error Handling

```scala
// FunctionalErrorHandling.scala

// ===== Typed errors with Either =====
sealed trait AppError extends Product with Serializable
case class NotFound(resource: String, id: String)  extends AppError
case class ValidationError(field: String, msg: String) extends AppError
case class DatabaseError(cause: Throwable)          extends AppError
case class ExternalServiceError(service: String, cause: String) extends AppError

type AppResult[A] = Either[AppError, A]

// ===== EitherT for stacking Future + Either =====
import cats.data.EitherT
import scala.concurrent.{ExecutionContext, Future}

type AppResultF[A] = EitherT[Future, AppError, A]

class OrderServiceEither()(implicit ec: ExecutionContext) {
  
  private def findOrderF(id: Long): AppResultF[Order] =
    EitherT(Future.successful(
      if (id > 0) Right(Order(id, 1L, 99.99, "pending"))
      else Left(NotFound("Order", id.toString))
    ))
  
  private def validateOrder(order: Order): AppResultF[Order] =
    EitherT.fromEither(
      if (order.total > 0) Right(order)
      else Left(ValidationError("total", "Must be positive"))
    )
  
  def processOrder(id: Long): AppResultF[String] =
    for {
      order     <- findOrderF(id)
      validated <- validateOrder(order)
    } yield s"Processed order ${validated.id} for total ${validated.total}"
  
  // Convert to plain Future at boundary
  def handleProcessOrder(id: Long): Future[Either[AppError, String]] =
    processOrder(id).value
}

// ===== Validated for accumulating errors =====
import cats.data.{Validated, ValidatedNec}
import cats.syntax.validated._
import cats.syntax.apply._

case class CreateOrderRequest(customerId: Long, total: Double, items: List[String])

object OrderValidator {
  type ValidationResult[A] = ValidatedNec[String, A]
  
  def validateCustomerId(id: Long): ValidationResult[Long] =
    if (id > 0) id.validNec
    else "customerId must be positive".invalidNec
  
  def validateTotal(total: Double): ValidationResult[Double] =
    if (total > 0) total.validNec
    else "total must be positive".invalidNec
  
  def validateItems(items: List[String]): ValidationResult[List[String]] =
    if (items.nonEmpty) items.validNec
    else "items must not be empty".invalidNec
  
  // Accumulate ALL errors (unlike Either which fails fast)
  def validate(req: CreateOrderRequest): ValidationResult[CreateOrderRequest] = (
    validateCustomerId(req.customerId),
    validateTotal(req.total),
    validateItems(req.items)
  ).mapN(CreateOrderRequest)
}

// Usage
object ErrorHandlingDemo {
  def demo(): Unit = {
    val valid = CreateOrderRequest(1L, 99.99, List("product-1"))
    val invalid = CreateOrderRequest(-1L, -5.0, List.empty)
    
    println(OrderValidator.validate(valid))
    // Valid(CreateOrderRequest(1,99.99,List(product-1)))
    
    println(OrderValidator.validate(invalid))
    // Invalid(Chain(customerId must be positive, total must be positive, items must not be empty))
  }
}
```

---

## Step 927: Data Validation with Refined Types

```scala
// RefinedTypes.scala

// Refined types: types that carry compile-time/runtime constraints
// Prevents invalid data from entering the system

// ===== Manual refined types (no library) =====
final class PositiveInt private(val value: Int) extends AnyVal
object PositiveInt {
  def apply(n: Int): Either[String, PositiveInt] =
    if (n > 0) Right(new PositiveInt(n))
    else Left(s"Expected positive int, got $n")
  
  def unsafeApply(n: Int): PositiveInt = apply(n).fold(sys.error, identity)
}

final class NonEmptyString private(val value: String) extends AnyVal
object NonEmptyString {
  def apply(s: String): Either[String, NonEmptyString] =
    if (s.nonEmpty) Right(new NonEmptyString(s))
    else Left("Expected non-empty string")
}

final class Email private(val value: String) extends AnyVal
object Email {
  private val pattern = """^[^@\s]+@[^@\s]+\.[^@\s]+$""".r
  def apply(s: String): Either[String, Email] =
    if (pattern.matches(s)) Right(new Email(s))
    else Left(s"Invalid email: $s")
}

// ===== Domain with refined types =====
case class CreateUserRequest(
  name: NonEmptyString,
  email: Email,
  age: PositiveInt
)

// Parse at boundary, use refined types internally
object UserParser {
  def parse(name: String, email: String, age: Int): Either[List[String], CreateUserRequest] = {
    val nameResult  = NonEmptyString(name).left.map(List(_))
    val emailResult = Email(email).left.map(List(_))
    val ageResult   = PositiveInt(age).left.map(List(_))
    
    (nameResult, emailResult, ageResult) match {
      case (Right(n), Right(e), Right(a)) => Right(CreateUserRequest(n, e, a))
      case _ =>
        val errors = List(nameResult, emailResult, ageResult)
          .collect { case Left(errs) => errs }
          .flatten
        Left(errors)
    }
  }
}
```

---

## Step 928: Functional State Management

```scala
// FunctionalState.scala
import cats.data.State

// ===== State Monad =====
// State[S, A] represents a computation that reads/modifies state S and returns A

case class ShoppingCart(
  items: Map[String, Int],
  discount: Double = 0.0
)

type CartState[A] = State[ShoppingCart, A]

object CartOperations {
  
  def addItem(productId: String, qty: Int): CartState[Unit] =
    State.modify { cart =>
      val current = cart.items.getOrElse(productId, 0)
      cart.copy(items = cart.items + (productId -> (current + qty)))
    }
  
  def removeItem(productId: String): CartState[Boolean] =
    State { cart =>
      if (cart.items.contains(productId))
        (cart.copy(items = cart.items - productId), true)
      else
        (cart, false)
    }
  
  def applyDiscount(pct: Double): CartState[Unit] =
    State.modify(_.copy(discount = pct))
  
  def getTotal(prices: Map[String, Double]): CartState[Double] =
    State.inspect { cart =>
      val subtotal = cart.items.foldLeft(0.0) { case (sum, (id, qty)) =>
        sum + prices.getOrElse(id, 0.0) * qty
      }
      subtotal * (1 - cart.discount)
    }
  
  // Compose operations
  def checkout(prices: Map[String, Double]): CartState[Double] = for {
    _     <- addItem("prod-1", 2)
    _     <- addItem("prod-2", 1)
    _     <- applyDiscount(0.1)  // 10% off
    total <- getTotal(prices)
  } yield total
}

object CartDemo {
  def main(args: Array[String]): Unit = {
    val prices = Map("prod-1" -> 29.99, "prod-2" -> 49.99)
    val program = CartOperations.checkout(prices)
    
    val (finalCart, total) = program.run(ShoppingCart(Map.empty)).value
    println(s"Cart: ${finalCart.items}")
    println(f"Total: $$${total}%.2f (10%% discount applied)")
  }
}
```

---

## Step 929: Functional Configuration

```scala
// FunctionalConfig.scala
import pureconfig._
import pureconfig.generic.auto._

/*
# build.sbt
"com.github.pureconfig" %% "pureconfig" % "0.17.6"
*/

// ===== Typed configuration =====
case class DatabaseConfig(
  host: String,
  port: Int,
  name: String,
  username: String,
  password: String,
  maxPoolSize: Int
)

case class KafkaConfig(
  bootstrapServers: String,
  groupId: String,
  autoOffsetReset: String
)

case class HttpConfig(
  host: String,
  port: Int,
  timeout: Int
)

case class AppConfig(
  db: DatabaseConfig,
  kafka: KafkaConfig,
  http: HttpConfig
)

// application.conf
/*
db {
  host         = "localhost"
  host         = ${?DB_HOST}
  port         = 5432
  port         = ${?DB_PORT}
  name         = "myapp"
  username     = "user"
  password     = "secret"
  password     = ${?DB_PASSWORD}
  max-pool-size = 20
}

kafka {
  bootstrap-servers = "localhost:9092"
  bootstrap-servers = ${?KAFKA_BROKERS}
  group-id          = "order-service"
  auto-offset-reset = "earliest"
}

http {
  host    = "0.0.0.0"
  port    = 8080
  timeout = 30
}
*/

object ConfigLoader {
  import pureconfig.error.ConfigReaderFailures
  
  def load(): Either[ConfigReaderFailures, AppConfig] =
    ConfigSource.default.load[AppConfig]
  
  def loadOrThrow(): AppConfig =
    ConfigSource.default.loadOrThrow[AppConfig]
  
  // Load from specific file (for testing)
  def loadFromFile(path: String): Either[ConfigReaderFailures, AppConfig] =
    ConfigSource.file(path).load[AppConfig]
}
```

---

## Step 930: Functional Testing

```scala
// FunctionalTesting.scala
import org.scalatest.flatspec.AnyFlatSpec
import org.scalatest.matchers.should.Matchers

// ===== Test with pure functions =====
class OrderDomainSpec extends AnyFlatSpec with Matchers {
  
  "OrderDomain.canCancel" should "allow cancelling pending orders" in {
    import domain._
    val order = Order(OrderId(1L), CustomerId(1L), Money(9999), Pending)
    OrderDomain.canCancel(order) shouldBe Right(order.copy(status = Cancelled))
  }
  
  it should "reject cancelling shipped orders" in {
    import domain._
    val order = Order(OrderId(1L), CustomerId(1L), Money(9999), Shipped)
    OrderDomain.canCancel(order).isLeft shouldBe true
  }
}

// ===== Test with tagless final (Id monad) =====
class OrderServiceSpec extends AnyFlatSpec with Matchers {
  
  // Test implementation
  val testRepo = new InMemoryOrderRepo()
  val testEvents = new EventAlgebra[Id] {
    val published = scala.collection.mutable.ListBuffer.empty[(String, String)]
    def publish(event: String, payload: String): Id[Unit] = {
      published += ((event, payload)); ()
    }
  }
  
  import cats.Id
  implicit val monadId: cats.Monad[Id] = cats.catsInstancesForId
  
  val service = new OrderService[Id](testRepo, testEvents)
  
  "OrderService" should "create and retrieve order" in {
    val order = service.createOrder(1L, 99.99)
    order.customerId shouldBe 1L
    order.total shouldBe 99.99
    testEvents.published should have length 1
    testEvents.published.head._1 shouldBe "order.created"
  }
}

// ===== Property-based testing =====
import org.scalacheck.{Arbitrary, Gen, Properties}
import org.scalacheck.Prop.forAll

object OrderPricingSpec extends Properties("PricingService") {
  
  val positiveAmount = Gen.posNum[Double].map(BigDecimal(_))
  val orderItem = for {
    price <- positiveAmount
    qty   <- Gen.choose(1, 100)
  } yield OrderItem(entities.Product(1L, "test", price, 1000), qty)
  
  property("total is sum of subtotals") = forAll(Gen.listOf(orderItem)) { items =>
    val total = domain_services.PricingService.calculateTotal(items.toList)
    val expected = items.map(_.subtotal).sum
    total == expected
  }
  
  property("discount reduces total") = forAll(Gen.listOf(orderItem), Gen.choose(0.0, 0.5)) {
    (items, discount) =>
      val noDiscount = domain_services.PricingService.calculateTotal(items.toList, BigDecimal(0))
      val withDiscount = domain_services.PricingService.calculateTotal(items.toList, BigDecimal(discount))
      withDiscount <= noDiscount
  }
}
```

---

## สรุป Part 93: Functional Architecture

| Pattern | When to Use | Benefit |
|---------|-------------|---------|
| Hexagonal | All services | Testability |
| Tagless Final | Library/framework code | Flexibility |
| Reader Monad | DI without framework | Simplicity |
| State Monad | Complex state transitions | Purity |
| EitherT | Async + typed errors | Safety |
| Refined Types | Domain boundaries | Correctness |

---

## แบบฝึกหัด Part 93

1. **Hexagonal Architecture**: implement Order service ด้วย Hexagonal Architecture: ports `OrderRepository`, `PaymentGateway`, domain logic ที่ pure, infrastructure adapters ที่ pluggable

2. **Tagless Final**: refactor existing `OrderService` เป็น tagless final pattern ที่ abstract over `F[_]`, เขียน test ด้วย `Id`, production ด้วย `Future`

3. **Validated**: implement request validator สำหรับ `CreateOrderRequest` ที่ accumulate errors ทั้งหมด (ไม่ fail fast) และ return `ValidatedNec[AppError, CreateOrderRequest]`

4. **Reader Monad**: implement `Reader[Dependencies, A]` สำหรับ notification service ที่ compose email + SMS + push notifications

5. **Property Testing**: เขียน ScalaCheck properties สำหรับ `PricingService` ที่ verify: discount never increases total, empty items = zero total, total is monotonic in quantities

---

## ไปต่อ: Part 94 — Cats & ZIO
[→ Part 94: Cats and ZIO](./part-94-cats-and-zio.md)
