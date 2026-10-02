# Part 81: Microservices Design — Steps 801-810

## บทนำ: Microservices Architecture

Microservices คือ architectural style ที่ applications ถูกสร้างเป็น collection of small, independently deployable services แต่ละ service มี business capability เฉพาะ

---

## Step 801: Service Decomposition

```scala
// ServiceDecomposition.scala
// แนวทางการแยก services จาก monolith

/*
=== Monolith E-Commerce ===
┌─────────────────────────────────┐
│  ECommerce Application          │
│  - User management              │
│  - Product catalog              │
│  - Order processing             │
│  - Payment                      │
│  - Inventory                    │
│  - Notifications                │
│  - Analytics                    │
└─────────────────────────────────┘

=== Microservices Decomposition ===
┌─────────┐ ┌─────────┐ ┌─────────┐
│ User    │ │Product  │ │ Order   │
│ Service │ │ Service │ │ Service │
└────┬────┘ └────┬────┘ └────┬────┘
     │           │           │
┌────┴──────────────────────┴────┐
│         Message Broker         │
│    (Kafka / RabbitMQ)          │
└────┬──────────────────────┬────┘
     │                      │
┌────┴────┐          ┌──────┴─────┐
│Payment  │          │Notification│
│Service  │          │Service     │
└─────────┘          └────────────┘

Decomposition Strategies:
1. By Business Domain (DDD Bounded Context)
2. By Subdomain
3. By Data ownership
4. By Team structure (Conway's Law)
*/

// ===== Domain Model =====
// User Domain
case class User(
  id: String,
  email: String,
  name: String,
  createdAt: java.time.Instant
)

case class UserProfile(
  userId: String,
  address: Address,
  preferences: Map[String, String]
)

// Product Domain
case class Product(
  id: String,
  name: String,
  price: BigDecimal,
  currency: String,
  category: String,
  stockQuantity: Int
)

// Order Domain
case class OrderItem(
  productId: String,
  quantity: Int,
  unitPrice: BigDecimal
)

case class Order(
  id: String,
  customerId: String,
  items: List[OrderItem],
  status: OrderStatus,
  totalAmount: BigDecimal,
  currency: String,
  createdAt: java.time.Instant
)

sealed trait OrderStatus
object OrderStatus {
  case object Pending   extends OrderStatus
  case object Confirmed extends OrderStatus
  case object Shipped   extends OrderStatus
  case object Delivered extends OrderStatus
  case object Cancelled extends OrderStatus
}

// Payment Domain
case class Payment(
  id: String,
  orderId: String,
  amount: BigDecimal,
  currency: String,
  method: String,
  status: PaymentStatus,
  processedAt: java.time.Instant
)

sealed trait PaymentStatus
object PaymentStatus {
  case object Pending   extends PaymentStatus
  case object Completed extends PaymentStatus
  case object Failed    extends PaymentStatus
  case object Refunded  extends PaymentStatus
}

case class Address(
  street: String,
  city: String,
  country: String,
  postalCode: String
)
```

---

## Step 802: Service API Design

```scala
// ServiceAPI.scala
import akka.actor.typed.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.model._
import akka.http.scaladsl.marshallers.sprayjson.SprayJsonSupport._
import spray.json._
import spray.json.DefaultJsonProtocol._
import scala.concurrent.{Future, ExecutionContext}

// ===== JSON Formats =====
object JsonFormats {
  implicit val productFormat: RootJsonFormat[Product] = jsonFormat6(Product)
  implicit val orderItemFormat: RootJsonFormat[OrderItem] = jsonFormat3(OrderItem)
}

// ===== Service Interfaces =====
trait ProductService {
  def findById(id: String): Future[Option[Product]]
  def list(category: Option[String], page: Int, size: Int): Future[List[Product]]
  def create(product: Product): Future[Product]
  def update(id: String, product: Product): Future[Option[Product]]
  def delete(id: String): Future[Boolean]
  def checkStock(id: String, quantity: Int): Future[Boolean]
}

// ===== REST API Routes =====
class ProductRoutes(service: ProductService)(implicit ec: ExecutionContext) {
  import JsonFormats._
  
  val routes =
    pathPrefix("api" / "v1" / "products") {
      concat(
        // GET /api/v1/products
        (get & pathEnd) {
          parameters("category".optional, "page".as[Int].withDefault(1), "size".as[Int].withDefault(20)) {
            (category, page, size) =>
              onSuccess(service.list(category, page, size)) { products =>
                complete(products)
              }
          }
        },
        
        // GET /api/v1/products/{id}
        (get & path(Segment)) { id =>
          onSuccess(service.findById(id)) {
            case Some(product) => complete(product)
            case None          => complete(StatusCodes.NotFound, s"""{"error":"Product $id not found"}""")
          }
        },
        
        // POST /api/v1/products
        (post & pathEnd & entity(as[Product])) { product =>
          onSuccess(service.create(product)) { created =>
            complete(StatusCodes.Created, created)
          }
        },
        
        // PUT /api/v1/products/{id}
        (put & path(Segment) & entity(as[Product])) { (id, product) =>
          onSuccess(service.update(id, product)) {
            case Some(updated) => complete(updated)
            case None          => complete(StatusCodes.NotFound)
          }
        },
        
        // DELETE /api/v1/products/{id}
        (delete & path(Segment)) { id =>
          onSuccess(service.delete(id)) { deleted =>
            if (deleted) complete(StatusCodes.NoContent)
            else         complete(StatusCodes.NotFound)
          }
        },
        
        // GET /api/v1/products/{id}/stock?quantity=5
        (get & path(Segment / "stock")) { id =>
          parameter("quantity".as[Int]) { qty =>
            onSuccess(service.checkStock(id, qty)) { available =>
              complete(s"""{"available": $available}""")
            }
          }
        }
      )
    }
}
```

