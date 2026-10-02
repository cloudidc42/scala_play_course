# Part 72: Spark RDD Deep Dive — Steps 711-720

## บทนำ: RDD คืออะไร?

RDD (Resilient Distributed Dataset) เป็น core abstraction ของ Spark ที่เป็น immutable, distributed collection ของ objects ที่สามารถประมวลผลแบบ parallel ได้ "Resilient" หมายถึงสามารถ recover จาก node failure ได้อัตโนมัติผ่าน lineage

---

## Step 711: RDD Fundamentals

### การสร้าง RDD

```scala
// RDDCreation.scala
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.rdd.RDD

object RDDCreation {
  def main(args: Array[String]): Unit = {
    val sc = new SparkContext(new SparkConf()
      .setMaster("local[*]")
      .setAppName("RDD Creation"))
    
    sc.setLogLevel("WARN")
    
    // ===== วิธีที่ 1: parallelize จาก collection =====
    val rdd1: RDD[Int] = sc.parallelize(1 to 10)
    println(s"rdd1 partitions: ${rdd1.getNumPartitions}")  // จำนวน partitions
    
    // กำหนดจำนวน partitions เอง
    val rdd2: RDD[Int] = sc.parallelize(1 to 10, 4)
    println(s"rdd2 partitions: ${rdd2.getNumPartitions}") // 4
    
    // ===== วิธีที่ 2: อ่านจากไฟล์ =====
    // Text file - แต่ละบรรทัดเป็น 1 element
    // val textRDD = sc.textFile("/path/to/file.txt")
    // val textRDD = sc.textFile("/path/to/dir/")  // อ่านทุกไฟล์ใน directory
    // val textRDD = sc.textFile("s3://bucket/path/*.csv")  // S3
    
    // wholeTextFiles - อ่านทั้งไฟล์เป็น (filename, content)
    // val filesRDD = sc.wholeTextFiles("/path/to/dir/")
    
    // ===== วิธีที่ 3: จาก DataFrame/Dataset =====
    // val dfRDD = someDataFrame.rdd
    
    // ===== วิธีที่ 4: สร้าง empty RDD =====
    val emptyRDD: RDD[String] = sc.emptyRDD[String]
    
    // ===== วิธีที่ 5: จาก Sequence ของ RDDs =====
    val rddA = sc.parallelize(1 to 5)
    val rddB = sc.parallelize(6 to 10)
    val unionRDD = rddA.union(rddB)
    println(s"Union: ${unionRDD.collect().mkString(", ")}")
    
    // แสดง properties ของ RDD
    println(s"""
      |RDD Properties:
      |  Partitions: ${rdd1.getNumPartitions}
      |  Storage Level: ${rdd1.getStorageLevel}
      |  Dependencies: ${rdd1.dependencies.map(_.getClass.getSimpleName).mkString(", ")}
    """.stripMargin)
    
    sc.stop()
  }
}
```

---

## Step 712: Transformations — Lazy Operations

