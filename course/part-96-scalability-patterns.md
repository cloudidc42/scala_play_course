# Part 96: Scalability Patterns — Steps 951-960

## บทนำ: Scalability

Scalability คือความสามารถในการ handle load ที่เพิ่มขึ้น — horizontal scaling, caching, database sharding, event-driven architecture, CQRS

---

## Step 951: Horizontal vs Vertical Scaling

```scala
// ScalingConcepts.scala

/*
===== Scaling Strategies =====

Vertical Scaling (Scale Up):
  + Simpler, no code changes
  + No distribution complexity
  - Expensive, has ceiling
  - Single point of failure
  Use for: databases (initially), monoliths

Horizontal Scaling (Scale Out):
  + Unlimited (theoretically)
  + Fault tolerant
  - Requires stateless design
  - Need load balancer
  - Session handling complexity
  Use for: web services, microservices, Spark workers

===== Design for Horizontal Scaling =====

1. Stateless services (no local state)
2. Externalize state (Redis, DB, Kafka)
3. Consistent hashing for routing
4. Idempotent operations
5. Graceful degradation
*/

// ===== Stateless service design =====

// ❌ Stateful (not horizontally scalable)
object StatefulOrderService {
  // Local cache — different on each instance!
  private val pendingOrders = scala.collection.mutable.Map.empty[Long, Order]
  
  def addPending(order: Order): Unit = pendingOrders(order.id) = order
  def getPending(id: Long): Option[Order] = pendingOrders.get(id)
}

// ✅ Stateless (all state in Redis)
class StatelessOrderService(redis: RedisClient) {
  
  def setPending(order: Order): Unit =
    redis.setex(s"pending:${order.id}", 3600, serializeOrder(order))
  
  def getPending(id: Long): Option[Order] =
    redis.get(s"pending:$id").map(deserializeOrder)
  
  private def serializeOrder(order: Order): String   = order.toString
  private def deserializeOrder(s: String): Order     = ???  // JSON deserialization
}

case class Order(id: Long, customerId: Long, total: Double, status: String)
class RedisClient {
  def setex(key: String, seconds: Int, value: String): Unit = ()
  def get(key: String): Option[String] = None
}
```

---

## Step 952: Consistent Hashing

```scala
// ConsistentHashing.scala
import java.security.MessageDigest
import scala.collection.immutable.TreeMap

// Consistent hashing: add/remove nodes with minimal key redistribution

class ConsistentHashRing(virtualNodes: Int = 100) {
  
  private var ring = TreeMap.empty[Long, String]  // hash → node
  
  def addNode(node: String): Unit = {
    (0 until virtualNodes).foreach { i =>
      val hash = hashKey(s"$node:$i")
      ring = ring + (hash -> node)
    }
    println(s"Added node: $node (${virtualNodes} virtual nodes)")
  }
  
  def removeNode(node: String): Unit = {
    (0 until virtualNodes).foreach { i =>
      val hash = hashKey(s"$node:$i")
      ring = ring - hash
    }
    println(s"Removed node: $node")
  }
  
  def getNode(key: String): Option[String] = {
    if (ring.isEmpty) return None
    
    val hash = hashKey(key)
    
    // Find first node with hash >= key's hash (clockwise)
    ring.from(hash).headOption
      .orElse(ring.headOption)  // wrap around
      .map(_._2)
  }
  
  def getNodes(key: String, n: Int): List[String] = {
    if (ring.isEmpty) return Nil
    
    val hash = hashKey(key)
    val startFrom = ring.from(hash)
    val wrapped = startFrom ++ ring.until(hash)  // full ring
    
    wrapped.values.distinct.take(n).toList
  }
  
  private def hashKey(key: String): Long = {
    val md = MessageDigest.getInstance("MD5")
    val bytes = md.digest(key.getBytes("UTF-8"))
    math.abs(java.nio.ByteBuffer.wrap(bytes).getLong)
  }
  
  def showDistribution(keys: List[String]): Unit = {
    val distribution = keys.groupBy(k => getNode(k).getOrElse("none"))
    distribution.foreach { case (node, ks) =>
      println(f"$node: ${ks.size} keys (${ks.size * 100.0 / keys.size}%.1f%%)")
    }
  }
}

object ConsistentHashDemo {
  def main(args: Array[String]): Unit = {
    val ring = new ConsistentHashRing(100)
    
    ring.addNode("cache-1")
    ring.addNode("cache-2")
    ring.addNode("cache-3")
    
    val keys = (1 to 1000).map(i => s"user:$i").toList
    ring.showDistribution(keys)
    
    // Add node — only ~25% of keys move
    ring.addNode("cache-4")
    ring.showDistribution(keys)
  }
}
```

