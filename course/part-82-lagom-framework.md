# Part 82: Lagom Framework — Steps 811-820

## บทนำ: Lagom

Lagom เป็น opinionated microservices framework สำหรับ Scala/Java ที่สร้างบน Akka, Play, และ Kafka โดยมี built-in support สำหรับ CQRS, Event Sourcing, และ reactive patterns

---

## Step 811: Lagom Project Setup

```scala
// build.sbt สำหรับ Lagom
lazy val root = (project in file("."))
  .aggregate(
    `hello-api`,
    `hello-impl`,
    `order-api`,
    `order-impl`
  )

val lagomVersion = "1.6.7"

lazy val `hello-api` = (project in file("hello-api"))
  .settings(
    libraryDependencies ++= Seq(
      lagomScaladslApi
    )
  )

lazy val `hello-impl` = (project in file("hello-impl"))
  .enablePlugins(LagomScala)
  .settings(
    libraryDependencies ++= Seq(
      lagomScaladslPersistenceCassandra,
      lagomScaladslKafkaBroker,
      lagomScaladslTestKit
    )
  )
  .settings(lagomForkedTestSettings)
  .dependsOn(`hello-api`)

lazy val `order-api` = (project in file("order-api"))
  .settings(
    libraryDependencies ++= Seq(lagomScaladslApi)
  )

lazy val `order-impl` = (project in file("order-impl"))
  .enablePlugins(LagomScala)
  .settings(
    libraryDependencies ++= Seq(
      lagomScaladslPersistenceCassandra,
      lagomScaladslKafkaBroker
    )
  )
  .dependsOn(`order-api`)
```

---

## Step 812: Service API Definition

```scala
// hello-api/src/main/scala/com/example/hello/api/HelloService.scala
package com.example.hello.api

import akka.{Done, NotUsed}
import com.lightbend.lagom.scaladsl.api._
import com.lightbend.lagom.scaladsl.api.broker.Topic
import com.lightbend.lagom.scaladsl.api.broker.kafka.{KafkaProperties, PartitionKeyStrategy}
import play.api.libs.json.{Format, Json}

// ===== Data Models =====
case class GreetingMessage(message: String)
object GreetingMessage {
  implicit val format: Format[GreetingMessage] = Json.format[GreetingMessage]
}

case class UserGreeting(id: String, message: String)
object UserGreeting {
  implicit val format: Format[UserGreeting] = Json.format[UserGreeting]
}

// ===== Events (Published to Kafka) =====
sealed trait GreetingEvent
case class GreetingChanged(id: String, message: String) extends GreetingEvent
object GreetingChanged {
  implicit val format: Format[GreetingChanged] = Json.format[GreetingChanged]
}

// ===== Service Interface =====
trait HelloService extends Service {
  
  // GET /api/hello/:id
  def hello(id: String): ServiceCall[NotUsed, String]
  
  // POST /api/hello/:id
  def useGreeting(id: String): ServiceCall[GreetingMessage, Done]
  
  // Topic subscription
  def greetingsTopic(): Topic[GreetingChanged]
  
  // Service descriptor
  override final def descriptor: Descriptor = {
    import Service._
    named("hello")
      .withCalls(
        pathCall("/api/hello/:id", hello _),
        pathCall("/api/hello/:id", useGreeting _)
      )
      .withTopics(
        topic("greeting-changed", greetingsTopic _)
          .addProperty(
            KafkaProperties.partitionKeyStrategy,
            PartitionKeyStrategy[GreetingChanged](_.id)
          )
      )
      .withAutoAcl(true)
  }
}
```

---

## Step 813: Persistent Entity