---

## Step 803: Inter-service Communication

```scala
// InterServiceComm.scala
import akka.actor.typed.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.model._
import akka.http.scaladsl.unmarshalling.Unmarshal
import spray.json._
import spray.json.DefaultJsonProtocol._
import scala.concurrent.{Future, ExecutionContext}

// ===== HTTP Client สำหรับ Service-to-Service =====
class ServiceClient(baseUrl: String)(implicit system: ActorSystem[_], ec: ExecutionContext) {
  
  def get[T: JsonReader](path: String): Future[T] = {
    Http().singleRequest(HttpRequest(uri = s"$baseUrl$path"))
      .flatMap { response =>
        response.status match {
          case StatusCodes.OK =>
            Unmarshal(response.entity).to[String].map { body =>
              body.parseJson.convertTo[T]
            }
          case status =>
            Future.failed(new RuntimeException(s"Request failed: $status"))
        }
      }
  }
  
  def post[A: JsonWriter, B: JsonReader](path: String, body: A): Future[B] = {
    val request = HttpRequest(
      method = HttpMethods.POST,
      uri    = s"$baseUrl$path",
      entity = HttpEntity(ContentTypes.`application/json`, body.toJson.toString)
    )
    
    Http().singleRequest(request)
      .flatMap { response =>
        Unmarshal(response.entity).to[String].map { body =>
          body.parseJson.convertTo[B]
        }
      }
  }
}

// ===== Circuit Breaker Pattern =====
import akka.pattern.CircuitBreaker
import scala.concurrent.duration._

class ResilientServiceClient(
  baseUrl: String,
  circuitBreaker: CircuitBreaker
)(implicit system: ActorSystem[_], ec: ExecutionContext) {
  
  private val http = new ServiceClient(baseUrl)
  
  def get[T: JsonReader](path: String): Future[T] = {
    circuitBreaker.withCircuitBreaker(http.get[T](path))
  }
}

// สร้าง CircuitBreaker
// val cb = new CircuitBreaker(
//   scheduler     = system.scheduler,
//   maxFailures   = 5,
//   callTimeout   = 5.seconds,
//   resetTimeout  = 30.seconds
// )

// ===== Retry Pattern =====
import akka.stream.scaladsl._
import scala.util.{Try, Success, Failure}

object RetryHelper {
  def retry[T](maxRetries: Int, delay: FiniteDuration)(f: => Future[T])
              (implicit ec: ExecutionContext, scheduler: akka.actor.Scheduler): Future[T] = {
    f.recoverWith { case ex if maxRetries > 0 =>
      println(s"Retry attempt ${maxRetries}, error: ${ex.getMessage}")
      akka.pattern.after(delay, scheduler)(retry(maxRetries - 1, delay)(f))
    }
  }
}

// ===== Event-Driven Communication =====
// Kafka producer สำหรับ publish events
import org.apache.kafka.clients.producer.{KafkaProducer, ProducerRecord}
import java.util.Properties

class EventPublisher(brokers: String) {
  private val props = new Properties()
  props.put("bootstrap.servers", brokers)
  props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer")
  props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer")
  
  private val producer = new KafkaProducer[String, String](props)
  
  def publish(topic: String, key: String, event: String): Unit = {
    producer.send(new ProducerRecord(topic, key, event))
  }
  
  def close(): Unit = producer.close()
}

// Domain events
case class OrderCreatedEvent(orderId: String, customerId: String, totalAmount: Double)
case class OrderStatusChangedEvent(orderId: String, oldStatus: String, newStatus: String)
case class PaymentProcessedEvent(paymentId: String, orderId: String, status: String)
```

---

## Step 804: CQRS Pattern

