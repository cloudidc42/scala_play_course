# Part 37: Akka Streams

## Steps 361-370: Source, Flow, Sink, Backpressure, Graphs, Fan-out/Fan-in

---

## Step 361: Akka Streams พื้นฐาน

```scala
// build.sbt:
// libraryDependencies += "com.typesafe.akka" %% "akka-stream" % "2.7.0"

import akka.actor.ActorSystem
import akka.stream._
import akka.stream.scaladsl._
import scala.concurrent._
import scala.concurrent.duration._

/*
 * Akka Streams = reactive streams implementation
 * Components:
 *   Source[Out, Mat]     — produces elements
 *   Flow[In, Out, Mat]   — transforms elements
 *   Sink[In, Mat]        — consumes elements
 *   Mat                  — materialized value (result after running)
 *
 * Backpressure: downstream controls upstream speed
 * "I'm ready for more" signal flows upstream
 */

object StreamsBasics extends App {
  implicit val system = ActorSystem("streams-basics")
  implicit val ec = system.dispatcher
  
  // ===== Source: data producer =====
  val fromRange: Source[Int, NotUsed] = Source(1 to 10)
  val fromList: Source[String, NotUsed] = Source(List("a", "b", "c"))
  val fromSingle: Source[Int, NotUsed] = Source.single(42)
  val fromEmpty: Source[Nothing, NotUsed] = Source.empty
  val fromTick: Source[Int, akka.actor.Cancellable] = 
    Source.tick(0.seconds, 100.millis, 1)  // Periodic
  val fromIterable: Source[Int, NotUsed] = Source.fromIterator(() => Iterator.from(1))
  
  // ===== Flow: transformation =====
  val double: Flow[Int, Int, NotUsed] = Flow[Int].map(_ * 2)
  val filterEven: Flow[Int, Int, NotUsed] = Flow[Int].filter(_ % 2 == 0)
  val toString2: Flow[Int, String, NotUsed] = Flow[Int].map(_.toString)
  
  // ===== Sink: consumer =====
  val toList: Sink[Int, Future[List[Int]]] = Sink.seq[Int].mapMaterializedValue(_.map(_.toList))
  val toSum: Sink[Int, Future[Int]] = Sink.fold[Int, Int](0)(_ + _)
  val toPrint: Sink[Any, Future[akka.Done]] = Sink.foreach(println)
  val ignore: Sink[Any, Future[akka.Done]] = Sink.ignore
  
  println("=== Basic Stream Pipeline ===")
  
  // Connect and run
  val result1: Future[akka.Done] = Source(1 to 10)
    .map(_ * 2)
    .filter(_ % 3 == 0)
    .runWith(Sink.foreach(n => println(s"  $n")))
  
  Await.ready(result1, 5.seconds)
  
  // Materialized values
  val sum: Future[Int] = Source(1 to 100)
    .runWith(Sink.fold(0)(_ + _))
  println(s"Sum 1-100: ${Await.result(sum, 2.seconds)}")
  
  val items: Future[Seq[Int]] = Source(1 to 10)
    .filter(_ % 2 == 0)
    .runWith(Sink.seq)
  println(s"Even items: ${Await.result(items, 2.seconds)}")
  
  // Composing streams
  val processedStream = Source(1 to 20)
    .via(Flow[Int].filter(_ % 2 == 0))      // Keep even
    .via(Flow[Int].map(_ * 3))              // Multiply by 3
    .via(Flow[Int].take(5))                 // First 5
    .to(Sink.foreach(n => print(s"$n ")))   // Print
  
  processedStream.run()
  Thread.sleep(100)
  println()
  
  system.terminate()
}
```

---

## Step 362: Backpressure และ Throttling

