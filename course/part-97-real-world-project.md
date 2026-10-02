# Part 97: Real-World E-commerce Project — Steps 961-970

## บทนำ: E-commerce Backend

Real-world e-commerce backend ด้วย Scala ครอบคลุม: Products, Orders, Payments, Users, Authentication — production-grade architecture

```scala
// build.sbt
name := "ecommerce-backend"
scalaVersion := "2.13.14"

val AkkaVersion     = "2.9.3"
val AkkaHttpVersion = "10.6.3"

libraryDependencies ++= Seq(
  "com.typesafe.akka" %% "akka-actor-typed"         % AkkaVersion,
  "com.typesafe.akka" %% "akka-stream"               % AkkaVersion,
  "com.typesafe.akka" %% "akka-http"                 % AkkaHttpVersion,
  "com.typesafe.akka" %% "akka-http-spray-json"      % AkkaHttpVersion,
  "org.tpolecat"      %% "doobie-core"               % "1.0.0-RC4",
  "org.tpolecat"      %% "doobie-hikari"             % "1.0.0-RC4",
  "org.tpolecat"      %% "doobie-postgres"           % "1.0.0-RC4",
  "org.mindrot"        % "jbcrypt"                   % "0.4",
  "com.github.benmanes.caffeine" % "caffeine"        % "3.1.8",
  "io.prometheus"      % "simpleclient"              % "0.16.0",
  "io.prometheus"      % "simpleclient_hotspot"      % "0.16.0"
)
```

---

## Step 961: Domain Model

```scala
// domain/Models.scala
package domain

import java.time.Instant

// ===== User Domain =====
case class UserId(value: Long)   extends AnyVal
case class Email(value: String)  extends AnyVal
case class Username(value: String) extends AnyVal

case class User(
  id: UserId,
  email: Email,
  username: Username,
  passwordHash: String,
  roles: Set[String] = Set("user"),
  createdAt: Instant = Instant.now(),
  updatedAt: Instant = Instant.now()
)

// ===== Product Domain =====
case class ProductId(value: Long) extends AnyVal
case class Money(cents: Long)     extends AnyVal {
  def +(other: Money): Money    = Money(cents + other.cents)
  def *(qty: Int): Money        = Money(cents * qty)
  def toDouble: Double          = cents / 100.0
  override def toString: String = f"${toDouble}%.2f"
}

case class Product(
  id: ProductId,
  name: String,
  description: String,
  price: Money,
  stock: Int,
  category: String,
  imageUrl: Option[String] = None,
  active: Boolean = true
)

// ===== Order Domain =====
case class OrderId(value: Long) extends AnyVal

sealed trait OrderStatus
case object Pending    extends OrderStatus
case object Confirmed  extends OrderStatus
case object Processing extends OrderStatus
case object Shipped    extends OrderStatus
case object Delivered  extends OrderStatus
case object Cancelled  extends OrderStatus
case object Refunded   extends OrderStatus

case class OrderItem(
  productId: ProductId,
  productName: String,
  quantity: Int,
  unitPrice: Money
) {
  def subtotal: Money = unitPrice * quantity
}

case class Order(
  id: OrderId,
  userId: UserId,
  items: List[OrderItem],
  status: OrderStatus,
  shippingAddress: Address,
  createdAt: Instant = Instant.now(),
  updatedAt: Instant = Instant.now()
) {
  def total: Money = items.map(_.subtotal).fold(Money(0))(_ + _)
  def itemCount: Int = items.map(_.quantity).sum
}

case class Address(
  street: String,
  city: String,
  country: String,
  postalCode: String
)

// ===== Payment Domain =====
case class PaymentId(value: String) extends AnyVal

sealed trait PaymentStatus
case object PaymentPending   extends PaymentStatus
case object PaymentSucceeded extends PaymentStatus
case object PaymentFailed    extends PaymentStatus
case object PaymentRefunded  extends PaymentStatus

case class Payment(
  id: PaymentId,
  orderId: OrderId,
  amount: Money,
  status: PaymentStatus,
  gateway: String,
  gatewayRef: Option[String] = None,
  createdAt: Instant = Instant.now()
)

// ===== Domain Errors =====
sealed trait AppError extends Product with Serializable
case class NotFoundError(resource: String, id: String)       extends AppError
case class ValidationError(errors: List[String])             extends AppError
case class AuthenticationError(message: String)              extends AppError
case class AuthorizationError(message: String)               extends AppError
case class ConflictError(message: String)                    extends AppError
case class InternalError(message: String, cause: Throwable)  extends AppError
case class PaymentError(message: String, code: String)       extends AppError
case class InsufficientStockError(productId: ProductId, requested: Int, available: Int) extends AppError
```