```scala
// RDDTransformations.scala
import org.apache.spark.SparkContext
import org.apache.spark.rdd.RDD

object RDDTransformations {
  def main(args: Array[String]): Unit = {
    val sc = new SparkContext("local[*]", "RDD Transformations")
    sc.setLogLevel("WARN")
    
    val numbers = sc.parallelize(List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10))
    
    // ===== Narrow Transformations (ไม่ต้อง shuffle) =====
    
    // map: แปลงทุก element
    val doubled: RDD[Int] = numbers.map(_ * 2)
    
    // flatMap: แปลง element แล้ว flatten
    val words = sc.parallelize(List("Hello World", "Spark is great"))
    val wordList: RDD[String] = words.flatMap(_.split(" "))
    
    // filter: กรอง elements
    val evens: RDD[Int] = numbers.filter(_ % 2 == 0)
    
    // mapPartitions: ทำงานระดับ partition (efficient กว่า map)
    val processedPartitions: RDD[Int] = numbers.mapPartitions { iter =>
      // ทำงานกับ partition ทั้งหมดครั้งเดียว
      // ดีสำหรับ expensive setup (DB connection, etc.)
      iter.map(_ * 3)
    }
    
    // mapPartitionsWithIndex: รู้ partition index ด้วย
    val withIndex: RDD[String] = numbers.mapPartitionsWithIndex { (idx, iter) =>
      iter.map(x => s"partition$idx: $x")
    }
    
    // glom: รวม elements ในแต่ละ partition เป็น Array
    val glommed: RDD[Array[Int]] = numbers.glom()
    
    // union: รวม 2 RDDs
    val moreNumbers = sc.parallelize(11 to 20)
    val allNumbers: RDD[Int] = numbers.union(moreNumbers)
    
    // ===== Wide Transformations (ต้อง shuffle) =====
    
    // groupBy: จัดกลุ่มตาม key
    val grouped: RDD[(Boolean, Iterable[Int])] = numbers.groupBy(_ % 2 == 0)
    
    // sortBy: เรียงลำดับ
    val sortedDesc: RDD[Int] = numbers.sortBy(x => x, ascending = false)
    
    // distinct: ลบ duplicates
    val withDups = sc.parallelize(List(1, 1, 2, 2, 3, 3))
    val unique: RDD[Int] = withDups.distinct()
    
    // repartition: เพิ่ม/ลด partitions (shuffle)
    val repartitioned: RDD[Int] = numbers.repartition(4)
    
    // coalesce: ลด partitions (ไม่ shuffle ถ้า shuffle=false)
    val coalesced: RDD[Int] = repartitioned.coalesce(2)
    
    // sample: สุ่มตัวอย่าง
    val sampled: RDD[Int] = numbers.sample(withReplacement = false, fraction = 0.5)
    
    // ===== Pair RDD Transformations =====
    
    val pairs = sc.parallelize(List(
      ("cat", 1), ("dog", 2), ("cat", 3), ("bird", 1), ("dog", 4)
    ))
    
    // reduceByKey: รวม values ที่มี key เดียวกัน
    val sumByKey: RDD[(String, Int)] = pairs.reduceByKey(_ + _)
    
    // groupByKey: จัดกลุ่ม values ตาม key
    val groupedByKey: RDD[(String, Iterable[Int])] = pairs.groupByKey()
    
    // mapValues: แปลง value โดยไม่เปลี่ยน key
    val doubledValues: RDD[(String, Int)] = pairs.mapValues(_ * 2)
    
    // sortByKey: เรียงตาม key
    val sortedPairs: RDD[(String, Int)] = pairs.sortByKey()
    
    // countByKey: นับจำนวน values ต่อ key
    val countMap: Map[String, Long] = pairs.countByKey()
    
    // join: join 2 pair RDDs ที่มี key เดียวกัน
    val scores = sc.parallelize(List(("Alice", 90), ("Bob", 85), ("Charlie", 78)))
    val grades = sc.parallelize(List(("Alice", "A"), ("Bob", "B")))
    val joined: RDD[(String, (Int, String))] = scores.join(grades)
    
    // leftOuterJoin, rightOuterJoin, fullOuterJoin
    val leftJoined: RDD[(String, (Int, Option[String]))] = scores.leftOuterJoin(grades)
    
    // ===== Actions (Trigger execution) =====
    println("=== Actions ===")
    println(s"count: ${numbers.count()}")
    println(s"first: ${numbers.first()}")
    println(s"collect: ${doubled.collect().mkString(", ")}")
    println(s"take(3): ${numbers.take(3).mkString(", ")}")
    println(s"top(3): ${numbers.top(3).mkString(", ")}")
    println(s"sum: ${numbers.sum()}")
    println(s"mean: ${numbers.mean()}")
    println(s"stdev: ${numbers.stdev()}")
    
    // reduce: รวม elements
    val sum = numbers.reduce(_ + _)
    println(s"reduce sum: $sum")
    
    // fold: เหมือน reduce แต่มี initial value
    val foldSum = numbers.fold(0)(_ + _)
    println(s"fold sum: $foldSum")
    
    // aggregate: flexible reduction
    val (total, count) = numbers.aggregate((0, 0))(
      seqOp  = { case ((sum, cnt), x) => (sum + x, cnt + 1) },
      combOp = { case ((sum1, cnt1), (sum2, cnt2)) => (sum1 + sum2, cnt1 + cnt2) }
    )
    println(s"aggregate: total=$total, count=$count, avg=${total.toDouble/count}")
    
    // foreach: ทำ action ต่อแต่ละ element
    println("Evens:")
    evens.foreach(x => print(s" $x"))
    println()
    
    // saveAsTextFile: บันทึกเป็น text
    // numbers.saveAsTextFile("/tmp/rdd-output")
    
    sc.stop()
  }
}
```

---

## Step 713: Lazy Evaluation คืออะไร?

```scala
// LazyEvaluationDemo.scala
import org.apache.spark.{SparkConf, SparkContext}
import java.time.Instant

object LazyEvaluationDemo {
  def main(args: Array[String]): Unit = {
    val sc = new SparkContext(new SparkConf()
      .setMaster("local[*]")
      .setAppName("Lazy Evaluation"))
    
    sc.setLogLevel("WARN")
    
    println(s"[${now()}] Creating RDD...")
    val data = sc.parallelize(1 to 1000000)
    // ยังไม่ทำอะไรเลย
    
    println(s"[${now()}] Adding transformations...")
    val step1 = data.map { x =>
      // println(s"Processing $x")  // uncomment เพื่อดูว่า ยังไม่รัน
      x * 2
    }
    val step2 = step1.filter(_ > 100000)
    val step3 = step2.map(_ + 1)
    // ยังไม่รัน! แค่สร้าง DAG
    
    println(s"[${now()}] Transformation chain created (no execution yet)")
    
    // ตอนนี้ถึงจะ execute
    println(s"[${now()}] Triggering action...")
    val count = step3.count()
    println(s"[${now()}] Count: $count")
    
    // Action ครั้งที่ 2 - จะ re-execute transformation chain!
    println(s"[${now()}] Second action...")
    val sum = step3.sum()
    println(s"[${now()}] Sum: $sum")
    
    // ป้องกัน re-execution ด้วย cache/persist
    println(s"[${now()}] Caching...")
    step3.cache() // persist in memory
    
    // Action แรกหลัง cache - ยังต้อง compute
    val count2 = step3.count()
    println(s"[${now()}] Count (cached): $count2")
    
    // Action ครั้งต่อไป - อ่านจาก cache ไม่ต้อง re-compute
    val sum2 = step3.sum()
    println(s"[${now()}] Sum (from cache): $sum2")
    
    // ดู lineage
    println("\n=== RDD Lineage ===")
    println(step3.toDebugString)
    
    sc.stop()
  }
  
  def now() = Instant.now().toString
}
```

