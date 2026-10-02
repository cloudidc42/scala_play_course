# Part 32: Execution Contexts

## Steps 311-320: ThreadPools, ExecutionContext, Blocking I/O, Custom Schedulers

---

## Step 311: ExecutionContext พื้นฐาน

```scala
import scala.concurrent._
import scala.concurrent.duration._
import java.util.concurrent.{Executors, ThreadPoolExecutor, TimeUnit}

object ExecutionContextBasics extends App {
  
  // ExecutionContext = where Futures run (thread pool)
  
  // 1. Global (ForkJoin) — default, good for CPU-bound
  val global: ExecutionContext = ExecutionContext.global
  
  // 2. Single thread
  val single: ExecutionContext = ExecutionContext.fromExecutor(
    Executors.newSingleThreadExecutor()
  )
  
  // 3. Fixed thread pool
  val fixed: ExecutionContext = ExecutionContext.fromExecutorService(
    Executors.newFixedThreadPool(4)
  )
  
  // 4. Cached thread pool — creates threads as needed (for I/O-bound)
  val cached: ExecutionContext = ExecutionContext.fromExecutorService(
    Executors.newCachedThreadPool()
  )
  
  // 5. Named threads (better for debugging)
  val namedPool = Executors.newFixedThreadPool(2, new java.util.concurrent.ThreadFactory {
    private var count = 0
    def newThread(r: Runnable): Thread = {
      count += 1
      val t = new Thread(r, s"worker-$count")
      t.setDaemon(true)
      t
    }
  })
  val named: ExecutionContext = ExecutionContext.fromExecutorService(namedPool)
  
  // Observe which thread runs each Future
  def identifyThread(label: String)(implicit ec: ExecutionContext): Future[String] = 
    Future {
      val threadName = Thread.currentThread().getName
      s"$label on $threadName"
    }
  
  println("=== Thread Pool Demo ===")
  
  implicit val ec = fixed
  
  val futures = List(
    identifyThread("task-1"),
    identifyThread("task-2"),
    identifyThread("task-3"),
    identifyThread("task-4")
  )
  
  val all = Future.sequence(futures)
  Await.result(all, 5.seconds).foreach(println)
  
  // Different EC for different work types
  println("\n=== Separate EC for I/O ===")
  
  val ioEC: ExecutionContext = ExecutionContext.fromExecutorService(
    Executors.newFixedThreadPool(20)  // Large pool for I/O blocking
  )
  
  val cpuEC: ExecutionContext = ExecutionContext.global  // ForkJoin for CPU
  
  // I/O operations on ioEC
  def readDataAsync(id: Int): Future[String] = Future {
    Thread.sleep(100)  // Simulate I/O
    s"data-$id"
  }(ioEC)
  
  // CPU operations on cpuEC
  def processData(data: String): Future[String] = Future {
    // Heavy computation
    val processed = data.toUpperCase.split("-").mkString("_")
    processed
  }(cpuEC)
  
  val pipeline = for {
    data      <- readDataAsync(42)(ioEC)
    processed <- processData(data)(cpuEC)
  } yield processed
  
  println(s"Pipeline result: ${Await.result(pipeline, 5.seconds)}")
  
  // Shutdown thread pools
  namedPool.shutdown()
  namedPool.awaitTermination(5, TimeUnit.SECONDS)
}
```

---

## Step 312: Blocking I/O สำหรับ Future

```scala
import scala.concurrent._
import scala.concurrent.duration._
import ExecutionContext.Implicits.global

object BlockingIODemo extends App {
  
  // WRONG: blocking in global thread pool starves other tasks
  def badBlocking(id: Int): Future[String] = Future {
    Thread.sleep(1000)  // Blocks a global thread!
    s"result-$id"
  }
  
  // CORRECT: wrap blocking code with blocking { }
  def goodBlocking(id: Int): Future[String] = Future {
    blocking {  // Tells ForkJoinPool to compensate with extra thread
      Thread.sleep(1000)
      s"result-$id"
    }
  }
  
  // BEST: use dedicated I/O thread pool
  val ioPool = ExecutionContext.fromExecutorService(
    Executors.newCachedThreadPool()
  )
  
  def bestBlocking(id: Int): Future[String] = Future {
    Thread.sleep(1000)  // OK — dedicated I/O pool
    s"result-$id"
  }(ioPool)
  
  import java.util.concurrent.Executors
  
  println("=== Blocking I/O Patterns ===")
  
  // Demonstrate: run 10 "blocking" tasks
  val start = System.currentTimeMillis()
  val results = Await.result(
    Future.sequence((1 to 5).map(bestBlocking).toList),
    15.seconds
  )
  println(s"5 tasks done in ${System.currentTimeMillis() - start}ms (parallel in I/O pool)")
  
  // Database simulation with connection pool
  class ConnectionPool(size: Int) {
    private val semaphore = new java.util.concurrent.Semaphore(size)
    private val ioEC = ExecutionContext.fromExecutorService(
      Executors.newFixedThreadPool(size)
    )
    
    def execute[A](query: String)(block: => A): Future[A] = Future {
      semaphore.acquire()
      try {
        println(s"  [DB] Executing: $query on ${Thread.currentThread.getName}")
        Thread.sleep(100)  // Simulate query time
        block
      } finally {
        semaphore.release()
      }
    }(ioEC)
    
    def shutdown(): Unit = ioEC match {
      case ec: java.util.concurrent.ExecutorService => ec.shutdown()
      case _ => ()
    }
  }
  
  val pool = new ConnectionPool(3)
  
  println("\n=== Connection Pool ===")
  val queries = (1 to 6).map(i =>
    pool.execute(s"SELECT * FROM table_$i") {
      Map("id" -> i, "value" -> s"row-$i")
    }
  )
  
  val queryResults = Await.result(Future.sequence(queries.toList), 10.seconds)
  queryResults.foreach(r => println(s"  $r"))
  
  Thread.sleep(200)
}
```

