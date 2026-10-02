# Part 92: Profiling & Benchmarks — Steps 911-920

## บทนำ: Profiling

Profiling คือการวัดและวิเคราะห์ performance ของโปรแกรมเพื่อหา bottleneck — JMH สำหรับ micro-benchmarks, Async Profiler สำหรับ CPU/memory profiling

---

## Step 911: JMH Micro-benchmarks

```scala
// OrderBenchmark.scala
// JMH = Java Microbenchmark Harness

/*
# build.sbt
"org.openjdk.jmh" % "jmh-core"          % "1.37" % "test"
"org.openjdk.jmh" % "jmh-generator-annprocess" % "1.37" % "test"
*/

import org.openjdk.jmh.annotations._
import org.openjdk.jmh.infra.Blackhole
import java.util.concurrent.TimeUnit

@State(Scope.Benchmark)
@BenchmarkMode(Array(Mode.Throughput, Mode.AverageTime))
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 10, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(value = 2, jvmArgs = Array("-Xms2g", "-Xmx2g", "-XX:+UseG1GC"))
class CollectionBenchmark {
  
  @Param(Array("1000", "10000", "100000"))
  var size: Int = _
  
  var list: List[Int] = _
  var vector: Vector[Int] = _
  var array: Array[Int] = _
  
  @Setup(Level.Trial)
  def setup(): Unit = {
    list   = (1 to size).toList
    vector = (1 to size).toVector
    array  = (1 to size).toArray
  }
  
  @Benchmark
  def listSum(bh: Blackhole): Unit = {
    bh.consume(list.sum)
  }
  
  @Benchmark
  def vectorSum(bh: Blackhole): Unit = {
    bh.consume(vector.sum)
  }
  
  @Benchmark
  def arraySum(bh: Blackhole): Unit = {
    var sum = 0
    var i = 0
    while (i < array.length) { sum += array(i); i += 1 }
    bh.consume(sum)
  }
  
  @Benchmark
  def listFilter(bh: Blackhole): Unit = {
    bh.consume(list.filter(_ % 2 == 0))
  }
  
  @Benchmark
  def vectorFilter(bh: Blackhole): Unit = {
    bh.consume(vector.filter(_ % 2 == 0))
  }
  
  @Benchmark
  def listMapFlatMap(bh: Blackhole): Unit = {
    bh.consume(list.map(_ * 2).filter(_ > 100).take(10))
  }
  
  // Using view to avoid intermediate collections
  @Benchmark
  def listViewMapFlatMap(bh: Blackhole): Unit = {
    bh.consume(list.view.map(_ * 2).filter(_ > 100).take(10).toList)
  }
}

@State(Scope.Thread)
class StringBenchmark {
  
  val items: List[String] = List.fill(1000)("hello")
  
  @Benchmark
  def concatenation(): String = items.foldLeft("")(_ + ", " + _)
  
  @Benchmark
  def mkString(): String = items.mkString(", ")
  
  @Benchmark
  def stringBuilder(): String = {
    val sb = new StringBuilder
    items.foreach { s =>
      if (sb.nonEmpty) sb.append(", ")
      sb.append(s)
    }
    sb.toString()
  }
}

// Run benchmarks:
// sbt "jmh:run -i 10 -wi 5 -f2 -t1 .*CollectionBenchmark.*"
// Options: -i iterations, -wi warmup iterations, -f forks, -t threads
```

---

## Step 912: Async Profiler Setup

```bash
#!/bin/bash
# setup-profiler.sh

# Download Async Profiler
ASYNC_PROFILER_VERSION="3.0"
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)

wget "https://github.com/async-profiler/async-profiler/releases/download/v${ASYNC_PROFILER_VERSION}/async-profiler-${ASYNC_PROFILER_VERSION}-${OS}-${ARCH}.tar.gz"
tar xzf async-profiler-*.tar.gz
cd async-profiler-*/

# Profile running JVM
# 1. Find PID
PID=$(jps | grep "Main" | awk '{print $1}')

# 2. CPU profiling (30 seconds, flame graph output)
./bin/asprof -d 30 -o flamegraph -f /tmp/flamegraph-cpu.html $PID
echo "CPU flame graph: /tmp/flamegraph-cpu.html"

# 3. Allocation profiling
./bin/asprof -e alloc -d 30 -o flamegraph -f /tmp/flamegraph-alloc.html $PID
echo "Allocation flame graph: /tmp/flamegraph-alloc.html"

# 4. Wall-clock profiling (shows all threads including blocked)
./bin/asprof -e wall -d 30 -o flamegraph -f /tmp/flamegraph-wall.html $PID

# 5. Lock contention
./bin/asprof -e lock -d 30 -o flamegraph -f /tmp/flamegraph-lock.html $PID
```

