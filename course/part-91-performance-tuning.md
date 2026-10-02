# Part 91: Performance Tuning — Steps 901-910

## บทนำ: Performance Tuning

Performance tuning ใน Scala ครอบคลุม JVM tuning, GC selection, memory management, thread pool configuration และ Scala-specific optimizations

---

## Step 901: JVM Flags สำหรับ Production

```bash
# jvm-production.sh
# JVM flags สำหรับ Scala production services

java \
  # Memory
  -Xms2g -Xmx2g \
  -XX:MaxMetaspaceSize=512m \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  \
  # GC: G1GC (default Java 9+, good for latency)
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:G1HeapRegionSize=16m \
  -XX:G1NewSizePercent=30 \
  -XX:G1MaxNewSizePercent=40 \
  -XX:InitiatingHeapOccupancyPercent=45 \
  \
  # GC: ZGC (Java 15+, ultra-low latency, for high-traffic APIs)
  # -XX:+UseZGC \
  # -XX:SoftMaxHeapSize=1800m \
  \
  # GC: Shenandoah (OpenJDK, concurrent)
  # -XX:+UseShenandoahGC \
  \
  # JIT Compilation
  -XX:+TieredCompilation \
  -XX:ReservedCodeCacheSize=256m \
  -XX:+UseStringDeduplication \
  \
  # Monitoring
  -XX:+PrintGCDetails \
  -XX:+PrintGCDateStamps \
  -Xloggc:/logs/gc.log \
  -XX:+UseGCLogFileRotation \
  -XX:NumberOfGCLogFiles=5 \
  -XX:GCLogFileSize=20m \
  \
  # Crash & OOM
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/dumps/heap.hprof \
  -XX:+ExitOnOutOfMemoryError \
  \
  # Threading
  -XX:ParallelGCThreads=4 \
  -XX:ConcGCThreads=2 \
  \
  # Performance
  -server \
  -XX:+OptimizeStringConcat \
  -XX:+UseCompressedOops \
  -XX:+UseCompressedClassPointers \
  \
  -jar app.jar
```

---

## Step 902: GC Tuning

```scala
// GCTuningGuide.scala

/*
===== GC Selection Guide =====

| GC           | Java | Pause | Throughput | Use Case                   |
|--------------|------|-------|------------|----------------------------|
| SerialGC     | All  | High  | High       | Single core, small heap    |
| ParallelGC   | All  | Med   | Highest    | Batch jobs (Spark)         |
| G1GC         | 9+   | Low   | High       | General purpose (default)  |
| ZGC          | 15+  | <1ms  | Med        | Low-latency APIs           |
| Shenandoah   | 11+  | <1ms  | Med        | Low-latency, OpenJDK       |
| EpsilonGC    | 11+  | None  | N/A        | Short-lived, testing       |

===== GC Tuning Steps =====

1. Baseline: Run with defaults, record metrics
2. Identify: Is it GC pause time or throughput?
3. Tune: Adjust heap sizes, region sizes
4. Measure: Compare before/after
5. Iterate

===== Key GC Metrics =====
- GC pause time (P99 < 200ms for most apps)
- GC throughput (> 95% time in app code)
- GC frequency (too frequent = heap too small)
- Allocation rate (high = objects created too fast)
- Promotion rate (high = objects promoted to old too fast)
*/

object GCAnalysis {
  // Analyze GC log programmatically
  case class GCEvent(
    timestamp: Long,
    pauseMs: Double,
    beforeMB: Long,
    afterMB: Long,
    heapMB: Long
  )
  
  def parseGCLog(logPath: String): List[GCEvent] = {
    // Simple parser for GC log format
    import scala.io.Source
    val pattern = """(\d+\.\d+): \[GC.*?(\d+\.\d+)ms\]""".r
    
    Source.fromFile(logPath).getLines()
      .flatMap(line => pattern.findFirstMatchIn(line))
      .map(m => GCEvent(
        timestamp = m.group(1).toDouble.toLong,
        pauseMs   = m.group(2).toDouble,
        beforeMB  = 0L,
        afterMB   = 0L,
        heapMB    = 0L
      ))
      .toList
  }
  
  def analyzeGC(events: List[GCEvent]): Map[String, Double] = {
    if (events.isEmpty) return Map.empty
    
    Map(
      "count"        -> events.size,
      "avgPauseMs"   -> events.map(_.pauseMs).sum / events.size,
      "maxPauseMs"   -> events.map(_.pauseMs).max,
      "p99PauseMs"   -> percentile(events.map(_.pauseMs).sorted, 0.99)
    )
  }
  
  def percentile(sorted: List[Double], p: Double): Double = {
    val idx = (sorted.size * p).toInt
    sorted(idx min (sorted.size - 1))
  }
}
```