---

## Step 962: Repository Layer

```scala
// repository/OrderRepository.scala
package repository

import domain._
import doobie._
import doobie.implicits._
import doobie.postgres.implicits._
import cats.effect.IO
import java.time.Instant

class OrderRepository(xa: Transactor[IO]) {
  
  def findById(id: OrderId): IO[Option[Order]] = {
    sql"""
      SELECT o.id, o.user_id, o.status, o.street, o.city, o.country, o.postal_code, o.created_at, o.updated_at
      FROM orders o
      WHERE o.id = ${id.value}
    """.query[(Long, Long, String, String, String, String, String, Instant, Instant)]
      .option
      .flatMap {
        case None => doobie.free.connection.pure(Option.empty[Order])
        case Some((oid, uid, status, street, city, country, postal, created, updated)) =>
          sql"""
            SELECT product_id, product_name, quantity, unit_price_cents
            FROM order_items WHERE order_id = $oid
          """.query[(Long, String, Int, Long)]
            .to[List]
            .map { items =>
              Some(Order(
                id       = OrderId(oid),
                userId   = UserId(uid),
                items    = items.map { case (pid, name, qty, price) =>
                  OrderItem(ProductId(pid), name, qty, Money(price))
                },
                status   = parseStatus(status),
                shippingAddress = Address(street, city, country, postal),
                createdAt = created,
                updatedAt = updated
              ))
            }
      }
      .transact(xa)
  }
  
  def save(order: Order): IO[Order] = {
    val upsertOrder = sql"""
      INSERT INTO orders (id, user_id, status, street, city, country, postal_code, created_at, updated_at)
      VALUES (${order.id.value}, ${order.userId.value}, ${statusToString(order.status)},
              ${order.shippingAddress.street}, ${order.shippingAddress.city},
              ${order.shippingAddress.country}, ${order.shippingAddress.postalCode},
              ${order.createdAt}, ${order.updatedAt})
      ON CONFLICT (id) DO UPDATE SET
        status     = EXCLUDED.status,
        updated_at = EXCLUDED.updated_at
    """.update.run
    
    val deleteItems = sql"DELETE FROM order_items WHERE order_id = ${order.id.value}".update.run
    
    val insertItems = Update[(Long, Long, String, Int, Long)](
      "INSERT INTO order_items (order_id, product_id, product_name, quantity, unit_price_cents) VALUES (?, ?, ?, ?, ?)"
    ).updateMany(order.items.map(item =>
      (order.id.value, item.productId.value, item.productName, item.quantity, item.unitPrice.cents)
    ))
    
    (upsertOrder *> deleteItems *> insertItems).transact(xa).map(_ => order)
  }
  
  def findByUser(userId: UserId, page: Int, pageSize: Int): IO[List[Order]] = {
    sql"""
      SELECT id FROM orders
      WHERE user_id = ${userId.value}
      ORDER BY created_at DESC
      LIMIT $pageSize OFFSET ${(page - 1) * pageSize}
    """.query[Long].to[List].transact(xa)
      .flatMap(ids => ids.traverse(id => findById(OrderId(id))))
      .map(_.flatten)
  }
  
  private def parseStatus(s: String): OrderStatus = s match {
    case "pending"    => Pending
    case "confirmed"  => Confirmed
    case "processing" => Processing
    case "shipped"    => Shipped
    case "delivered"  => Delivered
    case "cancelled"  => Cancelled
    case "refunded"   => Refunded
    case _            => Pending
  }
  
  private def statusToString(s: OrderStatus): String = s match {
    case Pending    => "pending"
    case Confirmed  => "confirmed"
    case Processing => "processing"
    case Shipped    => "shipped"
    case Delivered  => "delivered"
    case Cancelled  => "cancelled"
    case Refunded   => "refunded"
  }
  
  import cats.implicits._
}
```