```scala
// AgentProfiler.scala — Add profiler as Java agent

/*
# JVM flags for continuous profiling
-agentpath:/opt/async-profiler/lib/libasyncProfiler.so=start,event=cpu,file=/logs/cpu-profile.jfr,interval=10ms

# Or Java Flight Recorder (JFR) — built into JDK 11+
-XX:StartFlightRecording=duration=120s,filename=/logs/recording.jfr,settings=profile

# Analyze with JMC (Java Mission Control)
*/

object ProfilerHelper {
  
  // Programmatic JFR recording
  def startJFRRecording(name: String, durationSeconds: Int): Unit = {
    import jdk.jfr._
    
    val config = Configuration.getConfiguration("profile")
    val recording = new Recording(config)
    recording.setName(name)
    recording.setDuration(java.time.Duration.ofSeconds(durationSeconds))
    recording.setDestination(java.nio.file.Path.of(s"/logs/${name}.jfr"))
    recording.start()
    
    println(s"JFR recording '$name' started for ${durationSeconds}s")
  }
  
  // CPU time tracking
  def measureCPU[T](label: String)(f: => T): T = {
    val bean = java.lang.management.ManagementFactory.getThreadMXBean
    val cpuStart  = bean.getCurrentThreadCpuTime
    val wallStart = System.nanoTime()
    
    val result = f
    
    val cpuMs  = (bean.getCurrentThreadCpuTime - cpuStart) / 1_000_000.0
    val wallMs = (System.nanoTime() - wallStart) / 1_000_000.0
    
    println(f"[$label] CPU: ${cpuMs}%.2f ms, Wall: ${wallMs}%.2f ms, CPU%: ${cpuMs/wallMs*100}%.1f%%")
    result
  }
}
```

---

## Step 913: Memory Profiling

```scala
// MemoryProfiling.scala

object MemoryProfiling {
  
  // ===== Memory usage analysis =====
  def printMemoryUsage(label: String): Unit = {
    val rt = Runtime.getRuntime
    System.gc()
    Thread.sleep(100)
    
    val used  = (rt.totalMemory - rt.freeMemory) / (1024 * 1024)
    val total = rt.totalMemory / (1024 * 1024)
    val max   = rt.maxMemory / (1024 * 1024)
    
    println(f"[$label] Used: ${used}MB / Total: ${total}MB / Max: ${max}MB")
  }
  
  // ===== Object size estimation =====
  // Use Java Agent or jol-core
  /*
  import org.openjdk.jol.info.ClassLayout
  import org.openjdk.jol.info.GraphLayout
  
  case class SmallClass(x: Int, y: Int)
  case class LargeClass(data: Array[Byte], name: String, values: List[Int])
  
  def analyzeObjectSizes(): Unit = {
    println(ClassLayout.parseClass(classOf[SmallClass]).toPrintable())
    
    val large = LargeClass(new Array[Byte](1024), "test", List(1,2,3))
    println(GraphLayout.parseInstance(large).toFootprint())
  }
  */
  
  // ===== Detect memory leaks =====
  // Common patterns that cause leaks:
  
  // 1. Static/object-level caches without eviction
  // ❌ Unbounded cache
  private val cache = scala.collection.mutable.HashMap.empty[String, Array[Byte]]
  
  // ✅ Bounded cache with eviction
  private val boundedCache = com.github.benmanes.caffeine.cache.Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterAccess(10, java.util.concurrent.TimeUnit.MINUTES)
    .build[String, Array[Byte]]()
  
  // 2. ThreadLocal without removal
  // ❌ Leak in thread pool
  // val tl = new ThreadLocal[List[String]]()
  // tl.set(List.fill(10000)("big string"))
  // // Never called tl.remove() → leaks when threads are reused
  
  // 3. Listeners/callbacks not removed
  
  // ===== Heap dump analysis =====
  /*
  # Trigger heap dump
  jcmd <PID> VM.heap_dump /tmp/heapdump.hprof
  
  # Or on OOM
  -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/
  
  # Analyze with Eclipse MAT or JVisualVM
  # Look for:
  # - Leak suspects (objects growing without bound)
  # - Dominator tree (which objects hold most memory)
  # - Object histogram (count by class)
  */
}
```