---

## Step 714: Partitioning Strategy

```scala
// PartitioningDemo.scala
import org.apache.spark.{SparkConf, SparkContext, HashPartitioner, RangePartitioner}
import org.apache.spark.rdd.RDD

object PartitioningDemo {
  def main(args: Array[String]): Unit = {
    val sc = new SparkContext(new SparkConf()
      .setMaster("local[4]")
      .setAppName("Partitioning"))
    
    sc.setLogLevel("WARN")
    
    // ===== ตรวจสอบ partitions =====
    val data = sc.parallelize(1 to 20, 4)
    
    // ดู distribution ของ data ในแต่ละ partition
    val distribution = data.mapPartitionsWithIndex { (idx, iter) =>
      val elements = iter.toList
      Iterator((idx, elements.length, elements))
    }.collect()
    
    distribution.foreach { case (idx, count, elements) =>
      println(s"Partition $idx: $count elements = [${elements.mkString(", ")}]")
    }
    
    // ===== HashPartitioner =====
    val pairs = sc.parallelize(List(
      ("alice", 100), ("bob", 200), ("charlie", 150),
      ("alice", 50),  ("bob", 300), ("dave", 75)
    ))
    
    // HashPartitioner: hash(key) % numPartitions
    val hashPartitioned = pairs.partitionBy(new HashPartitioner(3))
    
    println("\n=== Hash Partitioned ===")
    hashPartitioned.mapPartitionsWithIndex { (idx, iter) =>
      val pairs = iter.toList
      Iterator(s"Partition $idx: ${pairs.map(_._1).mkString(", ")}")
    }.collect().foreach(println)
    
    // ===== RangePartitioner =====
    val sortedPairs = pairs.sortByKey() // requires RangePartitioner
    
    // สร้าง RangePartitioner เอง
    val rangePartitioner = new RangePartitioner(3, pairs)
    val rangePartitioned = pairs.partitionBy(rangePartitioner)
    
    println("\n=== Range Partitioned ===")
    rangePartitioned.mapPartitionsWithIndex { (idx, iter) =>
      val ps = iter.toList
      Iterator(s"Partition $idx: ${ps.map(_._1).mkString(", ")}")
    }.collect().foreach(println)
    
    // ===== Custom Partitioner =====
    class CategoryPartitioner(numParts: Int) extends org.apache.spark.Partitioner {
      override def numPartitions: Int = numParts
      
      override def getPartition(key: Any): Int = key match {
        case "alice"   => 0
        case "bob"     => 1
        case "charlie" => 2
        case _         => 0
      }
    }
    
    val customPartitioned = pairs.partitionBy(new CategoryPartitioner(3))
    
    println("\n=== Custom Partitioned ===")
    customPartitioned.mapPartitionsWithIndex { (idx, iter) =>
      val ps = iter.toList
      Iterator(s"Partition $idx: ${ps.map(_._1).mkString(", ")}")
    }.collect().foreach(println)
    
    // ===== Partition count recommendations =====
    println(s"""
      |Partitioning Guidelines:
      |  - Default: 2-4 per CPU core
      |  - Large datasets: enough partitions to fill executors
      |  - Too few: underutilization of cluster
      |  - Too many: overhead of task scheduling
      |  - Rule of thumb: (cluster cores * 2) to (cluster cores * 4)
      |
      |  Current cores: ${Runtime.getRuntime.availableProcessors()}
      |  Recommended partitions: ${Runtime.getRuntime.availableProcessors() * 2} to ${Runtime.getRuntime.availableProcessors() * 4}
    """.stripMargin)
    
    sc.stop()
  }
}
```

---

## Step 715: Persistence และ Caching