```scala
// CQRSPattern.scala
import scala.concurrent.{Future, ExecutionContext}
import java.util.concurrent.atomic.AtomicLong

// ===== Command Model =====
sealed trait OrderCommand
case class CreateOrderCommand(customerId: String, items: List[OrderItem]) extends OrderCommand
case class ConfirmOrderCommand(orderId: String) extends OrderCommand
case class CancelOrderCommand(orderId: String, reason: String) extends OrderCommand
case class ShipOrderCommand(orderId: String, trackingNumber: String) extends OrderCommand

// Command results
sealed trait CommandResult
case class CommandSuccess(id: String, message: String) extends CommandResult
case class CommandFailure(error: String) extends CommandResult

// ===== Write Side (Command Handler) =====
class OrderCommandHandler(
  eventStore: EventStore,
  publisher: EventPublisher
)(implicit ec: ExecutionContext) {
  
  def handle(command: OrderCommand): Future[CommandResult] = command match {
    
    case CreateOrderCommand(customerId, items) =>
      val orderId = java.util.UUID.randomUUID().toString
      val totalAmount = items.map(i => i.unitPrice * i.quantity).sum
      
      val order = Order(
        id           = orderId,
        customerId   = customerId,
        items        = items,
        status       = OrderStatus.Pending,
        totalAmount  = totalAmount,
        currency     = "USD",
        createdAt    = java.time.Instant.now()
      )
      
      // Save event
      val event = OrderCreatedEvent(orderId, customerId, totalAmount.toDouble)
      eventStore.save(orderId, event.getClass.getSimpleName, event.toString)
      
      // Publish to Kafka
      publisher.publish("order-events", orderId, event.toString)
      
      Future.successful(CommandSuccess(orderId, "Order created"))
    
    case ConfirmOrderCommand(orderId) =>
      eventStore.save(orderId, "OrderConfirmed", s"""{"orderId":"$orderId"}""")
      publisher.publish("order-events", orderId, s"""{"type":"OrderConfirmed","orderId":"$orderId"}""")
      Future.successful(CommandSuccess(orderId, "Order confirmed"))
    
    case CancelOrderCommand(orderId, reason) =>
      eventStore.save(orderId, "OrderCancelled", s"""{"orderId":"$orderId","reason":"$reason"}""")
      Future.successful(CommandSuccess(orderId, "Order cancelled"))
    
    case ShipOrderCommand(orderId, trackingNumber) =>
      eventStore.save(orderId, "OrderShipped", s"""{"orderId":"$orderId","tracking":"$trackingNumber"}""")
      Future.successful(CommandSuccess(orderId, "Order shipped"))
  }
}

// ===== Event Store =====
class EventStore {
  private val events = scala.collection.mutable.ListBuffer[(String, String, String, java.time.Instant)]()
  
  def save(aggregateId: String, eventType: String, data: String): Unit = {
    events += ((aggregateId, eventType, data, java.time.Instant.now()))
  }
  
  def getEvents(aggregateId: String): List[(String, String, String, java.time.Instant)] = {
    events.filter(_._1 == aggregateId).toList
  }
}

// ===== Query Model (Read Side) =====
// Denormalized read model สำหรับ performance
case class OrderReadModel(
  orderId: String,
  customerId: String,
  customerName: String,
  status: String,
  totalAmount: Double,
  itemCount: Int,
  createdAt: String
)

class OrderQueryRepository {
  // In-memory read store (production ใช้ Elasticsearch, Redis ฯลฯ)
  private val orders = scala.collection.mutable.Map[String, OrderReadModel]()
  
  def findById(orderId: String): Option[OrderReadModel] = orders.get(orderId)
  
  def findByCustomer(customerId: String): List[OrderReadModel] =
    orders.values.filter(_.customerId == customerId).toList
  
  def findByStatus(status: String): List[OrderReadModel] =
    orders.values.filter(_.status == status).toList
  
  // Projections — update read model from events
  def applyOrderCreated(event: OrderCreatedEvent): Unit = {
    orders(event.orderId) = OrderReadModel(
      orderId      = event.orderId,
      customerId   = event.customerId,
      customerName = "Unknown", // join from user service
      status       = "pending",
      totalAmount  = event.totalAmount,
      itemCount    = 0,
      createdAt    = java.time.Instant.now().toString
    )
  }
  
  def applyStatusChanged(event: OrderStatusChangedEvent): Unit = {
    orders.get(event.orderId).foreach { order =>
      orders(event.orderId) = order.copy(status = event.newStatus)
    }
  }
}

// ===== Main =====
object CQRSDemo {
  def main(args: Array[String]): Unit = {
    import scala.concurrent.ExecutionContext.Implicits.global
    
    val eventStore    = new EventStore()
    val publisher     = new EventPublisher("localhost:9092")
    val commandHandler = new OrderCommandHandler(eventStore, publisher)
    val queryRepo     = new OrderQueryRepository()
    
    // Write: create order
    val result = commandHandler.handle(CreateOrderCommand(
      customerId = "C001",
      items = List(OrderItem("P001", 2, BigDecimal(999.99)))
    ))
    
    result.foreach { r =>
      println(s"Command result: $r")
      
      r match {
        case CommandSuccess(orderId, _) =>
          // Update read model
          queryRepo.applyOrderCreated(OrderCreatedEvent(orderId, "C001", 1999.98))
          
          // Read: query
          queryRepo.findById(orderId).foreach { order =>
            println(s"Read model: $order")
          }
        case _ =>
      }
    }
    
    Thread.sleep(1000)
    publisher.close()
  }
}
```