---

## Step 914: Scala Benchmark Patterns

```scala
// ScalaBenchmarkPatterns.scala
import org.openjdk.jmh.annotations._
import java.util.concurrent.TimeUnit

@State(Scope.Benchmark)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@BenchmarkMode(Array(Mode.AverageTime))
class OptimizationBenchmark {
  
  val data = (1 to 10000).toVector
  
  // ===== Pattern 1: for comprehension vs map/flatMap =====
  
  @Benchmark
  def forComprehension(): Vector[Int] = {
    for {
      x <- data
      if x % 2 == 0
    } yield x * 2
  }
  
  @Benchmark
  def mapFilter(): Vector[Int] = {
    data.filter(_ % 2 == 0).map(_ * 2)
  }
  
  @Benchmark
  def withView(): Vector[Int] = {
    data.view.filter(_ % 2 == 0).map(_ * 2).toVector
  }
  
  // ===== Pattern 2: Option chains =====
  
  val map = Map(1 -> "a", 2 -> "b")
  
  @Benchmark
  def optionFlatMap(): Option[Int] = {
    map.get(1).flatMap(s => if (s.nonEmpty) Some(s.length) else None)
  }
  
  @Benchmark
  def optionForComp(): Option[Int] = {
    for {
      s <- map.get(1)
      if s.nonEmpty
    } yield s.length
  }
  
  // ===== Pattern 3: Recursion vs loop =====
  
  @Benchmark
  def foldLeftSum(): Long = data.map(_.toLong).foldLeft(0L)(_ + _)
  
  @Benchmark
  def whileLoopSum(): Long = {
    val arr = data.toArray
    var sum = 0L
    var i   = 0
    while (i < arr.length) { sum += arr(i); i += 1 }
    sum
  }
  
  @Benchmark
  def sumMethod(): Long = data.map(_.toLong).sum
}
```

---

## Step 915: Spark Performance

```scala
// SparkPerformance.scala
import org.apache.spark.sql.{SparkSession, Dataset, Row}
import org.apache.spark.sql.functions._

class SparkPerformanceGuide(spark: SparkSession) {
  import spark.implicits._
  
  // ===== Partition tuning =====
  def partitionTuning(): Unit = {
    val df = spark.read.parquet("s3://data/orders/")
    
    // Check current partitions
    println(s"Partitions: ${df.rdd.getNumPartitions}")
    
    // Rule of thumb: target 128MB per partition
    // Repartition for better parallelism
    val repartitioned = df.repartition(200)
    
    // Coalesce (reduce partitions, no full shuffle)
    val coalesced = df.coalesce(50)
    
    // Repartition by specific column (for joins/groupBy)
    val byCustomer = df.repartition(200, col("customer_id"))
  }
  
  // ===== Broadcast join for small tables =====
  def broadcastJoin(): Unit = {
    val orders   = spark.read.parquet("s3://data/orders/")
    val products = spark.read.parquet("s3://data/products/")  // small
    
    // Without broadcast: shuffle join (expensive)
    val withoutBroadcast = orders.join(products, "product_id")
    
    // With broadcast: distribute small table to all executors
    val withBroadcast = orders.join(broadcast(products), "product_id")
    
    // Auto-broadcast threshold (default 10MB)
    spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "50mb")
  }
  
  // ===== Avoid shuffles =====
  def avoidShuffles(): Unit = {
    val df = spark.read.parquet("s3://data/orders/")
    
    // ❌ Shuffles: groupBy, join, distinct, repartition
    val grouped = df.groupBy("customer_id").agg(sum("total"))
    
    // ✅ Pre-partition data by frequently-joined column
    // Write with partitionBy:
    df.write
      .mode("overwrite")
      .partitionBy("customer_id")
      .parquet("s3://data/orders-partitioned/")
    
    // Future reads will prune partitions automatically
  }
  
  // ===== Cache/persist =====
  def cacheStrategy(): Unit = {
    val df = spark.read.parquet("s3://data/large-table/")
    
    import org.apache.spark.storage.StorageLevel
    
    // Cache in memory (fast but limited)
    df.cache()
    
    // Persist with specific level
    df.persist(StorageLevel.MEMORY_AND_DISK_SER)  // serialize to save memory
    
    // Unpersist when done
    df.unpersist()
  }
  
  // ===== Predicate pushdown =====
  def predicatePushdown(): Unit = {
    val df = spark.read.parquet("s3://data/orders/")
    
    // ✅ Filter early — Spark pushes to parquet file level
    val filtered = df
      .filter(col("order_date") >= "2024-01-01")  // pushed to reader
      .filter(col("status") === "COMPLETED")
      .select("id", "customer_id", "total")  // column pruning
    
    // Show execution plan
    filtered.explain(extended = true)
  }
  
  // ===== Adaptive Query Execution (AQE) =====
  def enableAQE(): Unit = {
    spark.conf.set("spark.sql.adaptive.enabled", "true")
    spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
    spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
  }
}
```