```scala
// hello-impl/src/main/scala/com/example/hello/impl/HelloEntity.scala
package com.example.hello.impl

import akka.actor.typed.Behavior
import akka.cluster.sharding.typed.scaladsl._
import akka.persistence.typed.PersistenceId
import akka.persistence.typed.scaladsl.{Effect, EventSourcedBehavior, ReplyEffect, RetentionCriteria}
import play.api.libs.json._
import com.lightbend.lagom.scaladsl.persistence._

// ===== Commands =====
sealed trait HelloCommand

case class Hello(id: String, replyTo: akka.actor.typed.ActorRef[String])
    extends HelloCommand

case class UseGreetingMessage(
    message: String,
    replyTo: akka.actor.typed.ActorRef[akka.Done]
) extends HelloCommand

// ===== Events =====
sealed trait HelloEvent extends AggregateEvent[HelloEvent] {
  override def aggregateTag: AggregateEventShards[HelloEvent] = HelloEvent.Tag
}

object HelloEvent {
  val Tag: AggregateEventShards[HelloEvent] =
    AggregateEventTag.sharded[HelloEvent](numShards = 10)
}

case class GreetingMessageChanged(message: String) extends HelloEvent

object GreetingMessageChanged {
  implicit val format: Format[GreetingMessageChanged] = Json.format[GreetingMessageChanged]
}

// ===== State =====
case class HelloState(message: String, timestamp: String) {
  
  def applyEvent(event: HelloEvent): HelloState = event match {
    case GreetingMessageChanged(msg) =>
      copy(message = msg, timestamp = java.time.Instant.now().toString)
  }
}

object HelloState {
  val initial: HelloState = HelloState("Hello", java.time.Instant.now().toString)
  implicit val format: Format[HelloState] = Json.format[HelloState]
}

// ===== Entity Behavior =====
object HelloBehavior {
  
  def create(persistenceId: PersistenceId): Behavior[HelloCommand] =
    EventSourcedBehavior
      .withEnforcedReplies[HelloCommand, HelloEvent, HelloState](
        persistenceId  = persistenceId,
        emptyState     = HelloState.initial,
        commandHandler = (state, cmd) => handleCommand(state, cmd),
        eventHandler   = (state, evt) => state.applyEvent(evt)
      )
      .withRetention(RetentionCriteria.snapshotEvery(numberOfEvents = 100, keepNSnapshots = 2))
  
  private def handleCommand(state: HelloState, cmd: HelloCommand): ReplyEffect[HelloEvent, HelloState] =
    cmd match {
      case Hello(id, replyTo) =>
        Effect.reply(replyTo)(s"${state.message}, $id!")
      
      case UseGreetingMessage(message, replyTo) =>
        Effect
          .persist(GreetingMessageChanged(message))
          .thenReply(replyTo)(_ => akka.Done)
    }
}
```

---

## Step 814: Service Implementation

```scala
// hello-impl/src/main/scala/com/example/hello/impl/HelloServiceImpl.scala
package com.example.hello.impl

import akka.{Done, NotUsed}
import com.lightbend.lagom.scaladsl.api.ServiceCall
import com.lightbend.lagom.scaladsl.api.broker.Topic
import com.lightbend.lagom.scaladsl.broker.TopicProducer
import com.lightbend.lagom.scaladsl.persistence.{EventStreamElement, PersistentEntityRegistry}
import com.example.hello.api._
import scala.concurrent.ExecutionContext

class HelloServiceImpl(
  persistentEntityRegistry: PersistentEntityRegistry
)(implicit ec: ExecutionContext) extends HelloService {
  
  override def hello(id: String): ServiceCall[NotUsed, String] = ServiceCall { _ =>
    // Get entity ref
    val ref = persistentEntityRegistry.refFor[HelloEntity](id)
    
    // Send command
    ref.ask(Hello(id, _))
  }
  
  override def useGreeting(id: String): ServiceCall[GreetingMessage, Done] = ServiceCall { request =>
    val ref = persistentEntityRegistry.refFor[HelloEntity](id)
    ref.ask(UseGreetingMessage(request.message, _))
  }
  
  // Publish events to Kafka topic
  override def greetingsTopic(): Topic[GreetingChanged] =
    TopicProducer.singleStreamWithOffset { fromOffset =>
      persistentEntityRegistry.eventStream(HelloEvent.Tag, fromOffset)
        .map { case EventStreamElement(id, GreetingMessageChanged(body), offset) =>
          (GreetingChanged(id, body), offset)
        }
    }
}
```

---

## Step 815: Application Loader