---

## Step 805: Saga Pattern

```scala
// SagaPattern.scala
import scala.concurrent.{Future, ExecutionContext}

// ===== Saga for Order Processing =====
// Orchestrator-based Saga

sealed trait SagaStep
case class ReserveInventory(orderId: String, items: List[OrderItem]) extends SagaStep
case class ProcessPayment(orderId: String, amount: BigDecimal) extends SagaStep
case class ConfirmOrder(orderId: String) extends SagaStep
case class SendConfirmationEmail(orderId: String, customerId: String) extends SagaStep

// Compensating actions
sealed trait CompensatingAction
case class ReleaseInventory(orderId: String) extends CompensatingAction
case class RefundPayment(orderId: String) extends CompensatingAction
case class CancelOrder(orderId: String) extends CompensatingAction

// Saga state
case class SagaState(
  sagaId: String,
  orderId: String,
  completedSteps: List[String],
  status: String // running | completed | compensating | failed
)

class OrderSagaOrchestrator(implicit ec: ExecutionContext) {
  
  def execute(orderId: String, customerId: String, items: List[OrderItem], amount: BigDecimal): Future[String] = {
    val sagaId = java.util.UUID.randomUUID().toString
    println(s"Starting saga $sagaId for order $orderId")
    
    val completedSteps = scala.collection.mutable.ListBuffer[CompensatingAction]()
    
    // Step 1: Reserve inventory
    reserveInventory(orderId, items).flatMap { _ =>
      completedSteps += ReleaseInventory(orderId)
      println(s"[$sagaId] Inventory reserved")
      
      // Step 2: Process payment
      processPayment(orderId, amount).flatMap { _ =>
        completedSteps += RefundPayment(orderId)
        println(s"[$sagaId] Payment processed")
        
        // Step 3: Confirm order
        confirmOrder(orderId).flatMap { _ =>
          completedSteps += CancelOrder(orderId)
          println(s"[$sagaId] Order confirmed")
          
          // Step 4: Send notification
          sendNotification(orderId, customerId).map { _ =>
            println(s"[$sagaId] Notification sent")
            println(s"[$sagaId] Saga completed successfully!")
            sagaId
          }
        }
      }
    }.recoverWith { case ex =>
      println(s"[$sagaId] Step failed: ${ex.getMessage}")
      println(s"[$sagaId] Starting compensation...")
      
      // Compensate in reverse order
      compensate(sagaId, completedSteps.reverse.toList).map { _ =>
        throw ex
      }
    }
  }
  
  private def compensate(sagaId: String, actions: List[CompensatingAction]): Future[Unit] = {
    actions.foldLeft(Future.successful(())) { case (acc, action) =>
      acc.flatMap { _ =>
        action match {
          case ReleaseInventory(orderId) =>
            println(s"[$sagaId] Compensating: releasing inventory for $orderId")
            Future.successful(())
          case RefundPayment(orderId) =>
            println(s"[$sagaId] Compensating: refunding payment for $orderId")
            Future.successful(())
          case CancelOrder(orderId) =>
            println(s"[$sagaId] Compensating: cancelling order $orderId")
            Future.successful(())
        }
      }
    }
  }
  
  // Step implementations (simulate with Future)
  private def reserveInventory(orderId: String, items: List[OrderItem]): Future[Unit] =
    Future.successful(println(s"  Reserving inventory..."))
  
  private def processPayment(orderId: String, amount: BigDecimal): Future[Unit] =
    Future.successful(println(s"  Processing payment of $amount..."))
  
  private def confirmOrder(orderId: String): Future[Unit] =
    Future.successful(println(s"  Confirming order..."))
  
  private def sendNotification(orderId: String, customerId: String): Future[Unit] =
    Future.successful(println(s"  Sending notification to $customerId..."))
}

object SagaDemo {
  def main(args: Array[String]): Unit = {
    import scala.concurrent.ExecutionContext.Implicits.global
    import scala.concurrent.Await
    import scala.concurrent.duration._
    
    val saga = new OrderSagaOrchestrator()
    
    println("=== Happy Path ===")
    val result = Await.result(
      saga.execute("O001", "C001", List(OrderItem("P001", 2, BigDecimal(999.99))), BigDecimal(1999.98)),
      10.seconds
    )
    println(s"Saga completed: $result")
  }
}
```