```scala
// PersistenceDemo.scala
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.storage.StorageLevel

object PersistenceDemo {
  def main(args: Array[String]): Unit = {
    val sc = new SparkContext(new SparkConf()
      .setMaster("local[*]")
      .setAppName("Persistence"))
    
    sc.setLogLevel("WARN")
    
    val data = sc.parallelize(1 to 1000000)
    
    // ===== Storage Levels =====
    
    // MEMORY_ONLY (default cache()): เก็บใน RAM เป็น Java objects
    // ถ้า RAM เต็ม บาง partitions จะถูก drop และ recomputed เมื่อต้องการ
    val rdd1 = data.map(_ * 2).persist(StorageLevel.MEMORY_ONLY)
    
    // MEMORY_AND_DISK: เก็บใน RAM, spill ไป disk ถ้าเต็ม
    val rdd2 = data.map(_ * 3).persist(StorageLevel.MEMORY_AND_DISK)
    
    // MEMORY_ONLY_SER: เก็บใน RAM แบบ serialized (ประหยัด memory กว่าแต่ CPU มากกว่า)
    val rdd3 = data.map(_ * 4).persist(StorageLevel.MEMORY_ONLY_SER)
    
    // MEMORY_AND_DISK_SER: เก็บใน RAM แบบ serialized, spill ไป disk
    val rdd4 = data.map(_ * 5).persist(StorageLevel.MEMORY_AND_DISK_SER)
    
    // DISK_ONLY: เก็บบน disk เท่านั้น
    val rdd5 = data.map(_ * 6).persist(StorageLevel.DISK_ONLY)
    
    // OFF_HEAP: เก็บนอก JVM heap (ต้องตั้ง spark.memory.offHeap.enabled = true)
    // val rdd6 = data.persist(StorageLevel.OFF_HEAP)
    
    // Replicated versions (2 copies across nodes)
    val rdd7 = data.persist(StorageLevel.MEMORY_AND_DISK_2)
    
    // ===== Timing comparison =====
    def time[T](label: String)(f: => T): T = {
      val start = System.currentTimeMillis()
      val result = f
      val elapsed = System.currentTimeMillis() - start
      println(s"$label: ${elapsed}ms")
      result
    }
    
    // First count - must compute
    time("First count (no cache)") { data.map(_ * 2).count() }
    
    // Cache RDD
    val cachedData = data.map(_ * 2).cache()
    time("First count (trigger caching)") { cachedData.count() }
    
    // Second count - reads from cache
    time("Second count (from cache)") { cachedData.count() }
    time("Third count (from cache)")  { cachedData.count() }
    
    // ===== เมื่อไหร่ควร cache? =====
    /*
    ควร cache เมื่อ:
    1. RDD ถูกใช้หลายครั้ง (multiple actions)
    2. Computation ใช้เวลานาน (complex transformations)
    3. Dataset พอดีใน memory
    
    ไม่ควร cache เมื่อ:
    1. RDD ใช้แค่ครั้งเดียว
    2. Dataset ใหญ่กว่า available memory มาก
    3. RDD recompute ได้เร็ว
    */
    
    // ===== Storage Level Comparison =====
    println("""
      Storage Level Comparison:
      ┌──────────────────────┬────────┬──────┬──────────┬─────────────┐
      │ Storage Level        │ Space  │ CPU  │ In Memory│ On Disk     │
      ├──────────────────────┼────────┼──────┼──────────┼─────────────┤
      │ MEMORY_ONLY          │ High   │ Low  │ Deserialized│ No       │
      │ MEMORY_ONLY_SER      │ Low    │ High │ Serialized  │ No       │
      │ MEMORY_AND_DISK      │ High   │ Med  │ Deserialized│ Yes (ser)│
      │ MEMORY_AND_DISK_SER  │ Low    │ High │ Serialized  │ Yes (ser)│
      │ DISK_ONLY            │ Low    │ High │ No          │ Yes (ser)│
      └──────────────────────┴────────┴──────┴──────────┴─────────────┘
    """)
    
    // unpersist เมื่อไม่ใช้แล้ว
    cachedData.unpersist()
    
    // ดู cached RDDs
    sc.getPersistentRDDs.foreach { case (id, rdd) =>
      println(s"Persistent RDD: id=$id, name=${rdd.name}, level=${rdd.getStorageLevel}")
    }
    
    sc.stop()
  }
}
```

---

## Step 716: Pair RDDs และ Key-Value Operations