```scala
// hello-impl/src/main/scala/com/example/hello/impl/HelloApplication.scala
package com.example.hello.impl

import com.lightbend.lagom.scaladsl.api.ServiceLocator
import com.lightbend.lagom.scaladsl.api.ServiceLocator.NoServiceLocator
import com.lightbend.lagom.scaladsl.broker.kafka.LagomKafkaComponents
import com.lightbend.lagom.scaladsl.devmode.LagomDevModeComponents
import com.lightbend.lagom.scaladsl.persistence.cassandra.CassandraPersistenceComponents
import com.lightbend.lagom.scaladsl.server._
import com.softwaremill.macwire._
import play.api.libs.ws.ahc.AhcWSComponents
import com.example.hello.api.HelloService

class HelloLoader extends LagomApplicationLoader {
  
  override def load(context: LagomApplicationContext): LagomApplication =
    new HelloApplication(context) {
      override def serviceLocator: ServiceLocator = NoServiceLocator
    }
  
  override def loadDevMode(context: LagomApplicationContext): LagomApplication =
    new HelloApplication(context) with LagomDevModeComponents
  
  override def describeService: Some[Descriptor] = Some(readDescriptor[HelloService])
}

abstract class HelloApplication(context: LagomApplicationContext)
    extends LagomApplication(context)
    with CassandraPersistenceComponents
    with LagomKafkaComponents
    with AhcWSComponents {
  
  override lazy val lagomServer: LagomServer =
    serverFor[HelloService](wire[HelloServiceImpl])
  
  // Register entity
  persistentEntityRegistry.register(wire[HelloEntity])
  
  // Serializer registry
  override lazy val jsonSerializerRegistry: JsonSerializerRegistry =
    HelloSerializerRegistry
}

// Serializer registry
import com.lightbend.lagom.scaladsl.playjson.{JsonSerializer, JsonSerializerRegistry}
import scala.collection.immutable.Seq

object HelloSerializerRegistry extends JsonSerializerRegistry {
  override def serializers: Seq[JsonSerializer[_]] = Seq(
    JsonSerializer[GreetingMessageChanged],
    JsonSerializer[HelloState]
  )
}
```

---

## Step 816: Testing Lagom Services

```scala
// hello-impl/src/test/scala/com/example/hello/impl/HelloServiceSpec.scala
package com.example.hello.impl

import akka.NotUsed
import com.example.hello.api._
import com.lightbend.lagom.scaladsl.server.LocalServiceLocator
import com.lightbend.lagom.scaladsl.testkit.ServiceTest
import org.scalatest.matchers.should.Matchers
import org.scalatest.wordspec.AsyncWordSpec
import org.scalatest.BeforeAndAfterAll

class HelloServiceSpec extends AsyncWordSpec with Matchers with BeforeAndAfterAll {
  
  private val server = ServiceTest.startServer(ServiceTest.defaultSetup.withClusters()) { ctx =>
    new HelloApplication(ctx) with LocalServiceLocator
  }
  
  private val client = server.serviceClient.implement[HelloService]
  
  override def afterAll(): Unit = server.stop()
  
  "HelloService" should {
    
    "return default greeting" in {
      client.hello("Alice").invoke().map { answer =>
        answer should ===("Hello, Alice!")
      }
    }
    
    "return custom greeting after change" in {
      for {
        _ <- client.useGreeting("Bob").invoke(GreetingMessage("Hi"))
        answer <- client.hello("Bob").invoke()
      } yield {
        answer should ===("Hi, Bob!")
      }
    }
    
    "return greeting per entity" in {
      for {
        _      <- client.useGreeting("Charlie").invoke(GreetingMessage("Sawadee"))
        answer <- client.hello("Charlie").invoke()
      } yield {
        answer should ===("Sawadee, Charlie!")
      }
    }
  }
}
```

---

## Step 817: Inter-service Communication in Lagom