---

## Step 806: API Gateway Pattern

```scala
// APIGateway.scala
import akka.actor.typed.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.model._
import akka.http.scaladsl.model.headers._
import scala.concurrent.ExecutionContext

// ===== API Gateway =====
// รวม routes จากหลาย microservices และเพิ่ม cross-cutting concerns

class ApiGateway(
  userServiceUrl: String,
  productServiceUrl: String,
  orderServiceUrl: String
)(implicit system: ActorSystem[_], ec: ExecutionContext) {
  
  // ===== Auth Middleware =====
  def authenticated: Directive1[String] = {
    optionalHeaderValueByName("Authorization").flatMap {
      case Some(token) if token.startsWith("Bearer ") =>
        val jwt = token.drop(7)
        validateToken(jwt) match {
          case Some(userId) => provide(userId)
          case None         => complete(StatusCodes.Unauthorized, """{"error":"Invalid token"}""")
        }
      case _ => complete(StatusCodes.Unauthorized, """{"error":"Missing token"}""")
    }
  }
  
  def validateToken(token: String): Option[String] = {
    // ใน production ใช้ JWT library
    if (token == "valid-token") Some("user-123") else None
  }
  
  // ===== Rate Limiting =====
  private val requestCounts = scala.collection.mutable.Map[String, (Long, Int)]()
  val RATE_LIMIT = 100 // requests per minute
  
  def rateLimited(userId: String): Boolean = {
    val now = System.currentTimeMillis() / 60000 // current minute
    val (lastMinute, count) = requestCounts.getOrElse(userId, (0L, 0))
    
    if (lastMinute != now) {
      requestCounts(userId) = (now, 1)
      false
    } else if (count >= RATE_LIMIT) {
      true
    } else {
      requestCounts(userId) = (now, count + 1)
      false
    }
  }
  
  // ===== CORS =====
  val corsHeaders = List(
    `Access-Control-Allow-Origin`.*,
    `Access-Control-Allow-Methods`(HttpMethods.GET, HttpMethods.POST, HttpMethods.PUT, HttpMethods.DELETE),
    `Access-Control-Allow-Headers`("Authorization", "Content-Type")
  )
  
  // ===== Routes =====
  val routes =
    respondWithHeaders(corsHeaders) {
      concat(
        // Health check (no auth)
        path("health") {
          get { complete("""{"status":"healthy"}""") }
        },
        
        // Authenticated routes
        authenticated { userId =>
          if (rateLimited(userId)) {
            complete(StatusCodes.TooManyRequests, """{"error":"Rate limit exceeded"}""")
          } else {
            concat(
              pathPrefix("users")    { proxyTo(userServiceUrl) },
              pathPrefix("products") { proxyTo(productServiceUrl) },
              pathPrefix("orders")   { proxyTo(orderServiceUrl) }
            )
          }
        }
      )
    }
  
  // Proxy request ไปยัง service
  def proxyTo(serviceUrl: String) = extractRequest { request =>
    val forwardUri = s"$serviceUrl${request.uri.path}?${request.uri.rawQueryString.getOrElse("")}"
    
    onSuccess(Http().singleRequest(request.withUri(forwardUri))) { response =>
      complete(response)
    }
  }
}
```

---

## Step 807: Service Discovery