```scala
// PairRDDDemo.scala
import org.apache.spark.{SparkConf, SparkContext}

object PairRDDDemo {
  def main(args: Array[String]): Unit = {
    val sc = new SparkContext(new SparkConf()
      .setMaster("local[*]")
      .setAppName("Pair RDD Demo"))
    
    sc.setLogLevel("WARN")
    
    // Sales data: (region, amount)
    val sales = sc.parallelize(List(
      ("North", 1000.0), ("South", 1500.0), ("East",  800.0),
      ("North", 2000.0), ("South", 1200.0), ("West",  900.0),
      ("East",  1100.0), ("North", 1800.0), ("West",  1300.0),
      ("South", 900.0),  ("East",  1400.0), ("West",  1700.0)
    ))
    
    // ===== Basic Pair Operations =====
    
    // keys และ values
    val regions = sales.keys.distinct().collect()
    println(s"Regions: ${regions.mkString(", ")}")
    
    // reduceByKey: รวม values ด้วย function (efficient, ไม่ shuffle ทั้งหมด)
    val totalByRegion = sales.reduceByKey(_ + _)
    println("\n=== Total by Region ===")
    totalByRegion.sortByKey().collect().foreach { case (region, total) =>
      println(f"  $region%-10s: $total%.2f")
    }
    
    // groupByKey: รวม values เป็น Iterable (ไม่ efficient เท่า reduceByKey)
    // ใช้เมื่อต้องการ list ของ values ไม่ใช่แค่ aggregate
    val salesByRegion = sales.groupByKey()
    println("\n=== Sales List by Region ===")
    salesByRegion.collect().foreach { case (region, amounts) =>
      println(s"  $region: ${amounts.toList.map(f"%.0f".format(_)).mkString(", ")}")
    }
    
    // aggregateByKey: flexible aggregation
    val (totalSales, countSales) = (0.0, 0)
    val statsPerRegion = sales.aggregateByKey((0.0, 0))(
      seqOp  = { case ((sum, cnt), amount) => (sum + amount, cnt + 1) },
      combOp = { case ((sum1, c1), (sum2, c2)) => (sum1 + sum2, c1 + c2) }
    )
    
    println("\n=== Stats by Region ===")
    statsPerRegion.mapValues { case (sum, cnt) =>
      (sum, cnt, sum / cnt)
    }.sortByKey().collect().foreach { case (region, (total, count, avg)) =>
      println(f"  $region%-10s: total=$total%.0f, count=$count, avg=$avg%.0f")
    }
    
    // combineByKey: most flexible (other operations built on this)
    type Stats = (Double, Int, Double, Double) // sum, count, min, max
    val comprehensiveStats = sales.combineByKey(
      createCombiner = (amount: Double) => (amount, 1, amount, amount),
      mergeValue     = (stats: Stats, amount: Double) => {
        val (sum, cnt, min, max) = stats
        (sum + amount, cnt + 1, math.min(min, amount), math.max(max, amount))
      },
      mergeCombiners = (s1: Stats, s2: Stats) => {
        val (sum1, cnt1, min1, max1) = s1
        val (sum2, cnt2, min2, max2) = s2
        (sum1 + sum2, cnt1 + cnt2, math.min(min1, min2), math.max(max1, max2))
      }
    )
    
    println("\n=== Comprehensive Stats ===")
    comprehensiveStats.sortByKey().collect().foreach { case (region, (sum, cnt, min, max)) =>
      println(f"  $region%-10s: sum=$sum%.0f, cnt=$cnt, min=$min%.0f, max=$max%.0f, avg=${sum/cnt}%.0f")
    }
    
    // ===== Joins =====
    val budgets = sc.parallelize(List(
      ("North", 15000.0),
      ("South", 20000.0),
      ("East",  12000.0)
    ))
    
    // inner join
    val innerJoin = totalByRegion.join(budgets)
    println("\n=== Inner Join (Sales vs Budget) ===")
    innerJoin.collect().foreach { case (region, (sales, budget)) =>
      val pct = sales / budget * 100
      println(f"  $region%-10s: sales=$sales%.0f, budget=$budget%.0f, achievement=$pct%.1f%%")
    }
    
    // left outer join - ทุก region ใน totalByRegion
    val leftJoin = totalByRegion.leftOuterJoin(budgets)
    println("\n=== Left Outer Join ===")
    leftJoin.sortByKey().collect().foreach { case (region, (sales, budgetOpt)) =>
      budgetOpt match {
        case Some(budget) => println(f"  $region%-10s: sales=$sales%.0f, budget=$budget%.0f")
        case None         => println(f"  $region%-10s: sales=$sales%.0f, NO BUDGET")
      }
    }
    
    sc.stop()
  }
}
```

---

## Step 717: Shuffle Operations และ Performance