```scala
// order-api/src/main/scala/com/example/order/api/OrderService.scala
package com.example.order.api

import akka.{Done, NotUsed}
import com.lightbend.lagom.scaladsl.api._
import com.lightbend.lagom.scaladsl.api.broker.Topic
import play.api.libs.json._

case class CreateOrderRequest(
  customerId: String,
  items: List[OrderItemRequest]
)
object CreateOrderRequest {
  implicit val format: Format[CreateOrderRequest] = Json.format[CreateOrderRequest]
}

case class OrderItemRequest(productId: String, quantity: Int)
object OrderItemRequest {
  implicit val format: Format[OrderItemRequest] = Json.format[OrderItemRequest]
}

case class OrderResponse(
  orderId: String,
  status: String,
  total: Double
)
object OrderResponse {
  implicit val format: Format[OrderResponse] = Json.format[OrderResponse]
}

sealed trait OrderEvent
case class OrderCreated(orderId: String, customerId: String) extends OrderEvent
case class OrderCompleted(orderId: String) extends OrderEvent

object OrderCreated {
  implicit val format: Format[OrderCreated] = Json.format[OrderCreated]
}

trait OrderService extends Service {
  
  def createOrder: ServiceCall[CreateOrderRequest, OrderResponse]
  def getOrder(id: String): ServiceCall[NotUsed, OrderResponse]
  def cancelOrder(id: String): ServiceCall[NotUsed, Done]
  
  def orderEvents(): Topic[OrderEvent]
  
  override def descriptor: Descriptor = {
    import Service._
    named("order-service")
      .withCalls(
        pathCall("/api/orders", createOrder),
        pathCall("/api/orders/:id", getOrder _),
        pathCall("/api/orders/:id/cancel", cancelOrder _)
      )
      .withTopics(
        topic("order-events", orderEvents _)
      )
      .withAutoAcl(true)
  }
}
```

```scala
// order-impl/src/main/scala/com/example/order/impl/OrderServiceImpl.scala
package com.example.order.impl

import akka.{Done, NotUsed}
import com.example.hello.api.{HelloService, GreetingMessage}
import com.example.order.api._
import com.lightbend.lagom.scaladsl.api.ServiceCall
import com.lightbend.lagom.scaladsl.api.broker.Topic
import com.lightbend.lagom.scaladsl.broker.TopicProducer
import com.lightbend.lagom.scaladsl.persistence.PersistentEntityRegistry
import scala.concurrent.{ExecutionContext, Future}

// Calling another Lagom service
class OrderServiceImpl(
  persistentEntityRegistry: PersistentEntityRegistry,
  helloService: HelloService // inject other service
)(implicit ec: ExecutionContext) extends OrderService {
  
  override def createOrder: ServiceCall[CreateOrderRequest, OrderResponse] = ServiceCall { request =>
    val orderId = java.util.UUID.randomUUID().toString
    
    // Call HelloService (another microservice)
    helloService.hello(request.customerId).invoke().flatMap { greeting =>
      println(s"Creating order for customer: $greeting")
      
      // Create order entity
      val ref = persistentEntityRegistry.refFor[OrderEntity](orderId)
      
      ref.ask(CreateOrderCommand(
        orderId    = orderId,
        customerId = request.customerId,
        items      = request.items.map(i => OrderItemData(i.productId, i.quantity, 100.0)) // fake price
      )).map { _ =>
        OrderResponse(orderId, "pending", request.items.map(_.quantity * 100.0).sum)
      }
    }
  }
  
  override def getOrder(id: String): ServiceCall[NotUsed, OrderResponse] = ServiceCall { _ =>
    val ref = persistentEntityRegistry.refFor[OrderEntity](id)
    ref.ask(GetOrderCommand(_)).map {
      case Some(state) => OrderResponse(id, state.status, state.total)
      case None        => throw com.lightbend.lagom.scaladsl.api.transport.NotFound(s"Order $id not found")
    }
  }
  
  override def cancelOrder(id: String): ServiceCall[NotUsed, Done] = ServiceCall { _ =>
    val ref = persistentEntityRegistry.refFor[OrderEntity](id)
    ref.ask(CancelOrderCommand(_))
  }
  
  // Subscribe to HelloService events
  def subscribeToGreetingChanges(): Unit = {
    helloService.greetingsTopic().subscribe
      .atLeastOnce {
        akka.stream.scaladsl.Flow[com.example.hello.api.GreetingChanged].map { event =>
          println(s"Greeting changed for ${event.id}: ${event.message}")
          Done
        }
      }
  }
  
  override def orderEvents(): Topic[OrderEvent] =
    TopicProducer.singleStreamWithOffset { fromOffset =>
      persistentEntityRegistry.eventStream(OrderEvent.Tag, fromOffset)
        .collect {
          case akka.stream.scaladsl.Source =>
            ??? // map events to API events
        }
    }
}
```

---

## Step 818: Lagom Read-Side Processor