---

## Step 953: Caching at Scale

```scala
// CachingAtScale.scala
import scala.concurrent.{ExecutionContext, Future}
import scala.concurrent.duration._

// ===== Cache patterns =====

class DistributedCacheService()(implicit ec: ExecutionContext) {
  
  // ===== Cache-Aside (Lazy Loading) =====
  def getCacheAside[V](key: String)(load: => Future[V]): Future[V] = {
    redisGet(key).flatMap {
      case Some(v) => Future.successful(v.asInstanceOf[V])
      case None    =>
        load.flatMap { v =>
          redisSet(key, v, 5.minutes).map(_ => v)
        }
    }
  }
  
  // ===== Write-Through =====
  def setWriteThrough[V](key: String, value: V)(persist: V => Future[Unit]): Future[Unit] =
    persist(value).flatMap(_ => redisSet(key, value, 5.minutes))
  
  // ===== Write-Behind (async) =====
  def setWriteBehind[V](key: String, value: V)(persist: V => Future[Unit]): Future[Unit] = {
    redisSet(key, value, 5.minutes).map { _ =>
      // Async DB write — fire and forget
      persist(value).recover { case ex =>
        println(s"Write-behind failed: ${ex.getMessage}")
      }
    }
  }
  
  // ===== Cache stampede prevention =====
  private val inFlight = scala.collection.concurrent.TrieMap.empty[String, Future[Any]]
  
  def getOrLoad[V](key: String)(load: => Future[V]): Future[V] = {
    redisGet(key).flatMap {
      case Some(v) => Future.successful(v.asInstanceOf[V])
      case None    =>
        // Single-flight: only one request loads from DB
        inFlight.getOrElseUpdate(key, {
          val f = load.flatMap { v => redisSet(key, v, 5.minutes).map(_ => v) }
          f.onComplete(_ => inFlight.remove(key))
          f
        }).asInstanceOf[Future[V]]
    }
  }
  
  // Simulated Redis operations
  private val store = scala.collection.concurrent.TrieMap.empty[String, Any]
  private def redisGet(key: String): Future[Option[Any]] = Future.successful(store.get(key))
  private def redisSet(key: String, value: Any, ttl: Duration): Future[Unit] =
    Future.successful { store(key) = value }
}

// ===== Multi-level caching =====
class MultiLevelCache[V](
  l1: com.github.benmanes.caffeine.cache.Cache[String, V],
  l2: DistributedCacheService
)(implicit ec: ExecutionContext) {
  
  def get(key: String)(load: => Future[V]): Future[V] = {
    Option(l1.getIfPresent(key)) match {
      case Some(v) => Future.successful(v)  // L1 hit
      case None    =>
        l2.getCacheAside(key)(load).map { v =>
          l1.put(key, v)  // Populate L1
          v
        }
    }
  }
}
```

---

## Step 954: Database Sharding