```scala
// ShuffleDemo.scala
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.rdd.RDD

object ShuffleDemo {
  def main(args: Array[String]): Unit = {
    val sc = new SparkContext(new SparkConf()
      .setMaster("local[*]")
      .setAppName("Shuffle Demo")
      .set("spark.shuffle.compress", "true")
      .set("spark.shuffle.spill.compress", "true"))
    
    sc.setLogLevel("WARN")
    
    // ===== Operations ที่ต้อง Shuffle =====
    /*
    Wide Transformations (trigger shuffle):
    - groupByKey
    - reduceByKey
    - sortByKey
    - join / cogroup
    - repartition
    - distinct
    - intersection
    - subtract
    */
    
    val data = sc.parallelize((1 to 100000).map(i => (i % 100, i)))
    
    // Shuffle เกิดขึ้นเพราะต้อง bring data with same key to same partition
    def timeOp[T](label: String)(f: => T): T = {
      val start = System.currentTimeMillis()
      val result = f
      println(s"$label: ${System.currentTimeMillis() - start}ms")
      result
    }
    
    // groupByKey vs reduceByKey
    // groupByKey: shuffle ทั้งหมดก่อน แล้วค่อย group
    timeOp("groupByKey + sum") {
      data.groupByKey().mapValues(_.sum).count()
    }
    
    // reduceByKey: combine locally ก่อน แล้ว shuffle (เร็วกว่า)
    timeOp("reduceByKey") {
      data.reduceByKey(_ + _).count()
    }
    
    // ===== Shuffle Partitions =====
    println(s"Default shuffle partitions: ${data.reduceByKey(_ + _).getNumPartitions}")
    
    // ===== Avoiding Unnecessary Shuffles =====
    
    // BAD: filter หลัง groupBy ทำ shuffle ก่อน filter
    val badApproach = data.groupByKey().filter { case (k, _) => k < 50 }
    
    // GOOD: filter ก่อน groupBy ลด data ที่ต้อง shuffle
    val goodApproach = data.filter { case (k, _) => k < 50 }.groupByKey()
    
    timeOp("Bad approach (shuffle then filter)") { badApproach.count() }
    timeOp("Good approach (filter then shuffle)") { goodApproach.count() }
    
    // ===== Broadcast Join =====
    // เมื่อ join กับ small dataset ให้ broadcast แทน shuffle join
    val smallLookup = Map(
      1 -> "One", 2 -> "Two", 3 -> "Three"
    )
    
    // Broadcast variable - ส่งไปยัง executors ครั้งเดียว
    val broadcastLookup = sc.broadcast(smallLookup)
    
    val largeData = sc.parallelize(1 to 1000000)
    val enriched = largeData.flatMap { i =>
      broadcastLookup.value.get(i % 4) match {
        case Some(name) => Some((i, name))
        case None       => None
      }
    }
    
    println(s"Enriched count: ${enriched.count()}")
    
    // cleanup broadcast
    broadcastLookup.unpersist()
    broadcastLookup.destroy()
    
    sc.stop()
  }
}
```

---

## Step 718: Advanced RDD Operations

```scala
// AdvancedRDDOps.scala
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.rdd.RDD

object AdvancedRDDOps {
  def main(args: Array[String]): Unit = {
    val sc = new SparkContext(new SparkConf()
      .setMaster("local[*]")
      .setAppName("Advanced RDD"))
    
    sc.setLogLevel("WARN")
    
    // ===== zip =====
    val keys   = sc.parallelize(List("a", "b", "c", "d"))
    val values = sc.parallelize(List(1, 2, 3, 4))
    val zipped = keys.zip(values)
    println(s"Zipped: ${zipped.collect().mkString(", ")}")
    
    // ===== zipWithIndex =====
    val items = sc.parallelize(List("apple", "banana", "cherry"))
    val indexed = items.zipWithIndex()
    indexed.collect().foreach { case (item, idx) =>
      println(s"  $idx: $item")
    }
    
    // ===== zipWithUniqueId =====
    val uniqueIds = items.zipWithUniqueId()
    uniqueIds.collect().foreach { case (item, id) =>
      println(s"  id=$id: $item")
    }
    
    // ===== cartesian (cross join) =====
    val colors = sc.parallelize(List("red", "blue"))
    val sizes  = sc.parallelize(List("S", "M", "L"))
    val combinations = colors.cartesian(sizes)
    println(s"\nCombinations: ${combinations.collect().mkString(", ")}")
    
    // ===== cogroup =====
    val purchases = sc.parallelize(List(
      ("Alice", "Book"), ("Bob", "Pen"), ("Alice", "Laptop"), ("Charlie", "Phone")
    ))
    val addresses = sc.parallelize(List(
      ("Alice", "Bangkok"), ("Bob", "Chiang Mai")
    ))
    
    val cogrouped = purchases.cogroup(addresses)
    println("\n=== CoGrouped ===")
    cogrouped.collect().foreach { case (person, (items, addrs)) =>
      println(s"  $person: items=${items.toList}, address=${addrs.toList}")
    }
    
    // ===== intersection และ subtract =====
    val set1 = sc.parallelize(List(1, 2, 3, 4, 5))
    val set2 = sc.parallelize(List(3, 4, 5, 6, 7))
    
    println(s"\nIntersection: ${set1.intersection(set2).collect().sorted.mkString(", ")}")
    println(s"Subtract (set1 - set2): ${set1.subtract(set2).collect().sorted.mkString(", ")}")
    
    // ===== treeReduce and treeAggregate (better for deep recursion) =====
    val largeRDD = sc.parallelize(1 to 1000000)
    
    // treeReduce: logarithmic depth reduction (avoid StackOverflow)
    val sum = largeRDD.treeReduce(_ + _, depth = 4)
    println(s"\ntreeReduce sum: $sum")
    
    // ===== Numeric Operations =====
    val nums = sc.parallelize(List(1.0, 2.0, 3.0, 4.0, 5.0))
    val stats = nums.stats()
    println(s"""
      |Statistics:
      |  count: ${stats.count}
      |  mean:  ${stats.mean}
      |  stdev: ${stats.stdev}
      |  min:   ${stats.min}
      |  max:   ${stats.max}
    """.stripMargin)
    
    sc.stop()
  }
}
```