```scala
// OrderReadSideProcessor.scala
package com.example.order.impl

import akka.Done
import com.lightbend.lagom.scaladsl.persistence._
import com.lightbend.lagom.scaladsl.persistence.cassandra._
import com.datastax.oss.driver.api.core.cql.BoundStatement
import scala.concurrent.{ExecutionContext, Future}

class OrderReadSideProcessor(
  readSide: CassandraReadSide
)(implicit ec: ExecutionContext) extends ReadSideProcessor[OrderEvent] {
  
  override def buildHandler(): ReadSideProcessor.ReadSideHandler[OrderEvent] =
    readSide.builder[OrderEvent]("order-read-side")
      .setGlobalPrepare(createTable)
      .setPrepare(_ => prepareStatements)
      .setEventHandler[OrderCreatedEvent](processOrderCreated)
      .setEventHandler[OrderStatusChangedEvent](processStatusChanged)
      .build()
  
  override def aggregateTags: Set[AggregateEventTag[OrderEvent]] = OrderEvent.Tag.allTags
  
  // ===== CQL Statements =====
  private val CREATE_TABLE = """
    CREATE TABLE IF NOT EXISTS order_summary (
      order_id text PRIMARY KEY,
      customer_id text,
      status text,
      total double,
      created_at timestamp,
      updated_at timestamp
    )
  """
  
  private var insertOrderStatement: BoundStatement = _
  private var updateStatusStatement: BoundStatement = _
  
  private def createTable(session: CassandraSession): Future[Done] = {
    session.executeCreateTable(CREATE_TABLE)
  }
  
  private def prepareStatements(session: CassandraSession): Future[Done] = {
    for {
      insert <- session.prepare("INSERT INTO order_summary (order_id, customer_id, status, total, created_at) VALUES (?, ?, ?, ?, ?)")
      update <- session.prepare("UPDATE order_summary SET status = ?, updated_at = ? WHERE order_id = ?")
    } yield {
      insertOrderStatement = insert.bind()
      updateStatusStatement = update.bind()
      Done
    }
  }
  
  private def processOrderCreated(
    session: CassandraSession,
    event: EventStreamElement[OrderCreatedEvent]
  ): Future[List[BoundStatement]] = {
    val e = event.event
    Future.successful(List(
      insertOrderStatement
        .setString("order_id",    e.orderId)
        .setString("customer_id", e.customerId)
        .setString("status",      "pending")
        .setDouble("total",       e.total)
        .setInstant("created_at", java.time.Instant.now())
    ))
  }
  
  private def processStatusChanged(
    session: CassandraSession,
    event: EventStreamElement[OrderStatusChangedEvent]
  ): Future[List[BoundStatement]] = {
    val e = event.event
    Future.successful(List(
      updateStatusStatement
        .setString("status",     e.newStatus)
        .setInstant("updated_at", java.time.Instant.now())
        .setString("order_id",   e.orderId)
    ))
  }
}
```

---

## Step 819: Lagom Pub/Sub

```scala
// PubSubExample.scala
package com.example.hello.impl

import akka.stream.scaladsl.Source
import com.lightbend.lagom.scaladsl.pubsub.{PubSubRegistry, TopicId}
import com.example.hello.api.GreetingMessage
import scala.concurrent.ExecutionContext

// ===== In-cluster Pub/Sub =====
class GreetingEventService(pubSub: PubSubRegistry)(implicit ec: ExecutionContext) {
  
  // Create topic
  private val greetingTopic = pubSub.refFor(TopicId[GreetingMessage])
  
  def publishGreeting(greeting: GreetingMessage): Unit = {
    greetingTopic ! greeting
  }
  
  def subscribeToGreetings(): Source[GreetingMessage, _] = {
    greetingTopic.subscriber
  }
}

// ===== Streaming Service Call =====
// hello-api/src/main/scala/com/example/hello/api/HelloService.scala (extended)
import akka.stream.scaladsl.Source

// Streaming endpoint
// def greetingStream: ServiceCall[Source[String, NotUsed], Source[String, NotUsed]]

// Service descriptor addition:
// namedCall("/api/hello/stream", greetingStream)

// ===== WebSocket-like Streaming =====
class StreamingHelloServiceImpl extends HelloService {
  
  def greetingStream = ServiceCall[Source[String, NotUsed], Source[String, NotUsed]] { request =>
    import scala.concurrent.Future
    Future.successful(request.map(name => s"Hello, $name!"))
  }
}
```

---

## Step 820: Production Config