```scala
// DatabaseSharding.scala

// ===== Horizontal sharding: distribute data across multiple DBs =====

class ShardRouter(shardCount: Int) {
  
  def getShardId(customerId: Long): Int = (customerId % shardCount).toInt
  
  def getConnectionString(customerId: Long): String = {
    val shard = getShardId(customerId)
    s"jdbc:postgresql://shard-$shard.db.internal/orders_$shard"
  }
}

class ShardedOrderRepository(shardConnections: Map[Int, javax.sql.DataSource]) {
  
  private val router = new ShardRouter(shardConnections.size)
  
  def findOrder(orderId: Long, customerId: Long): Option[Order] = {
    val shardId = router.getShardId(customerId)
    val ds = shardConnections(shardId)
    
    val conn = ds.getConnection()
    try {
      val stmt = conn.prepareStatement("SELECT * FROM orders WHERE id = ? AND customer_id = ?")
      stmt.setLong(1, orderId)
      stmt.setLong(2, customerId)
      val rs = stmt.executeQuery()
      if (rs.next()) Some(Order(rs.getLong("id"), rs.getLong("customer_id"), rs.getDouble("total"), rs.getString("status")))
      else None
    } finally conn.close()
  }
  
  def saveOrder(order: Order): Order = {
    val shardId = router.getShardId(order.customerId)
    val ds = shardConnections(shardId)
    
    val conn = ds.getConnection()
    try {
      val stmt = conn.prepareStatement(
        "INSERT INTO orders (id, customer_id, total, status) VALUES (?, ?, ?, ?) ON CONFLICT (id) DO UPDATE SET status = EXCLUDED.status"
      )
      stmt.setLong(1, order.id)
      stmt.setLong(2, order.customerId)
      stmt.setDouble(3, order.total)
      stmt.setString(4, order.status)
      stmt.executeUpdate()
      order
    } finally conn.close()
  }
  
  // Cross-shard query (expensive — avoid if possible)
  def findOrdersByDate(from: java.time.LocalDate, to: java.time.LocalDate): List[Order] = {
    import scala.concurrent.{ExecutionContext, Future, Await}
    import scala.concurrent.duration._
    import scala.concurrent.ExecutionContext.Implicits.global
    
    val futures = shardConnections.values.map { ds =>
      Future {
        val conn = ds.getConnection()
        try {
          // Query each shard
          List.empty[Order]  // simplified
        } finally conn.close()
      }
    }
    
    Await.result(Future.sequence(futures), 30.seconds).flatten.toList
  }
}
```

---

## Step 955: Event-Driven Architecture

```scala
// EventDrivenArchitecture.scala
import akka.actor.typed.ActorSystem
import akka.actor.typed.scaladsl.Behaviors
import akka.kafka.ProducerSettings
import akka.kafka.scaladsl.Producer
import akka.stream.scaladsl._
import org.apache.kafka.clients.producer.ProducerRecord
import org.apache.kafka.common.serialization.StringSerializer
import scala.concurrent.Future

// ===== Event-driven order processing =====

case class OrderPlacedEvent(orderId: Long, customerId: Long, total: Double, items: List[String])
case class PaymentProcessedEvent(orderId: Long, chargeId: String, amount: Double)
case class InventoryReservedEvent(orderId: Long, items: List[String])
case class OrderFulfilledEvent(orderId: Long)

// Each service reacts to events independently
class OrderEventProcessor(implicit system: ActorSystem[_]) {
  
  val producerSettings = ProducerSettings(system, new StringSerializer, new StringSerializer)
    .withBootstrapServers("kafka:9092")
  
  def publishEvent[E](topic: String, key: String, event: E)(implicit enc: spray.json.JsonWriter[E]): Future[Unit] = {
    import spray.json._
    Source.single(new ProducerRecord[String, String](topic, key, event.toJson.compactPrint))
      .runWith(Producer.plainSink(producerSettings))
      .map(_ => ())
  }
}

// ===== Saga pattern (event-driven) =====
object OrderSaga {
  
  // State machine
  sealed trait SagaState
  case object Initiated      extends SagaState
  case object PaymentPending extends SagaState
  case object PaymentFailed  extends SagaState
  case object InventoryPending extends SagaState
  case object Completed      extends SagaState
  case object Compensating   extends SagaState
  case object Failed         extends SagaState
  
  // Commands
  sealed trait SagaCommand
  case class StartOrder(orderId: Long, customerId: Long, total: Double) extends SagaCommand
  case class PaymentResult(orderId: Long, success: Boolean, chargeId: Option[String]) extends SagaCommand
  case class InventoryResult(orderId: Long, success: Boolean) extends SagaCommand
  
  def transition(state: SagaState, cmd: SagaCommand): SagaState = (state, cmd) match {
    case (Initiated, _: StartOrder)                            => PaymentPending
    case (PaymentPending, PaymentResult(_, true, _))           => InventoryPending
    case (PaymentPending, PaymentResult(_, false, _))          => Failed
    case (InventoryPending, InventoryResult(_, true))          => Completed
    case (InventoryPending, InventoryResult(_, false))         => Compensating
    case _                                                     => state
  }
}
```