```scala
// ServiceDiscovery.scala
import scala.concurrent.{Future, ExecutionContext}

// ===== Service Registry =====
case class ServiceInstance(
  id: String,
  name: String,
  host: String,
  port: Int,
  metadata: Map[String, String] = Map.empty,
  status: String = "UP"
)

class ServiceRegistry {
  private val services = scala.collection.mutable.Map[String, List[ServiceInstance]]()
  
  def register(instance: ServiceInstance): Unit = {
    val existing = services.getOrElse(instance.name, List.empty)
    services(instance.name) = existing :+ instance
    println(s"Registered: ${instance.name} at ${instance.host}:${instance.port}")
  }
  
  def deregister(instanceId: String): Unit = {
    services.foreach { case (name, instances) =>
      services(name) = instances.filterNot(_.id == instanceId)
    }
  }
  
  def discover(serviceName: String): List[ServiceInstance] = {
    services.getOrElse(serviceName, List.empty)
           .filter(_.status == "UP")
  }
  
  def updateStatus(instanceId: String, status: String): Unit = {
    services.foreach { case (name, instances) =>
      services(name) = instances.map { i =>
        if (i.id == instanceId) i.copy(status = status) else i
      }
    }
  }
}

// ===== Load Balancer =====
trait LoadBalancer {
  def select(instances: List[ServiceInstance]): Option[ServiceInstance]
}

class RoundRobinLoadBalancer extends LoadBalancer {
  private var index = new java.util.concurrent.atomic.AtomicLong(0)
  
  def select(instances: List[ServiceInstance]): Option[ServiceInstance] = {
    if (instances.isEmpty) None
    else {
      val i = (index.getAndIncrement() % instances.size).toInt
      Some(instances(i))
    }
  }
}

class RandomLoadBalancer extends LoadBalancer {
  private val random = new scala.util.Random()
  
  def select(instances: List[ServiceInstance]): Option[ServiceInstance] = {
    if (instances.isEmpty) None
    else Some(instances(random.nextInt(instances.size)))
  }
}

// ===== Service Client with Discovery =====
class DiscoverableServiceClient(
  registry: ServiceRegistry,
  loadBalancer: LoadBalancer
)(implicit ec: ExecutionContext) {
  
  def call(serviceName: String, path: String): Future[String] = {
    registry.discover(serviceName) match {
      case Nil => Future.failed(new RuntimeException(s"No instances for: $serviceName"))
      case instances =>
        loadBalancer.select(instances) match {
          case None => Future.failed(new RuntimeException("No available instance"))
          case Some(instance) =>
            val url = s"http://${instance.host}:${instance.port}$path"
            println(s"Calling $serviceName at $url")
            // HTTP call here
            Future.successful(s"Response from ${instance.id}")
        }
    }
  }
}

object ServiceDiscoveryDemo {
  def main(args: Array[String]): Unit = {
    import scala.concurrent.ExecutionContext.Implicits.global
    import scala.concurrent.Await
    import scala.concurrent.duration._
    
    val registry = new ServiceRegistry()
    val lb = new RoundRobinLoadBalancer()
    
    // Register service instances
    registry.register(ServiceInstance("product-1", "product-service", "10.0.1.1", 8080))
    registry.register(ServiceInstance("product-2", "product-service", "10.0.1.2", 8080))
    registry.register(ServiceInstance("product-3", "product-service", "10.0.1.3", 8080))
    
    // Simulate one instance going down
    registry.updateStatus("product-2", "DOWN")
    
    val client = new DiscoverableServiceClient(registry, lb)
    
    // Make 5 calls — should round-robin between product-1 and product-3
    val calls = (1 to 5).map { i =>
      client.call("product-service", "/api/v1/products")
    }
    
    calls.foreach { f =>
      val result = Await.result(f, 5.seconds)
      println(s"Got: $result")
    }
  }
}
```

---

## Step 808: Health Checks