```hocon
# application.conf สำหรับ Lagom Production

play.application.loader = com.example.hello.impl.HelloLoader

# Cassandra
cassandra.default {
  contact-points = ["cassandra1:9042", "cassandra2:9042"]
  keyspace = "hello_keyspace"
  authentication {
    username = ${?CASSANDRA_USER}
    password = ${?CASSANDRA_PASSWORD}
  }
  ssl.enabled = true
}

# Kafka
lagom.broker.kafka {
  brokers = ${?KAFKA_SERVICE_URI}
  brokers = "kafka:9092"
  
  client {
    default {
      kafka-clients {
        security.protocol = SASL_SSL
        sasl.mechanism = SCRAM-SHA-256
        sasl.jaas.config = ${?KAFKA_SASL_JAAS_CONFIG}
      }
    }
  }
}

# Akka Cluster
akka {
  cluster.seed-nodes = [
    "akka://application@10.0.1.1:2551",
    "akka://application@10.0.1.2:2551"
  ]
  
  remote.artery {
    canonical.hostname = ${?POD_IP}
    canonical.port = 2551
  }
}

# Lagom service locator (production uses Kubernetes/Consul)
lagom.service-locator {
  # Use Kubernetes DNS-based discovery
  type = "kubernetes"
}
```

```scala
// HelloEntity.scala (simplified version for demo)
package com.example.hello.impl

import com.lightbend.lagom.scaladsl.persistence._
import play.api.libs.json._

class HelloEntity extends PersistentEntity {
  override type Command = HelloCommand
  override type Event   = HelloEvent
  override type State   = HelloState
  
  override def initialState: HelloState = HelloState.initial
  
  override def behavior: Behavior = {
    case _ =>
      Actions()
        .onReadOnlyCommand[Hello, String] {
          case (Hello(id, _), ctx, state) =>
            ctx.reply(s"${state.message}, $id!")
        }
        .onCommand[UseGreetingMessage, akka.Done] {
          case (UseGreetingMessage(message, _), ctx, _) =>
            ctx.thenPersist(GreetingMessageChanged(message)) { _ =>
              ctx.reply(akka.Done)
            }
        }
        .onEvent {
          case (GreetingMessageChanged(message), state) =>
            state.copy(message = message, timestamp = java.time.Instant.now().toString)
        }
  }
}
```

---

## สรุป Part 82: Lagom Framework

| Component | Purpose | Implementation |
|-----------|---------|----------------|
| Service API | Interface definition | `trait XxxService extends Service` |
| Service Impl | Business logic | `class XxxServiceImpl extends XxxService` |
| Persistent Entity | Event sourcing | Extends `PersistentEntity` |
| Read Side | Query model | `ReadSideProcessor` + Cassandra/Elasticsearch |
| Topic | Event publishing | `TopicProducer.singleStreamWithOffset` |
| Pub/Sub | In-cluster events | `PubSubRegistry` |

### Lagom vs Plain Akka+Play

| Feature | Lagom | Plain Akka+Play |
|---------|-------|-----------------|
| Setup | Convention-based | Manual wiring |
| CQRS | Built-in | Custom |
| Dev mode | Hot reload | Manual |
| Service Discovery | Built-in | Custom |
| Kafka | Integrated | Manual |

---

## แบบฝึกหัด Part 82

1. **Hello World Service**: สร้าง Lagom service ที่รับ name และ language แล้ว return greeting ในภาษานั้น ใช้ Persistent Entity เก็บ preferred language per user

2. **Event Sourcing**: implement Shopping Cart ด้วย Lagom Persistent Entity ที่มี AddItem, RemoveItem, Checkout commands และ event replay

3. **Read Side**: สร้าง read-side processor ที่ project order events ไปยัง Cassandra table สำหรับ dashboard queries

4. **Service Communication**: สร้าง 2 services ที่ communicate กันทั้งแบบ synchronous (service call) และ asynchronous (Kafka topic)

5. **Integration Test**: เขียน full integration test สำหรับ Lagom service ที่ test API, persistence, และ topic publishing

---

## ไปต่อ: Part 83 — Docker และ Scala
ใน Part ถัดไปจะเรียน Dockerfile, multi-stage builds, sbt-native-packager

[→ Part 83: Docker and Scala](./part-83-docker-and-scala.md)