---

## Step 916: HTTP Service Benchmarking

```scala
// HTTPBenchmarking.scala
// wrk, hey, ab for HTTP load testing

/*
# Install wrk
brew install wrk  # macOS
apt install wrk   # Ubuntu

# Basic load test: 10 threads, 100 connections, 30 seconds
wrk -t10 -c100 -d30s http://localhost:8080/api/v1/orders

# With custom script
wrk -t10 -c100 -d30s -s wrk-post.lua http://localhost:8080/api/v1/orders

# wrk-post.lua
wrk.method = "POST"
wrk.body   = '{"customerId":"123","total":99.99}'
wrk.headers["Content-Type"] = "application/json"
wrk.headers["Authorization"] = "Bearer my-token"

# Results interpretation:
# Latency: avg, stdev, max, +/-stdev
# Req/Sec: avg, stdev, max, +/-stdev
# Transfer/sec
*/

// hey (Go-based HTTP load tester)
/*
# Install
go install github.com/rakyll/hey@latest

# Run
hey -n 10000 -c 100 -m GET http://localhost:8080/health

# POST with body
hey -n 10000 -c 100 -m POST \
  -H "Content-Type: application/json" \
  -d '{"customerId":"123","total":99.99}' \
  http://localhost:8080/api/v1/orders
*/

// Scala HTTP server for benchmarking
import akka.actor.typed.ActorSystem
import akka.actor.typed.scaladsl.Behaviors
import akka.http.scaladsl.Http
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.model.ContentTypes

object BenchmarkServer extends App {
  implicit val system: ActorSystem[Nothing] = ActorSystem(Behaviors.empty, "benchmark")
  
  val routes = concat(
    path("health") {
      get { complete("""{"status":"ok"}""") }
    },
    path("compute") {
      get {
        // CPU-bound work
        val result = (1 to 10000).sum
        complete(s"""{"result":$result}""")
      }
    },
    path("json") {
      get {
        import spray.json._
        import DefaultJsonProtocol._
        val data = Map("id" -> 1, "name" -> "test", "value" -> 99.99)
        complete(ContentTypes.`application/json`, data.toJson.compactPrint)
      }
    }
  )
  
  Http().newServerAt("0.0.0.0", 8080).bind(routes)
  println("Benchmark server started on :8080")
}
```

---

## Step 917: Database Query Profiling

```scala
// DatabaseQueryProfiling.scala
import doobie._
import doobie.implicits._
import cats.effect.IO
import scala.concurrent.duration._

class QueryProfiler(xa: Transactor[IO]) {
  
  // ===== Query execution time =====
  def profileQuery[A](name: String, query: ConnectionIO[A]): IO[A] = {
    val start = System.nanoTime()
    query.transact(xa).flatTap { _ =>
      IO {
        val ms = (System.nanoTime() - start) / 1_000_000.0
        println(f"[$name] ${ms}%.2f ms")
      }
    }
  }
  
  // ===== EXPLAIN ANALYZE =====
  def explainAnalyze(sql: String): IO[List[String]] = {
    Fragment.const(s"EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) $sql")
      .query[String]
      .to[List]
      .transact(xa)
  }
  
  // ===== Slow query log =====
  def enableSlowQueryLog(): ConnectionIO[Unit] = {
    // PostgreSQL: log queries slower than 1 second
    sql"SET log_min_duration_statement = 1000".update.run.void
  }
  
  // ===== N+1 query detection =====
  
  // ❌ N+1: 1 query for orders + N queries for each customer
  def naiveLoad(): IO[List[String]] = {
    val getOrders = sql"SELECT id, customer_id FROM orders LIMIT 100"
      .query[(Long, Long)].to[List]
    
    getOrders.transact(xa).flatMap { orders =>
      // This makes 100 DB queries!
      orders.traverse { case (_, customerId) =>
        sql"SELECT name FROM customers WHERE id = $customerId"
          .query[String].unique.transact(xa)
      }
    }
  }
  
  // ✅ Batch load: 2 queries total
  def batchLoad(): IO[List[(Long, String)]] = {
    sql"""
      SELECT o.id, c.name
      FROM orders o
      JOIN customers c ON o.customer_id = c.id
      LIMIT 100
    """.query[(Long, String)].to[List].transact(xa)
  }
}
```