---

## Step 956: CQRS at Scale

```scala
// CQRSAtScale.scala
import scala.concurrent.{ExecutionContext, Future}

// ===== Command side (write) =====
case class CreateOrderCmd(customerId: Long, items: List[OrderItemCmd])
case class OrderItemCmd(productId: Long, quantity: Int)

trait CommandHandler[C, E] {
  def handle(command: C): Future[List[E]]
}

class OrderCommandHandler(
  orderWriteRepo: OrderWriteRepository,
  eventBus: EventBus
)(implicit ec: ExecutionContext) extends CommandHandler[CreateOrderCmd, OrderEvent] {
  
  def handle(cmd: CreateOrderCmd): Future[List[OrderEvent]] = {
    val orderId = System.currentTimeMillis()
    val event = OrderCreatedEvent(
      orderId     = orderId,
      customerId  = cmd.customerId,
      items       = cmd.items.map(i => s"${i.productId}:${i.quantity}"),
      createdAt   = java.time.Instant.now()
    )
    
    for {
      _ <- orderWriteRepo.save(orderId, event)
      _ <- eventBus.publish("orders", event)
    } yield List(event)
  }
}

// ===== Query side (read) =====
// Denormalized read models optimized for queries

case class OrderSummaryView(
  orderId: Long,
  customerName: String,
  totalItems: Int,
  totalAmount: Double,
  status: String,
  createdAt: java.time.Instant
)

case class CustomerOrdersView(
  customerId: Long,
  customerName: String,
  totalOrders: Int,
  lifetimeValue: Double,
  lastOrderDate: java.time.Instant
)

class OrderQueryService(readDb: OrderReadDatabase)(implicit ec: ExecutionContext) {
  
  def getOrderSummary(orderId: Long): Future[Option[OrderSummaryView]] =
    readDb.findOrderSummary(orderId)
  
  def getCustomerOrders(customerId: Long, page: Int, size: Int): Future[List[OrderSummaryView]] =
    readDb.findOrdersByCustomer(customerId, page, size)
  
  def getCustomerStats(customerId: Long): Future[Option[CustomerOrdersView]] =
    readDb.findCustomerStats(customerId)
  
  // Complex query (read side is optimized for this)
  def getTopCustomers(limit: Int = 100): Future[List[CustomerOrdersView]] =
    readDb.findTopCustomersByLifetimeValue(limit)
}

// ===== Event projector (write → read sync) =====
class OrderProjector(readDb: OrderReadDatabase)(implicit ec: ExecutionContext) {
  
  def project(event: OrderEvent): Future[Unit] = event match {
    case OrderCreatedEvent(id, customerId, items, createdAt) =>
      readDb.insertOrderSummary(OrderSummaryView(
        orderId      = id,
        customerName = "Loading...",  // enriched async
        totalItems   = items.size,
        totalAmount  = 0.0,
        status       = "pending",
        createdAt    = createdAt
      ))
    
    case _ => Future.unit
  }
}

// Placeholder types
sealed trait OrderEvent
case class OrderCreatedEvent(orderId: Long, customerId: Long, items: List[String], createdAt: java.time.Instant) extends OrderEvent
trait OrderWriteRepository { def save(id: Long, event: OrderEvent): Future[Unit] }
trait OrderReadDatabase {
  def findOrderSummary(id: Long): Future[Option[OrderSummaryView]]
  def findOrdersByCustomer(cid: Long, page: Int, size: Int): Future[List[OrderSummaryView]]
  def findCustomerStats(cid: Long): Future[Option[CustomerOrdersView]]
  def findTopCustomersByLifetimeValue(limit: Int): Future[List[CustomerOrdersView]]
  def insertOrderSummary(view: OrderSummaryView): Future[Unit]
}
trait EventBus { def publish(topic: String, event: Any): Future[Unit] }
```