---

## Step 963: Service Layer

```scala
// service/OrderService.scala
package service

import domain._
import repository._
import scala.concurrent.{ExecutionContext, Future}
import cats.effect.IO
import cats.implicits._

class OrderService(
  orderRepo: OrderRepository,
  productRepo: ProductRepository,
  paymentService: PaymentService,
  inventoryService: InventoryService,
  eventPublisher: EventPublisher
) {
  
  def createOrder(
    userId: UserId,
    items: List[(ProductId, Int)],  // (productId, quantity)
    shippingAddress: Address
  ): IO[Either[AppError, Order]] = {
    for {
      // 1. Fetch products
      products <- items.traverse { case (pid, _) =>
        productRepo.findById(pid).map(_.toRight(NotFoundError("Product", pid.value.toString)))
      }
      
      // 2. Check errors
      result <- products.sequence match {
        case Left(err)      => IO.pure(Left(err))
        case Right(prods)   =>
          
          // 3. Build order items
          val orderItems = prods.zip(items.map(_._2)).map { case (product, qty) =>
            OrderItem(product.id, product.name, qty, product.price)
          }
          
          // 4. Check stock
          val stockErrors = prods.zip(items.map(_._2)).flatMap { case (product, qty) =>
            if (product.stock < qty) List(InsufficientStockError(product.id, qty, product.stock))
            else Nil
          }
          
          if (stockErrors.nonEmpty) {
            IO.pure(Left(stockErrors.head))
          } else {
            val order = Order(
              id              = OrderId(generateId()),
              userId          = userId,
              items           = orderItems,
              status          = Pending,
              shippingAddress = shippingAddress
            )
            
            for {
              // 5. Reserve inventory
              _     <- inventoryService.reserveItems(order.id, orderItems)
              // 6. Save order
              saved <- orderRepo.save(order)
              // 7. Publish event
              _     <- eventPublisher.publish(OrderCreated(saved))
            } yield Right(saved)
          }
      }
    } yield result
  }
  
  def cancelOrder(orderId: OrderId, userId: UserId, reason: String): IO[Either[AppError, Order]] = {
    orderRepo.findById(orderId).flatMap {
      case None => IO.pure(Left(NotFoundError("Order", orderId.value.toString)))
      
      case Some(order) if order.userId != userId =>
        IO.pure(Left(AuthorizationError("Not your order")))
      
      case Some(order) if !Set(Pending, Confirmed).contains(order.status) =>
        IO.pure(Left(ConflictError(s"Cannot cancel order in ${order.status} status")))
      
      case Some(order) =>
        val cancelled = order.copy(
          status    = Cancelled,
          updatedAt = java.time.Instant.now()
        )
        for {
          saved <- orderRepo.save(cancelled)
          _     <- inventoryService.releaseItems(orderId, order.items)
          _     <- eventPublisher.publish(OrderCancelled(saved, reason))
        } yield Right(saved)
    }
  }
  
  private def generateId(): Long = System.currentTimeMillis()
}

// Event types
sealed trait DomainEvent
case class OrderCreated(order: Order) extends DomainEvent
case class OrderCancelled(order: Order, reason: String) extends DomainEvent
case class OrderShipped(order: Order, trackingId: String) extends DomainEvent

// Service traits
trait ProductRepository {
  def findById(id: ProductId): IO[Option[Product]]
}

trait InventoryService {
  def reserveItems(orderId: OrderId, items: List[OrderItem]): IO[Unit]
  def releaseItems(orderId: OrderId, items: List[OrderItem]): IO[Unit]
}

trait PaymentService {
  def charge(userId: UserId, amount: Money): IO[Either[PaymentError, Payment]]
}

trait EventPublisher {
  def publish(event: DomainEvent): IO[Unit]
}
```

---

## Step 964: HTTP API Layer