---

## Step 903: Thread Pool Tuning

```scala
// ThreadPoolTuning.scala
import scala.concurrent.{ExecutionContext, Future}
import java.util.concurrent._

object ThreadPoolTuning {
  
  // ===== CPU-bound tasks =====
  // Use # of CPU cores
  val cpuBound: ExecutionContext = ExecutionContext.fromExecutorService(
    new ForkJoinPool(Runtime.getRuntime.availableProcessors())
  )
  
  // ===== IO-bound tasks =====
  // More threads than CPUs (waiting on IO)
  val ioBound: ExecutionContext = ExecutionContext.fromExecutorService(
    new ThreadPoolExecutor(
      /*corePoolSize*/  10,
      /*maxPoolSize*/   200,
      /*keepAlive*/     60L, TimeUnit.SECONDS,
      /*queue*/         new LinkedBlockingQueue[Runnable](1000),
      /*factory*/       new ThreadFactory {
        private val counter = new java.util.concurrent.atomic.AtomicInteger(0)
        def newThread(r: Runnable): Thread = {
          val t = new Thread(r, s"io-thread-${counter.getAndIncrement()}")
          t.setDaemon(true)
          t
        }
      },
      /*handler*/       new ThreadPoolExecutor.CallerRunsPolicy()
    )
  )
  
  // ===== Blocking IO =====
  // Dedicated pool for blocking calls (JDBC, file IO)
  val blockingIO: ExecutionContext = ExecutionContext.fromExecutorService(
    new ThreadPoolExecutor(
      5, 100, 60L, TimeUnit.SECONDS,
      new SynchronousQueue[Runnable](),
      new ThreadPoolExecutor.CallerRunsPolicy()
    )
  )
  
  // ===== Akka Dispatchers =====
  /*
  # application.conf
  
  # Default dispatcher (Akka managed fork-join)
  akka.actor.default-dispatcher {
    type = Dispatcher
    executor = "fork-join-executor"
    fork-join-executor {
      parallelism-min = 8
      parallelism-factor = 3.0
      parallelism-max = 64
    }
    throughput = 5  # process N messages before yielding
  }
  
  # Blocking IO dispatcher
  blocking-dispatcher {
    type = Dispatcher
    executor = "thread-pool-executor"
    thread-pool-executor {
      fixed-pool-size = 32
    }
  }
  */
  
  // ===== Choosing the right pool =====
  def demonstratePoolUsage(): Unit = {
    // CPU work: use cpuBound
    Future {
      (1 to 1000000).sum
    }(cpuBound)
    
    // HTTP calls: use ioBound
    Future {
      // Thread blocks waiting for HTTP response
      // Thread is occupied but CPU is idle
      Thread.sleep(100) // simulate HTTP call
    }(ioBound)
    
    // Database: use blockingIO (dedicated pool)
    Future {
      Thread.sleep(50) // simulate DB query
    }(blockingIO)
  }
}
```

---

## Step 904: Scala Collections Performance