---

## Step 918: Profiling in Production

```scala
// ProductionProfiling.scala

/*
===== Continuous Profiling =====

1. Pyroscope (open-source, lightweight)
   - eBPF-based sampling profiler
   - < 2% CPU overhead
   - Stores profiles over time

2. Datadog APM
   - Auto-instrumentation
   - Distributed tracing + profiling

3. Elastic APM

===== Pyroscope with Scala =====
*/

// build.sbt
// "io.pyroscope" % "agent" % "0.12.0"

object PyroscopeConfig {
  
  def configure(): Unit = {
    import io.pyroscope.agent.PyroscopeAgent
    import io.pyroscope.agent.Config
    import io.pyroscope.javaspy.EventType
    
    PyroscopeAgent.start(
      new Config.Builder()
        .setApplicationName("order-service")
        .setProfilingEvent(EventType.ITIMER)
        .setServerAddress("http://pyroscope:4040")
        .setLabels(Map(
          "region"  -> sys.env.getOrElse("AWS_REGION", "us-east-1"),
          "version" -> sys.env.getOrElse("APP_VERSION", "unknown")
        ).asJava)
        .build()
    )
    
    println("Pyroscope continuous profiling started")
  }
}

// ===== Thread dump analysis =====
object ThreadDumpAnalysis {
  
  def printThreadDump(): Unit = {
    val bean = java.lang.management.ManagementFactory.getThreadMXBean
    val threads = bean.dumpAllThreads(true, true)
    
    threads.foreach { info =>
      println(s"\n=== Thread: ${info.getThreadName} [${info.getThreadState}] ===")
      info.getStackTrace.take(5).foreach { elem =>
        println(s"  at $elem")
      }
    }
  }
  
  def detectDeadlocks(): Unit = {
    val bean = java.lang.management.ManagementFactory.getThreadMXBean
    val deadlockedIds = bean.findDeadlockedThreads()
    
    if (deadlockedIds != null && deadlockedIds.nonEmpty) {
      println(s"DEADLOCK DETECTED! Thread IDs: ${deadlockedIds.mkString(", ")}")
      val infos = bean.getThreadInfo(deadlockedIds)
      infos.foreach(info => println(s"Deadlocked thread: ${info.getThreadName}"))
    } else {
      println("No deadlocks detected")
    }
  }
  
  def printBlockedThreads(): Unit = {
    val bean = java.lang.management.ManagementFactory.getThreadMXBean
    val blocked = bean.dumpAllThreads(false, false)
      .filter(_.getThreadState == Thread.State.BLOCKED)
    
    println(s"Blocked threads: ${blocked.length}")
    blocked.foreach(t => println(s"  - ${t.getThreadName}: blocked on ${t.getLockName}"))
  }
}
```

---

## Step 919: Benchmark Results Analysis