```scala
import akka.actor.ActorSystem
import akka.stream._
import akka.stream.scaladsl._
import scala.concurrent._
import scala.concurrent.duration._

object BackpressureDemo extends App {
  implicit val system = ActorSystem("backpressure")
  implicit val ec = system.dispatcher
  
  // Fast producer, slow consumer — backpressure in action
  val fastProducer = Source(1 to 1000)
  
  val slowConsumer = Sink.foreach[Int] { n =>
    Thread.sleep(10)  // Slow processing
    if (n % 100 == 0) println(s"  Consumed: $n")
  }
  
  println("=== Backpressure ===")
  Await.ready(fastProducer.runWith(slowConsumer), 30.seconds)
  
  // Throttle — rate limiting
  println("\n=== Throttle ===")
  val throttled = Source(1 to 20)
    .throttle(5, 1.second)  // Max 5 elements per second
    .runWith(Sink.foreach(n => println(s"  $n at ${System.currentTimeMillis() % 10000}")))
  
  Await.ready(throttled, 10.seconds)
  
  // Buffer — absorb bursts
  println("\n=== Buffering ===")
  Source(1 to 50)
    .buffer(10, OverflowStrategy.backpressure)   // Buffer 10 elements
    .map { n =>
      Thread.sleep(20)  // Simulate slow processing
      n
    }
    .take(10)
    .runForeach(n => print(s"$n "))
    .onComplete(_ => println())
  
  Thread.sleep(500)
  
  // Overflow strategies
  println("\n=== Overflow Strategies ===")
  
  // dropHead — drop oldest when buffer full
  Source(1 to 100)
    .buffer(10, OverflowStrategy.dropHead)
    .throttle(1, 100.millis)
    .take(5)
    .runForeach(n => print(s"$n "))
    .onComplete(_ => println(" (dropHead)"))
  
  // dropBuffer — drop all when overflow
  Source(1 to 100)
    .buffer(10, OverflowStrategy.dropBuffer)
    .throttle(1, 100.millis)
    .take(5)
    .runForeach(n => print(s"$n "))
    .onComplete(_ => println(" (dropBuffer)"))
  
  Thread.sleep(2000)
  system.terminate()
}
```

---

## Step 363: Flows — Advanced Transformations

```scala
import akka.actor.ActorSystem
import akka.stream._
import akka.stream.scaladsl._
import scala.concurrent._
import scala.concurrent.duration._

object AdvancedFlows extends App {
  implicit val system = ActorSystem("flows")
  implicit val ec = system.dispatcher
  
  // mapAsync — async transformation with parallelism
  println("=== mapAsync ===")
  
  def fetchData(id: Int): Future[String] = Future {
    Thread.sleep(50 + scala.util.Random.nextInt(50))
    s"data-$id"
  }
  
  Await.ready(
    Source(1 to 10)
      .mapAsync(parallelism = 4)(fetchData)  // 4 concurrent fetches
      .runForeach(s => print(s"$s ")),
    10.seconds
  )
  println()
  
  // mapAsyncUnordered — for when order doesn't matter
  println("\n=== mapAsyncUnordered (faster, unordered) ===")
  Await.ready(
    Source(1 to 10)
      .mapAsyncUnordered(4)(fetchData)
      .runForeach(s => print(s"$s ")),
    10.seconds
  )
  println()
  
  // grouped — mini-batches
  println("\n=== Grouped ===")
  Await.ready(
    Source(1 to 20)
      .grouped(5)  // Batch 5 at a time
      .runForeach(batch => println(s"  Batch: $batch")),
    5.seconds
  )
  
  // groupedWithin — time or size window
  println("\n=== GroupedWithin ===")
  val start = System.currentTimeMillis()
  Await.ready(
    Source.tick(0.millis, 30.millis, 1)
      .take(15)
      .scan(0)(_ + _)  // Running counter
      .groupedWithin(4, 100.millis)  // Group by 4 items OR 100ms
      .runForeach(group => println(s"  ${System.currentTimeMillis() - start}ms: $group")),
    5.seconds
  )
  
  // sliding window
  println("\n=== Sliding ===")
  Await.ready(
    Source(1 to 10)
      .sliding(3, 1)  // Window of 3, step 1
      .runForeach(window => println(s"  window: $window")),
    5.seconds
  )
  
  // fold — accumulate
  val histogram: Future[Map[Int, Int]] = Source(
    List(1,2,1,3,2,1,4,3,2,1,5,4,3,2,1)
  ).runFold(Map.empty[Int, Int]) { (acc, n) =>
    acc + (n -> (acc.getOrElse(n, 0) + 1))
  }
  println(s"\n=== Histogram ===")
  println(s"  ${Await.result(histogram, 2.seconds)}")
  
  // scan — running aggregation
  println("\n=== Scan (running stats) ===")
  Await.ready(
    Source(List(3.0, 1.0, 4.0, 1.0, 5.0, 9.0, 2.0, 6.0))
      .scan((0, 0.0, 0.0)) { case ((count, sum, max), x) =>
        (count + 1, sum + x, Math.max(max, x))
      }
      .map { case (count, sum, max) =>
        if (count == 0) "start" else f"count=$count, avg=${sum/count}%.2f, max=$max"
      }
      .runForeach(s => println(s"  $s")),
    5.seconds
  )
  
  system.terminate()
}
```

---

## Step 364: Graphs — Fan-out and Fan-in