```scala
// CollectionsPerformance.scala

object CollectionsPerformance {
  
  // ===== Immutable vs Mutable =====
  
  // ❌ Slow: building large immutable list via prepend
  def slowBuild(n: Int): List[Int] = {
    var result = List.empty[Int]
    for (i <- 0 until n) {
      result = i :: result  // O(1) per op, but creates new list each time
    }
    result.reverse  // O(n)
  }
  
  // ✅ Fast: use ListBuffer then convert
  def fastBuild(n: Int): List[Int] = {
    val buffer = scala.collection.mutable.ListBuffer.empty[Int]
    for (i <- 0 until n) buffer += i
    buffer.toList  // single O(n) conversion
  }
  
  // ===== Vector vs List =====
  
  // List: O(1) prepend, O(n) random access
  // Vector: O(log n) for all ops, good for random access
  
  def vectorVsList(): Unit = {
    val n = 100000
    
    // Random access: Vector wins
    val vec = Vector.tabulate(n)(identity)
    val lst = List.tabulate(n)(identity)
    
    // vec(50000) is O(log n) ≈ O(1) in practice
    // lst(50000) is O(50000)
    println(s"Vector(50000): ${vec(50000)}")
  }
  
  // ===== HashMap vs TreeMap =====
  // HashMap: O(1) avg, no ordering
  // TreeMap: O(log n), sorted
  
  def mapComparison(): Unit = {
    val n = 100000
    val hashMap = Map.from((1 to n).map(i => i -> i.toString))
    val treeMap = scala.collection.immutable.SortedMap.from(hashMap)
    
    // hashMap.get(key) is O(1)
    // treeMap.get(key) is O(log n)
    // treeMap.range(1000, 2000) is efficient — hashMap cannot do this well
  }
  
  // ===== Avoid boxing: Use specialized collections =====
  
  // ❌ Slow: scala.collection.immutable.Map[Int, Int] boxes primitives
  val boxedMap: Map[Int, Int] = Map(1 -> 2, 3 -> 4)
  
  // ✅ Fast for numeric keys: use arrays or specialized libs
  // e.g., scala.collection.mutable.LongMap for Long keys
  val longMap = scala.collection.mutable.LongMap.empty[Int]
  longMap.update(1L, 100)
  
  // ===== Parallel collections =====
  def parallelExample(): Unit = {
    val data = (1 to 1000000).toVector
    
    // Sequential
    val seq = data.map(_ * 2)
    
    // Parallel (uses ForkJoinPool)
    val par = data.par.map(_ * 2).seq
    
    // Use .par only for CPU-intensive ops on large collections
    // Don't use .par for IO-bound ops (use Future instead)
  }
  
  // ===== @specialized annotation =====
  class GenericClass[A](val value: A)  // A gets boxed
  
  class SpecializedClass[@specialized(Int, Long, Double) A](val value: A)  // No boxing for primitives
}
```

---

## Step 905: Akka Performance

```scala
// AkkaPerformance.scala
import akka.actor.typed._
import akka.actor.typed.scaladsl._
import akka.stream.scaladsl._
import akka.util.ByteString
import scala.concurrent.duration._

object AkkaPerformance {
  
  // ===== Batching messages =====
  sealed trait WorkerMsg
  case class Process(items: Vector[Int]) extends WorkerMsg
  case class ProcessSingle(item: Int) extends WorkerMsg
  
  // ❌ Slow: send one message at a time
  def sendOneByOne(actor: ActorRef[WorkerMsg], items: List[Int]): Unit = {
    items.foreach(item => actor ! ProcessSingle(item))
  }
  
  // ✅ Fast: batch messages
  def sendBatched(actor: ActorRef[WorkerMsg], items: List[Int], batchSize: Int = 100): Unit = {
    items.grouped(batchSize).foreach { batch =>
      actor ! Process(batch.toVector)
    }
  }
  
  // ===== Stream backpressure =====
  def streamWithBackpressure()(implicit mat: akka.stream.Materializer): Unit = {
    Source(1 to 1000000)
      .map(_ * 2)
      .async  // asynchronous boundary — enables parallel stages
      .filter(_ % 2 == 0)
      .async
      .grouped(1000)  // batch downstream
      .runForeach(batch => println(s"Processed ${batch.size} items"))
  }
  
  // ===== Stream throughput optimization =====
  def highThroughputStream()(implicit mat: akka.stream.Materializer): Unit = {
    Source(1 to 10000000)
      // Process in parallel with mapAsyncUnordered (order doesn't matter)
      .mapAsyncUnordered(parallelism = 16)(n => scala.concurrent.Future.successful(n * 2))
      // Buffer to smooth backpressure
      .buffer(10000, akka.stream.OverflowStrategy.backpressure)
      // Batch for downstream processing
      .groupedWithin(1000, 100.milliseconds)
      .runForeach { batch =>
        // Process batch
      }
  }
  
  // ===== Avoid stashing too much =====
  // Stash is backed by a queue; large stashes waste memory
  // Use getOrElseUpdate or explicit message ordering instead
  
  // ===== Use ask pattern only when necessary =====
  // ask creates a new actor per call — use tell (!) when possible
}
```

---

## Step 906: Memory Optimization