```scala
// api/OrderRoutes.scala
package api

import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.server.Route
import akka.http.scaladsl.model.StatusCodes
import spray.json._
import spray.json.DefaultJsonProtocol._
import domain._
import service._
import scala.concurrent.ExecutionContext

class OrderRoutes(orderService: OrderService)(implicit ec: ExecutionContext) {
  
  // JSON formats
  implicit val addressFormat: RootJsonFormat[Address] = jsonFormat4(Address)
  implicit val moneyWriter: JsonWriter[Money] = (m: Money) => JsNumber(m.toDouble)
  implicit val orderItemWriter: JsonWriter[OrderItem] = (i: OrderItem) => JsObject(
    "productId"  -> JsNumber(i.productId.value),
    "name"       -> JsString(i.productName),
    "quantity"   -> JsNumber(i.quantity),
    "unitPrice"  -> JsNumber(i.unitPrice.toDouble),
    "subtotal"   -> JsNumber(i.subtotal.toDouble)
  )
  
  implicit val orderWriter: JsonWriter[Order] = (o: Order) => JsObject(
    "id"              -> JsNumber(o.id.value),
    "userId"          -> JsNumber(o.userId.value),
    "status"          -> JsString(o.status.toString),
    "items"           -> JsArray(o.items.map(_.toJson).toVector),
    "total"           -> JsNumber(o.total.toDouble),
    "itemCount"       -> JsNumber(o.itemCount),
    "shippingAddress" -> o.shippingAddress.toJson,
    "createdAt"       -> JsString(o.createdAt.toString)
  )
  
  val routes: Route = {
    import cats.effect.unsafe.implicits.global
    
    pathPrefix("api" / "v1" / "orders") {
      
      // POST /api/v1/orders
      post {
        entity(as[JsObject]) { body =>
          // Extract auth from header
          optionalHeaderValueByName("X-User-ID") {
            case None => complete(StatusCodes.Unauthorized, """{"error":"Not authenticated"}""")
            case Some(userIdStr) =>
              val userId = UserId(userIdStr.toLong)
              
              // Parse request
              val items = body.fields("items").asInstanceOf[JsArray].elements.map { item =>
                val o = item.asJsObject
                (ProductId(o.fields("productId").convertTo[Long]), o.fields("quantity").convertTo[Int])
              }.toList
              
              val address = body.fields("shippingAddress").convertTo[Address]
              
              val result = orderService.createOrder(userId, items, address).unsafeRunSync()
              
              result match {
                case Right(order)                      => complete(StatusCodes.Created, order.toJson)
                case Left(ValidationError(errors))     => complete(StatusCodes.BadRequest, JsObject("errors" -> JsArray(errors.map(JsString(_)).toVector)))
                case Left(NotFoundError(res, id))      => complete(StatusCodes.NotFound, JsObject("error" -> JsString(s"$res $id not found")))
                case Left(InsufficientStockError(pid, req, avail)) =>
                  complete(StatusCodes.Conflict, JsObject("error" -> JsString(s"Insufficient stock for product ${pid.value}: requested $req, available $avail")))
                case Left(err) => complete(StatusCodes.InternalServerError, JsObject("error" -> JsString(err.toString)))
              }
          }
        }
      } ~
      
      // GET /api/v1/orders/:id
      path(LongNumber) { orderId =>
        get {
          optionalHeaderValueByName("X-User-ID") {
            case None => complete(StatusCodes.Unauthorized)
            case Some(userIdStr) =>
              import cats.effect.unsafe.implicits.global
              orderService.cancelOrder(OrderId(orderId), UserId(userIdStr.toLong), "User requested") match {
                case _ => complete(StatusCodes.OK, "OK")
              }
          }
        }
      }
    }
  }
}
```

---

## Step 965: Authentication Middleware