```scala
import akka.actor.ActorSystem
import akka.stream._
import akka.stream.scaladsl._
import scala.concurrent._
import scala.concurrent.duration._

object GraphsDemo extends App {
  implicit val system = ActorSystem("graphs")
  implicit val ec = system.dispatcher
  
  // Broadcast — send to multiple sinks
  println("=== Broadcast ===")
  val broadcastResult = RunnableGraph.fromGraph(GraphDSL.create() { implicit builder =>
    import GraphDSL.Implicits._
    
    val source = Source(1 to 10)
    val broadcast = builder.add(Broadcast[Int](3))
    val evenSink = Sink.foreach[Int](n => print(s"even:$n "))
    val oddSink = Sink.foreach[Int](n => print(s"odd:$n "))
    val allSink = Sink.foreach[Int](n => print(s"all:$n "))
    
    source ~> broadcast
    broadcast.out(0).filter(_ % 2 == 0) ~> evenSink
    broadcast.out(1).filter(_ % 2 != 0) ~> oddSink
    broadcast.out(2) ~> allSink
    
    ClosedShape
  }).run()
  
  Thread.sleep(200)
  println()
  
  // Merge — combine multiple sources
  println("\n=== Merge ===")
  val mergeResult = RunnableGraph.fromGraph(GraphDSL.create() { implicit builder =>
    import GraphDSL.Implicits._
    
    val source1 = Source(List(1, 3, 5, 7, 9))
    val source2 = Source(List(2, 4, 6, 8, 10))
    val merge = builder.add(Merge[Int](2))
    val sink = Sink.foreach[Int](n => print(s"$n "))
    
    source1 ~> merge.in(0)
    source2 ~> merge.in(1)
    merge.out ~> sink
    
    ClosedShape
  }).run()
  
  Thread.sleep(200)
  println()
  
  // Zip — pair elements from two sources
  println("\n=== Zip ===")
  Await.ready(
    Source(List("Alice", "Bob", "Carol"))
      .zip(Source(List(30, 25, 35)))
      .runForeach { case (name, age) => println(s"  $name: $age") },
    5.seconds
  )
  
  // Balance — distribute to workers
  println("\n=== Balance (load balancing) ===")
  val balanceResult = RunnableGraph.fromGraph(GraphDSL.create(Sink.seq[String]) { implicit builder =>
    sink =>
    import GraphDSL.Implicits._
    
    val source = Source(1 to 20)
    val balance = builder.add(Balance[Int](3))
    val merge = builder.add(Merge[String](3))
    
    source ~> balance
    
    (0 to 2).foreach { workerId =>
      balance.out(workerId)
        .map(n => s"worker-$workerId:$n")
        ~> merge.in(workerId)
    }
    
    merge.out ~> sink.in
    ClosedShape
  })
  
  val results = Await.result(balanceResult.run(), 5.seconds)
  results.take(6).foreach(r => println(s"  $r"))
  
  // Complex graph — ETL pipeline
  println("\n=== ETL Pipeline ===")
  
  case class RawRecord(line: String)
  case class ParsedRecord(fields: List[String])
  case class EnrichedRecord(id: String, name: String, amount: Double, category: String)
  
  val etlResult = Source(List(
    "1,Alice,1500.00",
    "2,Bob,2500.50",
    "3,Carol,800.00",
    "invalid,line",
    "4,Dave,3000.00"
  ))
  .map(line => RawRecord(line))
  .map { r =>
    val parts = r.line.split(",")
    try ParsedRecord(parts.toList)
    catch { case _: Exception => ParsedRecord(List.empty) }
  }
  .filter(_.fields.size == 3)
  .map { p =>
    try {
      val amount = p.fields(2).toDouble
      val category = if (amount > 2000) "premium" else "standard"
      Some(EnrichedRecord(p.fields(0), p.fields(1), amount, category))
    } catch {
      case _: Exception => None
    }
  }
  .collect { case Some(r) => r }
  .runWith(Sink.seq)
  
  val records = Await.result(etlResult, 5.seconds)
  records.foreach(r => println(s"  ${r.id}: ${r.name}, ${r.amount}, ${r.category}"))
  
  system.terminate()
}
```

---

## Step 365-370: Stream Error Handling and Real-World Patterns