```scala
// MemoryOptimization.scala

object MemoryOptimization {
  
  // ===== Case class memory =====
  // Scala case class has overhead: boxing, AnyRef header
  
  // ❌ Large: boxing optional fields
  case class UserHeavy(
    id: Long,
    name: String,
    age: Option[Int],
    score: Option[Double]
  )
  
  // ✅ Lighter: use sentinel values or value classes
  case class UserLight(
    id: Long,
    name: String,
    age: Int = -1,       // -1 = not set
    score: Double = 0.0  // 0.0 = not set
  )
  
  // ===== Value classes (no heap allocation) =====
  class UserId(val value: Long) extends AnyVal
  class Email(val value: String) extends AnyVal
  
  // UserId is erased to Long at runtime — zero allocation!
  def processUser(id: UserId, email: Email): Unit = {
    println(s"User ${id.value} email: ${email.value}")
  }
  
  // ===== Lazy vals =====
  object ExpensiveConfig {
    // ❌ Eager: computed even if never used
    val eagerFibCache: Map[Int, Long] = (0 to 1000).foldLeft(Map(0 -> 0L, 1 -> 1L)) {
      case (acc, n) if n < 2 => acc
      case (acc, n)          => acc + (n -> (acc(n-1) + acc(n-2)))
    }
    
    // ✅ Lazy: computed only on first access, then cached
    lazy val lazyFibCache: Map[Int, Long] = (0 to 1000).foldLeft(Map(0 -> 0L, 1 -> 1L)) {
      case (acc, n) if n < 2 => acc
      case (acc, n)          => acc + (n -> (acc(n-1) + acc(n-2)))
    }
  }
  
  // ===== Object pooling =====
  class ByteBufferPool(poolSize: Int, bufferSize: Int) {
    private val pool = new java.util.concurrent.ArrayBlockingQueue[java.nio.ByteBuffer](poolSize)
    
    // Pre-fill pool
    (1 to poolSize).foreach(_ => pool.offer(java.nio.ByteBuffer.allocateDirect(bufferSize)))
    
    def acquire(): java.nio.ByteBuffer =
      Option(pool.poll()).getOrElse(java.nio.ByteBuffer.allocateDirect(bufferSize))
    
    def release(buf: java.nio.ByteBuffer): Unit = {
      buf.clear()
      pool.offer(buf)
    }
    
    def withBuffer[T](f: java.nio.ByteBuffer => T): T = {
      val buf = acquire()
      try f(buf) finally release(buf)
    }
  }
  
  // ===== Interning strings =====
  // For small-cardinality strings (statuses, codes), intern to save memory
  val STATUS_PENDING  = "pending".intern()
  val STATUS_ACTIVE   = "active".intern()
  val STATUS_INACTIVE = "inactive".intern()
  
  // ===== Off-heap storage for large data =====
  // Use Chronicle Map, MapDB, or direct ByteBuffer for large caches
}
```

---

## Step 907: Database Connection Pool

```scala
// ConnectionPoolConfig.scala
// HikariCP — fastest JDBC connection pool

/*
# application.conf
db {
  url      = "jdbc:postgresql://localhost/mydb"
  user     = "myuser"
  password = "secret"
  
  pool {
    maximumPoolSize      = 20    # Max connections
    minimumIdle          = 5     # Min idle connections
    connectionTimeout    = 30000 # Wait for connection (ms)
    idleTimeout          = 600000 # Close idle connections after 10m
    maxLifetime          = 1800000 # Max connection lifetime 30m
    keepaliveTime        = 60000  # Ping connection every 1m
    connectionTestQuery  = "SELECT 1"
    
    # Tune for workload
    # OLTP (many small queries):  maximumPoolSize = 2 * CPU + 1
    # Analytics (large queries):  maximumPoolSize = CPU
  }
}
*/

import com.zaxxer.hikari.{HikariConfig, HikariDataSource}
import javax.sql.DataSource

object ConnectionPoolFactory {
  
  def createPool(
    jdbcUrl: String,
    username: String,
    password: String,
    poolName: String = "main",
    maxPoolSize: Int = 20
  ): DataSource = {
    val config = new HikariConfig()
    config.setJdbcUrl(jdbcUrl)
    config.setUsername(username)
    config.setPassword(password)
    config.setPoolName(poolName)
    
    // Pool sizing: Hikari recommends 2-3x CPU for OLTP
    config.setMaximumPoolSize(maxPoolSize)
    config.setMinimumIdle(maxPoolSize / 4)
    
    // Timeouts
    config.setConnectionTimeout(30_000)
    config.setIdleTimeout(600_000)
    config.setMaxLifetime(1_800_000)
    config.setKeepaliveTime(60_000)
    
    // Health check
    config.setConnectionTestQuery("SELECT 1")
    config.setValidationTimeout(5_000)
    
    // Performance
    config.addDataSourceProperty("cachePrepStmts", "true")
    config.addDataSourceProperty("prepStmtCacheSize", "250")
    config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048")
    config.addDataSourceProperty("useServerPrepStmts", "true")
    
    // Prometheus metrics
    config.setMetricRegistry(io.prometheus.client.CollectorRegistry.defaultRegistry)
    
    new HikariDataSource(config)
  }
}
```