```scala
// middleware/AuthMiddleware.scala
package middleware

import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.server.{Directive1, Route}
import akka.http.scaladsl.model.StatusCodes
import domain._

case class AuthenticatedUser(userId: UserId, roles: Set[String])

class AuthMiddleware(jwtSecret: String) {
  
  val authenticate: Directive1[AuthenticatedUser] = {
    optionalHeaderValueByName("Authorization").flatMap {
      case Some(auth) if auth.startsWith("Bearer ") =>
        val token = auth.drop(7)
        validateToken(token) match {
          case Right(user) => provide(user)
          case Left(err)   => complete(StatusCodes.Unauthorized, s"""{"error":"$err"}""")
        }
      case _ => complete(StatusCodes.Unauthorized, """{"error":"Missing authorization header"}""")
    }
  }
  
  def requireRole(role: String)(inner: Route): Directive1[AuthenticatedUser] = {
    authenticate.flatMap { user =>
      if (user.roles.contains(role) || user.roles.contains("admin")) provide(user)
      else complete(StatusCodes.Forbidden, s"""{"error":"Requires role: $role"}""").flatMap(_ => provide(user))
    }
  }
  
  private def validateToken(token: String): Either[String, AuthenticatedUser] = {
    // Simplified validation (use proper JWT library in production)
    if (token == "valid-token-123") {
      Right(AuthenticatedUser(UserId(1L), Set("user")))
    } else {
      Left("Invalid token")
    }
  }
}
```

---

## Step 966: Database Schema

```sql
-- migrations/V1__create_tables.sql

CREATE TABLE users (
  id           BIGSERIAL PRIMARY KEY,
  email        VARCHAR(255) UNIQUE NOT NULL,
  username     VARCHAR(100) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  roles        TEXT[] DEFAULT ARRAY['user'],
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  updated_at   TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE products (
  id           BIGSERIAL PRIMARY KEY,
  name         VARCHAR(255) NOT NULL,
  description  TEXT,
  price_cents  BIGINT NOT NULL,
  stock        INTEGER NOT NULL DEFAULT 0,
  category     VARCHAR(100),
  image_url    VARCHAR(500),
  active       BOOLEAN DEFAULT TRUE,
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  updated_at   TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE orders (
  id           BIGINT PRIMARY KEY,
  user_id      BIGINT NOT NULL REFERENCES users(id),
  status       VARCHAR(50) NOT NULL DEFAULT 'pending',
  street       VARCHAR(255),
  city         VARCHAR(100),
  country      VARCHAR(100),
  postal_code  VARCHAR(20),
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  updated_at   TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE order_items (
  id                BIGSERIAL PRIMARY KEY,
  order_id          BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id        BIGINT NOT NULL REFERENCES products(id),
  product_name      VARCHAR(255) NOT NULL,
  quantity          INTEGER NOT NULL,
  unit_price_cents  BIGINT NOT NULL
);

CREATE TABLE payments (
  id           VARCHAR(50) PRIMARY KEY,
  order_id     BIGINT NOT NULL REFERENCES orders(id),
  amount_cents BIGINT NOT NULL,
  status       VARCHAR(50) NOT NULL DEFAULT 'pending',
  gateway      VARCHAR(50) NOT NULL,
  gateway_ref  VARCHAR(255),
  created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_orders_user_id    ON orders(user_id);
CREATE INDEX idx_orders_status     ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at);
CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_products_category ON products(category);
CREATE INDEX idx_products_active   ON products(active);
```

---

## Step 967: Payment Integration

```scala
// service/StripePaymentService.scala
package service

import domain._
import cats.effect.IO
import com.stripe.Stripe
import com.stripe.model.{Charge, PaymentIntent}
import com.stripe.param.PaymentIntentCreateParams

class StripePaymentService(apiKey: String) extends PaymentService {
  
  Stripe.apiKey = apiKey
  
  def charge(userId: UserId, amount: Money): IO[Either[PaymentError, Payment]] = IO {
    try {
      val params = PaymentIntentCreateParams.builder()
        .setAmount(amount.cents)
        .setCurrency("usd")
        .setAutomaticPaymentMethods(
          PaymentIntentCreateParams.AutomaticPaymentMethods.builder()
            .setEnabled(true)
            .build()
        )
        .putMetadata("userId", userId.value.toString)
        .build()
      
      val intent = PaymentIntent.create(params)
      
      Right(Payment(
        id         = PaymentId(intent.getId),
        orderId    = OrderId(0L),  // set by caller
        amount     = amount,
        status     = PaymentPending,
        gateway    = "stripe",
        gatewayRef = Some(intent.getId)
      ))
    } catch {
      case ex: com.stripe.exception.CardException =>
        Left(PaymentError(ex.getMessage, ex.getCode))
      case ex: Exception =>
        Left(PaymentError(ex.getMessage, "INTERNAL_ERROR"))
    }
  }
  
  // Mock for testing
  def chargeMock(userId: UserId, amount: Money): IO[Either[PaymentError, Payment]] = IO.pure {
    if (amount.cents > 0) {
      Right(Payment(
        id         = PaymentId(s"pi_mock_${System.currentTimeMillis()}"),
        orderId    = OrderId(0L),
        amount     = amount,
        status     = PaymentSucceeded,
        gateway    = "mock",
        gatewayRef = Some(s"ch_mock_${System.currentTimeMillis()}")
      ))
    } else {
      Left(PaymentError("Amount must be positive", "INVALID_AMOUNT"))
    }
  }
}
```