---

## Step 719: RDD Lineage และ Debugging

```scala
// LineageDebug.scala
import org.apache.spark.{SparkConf, SparkContext}

object LineageDebug {
  def main(args: Array[String]): Unit = {
    val sc = new SparkContext(new SparkConf()
      .setMaster("local[*]")
      .setAppName("Lineage Debug"))
    
    sc.setLogLevel("WARN")
    
    val rawData = sc.parallelize(
      List("2024-01-15,Electronics,iPhone,999.99",
           "2024-01-16,Clothing,T-Shirt,29.99",
           "2024-01-17,Electronics,Laptop,1299.99",
           "bad line here",
           "2024-01-18,Food,Coffee,5.99")
    ).setName("rawSalesData")
    
    // Parse line
    case class SaleRecord(date: String, category: String, product: String, price: Double)
    
    val parsedData = rawData
      .flatMap { line =>
        line.split(",") match {
          case Array(date, cat, prod, price) =>
            try Some(SaleRecord(date, cat, prod, price.toDouble))
            catch { case _: Exception => None }
          case _ => None
        }
      }
      .setName("parsedSalesData")
    
    val electronicsOnly = parsedData
      .filter(_.category == "Electronics")
      .setName("electronics")
    
    val expensiveItems = electronicsOnly
      .filter(_.price > 500)
      .setName("expensiveElectronics")
    
    // ===== Debug Lineage =====
    println("=== RDD Lineage (toDebugString) ===")
    println(expensiveItems.toDebugString)
    /*
    (4) expensiveElectronics MapPartitionsRDD[4] at setName at ...
     |  electronics MapPartitionsRDD[3] at setName at ...
     |  parsedSalesData MapPartitionsRDD[2] at setName at ...
     |  rawSalesData ParallelCollectionRDD[0] at parallelize at ...
    */
    
    // ===== Dependencies =====
    println("\n=== Dependencies ===")
    expensiveItems.dependencies.foreach { dep =>
      println(s"  Dependency: ${dep.getClass.getSimpleName} -> ${dep.rdd.name}")
    }
    
    // ===== Checkpoint for long lineage =====
    sc.setCheckpointDir("/tmp/spark-checkpoints")
    
    // Long transformation chain - risk of StackOverflow
    var longChain = sc.parallelize(1 to 1000)
    for (i <- 1 to 50) {
      longChain = longChain.map(_ + i)
      if (i % 10 == 0) {
        longChain.checkpoint() // ตัด lineage ทุก 10 steps
        longChain.count()      // materialize checkpoint
      }
    }
    
    println(s"\nLong chain result: ${longChain.sum()}")
    
    sc.stop()
  }
}
```

---

## Step 720: RDD Best Practices