---

## Step 908: Caching Strategies

```scala
// CachingStrategies.scala
import scala.concurrent.{ExecutionContext, Future}
import scala.concurrent.duration._
import java.util.concurrent.TimeUnit
import com.github.benmanes.caffeine.cache.{Caffeine, AsyncLoadingCache}

class CacheService()(implicit ec: ExecutionContext) {
  
  // ===== Local in-memory cache (Caffeine — fastest JVM cache) =====
  val localCache: AsyncLoadingCache[String, String] = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(5, TimeUnit.MINUTES)
    .refreshAfterWrite(1, TimeUnit.MINUTES)
    .recordStats()
    .buildAsync[String, String]((key, _) => {
      // Load function — called on cache miss
      java.util.concurrent.CompletableFuture.completedFuture(s"value-for-$key")
    })
  
  // ===== Redis cache (distributed) =====
  // Using Lettuce Redis client
  /*
  val redisClient = RedisClient.create("redis://localhost:6379")
  val connection = redisClient.connect()
  val commands = connection.async()
  
  def getFromRedis(key: String): Future[Option[String]] = {
    commands.get(key).toCompletableFuture.asScala.map(Option(_))
  }
  
  def setInRedis(key: String, value: String, ttl: Duration): Future[Unit] = {
    commands.setex(key, ttl.toSeconds, value).toCompletableFuture.asScala.map(_ => ())
  }
  */
  
  // ===== Multi-level cache (L1: local, L2: Redis) =====
  def getWithMultiLevelCache(key: String): Future[Option[String]] = {
    // L1: Check local
    val cached = localCache.getIfPresent(key)
    if (cached != null) {
      cached.toCompletableFuture.asScala.map(Some(_))
    } else {
      // L2: Check Redis (simulated)
      Future.successful(None).flatMap {
        case Some(v) =>
          // Populate L1
          localCache.put(key, java.util.concurrent.CompletableFuture.completedFuture(v))
          Future.successful(Some(v))
        case None =>
          // DB lookup
          Future.successful(Some(s"db-value-$key"))
      }
    }
  }
  
  // ===== Cache aside pattern =====
  def cacheAside[V](key: String, ttl: Duration)(loadFromDB: => Future[V]): Future[V] = {
    // 1. Check cache
    // 2. If miss: load from DB
    // 3. Store in cache
    // 4. Return value
    loadFromDB  // simplified
  }
  
  // ===== Cache invalidation strategies =====
  // TTL: simple, eventual consistency
  // Event-driven: on write, invalidate/update cache
  // Write-through: write to cache + DB simultaneously
  // Write-behind: write to cache first, DB asynchronously
}
```

---

## Step 909: Scala Code Optimizations

```scala
// ScalaOptimizations.scala

object ScalaOptimizations {
  
  // ===== @tailrec — stack-safe recursion =====
  import scala.annotation.tailrec
  
  // ❌ Stack overflow for large n
  def sumUnsafe(n: Long): Long =
    if (n <= 0) 0L else n + sumUnsafe(n - 1)
  
  // ✅ Tail-recursive, compiled to loop
  @tailrec
  def sumSafe(n: Long, acc: Long = 0L): Long =
    if (n <= 0) acc else sumSafe(n - 1, acc + n)
  
  // ===== Avoid closures capturing large objects =====
  
  class LargeObject(data: Array[Byte]) {
    // ❌ Bad: closure captures `this` (including data array)
    def badFilter(items: List[Int]): List[Int] = {
      val threshold = data.length  // captures `this`
      items.filter(_ > threshold)
    }
    
    // ✅ Good: extract to local val
    def goodFilter(items: List[Int]): List[Int] = {
      val threshold = data.length  // extract to avoid capturing `this`
      items.filter(_ > threshold)
    }
  }
  
  // ===== String builder instead of concatenation =====
  
  // ❌ Slow: O(n²) string concatenation
  def slowConcat(items: List[String]): String =
    items.foldLeft("")(_ + ", " + _)
  
  // ✅ Fast: O(n) StringBuilder
  def fastConcat(items: List[String]): String = {
    val sb = new StringBuilder
    items.foreach { item =>
      if (sb.nonEmpty) sb.append(", ")
      sb.append(item)
    }
    sb.toString()
  }
  
  // ✅ Even simpler
  def mkStringConcat(items: List[String]): String = items.mkString(", ")
  
  // ===== Pattern matching compilation =====
  
  sealed trait Shape
  case class Circle(r: Double) extends Shape
  case class Rectangle(w: Double, h: Double) extends Shape
  case class Triangle(b: Double, h: Double) extends Shape
  
  // Scala compiles sealed hierarchy match to efficient tableswitch/lookupswitch
  def area(shape: Shape): Double = shape match {
    case Circle(r)        => Math.PI * r * r
    case Rectangle(w, h)  => w * h
    case Triangle(b, h)   => 0.5 * b * h
  }
  
  // ===== Avoid repeated map lookups =====
  
  // ❌ Double lookup
  def processMap(m: Map[String, Int], key: String): Option[Int] = {
    if (m.contains(key)) Some(m(key)) else None
  }
  
  // ✅ Single lookup
  def processMapOpt(m: Map[String, Int], key: String): Option[Int] = m.get(key)
  
  // ===== inline macro for hot paths =====
  // Scala 3 inline keyword inlines at compile time
  inline def square(x: Double): Double = x * x  // always inlined
}
```