```scala
// BenchmarkAnalysis.scala

object BenchmarkAnalysis {
  
  case class BenchmarkResult(
    name: String,
    mode: String,
    score: Double,
    error: Double,
    unit: String
  )
  
  def compareResults(baseline: BenchmarkResult, optimized: BenchmarkResult): Unit = {
    val improvement = (baseline.score - optimized.score) / baseline.score * 100
    
    println(s"=== Benchmark Comparison ===")
    println(f"Baseline:  ${baseline.name}  ${baseline.score}%.2f ± ${baseline.error}%.2f ${baseline.unit}")
    println(f"Optimized: ${optimized.name} ${optimized.score}%.2f ± ${optimized.error}%.2f ${optimized.unit}")
    println(f"Improvement: ${improvement}%.1f%%")
    
    if (improvement > 20) println("✓ Significant improvement!")
    else if (improvement > 0) println("→ Minor improvement")
    else println("✗ Regression!")
  }
  
  // Typical benchmark results for collections:
  val sampleResults = List(
    BenchmarkResult("listSum",         "avgt", 523.2, 12.3, "us/op"),
    BenchmarkResult("vectorSum",       "avgt", 312.5, 8.1,  "us/op"),
    BenchmarkResult("arraySum",        "avgt",  85.3, 2.4,  "us/op"),
    BenchmarkResult("concatenation",   "avgt", 8432.1, 245.0, "us/op"),
    BenchmarkResult("mkString",        "avgt",  421.3, 15.2, "us/op"),
    BenchmarkResult("stringBuilder",   "avgt",  398.7, 12.8, "us/op")
  )
  
  def printResultsTable(results: List[BenchmarkResult]): Unit = {
    println(f"${"Name"}%-30s ${"Score"}%12s ${"Error"}%10s ${"Unit"}%-15s")
    println("-" * 70)
    results.sortBy(_.score).foreach { r =>
      println(f"${r.name}%-30s ${r.score}%12.2f ${r.error}%10.2f ${r.unit}%-15s")
    }
  }
}
```

---

## Step 920: Performance CI Integration

```yaml
# .github/workflows/benchmarks.yml
name: Performance Benchmarks

on:
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # nightly

jobs:
  benchmark:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup JDK
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      
      - name: Cache SBT
        uses: actions/cache@v4
        with:
          path: ~/.sbt ~/.ivy2
          key: sbt-${{ hashFiles('build.sbt') }}
      
      - name: Run JMH benchmarks
        run: |
          sbt "jmh:run -i 5 -wi 3 -f1 -rf json -rff benchmark-results.json"
      
      - name: Compare with baseline
        run: |
          # Compare results with stored baseline
          if [ -f benchmark-baseline.json ]; then
            python3 scripts/compare_benchmarks.py \
              benchmark-baseline.json \
              benchmark-results.json \
              --threshold 10  # fail if > 10% regression
          fi
      
      - name: Store baseline (main branch only)
        if: github.ref == 'refs/heads/main'
        run: cp benchmark-results.json benchmark-baseline.json
      
      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: benchmark-results
          path: benchmark-results.json
      
      - name: Comment PR with results
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs')
            const results = JSON.parse(fs.readFileSync('benchmark-results.json'))
            const summary = results.map(r => 
              `| ${r.benchmark} | ${r.primaryMetric.score.toFixed(2)} ${r.primaryMetric.scoreUnit} |`
            ).join('\n')
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Benchmark Results\n| Benchmark | Score |\n|---|---|\n${summary}`
            })
```

---

## สรุป Part 92: Profiling & Benchmarks

| Tool | Use Case | Overhead |
|------|----------|---------|
| JMH | Micro-benchmarks | Test only |
| Async Profiler | CPU/alloc profiling | < 2% |
| JFR | Production profiling | < 1% |
| Gatling | Load testing | External |
| wrk/hey | HTTP benchmarking | External |
| Pyroscope | Continuous profiling | < 2% |

---

## แบบฝึกหัด Part 92

1. **JMH Benchmark**: เขียน benchmark เปรียบเทียบ 5 วิธีในการหา sum ของ List[Int]: foldLeft, reduce, sum, while loop, recursive

2. **Async Profiler**: profile Scala service ขณะ handling 100 req/s เป็นเวลา 60s และวิเคราะห์ flame graph หา top hot paths

3. **Memory Leak**: สร้าง service ที่มี memory leak (unbounded cache), profile ด้วย heap dump analysis, แก้ไข

4. **N+1 Detection**: implement query counter middleware ที่ detect N+1 queries โดย log warning เมื่อ query count > threshold ต่อ request

5. **CI Integration**: setup GitHub Actions ที่ run JMH benchmarks และ fail PR ถ้า performance regression > 15%

---

## ไปต่อ: Part 93 — Functional Architecture
[→ Part 93: Functional Architecture](./part-93-functional-architecture.md)