---

## Step 313-320: Custom Schedulers and Thread Management

```scala
import scala.concurrent._
import scala.concurrent.duration._
import java.util.concurrent._
import java.util.concurrent.atomic._

object CustomSchedulerDemo extends App {
  
  // Scheduled executor — run tasks at specific times
  val scheduler = Executors.newScheduledThreadPool(2)
  val ec = ExecutionContext.fromExecutorService(scheduler)
  
  // Schedule one-shot
  def scheduleOnce[A](delay: FiniteDuration)(f: => A): Future[A] = {
    val promise = Promise[A]()
    scheduler.schedule(
      new Runnable { def run(): Unit = promise.complete(scala.util.Try(f)) },
      delay.toMillis,
      TimeUnit.MILLISECONDS
    )
    promise.future
  }
  
  // Schedule periodic
  def scheduleAtFixedRate(delay: FiniteDuration, period: FiniteDuration)(f: => Unit): ScheduledFuture[_] = {
    scheduler.scheduleAtFixedRate(
      new Runnable { def run(): Unit = f },
      delay.toMillis,
      period.toMillis,
      TimeUnit.MILLISECONDS
    )
  }
  
  println("=== Custom Scheduler ===")
  
  val delayed = scheduleOnce(200.millis) {
    println(s"  Delayed task on ${Thread.currentThread.getName}")
    "done"
  }
  println(s"Scheduled, result: ${Await.result(delayed, 2.seconds)}")
  
  // Periodic task
  val counter = new AtomicInteger(0)
  val periodic = scheduleAtFixedRate(0.millis, 100.millis) {
    val n = counter.incrementAndGet()
    println(s"  Tick $n")
  }
  
  Thread.sleep(450)
  periodic.cancel(false)
  println(s"Total ticks: ${counter.get()}")
  
  // Work-stealing pool (Scala 2.13+)
  println("\n=== Work-Stealing Pool ===")
  
  val wsPool: ExecutorService = ForkJoinPool.commonPool()
  val wsEC = ExecutionContext.fromExecutorService(wsPool)
  
  // Recursive parallel work (suited for work-stealing)
  def parallelSum(arr: Array[Int], from: Int, to: Int)(implicit ec: ExecutionContext): Future[Long] = {
    if (to - from <= 1000) {
      Future(arr.slice(from, to).map(_.toLong).sum)
    } else {
      val mid = (from + to) / 2
      val leftF  = parallelSum(arr, from, mid)
      val rightF = parallelSum(arr, mid, to)
      for { l <- leftF; r <- rightF } yield l + r
    }
  }
  
  val bigArr = Array.tabulate(100000)(i => i)
  val start = System.currentTimeMillis()
  val sum = Await.result(parallelSum(bigArr, 0, bigArr.length)(wsEC), 10.seconds)
  println(s"Parallel sum: $sum (${System.currentTimeMillis() - start}ms)")
  
  val seqSum = bigArr.map(_.toLong).sum
  println(s"Sequential sum: $seqSum (match: ${sum == seqSum})")
  
  // Thread-local data
  val threadLocal = new ThreadLocal[String]() {
    override def initialValue(): String = "default"
  }
  
  println("\n=== Thread Local ===")
  val futures = (1 to 4).map { i =>
    Future {
      threadLocal.set(s"thread-${Thread.currentThread.getName}")
      Thread.sleep(50)
      s"TL value: ${threadLocal.get()}"
    }(ec)
  }
  
  Await.result(Future.sequence(futures.toList), 5.seconds).foreach(println)
  
  scheduler.shutdown()
}
```

---

## สรุป Part 32

| ExecutionContext | Thread Model | ใช้เมื่อ |
|----------------|-------------|---------|
| `global` (ForkJoin) | Work-stealing | CPU-bound, non-blocking |
| `newFixedThreadPool(n)` | Fixed size | Controlled concurrency |
| `newCachedThreadPool()` | Grows as needed | Many short I/O tasks |
| `newSingleThreadExecutor()` | Single thread | Sequential, ordered |
| `newScheduledThreadPool(n)` | Scheduled | Delayed/periodic tasks |
| `blocking { }` | Compensates | Temporary blocking in ForkJoin |

---

## แบบฝึกหัด Part 32

**ข้อ 1:** Implement thread pool monitor ที่ track queue size, active threads, completed tasks

**ข้อ 2:** สร้าง `PriorityExecutionContext` ที่ runs high-priority tasks first

**ข้อ 3:** Implement async rate limiter ด้วย scheduled executor

**ข้อ 4:** สร้าง `WorkQueue` ที่ batches small tasks เพื่อ reduce overhead

**ข้อ 5:** Implement graceful shutdown ที่ drain queue และ wait for in-flight tasks

---

➡️ ต่อไป: [Part 33 — Async Programming Patterns](part-33-async-programming-patterns.md)