---

## Step 910: Performance Testing

```scala
// PerformanceTesting.scala
// Use Gatling for load testing

/*
# build.sbt
"io.gatling.highcharts" % "gatling-charts-highcharts" % "3.9.5" % "test"
"io.gatling"            % "gatling-test-framework"    % "3.9.5" % "test"
*/

// src/test/scala/simulations/OrderSimulation.scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class OrderSimulation extends Simulation {
  
  val httpProtocol = http
    .baseUrl("https://api.myapp.com")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")
    .header("Authorization", "Bearer test-jwt-token")
  
  val feeder = csv("orders.csv").random
  
  val createOrder = exec(
    http("Create Order")
      .post("/api/v1/orders")
      .body(StringBody("""{"customerId":"${customerId}","total":${total}}"""))
      .check(status.is(201))
      .check(jsonPath("$.id").saveAs("orderId"))
  )
  
  val getOrder = exec(
    http("Get Order")
      .get("/api/v1/orders/${orderId}")
      .check(status.is(200))
      .check(responseTimeInMillis.lte(200))
  )
  
  val orderScenario = scenario("Order Flow")
    .feed(feeder)
    .exec(createOrder)
    .pause(1.second)
    .exec(getOrder)
  
  setUp(
    // Ramp up to 100 users over 30s, hold for 5 min
    orderScenario.inject(
      rampUsersPerSec(1) to 100 during 30.seconds,
      constantUsersPerSec(100) during 5.minutes,
      rampUsersPerSec(100) to 0 during 30.seconds
    )
  ).protocols(httpProtocol)
   .assertions(
     global.responseTime.percentile3.lte(500),  // P99 < 500ms
     global.successfulRequests.percent.gte(99)  // 99% success
   )
}
```

---

## สรุป Part 91: Performance Tuning

| Category | Tool/Technique | Impact |
|----------|----------------|--------|
| GC | ZGC/G1GC tuning | Low latency |
| Threads | Right pool size | Throughput |
| Collections | ListBuffer, Vector | Memory/Speed |
| Caching | Caffeine L1 + Redis L2 | Latency |
| DB Pool | HikariCP tuning | Throughput |
| Code | @tailrec, StringBuilder | CPU/Memory |

---

## แบบฝึกหัด Part 91

1. **GC Tuning**: setup JVM ด้วย G1GC, enable GC logging, run load test, และ analyze GC logs เพื่อหา GC bottleneck

2. **Thread Pool**: configure 3 thread pools (CPU-bound, IO-bound, blocking) สำหรับ web service ที่ handle 1000 req/s

3. **Collection Benchmark**: เปรียบเทียบ performance ของ List vs Vector vs ArrayBuffer สำหรับ operations: prepend, append, random access, iteration

4. **Caching**: implement 2-level cache (Caffeine L1 + simulated Redis L2) พร้อม cache hit rate monitoring

5. **Load Test**: เขียน Gatling simulation สำหรับ API ที่ ramp up จาก 1 ถึง 100 users และ assert P99 < 500ms

---

## ไปต่อ: Part 92 — Profiling & Benchmarks
[→ Part 92: Profiling and Benchmarks](./part-92-profiling-and-benchmarks.md)