```scala
// HealthChecks.scala
import akka.actor.typed.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.model._
import scala.concurrent.{Future, ExecutionContext}

// ===== Health Check Types =====
case class HealthStatus(
  status: String,          // healthy | degraded | unhealthy
  checks: Map[String, CheckResult],
  timestamp: String
)

case class CheckResult(
  status: String,
  message: String,
  responseTime: Long
)

// ===== Health Checkers =====
trait HealthChecker {
  def name: String
  def check(): Future[CheckResult]
}

class DatabaseHealthChecker(dbUrl: String)(implicit ec: ExecutionContext) extends HealthChecker {
  def name = "database"
  
  def check(): Future[CheckResult] = {
    val start = System.currentTimeMillis()
    Future {
      // Simulate DB ping
      Thread.sleep(10)
      val elapsed = System.currentTimeMillis() - start
      CheckResult("healthy", "Database is responsive", elapsed)
    }.recover { case ex =>
      CheckResult("unhealthy", ex.getMessage, -1)
    }
  }
}

class KafkaHealthChecker(brokers: String)(implicit ec: ExecutionContext) extends HealthChecker {
  def name = "kafka"
  
  def check(): Future[CheckResult] = {
    val start = System.currentTimeMillis()
    Future {
      // Simulate Kafka check
      Thread.sleep(5)
      val elapsed = System.currentTimeMillis() - start
      CheckResult("healthy", s"Kafka connected to $brokers", elapsed)
    }.recover { case ex =>
      CheckResult("unhealthy", ex.getMessage, -1)
    }
  }
}

class DiskHealthChecker(threshold: Double = 0.9)(implicit ec: ExecutionContext) extends HealthChecker {
  def name = "disk"
  
  def check(): Future[CheckResult] = Future {
    val roots = java.io.File.listRoots()
    val root = roots.headOption.getOrElse(new java.io.File("/"))
    val usedRatio = 1.0 - root.getFreeSpace.toDouble / root.getTotalSpace
    
    if (usedRatio > threshold) {
      CheckResult("unhealthy", s"Disk usage ${(usedRatio * 100).toInt}% > ${(threshold * 100).toInt}%", 0)
    } else {
      CheckResult("healthy", s"Disk usage ${(usedRatio * 100).toInt}%", 0)
    }
  }
}

// ===== Health Endpoint =====
class HealthEndpoint(checkers: List[HealthChecker])(implicit ec: ExecutionContext) {
  
  def checkAll(): Future[HealthStatus] = {
    import scala.concurrent.Future
    
    Future.sequence(checkers.map { checker =>
      checker.check().map(result => checker.name -> result)
    }).map { results =>
      val checksMap = results.toMap
      val overallStatus = if (checksMap.values.forall(_.status == "healthy")) "healthy"
                         else if (checksMap.values.exists(_.status == "unhealthy")) "unhealthy"
                         else "degraded"
      
      HealthStatus(
        status    = overallStatus,
        checks    = checksMap,
        timestamp = java.time.Instant.now().toString
      )
    }
  }
  
  // Kubernetes probes
  def livenessCheck(): Future[Boolean] = Future.successful(true)
  
  def readinessCheck(): Future[Boolean] = {
    checkAll().map(_.status != "unhealthy")
  }
}

object HealthCheckDemo {
  def main(args: Array[String]): Unit = {
    import scala.concurrent.ExecutionContext.Implicits.global
    import scala.concurrent.Await
    import scala.concurrent.duration._
    
    val checkers = List(
      new DatabaseHealthChecker("jdbc:postgresql://localhost:5432/mydb"),
      new KafkaHealthChecker("localhost:9092"),
      new DiskHealthChecker(0.9)
    )
    
    val healthEndpoint = new HealthEndpoint(checkers)
    
    val health = Await.result(healthEndpoint.checkAll(), 10.seconds)
    
    println(s"Overall: ${health.status}")
    health.checks.foreach { case (name, result) =>
      println(s"  $name: ${result.status} (${result.responseTime}ms) - ${result.message}")
    }
  }
}
```

---

## Step 809: Configuration Management

```scala
// application.conf
/*
# Order Service Configuration

app {
  name = "order-service"
  version = "1.2.3"
  environment = ${?APP_ENV}
  environment = "development"
}

server {
  host = "0.0.0.0"
  port = 8080
  port = ${?SERVER_PORT}
}

database {
  url      = "jdbc:postgresql://localhost:5432/orders"
  url      = ${?DB_URL}
  username = "orders_user"
  username = ${?DB_USER}
  password = "secret"
  password = ${?DB_PASSWORD}
  pool {
    min-connections = 5
    max-connections = 20
    connection-timeout = 30s
  }
}

kafka {
  brokers   = "localhost:9092"
  brokers   = ${?KAFKA_BROKERS}
  group-id  = "order-service"
  topics {
    order-events    = "order-events"
    payment-events  = "payment-events"
  }
}

services {
  user-service    = "http://user-service:8081"
  user-service    = ${?USER_SERVICE_URL}
  product-service = "http://product-service:8082"
  product-service = ${?PRODUCT_SERVICE_URL}
}

features {
  enable-new-payment-flow = false
  enable-new-payment-flow = ${?FEATURE_NEW_PAYMENT}
}
*/

// Config loader
import com.typesafe.config.{Config, ConfigFactory}

case class AppConfig(
  name: String,
  version: String,
  environment: String,
  serverPort: Int,
  dbUrl: String,
  kafkaBrokers: String,
  userServiceUrl: String,
  productServiceUrl: String
)

object ConfigLoader {
  
  def load(env: String = "development"): AppConfig = {
    val config = ConfigFactory.load()
      .withFallback(ConfigFactory.parseResources(s"application-$env.conf"))
      .resolve()
    
    AppConfig(
      name             = config.getString("app.name"),
      version          = config.getString("app.version"),
      environment      = config.getString("app.environment"),
      serverPort       = config.getInt("server.port"),
      dbUrl            = config.getString("database.url"),
      kafkaBrokers     = config.getString("kafka.brokers"),
      userServiceUrl   = config.getString("services.user-service"),
      productServiceUrl = config.getString("services.product-service")
    )
  }
}
```

---

## Step 810: Distributed Tracing