---

## Step 957: Queue-Based Load Leveling

```scala
// QueueBasedLoadLeveling.scala
import akka.actor.typed.ActorSystem
import akka.actor.typed.scaladsl.Behaviors
import akka.stream.scaladsl._
import scala.concurrent.duration._

// ===== Decouple producers from consumers via queue =====

class OrderProcessingPipeline(implicit system: ActorSystem[_]) {
  import system.executionContext
  
  // Simulated order queue (Kafka/SQS in production)
  val orderQueue = akka.stream.scaladsl.Source.queue[Order](bufferSize = 10000)
  
  def startProcessing(): Unit = {
    Source.tick(1.second, 100.millis, ())
      .map(_ => Order(System.currentTimeMillis(), 1L, 99.99, "pending"))
      // Buffer to handle bursts
      .buffer(1000, akka.stream.OverflowStrategy.backpressure)
      // Parallel processing
      .mapAsyncUnordered(parallelism = 20) { order =>
        processOrder(order)
      }
      // Batch results for DB write
      .groupedWithin(100, 500.millis)
      .mapAsync(5) { batch =>
        bulkSaveToDB(batch.toList)
      }
      .runWith(Sink.ignore)
  }
  
  private def processOrder(order: Order): scala.concurrent.Future[Order] = {
    scala.concurrent.Future {
      // Simulate processing
      Thread.sleep(50)
      order.copy(status = "processed")
    }(system.executionContext)
  }
  
  private def bulkSaveToDB(orders: List[Order]): scala.concurrent.Future[Unit] = {
    scala.concurrent.Future.successful {
      println(s"Saved batch of ${orders.size} orders")
    }
  }
}
```

---

## Step 958: Auto-Scaling

```yaml
# kubernetes/hpa.yaml — Kubernetes Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  
  minReplicas: 3
  maxReplicas: 50
  
  metrics:
    # Scale on CPU
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    
    # Scale on memory
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: "1500Mi"
    
    # Custom metric: requests per second
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
    
    # External metric: Kafka consumer lag
    - type: External
      external:
        metric:
          name: kafka_consumer_lag_sum
          selector:
            matchLabels:
              topic: orders
              group: order-processor
        target:
          type: AverageValue
          averageValue: "1000"
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # Wait 60s before scaling up again
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60  # Add at most 4 pods every 60s
    
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5m before scaling down
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60  # Remove at most 2 pods every 60s
```

---

## Step 959: Database Read Replicas

```scala
// ReadReplicas.scala
import com.zaxxer.hikari.{HikariConfig, HikariDataSource}
import javax.sql.DataSource

// ===== Read/Write splitting =====
class DataSourceRouter(
  writeDataSource: DataSource,
  readDataSources: List[DataSource]
) {
  
  private val readIndex = new java.util.concurrent.atomic.AtomicInteger(0)
  
  def getWriteDS: DataSource = writeDataSource
  
  // Round-robin across read replicas
  def getReadDS: DataSource = {
    val idx = readIndex.getAndIncrement() % readDataSources.size
    readDataSources(idx)
  }
}

// Repository using read/write split
class OrderRepositoryWithReplicas(router: DataSourceRouter) {
  
  // Writes go to primary
  def saveOrder(order: Order): Order = {
    val conn = router.getWriteDS.getConnection()
    try {
      // INSERT/UPDATE
      order
    } finally conn.close()
  }
  
  // Reads go to replica
  def findById(id: Long): Option[Order] = {
    val conn = router.getReadDS.getConnection()
    try {
      // SELECT — can be stale by replication lag (usually < 100ms)
      None
    } finally conn.close()
  }
  
  // Critical reads (must be fresh) → go to primary
  def findByIdConsistent(id: Long): Option[Order] = {
    val conn = router.getWriteDS.getConnection()
    try {
      None
    } finally conn.close()
  }
}
```