---

## Step 968: Testing Strategy

```scala
// test/OrderServiceSpec.scala
package test

import domain._
import service._
import cats.effect.IO
import cats.effect.unsafe.implicits.global
import org.scalatest.flatspec.AnyFlatSpec
import org.scalatest.matchers.should.Matchers

class OrderServiceSpec extends AnyFlatSpec with Matchers {
  
  // Test repositories
  val testProductRepo = new ProductRepository {
    private val products = Map(
      ProductId(1L) -> Product(ProductId(1L), "Widget", "Nice widget", Money(999), 100, "electronics"),
      ProductId(2L) -> Product(ProductId(2L), "Gadget", "Cool gadget", Money(1999), 5, "electronics")
    )
    def findById(id: ProductId): IO[Option[Product]] = IO.pure(products.get(id))
  }
  
  var savedOrders = Map.empty[OrderId, Order]
  val testOrderRepo = new OrderRepository(null) {
    override def save(order: Order): IO[Order] = IO.pure { savedOrders += (order.id -> order); order }
    override def findById(id: OrderId): IO[Option[Order]] = IO.pure(savedOrders.get(id))
    override def findByUser(uid: UserId, page: Int, size: Int): IO[List[Order]] =
      IO.pure(savedOrders.values.filter(_.userId == uid).toList)
  }
  
  val testInventory = new InventoryService {
    def reserveItems(orderId: OrderId, items: List[OrderItem]): IO[Unit] = IO.unit
    def releaseItems(orderId: OrderId, items: List[OrderItem]): IO[Unit] = IO.unit
  }
  
  val testPayment = new PaymentService {
    def charge(userId: UserId, amount: Money): IO[Either[PaymentError, Payment]] =
      IO.pure(Right(Payment(PaymentId("mock"), OrderId(0), amount, PaymentSucceeded, "mock")))
  }
  
  var publishedEvents = List.empty[DomainEvent]
  val testEvents = new EventPublisher {
    def publish(event: DomainEvent): IO[Unit] = IO.pure { publishedEvents = event :: publishedEvents }
  }
  
  val service = new OrderService(testOrderRepo, testProductRepo, testPayment, testInventory, testEvents)
  
  "OrderService.createOrder" should "create a pending order" in {
    val result = service.createOrder(
      UserId(1L),
      List((ProductId(1L), 2)),
      Address("123 Main St", "NYC", "US", "10001")
    ).unsafeRunSync()
    
    result.isRight shouldBe true
    result.map(_.status) shouldBe Right(Pending)
    result.map(_.total) shouldBe Right(Money(1998))
    publishedEvents.head shouldBe a[OrderCreated]
  }
  
  it should "fail when product not found" in {
    val result = service.createOrder(
      UserId(1L),
      List((ProductId(999L), 1)),
      Address("123 Main St", "NYC", "US", "10001")
    ).unsafeRunSync()
    
    result.isLeft shouldBe true
    result.left.get shouldBe a[NotFoundError]
  }
  
  it should "fail when insufficient stock" in {
    val result = service.createOrder(
      UserId(1L),
      List((ProductId(2L), 100)),  // only 5 in stock
      Address("123 Main St", "NYC", "US", "10001")
    ).unsafeRunSync()
    
    result.isLeft shouldBe true
    result.left.get shouldBe a[InsufficientStockError]
  }
}
```