```scala
// RDDBestPractices.scala
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.serializer.KryoSerializer

// สำหรับ Kryo serialization ต้อง register classes
class MyKryoRegistrator extends org.apache.spark.serializer.KryoRegistrator {
  override def registerClasses(kryo: com.esotericsoftware.kryo.Kryo): Unit = {
    kryo.register(classOf[Array[Int]])
    kryo.register(classOf[Array[String]])
    kryo.register(classOf[SaleItem])
  }
}

case class SaleItem(id: Int, name: String, price: Double, quantity: Int)

object RDDBestPractices {
  def main(args: Array[String]): Unit = {
    val sc = new SparkContext(new SparkConf()
      .setMaster("local[*]")
      .setAppName("RDD Best Practices")
      // ใช้ Kryo serialization แทน Java serialization (เร็วกว่า ~10x)
      .set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
      .set("spark.kryo.registrator", "MyKryoRegistrator")
      .set("spark.kryo.registrationRequired", "false")
    )
    
    sc.setLogLevel("WARN")
    
    // ===== Best Practice 1: ใช้ reduceByKey แทน groupByKey =====
    val sales = sc.parallelize((1 to 100000).map(i => (i % 10, i.toDouble)))
    
    // BAD: groupByKey shuffle ทั้งหมด
    // val badResult = sales.groupByKey().mapValues(_.sum)
    
    // GOOD: reduceByKey combine local first
    val goodResult = sales.reduceByKey(_ + _)
    
    // ===== Best Practice 2: filter ก่อน join =====
    val customers = sc.parallelize((1 to 10000).map(i => (i, s"Customer$i")))
    val orders    = sc.parallelize((1 to 100000).map(i => (i % 10000, i * 10.0)))
    
    // BAD: join แล้ว filter
    // val bad = customers.join(orders).filter(_._1 < 100)
    
    // GOOD: filter ก่อน join
    val smallCustomers = customers.filter(_._1 < 100)
    val relevantOrders = orders.filter(_._1 < 100)
    val joined = smallCustomers.join(relevantOrders)
    
    // ===== Best Practice 3: ใช้ mapPartitions สำหรับ expensive operations =====
    import java.sql.Connection
    
    // BAD: เปิด connection ทุก element
    // val bad = rdd.map { item =>
    //   val conn = openConnection()  // expensive!
    //   val result = query(conn, item)
    //   conn.close()
    //   result
    // }
    
    // GOOD: เปิด connection ครั้งเดียวต่อ partition
    val items = sc.parallelize(1 to 1000)
    val processed = items.mapPartitions { iter =>
      // Setup เกิดขึ้นครั้งเดียวต่อ partition
      // val conn = openConnection()
      val results = iter.map { item =>
        // ใช้ conn
        item * 2
      }
      // conn.close() - ควรใช้ try-finally
      results
    }
    
    // ===== Best Practice 4: ใช้ broadcast สำหรับ small lookup tables =====
    val lookup = Map(1 -> "A", 2 -> "B", 3 -> "C")
    val broadcastLookup = sc.broadcast(lookup)
    
    val enriched = items.map { i =>
      val key = i % 4
      val label = broadcastLookup.value.getOrElse(key, "Unknown")
      (i, label)
    }
    
    println(s"Enriched sample: ${enriched.take(5).mkString(", ")}")
    broadcastLookup.unpersist()
    
    // ===== Best Practice 5: กำหนด schema ล่วงหน้า =====
    // อย่าใช้ inferSchema ใน production (ช้าและอาจ infer ผิด)
    // ใช้ DataFrame/Dataset API แทน RDD เมื่อเป็นไปได้
    
    // ===== Best Practice 6: ระวัง closure serialization =====
    
    // BAD: serialize ทั้ง object
    class Config {
      val multiplier = 2
      val prefix = "item"
      // อาจมี connection, file handles ที่ serialize ไม่ได้
    }
    
    // val config = new Config()
    // val bad = items.map(i => config.multiplier * i) // serialize ทั้ง config object
    
    // GOOD: extract ค่าที่ต้องการก่อน
    val multiplier = 2 // serialize เฉพาะ primitive value
    val good = items.map(i => multiplier * i)
    
    println(s"Good: ${good.take(5).mkString(", ")}")
    
    sc.stop()
  }
}
```

---

## สรุป Part 72

| Operation | Type | Shuffle? | Use Case |
|-----------|------|----------|----------|
| map | Narrow | No | Element transformation |
| filter | Narrow | No | Element selection |
| flatMap | Narrow | No | One-to-many mapping |
| mapPartitions | Narrow | No | Partition-level ops |
| reduceByKey | Wide | Yes | Aggregation with combine |
| groupByKey | Wide | Yes | Full grouping |
| join | Wide | Yes | Combining datasets |
| sortByKey | Wide | Yes | Sorting |
| repartition | Wide | Yes | Change partition count |
| coalesce | Narrow (mostly) | No | Reduce partitions |

### Storage Level Selection Guide

| Storage Level | When to Use |
|---------------|-------------|
| MEMORY_ONLY | Small datasets, fast recompute |
| MEMORY_AND_DISK | Medium datasets, slow recompute |
| MEMORY_ONLY_SER | Large datasets, memory pressure |
| DISK_ONLY | Very large, not reused often |

### Key Takeaways
1. ใช้ **lazy evaluation** ให้ประโยชน์ - filter ก่อน expensive operations
2. **reduceByKey** เร็วกว่า groupByKey เสมอ
3. **cache()** เฉพาะ RDD ที่ใช้หลายครั้ง
4. **broadcast** สำหรับ small lookup tables
5. **mapPartitions** สำหรับ expensive per-element setup

---

## แบบฝึกหัด Part 72

1. **Partition Analysis**: สร้าง RDD จากไฟล์ใหญ่ แล้ว analyze distribution ของ data ในแต่ละ partition ลองปรับจำนวน partitions และ compare performance

2. **Caching Benchmark**: สร้าง complex transformation chain แล้ว measure เวลาที่ใช้ run action ครั้งแรก vs ครั้งที่สองหลัง cache

3. **Custom Partitioner**: สร้าง custom partitioner ที่แบ่ง data ตาม first letter ของ string key แล้ว verify ว่า data distribute ถูกต้อง

4. **Optimization Challenge**: มี code ที่ใช้ groupByKey แล้ว map แปลงเป็น aggregate ให้ refactor ใช้ reduceByKey หรือ aggregateByKey แทน และ measure performance improvement

5. **Broadcast Join**: สร้าง large dataset และ small lookup table แล้ว implement broadcast join เปรียบเทียบกับ regular join ดู DAG ว่าต่างกันอย่างไร

---

## ไปต่อ: Part 73 — Spark DataFrames
ใน Part ถัดไปจะเรียน DataFrame API ที่ powerful กว่า RDD, schema management, และ Dataset[T]

[→ Part 73: Spark DataFrames](./part-73-spark-dataframes.md)