---

## Step 960: Backpressure & Flow Control

```scala
// BackpressureFlowControl.scala
import akka.stream.scaladsl._
import akka.actor.typed.ActorSystem
import akka.actor.typed.scaladsl.Behaviors
import scala.concurrent.duration._
import akka.stream.OverflowStrategy

class BackpressureDemo(implicit system: ActorSystem[_]) {
  
  // ===== Backpressure: slow consumer signals slow producer =====
  
  def backpressureExample(): Unit = {
    // Fast producer (1000/s)
    val fastProducer = Source.tick(0.seconds, 1.milli, "message")
    
    // Slow consumer (10/s)
    val slowConsumer = Sink.foreach[String] { msg =>
      Thread.sleep(100)  // simulate slow processing
    }
    
    // Without buffering: producer is naturally slowed (backpressure)
    fastProducer.runWith(slowConsumer)
  }
  
  // ===== Buffer strategies =====
  
  def bufferStrategies(): Unit = {
    val source = Source(1 to 10000)
    
    // Backpressure: block producer when buffer full (default, safe)
    source.buffer(1000, OverflowStrategy.backpressure)
    
    // Drop head: drop oldest messages when full
    source.buffer(1000, OverflowStrategy.dropHead)
    
    // Drop tail: drop newest messages when full
    source.buffer(1000, OverflowStrategy.dropTail)
    
    // Drop new: drop incoming when full
    source.buffer(1000, OverflowStrategy.dropNew)
    
    // Fail: fail stream when buffer full
    source.buffer(1000, OverflowStrategy.fail)
  }
  
  // ===== Throttle: rate limiting =====
  
  def throttleExample(): Unit = {
    Source(1 to 10000)
      .throttle(100, 1.second)          // max 100 elements/second
      .throttle(10, 100.milliseconds)   // max 10 elements/100ms
      .runForeach(n => println(s"Processing: $n"))
  }
  
  // ===== Conflate: skip intermediate values under load =====
  
  def conflateExample(): Unit = {
    // If consumer is slow, merge/skip intermediate values
    Source.tick(0.seconds, 10.millis, 1)
      .conflate(_ + _)  // sum skipped values
      .throttle(1, 1.second)  // slow consumer
      .runForeach(n => println(s"Conflated: $n"))
  }
}
```

---

## สรุป Part 96: Scalability Patterns

| Pattern | Problem Solved | Trade-off |
|---------|----------------|-----------|
| Horizontal scaling | Capacity | Stateless required |
| Consistent hashing | Even distribution | Complexity |
| CQRS | Read/write scaling | Eventual consistency |
| Sharding | DB capacity | Cross-shard queries |
| Event-driven | Decoupling | Debugging complexity |
| Read replicas | Read throughput | Replication lag |
| Backpressure | Consumer overload | Reduced throughput |

---

## แบบฝึกหัด Part 96

1. **Consistent Hashing**: implement consistent hash ring ที่มี 5 nodes, เพิ่ม/ลบ nodes และ verify ว่า < 30% ของ keys ต้องย้าย

2. **CQRS**: implement order system ด้วย separate command/query models, event projector ที่ sync write → read database

3. **Sharding**: implement `ShardedOrderRepository` ที่ route queries ไปยัง correct shard ตาม `customerId % shardCount`

4. **Backpressure**: สร้าง Akka Streams pipeline ที่ fast producer (1000/s) → backpressure → slow consumer (100/s) และ monitor buffer usage

5. **Auto-scaling**: เขียน HPA configuration สำหรับ order service ที่ scale based on Kafka consumer lag metric, พร้อม scale-up/down policies

---

## ไปต่อ: Part 97 — Real-World Project
[→ Part 97: Real-World E-commerce Project](./part-97-real-world-project.md)