```scala
import akka.actor.ActorSystem
import akka.stream._
import akka.stream.scaladsl._
import scala.concurrent._
import scala.concurrent.duration._
import scala.util.{Try, Success, Failure}

object StreamErrorHandling extends App {
  implicit val system = ActorSystem("stream-errors")
  implicit val ec = system.dispatcher
  
  println("=== Stream Error Handling ===")
  
  // recover — replace failed element
  Await.ready(
    Source(List("1", "abc", "3", "xyz", "5"))
      .map(s => Try(s.toInt))
      .collect {
        case Success(n) => n
        case Failure(_) => -1  // Default on parse error
      }
      .runForeach(n => print(s"$n ")),
    5.seconds
  )
  println()
  
  // supervisionStrategy — skip bad elements
  println("\n=== Supervision Strategy ===")
  
  implicit val strategy = ActorAttributes.supervisionStrategy { e =>
    println(s"  Stream error: ${e.getMessage}, resuming...")
    Supervision.Resume  // Skip element and continue
  }
  
  Await.ready(
    Source(List(1, 2, 0, 3, 0, 5))
      .map(n => 10 / n)  // Division by zero for 0s
      .withAttributes(strategy)
      .runForeach(n => print(s"$n ")),
    5.seconds
  )
  println()
  
  // recoverWithRetries
  println("\n=== recoverWithRetries ===")
  var attempt = 0
  Await.ready(
    Source.single(1)
      .flatMapConcat { _ =>
        attempt += 1
        println(s"  Attempt $attempt")
        if (attempt < 3) Source.failed(new Exception("Transient"))
        else Source.single("success")
      }
      .recoverWithRetries(5, {
        case _: Exception => Source.empty
      })
      .runForeach(println),
    10.seconds
  )
  
  // Log and metrics in stream
  println("\n=== Logging in Stream ===")
  Await.ready(
    Source(1 to 20)
      .log("before-filter")
      .filter(_ % 2 == 0)
      .log("after-filter")
      .map(_ * 10)
      .take(3)
      .runForeach(n => println(s"  Result: $n")),
    5.seconds
  )
  
  // File processing stream
  println("\n=== File Processing ===")
  import akka.util.ByteString
  
  val fileContent = "line 1: hello world\nline 2: scala streams\nline 3: backpressure"
  
  Await.ready(
    Source.single(ByteString(fileContent))
      .via(Framing.delimiter(ByteString("\n"), maximumFrameLength = 256))
      .map(_.utf8String)
      .zipWithIndex
      .map { case (line, idx) => s"${idx + 1}: $line" }
      .runForeach(println),
    5.seconds
  )
  
  // Kafka-like stream (simulated)
  println("\n=== Message Stream Processing ===")
  case class Message(key: String, value: String, partition: Int)
  
  val messageStream = Source(List(
    Message("user:1", """{"event":"login","userId":1}""", 0),
    Message("user:2", """{"event":"purchase","userId":2,"amount":100}""", 1),
    Message("user:1", """{"event":"logout","userId":1}""", 0),
    Message("order:1", """{"event":"created","orderId":1}""", 2),
    Message("order:1", """{"event":"paid","orderId":1}""", 2)
  ))
  
  Await.ready(
    messageStream
      .groupBy(3, _.partition)  // 3 sub-streams by partition
      .map { msg =>
        val eventType = msg.value.split(":")(1).split("\"")(1)
        s"${msg.partition}:${msg.key}:$eventType"
      }
      .mergeSubstreams
      .runForeach(println),
    5.seconds
  )
  
  system.terminate()
}
```

---

## สรุป Part 37

| Component | วัตถุประสงค์ | ตัวอย่าง |
|-----------|------------|---------|
| `Source[O,M]` | Produce data | `Source(1 to 10)`, `Source.tick` |
| `Flow[I,O,M]` | Transform | `Flow[Int].map(_ * 2)` |
| `Sink[I,M]` | Consume data | `Sink.fold`, `Sink.foreach` |
| `Broadcast[T]` | Fan-out | Split stream to N branches |
| `Merge[T]` | Fan-in | Combine N streams |
| `Balance[T]` | Load balance | Distribute to N workers |
| `Zip[A,B]` | Combine pairs | Pair elements from 2 sources |
| `buffer` | Absorb bursts | `buffer(100, OverflowStrategy.*)` |
| `throttle` | Rate limiting | `throttle(10, 1.second)` |
| `mapAsync` | Async transform | `mapAsync(4)(asyncFn)` |
| `groupedWithin` | Time windows | `groupedWithin(100, 1.second)` |

---

## แบบฝึกหัด Part 37

**ข้อ 1:** สร้าง log file processor ที่ parse, filter, aggregate log entries ด้วย streams

**ข้อ 2:** Implement real-time analytics pipeline: ingest events → compute metrics → publish results

**ข้อ 3:** สร้าง image processing pipeline ด้วย parallel mapAsync ที่ resize, compress, watermark

**ข้อ 4:** Implement stream-based CSV import ที่ validate, transform, และ batch insert ด้วย backpressure

**ข้อ 5:** สร้าง rate-limited API caller ที่ respect retry-after headers และ circuit breaker

---

➡️ ต่อไป: [Part 38 — Akka HTTP](part-38-akka-http.md)