```scala
// DistributedTracing.scala
import java.util.UUID

// ===== Simple Tracing (OpenTelemetry-like) =====
case class Span(
  traceId: String,
  spanId: String,
  parentSpanId: Option[String],
  operationName: String,
  startTime: Long,
  endTime: Option[Long],
  tags: Map[String, String],
  status: String
)

class Tracer(serviceName: String) {
  private val spans = scala.collection.mutable.ListBuffer[Span]()
  
  def startSpan(operation: String, parentSpanId: Option[String] = None,
                traceId: String = UUID.randomUUID().toString): Span = {
    val span = Span(
      traceId       = traceId,
      spanId        = UUID.randomUUID().toString,
      parentSpanId  = parentSpanId,
      operationName = operation,
      startTime     = System.currentTimeMillis(),
      endTime       = None,
      tags          = Map("service" -> serviceName),
      status        = "in_progress"
    )
    spans += span
    span
  }
  
  def finishSpan(span: Span, status: String = "ok", tags: Map[String, String] = Map.empty): Span = {
    val finished = span.copy(
      endTime = Some(System.currentTimeMillis()),
      status  = status,
      tags    = span.tags ++ tags
    )
    val idx = spans.indexWhere(_.spanId == span.spanId)
    if (idx >= 0) spans(idx) = finished
    finished
  }
  
  def report(): Unit = {
    println("\n=== Distributed Trace ===")
    spans.sortBy(_.startTime).foreach { span =>
      val duration = span.endTime.map(_ - span.startTime).getOrElse(-1L)
      val indent = if (span.parentSpanId.isDefined) "  " else ""
      println(s"$indent[${span.status}] ${span.operationName} (${duration}ms) traceId=${span.traceId.take(8)}...")
    }
  }
}

// ===== Trace Propagation Context =====
case class TraceContext(traceId: String, spanId: String)

def withTrace[T](tracer: Tracer, operation: String, parentCtx: Option[TraceContext] = None)(
  f: TraceContext => T
): T = {
  val span = tracer.startSpan(operation, parentCtx.map(_.spanId),
                               parentCtx.map(_.traceId).getOrElse(UUID.randomUUID().toString))
  try {
    val ctx = TraceContext(span.traceId, span.spanId)
    val result = f(ctx)
    tracer.finishSpan(span, "ok")
    result
  } catch {
    case ex: Exception =>
      tracer.finishSpan(span, "error", Map("error" -> ex.getMessage))
      throw ex
  }
}

object TracingDemo {
  def main(args: Array[String]): Unit = {
    val tracer = new Tracer("order-service")
    
    // Simulate distributed operation
    withTrace(tracer, "create-order") { ctx =>
      println(s"Processing order with trace ${ctx.traceId}")
      Thread.sleep(10)
      
      withTrace(tracer, "validate-inventory", Some(ctx)) { childCtx =>
        Thread.sleep(20)
        println("Inventory validated")
      }
      
      withTrace(tracer, "process-payment", Some(ctx)) { childCtx =>
        Thread.sleep(50)
        
        withTrace(tracer, "call-payment-gateway", Some(childCtx)) { grandChildCtx =>
          Thread.sleep(30)
          println("Payment gateway called")
        }
      }
      
      withTrace(tracer, "send-notification", Some(ctx)) { childCtx =>
        Thread.sleep(5)
        println("Notification sent")
      }
    }
    
    tracer.report()
  }
}
```

---

## สรุป Part 81: Microservices Design

| Pattern | Problem | Solution |
|---------|---------|----------|
| Decomposition | Monolith too complex | Split by domain |
| CQRS | Read/write scaling | Separate read/write models |
| Saga | Distributed transactions | Orchestrated compensations |
| API Gateway | Client complexity | Single entry point |
| Service Discovery | Dynamic instances | Registry + load balancer |
| Circuit Breaker | Cascade failures | Fail fast |
| Health Check | Kubernetes probes | /health, /ready |

### Communication Patterns
| Pattern | Latency | Coupling | Use Case |
|---------|---------|----------|----------|
| Sync HTTP | Low | Tight | Query operations |
| Async Kafka | Higher | Loose | Commands/events |
| gRPC | Very low | Tight | Internal services |

---

## แบบฝึกหัด Part 81

1. **Domain Decomposition**: แยก monolith e-commerce ออกเป็น 5 microservices พร้อม define bounded contexts, APIs, และ events

2. **CQRS Implementation**: implement CQRS pattern สำหรับ product catalog ที่มี high read/write ratio โดย read model ใช้ Elasticsearch

3. **Saga Orchestrator**: implement Order Processing Saga ที่ handle inventory, payment, shipping ด้วย proper compensation logic

4. **Service Mesh Simulation**: implement service discovery, load balancing, และ circuit breaker โดยไม่ใช้ external library

5. **Health Check System**: สร้าง health check framework ที่ check database, Kafka, external APIs, และ expose Kubernetes-compatible /health และ /ready endpoints

---

## ไปต่อ: Part 82 — Lagom Framework
ใน Part ถัดไปจะเรียน Lagom framework, persistent entities, และ pub/sub

[→ Part 82: Lagom Framework](./part-82-lagom-framework.md)