---

## Step 969: Application Server

```scala
// Main.scala
import akka.actor.typed.ActorSystem
import akka.actor.typed.scaladsl.Behaviors
import akka.http.scaladsl.Http
import akka.http.scaladsl.server.Directives._
import cats.effect.unsafe.implicits.global
import doobie.hikari.HikariTransactor
import cats.effect.IO
import scala.concurrent.ExecutionContext

object Main extends App {
  
  implicit val system: ActorSystem[Nothing] = ActorSystem(Behaviors.empty, "ecommerce")
  implicit val ec: ExecutionContext = system.executionContext
  
  // Database
  val transactor = HikariTransactor.newHikariTransactor[IO](
    driverClassName = "org.postgresql.Driver",
    url             = sys.env.getOrElse("DB_URL", "jdbc:postgresql://localhost/ecommerce"),
    user            = sys.env.getOrElse("DB_USER", "postgres"),
    pass            = sys.env.getOrElse("DB_PASSWORD", "postgres"),
    connectEC       = ec
  ).allocated.unsafeRunSync()._1
  
  // Repositories
  val orderRepo   = new repository.OrderRepository(transactor)
  val productRepo = new service.InMemoryProductRepo()
  
  // Services
  val inventoryService  = new service.SimpleInventoryService(productRepo)
  val paymentService    = new service.StripePaymentService(sys.env.getOrElse("STRIPE_KEY", "sk_test_xxx"))
  val eventPublisher    = new service.LogEventPublisher()
  val orderService      = new service.OrderService(orderRepo, productRepo, paymentService, inventoryService, eventPublisher)
  
  // Auth
  val authMiddleware = new middleware.AuthMiddleware(sys.env.getOrElse("JWT_SECRET", "secret"))
  
  // Routes
  val orderRoutes = new api.OrderRoutes(orderService)
  
  val allRoutes = {
    import akka.http.scaladsl.server.Directives._
    orderRoutes.routes ~
    path("health") { get { complete("""{"status":"healthy","service":"ecommerce"}""") } }
  }
  
  Http().newServerAt("0.0.0.0", 8080).bind(allRoutes).foreach { binding =>
    println(s"E-commerce API started on ${binding.localAddress}")
  }
  
  sys.addShutdownHook {
    system.terminate()
  }
}
```

---

## Step 970: Production Deployment

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      DB_URL: jdbc:postgresql://postgres:5432/ecommerce
      DB_USER: postgres
      DB_PASSWORD: ${DB_PASSWORD}
      JWT_SECRET: ${JWT_SECRET}
      STRIPE_KEY: ${STRIPE_KEY}
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1.0'
          memory: 1G

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ecommerce
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5

volumes:
  postgres_data:
```

---

## สรุป Part 97: Real-World E-commerce

| Layer | Technology | Purpose |
|-------|------------|---------|
| API | Akka HTTP | HTTP endpoints |
| Service | Pure Scala | Business logic |
| Repository | Doobie | DB access |
| DB | PostgreSQL | Persistence |
| Auth | JWT + BCrypt | Security |
| Payment | Stripe | Payments |
| Events | Kafka | Async integration |

---

## แบบฝึกหัด Part 97

1. **Product Service**: implement complete ProductService ด้วย CRUD operations, search by category, และ inventory management

2. **User Registration**: implement user registration flow: validate email uniqueness, hash password, generate JWT, return profile

3. **Order Checkout**: implement complete checkout flow: validate items, reserve inventory, charge payment, confirm order

4. **Integration Tests**: เขียน integration tests ที่ใช้ Testcontainers สำหรับ PostgreSQL จริง ทดสอบ complete order flow

5. **Admin Dashboard**: implement admin endpoints สำหรับ: list all orders, update order status, view revenue by date range

---

## ไปต่อ: Part 98 — Real-World Data Platform
[→ Part 98: Real-World Data Platform](./part-98-real-world-data-platform.md)
