# Part 71: Apache Spark Introduction — Steps 701-710

## บทนำ: Apache Spark คืออะไร?

Apache Spark เป็น unified analytics engine สำหรับ large-scale data processing ที่เร็วกว่า Hadoop MapReduce ถึง 100 เท่าใน memory และ 10 เท่าบน disk เขียนด้วย Scala และมี API สำหรับ Scala, Java, Python, R

---

## Step 701: ทำความเข้าใจ Apache Spark Architecture

### องค์ประกอบหลักของ Spark

```
┌─────────────────────────────────────────────────────┐
│                   Spark Application                   │
│  ┌─────────────┐                                      │
│  │   Driver    │  (SparkContext / SparkSession)        │
│  │  Program    │                                      │
│  └──────┬──────┘                                      │
│         │  schedules tasks                            │
│  ┌──────▼──────────────────────────────────────┐     │
│  │           Cluster Manager                    │     │
│  │   (Standalone / YARN / Mesos / Kubernetes)   │     │
│  └──────┬─────────────────────────┬────────────┘     │
│         │                         │                   │
│  ┌──────▼──────┐           ┌──────▼──────┐           │
│  │  Worker 1   │           │  Worker 2   │           │
│  │  ┌────────┐ │           │  ┌────────┐ │           │
│  │  │Executor│ │           │  │Executor│ │           │
│  │  │ Tasks  │ │           │  │ Tasks  │ │           │
│  │  └────────┘ │           │  └────────┘ │           │
│  └─────────────┘           └─────────────┘           │
└─────────────────────────────────────────────────────┘
```

### Spark Components
- **Spark Core**: RDD API, scheduling, memory management, fault recovery
- **Spark SQL**: DataFrames, Datasets, SQL queries
- **Spark Streaming**: Real-time data processing
- **MLlib**: Machine learning algorithms
- **GraphX**: Graph computation

---

## Step 702: การติดตั้ง Spark และ build.sbt

### build.sbt สำหรับ Spark 3.5.x

```scala
// build.sbt
ThisBuild / version := "0.1.0-SNAPSHOT"
ThisBuild / scalaVersion := "2.13.12"
ThisBuild / organization := "com.example"

lazy val root = (project in file("."))
  .settings(
    name := "spark-course",
    
    // Spark dependencies
    libraryDependencies ++= Seq(
      "org.apache.spark" %% "spark-core"      % "3.5.1" % "provided",
      "org.apache.spark" %% "spark-sql"        % "3.5.1" % "provided",
      "org.apache.spark" %% "spark-streaming"  % "3.5.1" % "provided",
      "org.apache.spark" %% "spark-mllib"      % "3.5.1" % "provided",
      
      // For testing
      "org.scalatest"    %% "scalatest"         % "3.2.17" % Test,
      
      // Logging
      "org.slf4j"         % "slf4j-api"         % "2.0.9",
      "ch.qos.logback"    % "logback-classic"   % "1.4.11"
    ),
    
    // Spark needs Java serialization settings
    javaOptions ++= Seq(
      "-Xmx4g",
      "-XX:+UseG1GC",
      "--add-opens=java.base/sun.nio.ch=ALL-UNNAMED",
      "--add-opens=java.base/java.nio=ALL-UNNAMED"
    ),
    
    // สำหรับ local testing
    fork := true,
    
    // Assembly plugin สำหรับ fat jar
    assembly / assemblyMergeStrategy := {
      case PathList("META-INF", xs @ _*) => MergeStrategy.discard
      case x => MergeStrategy.first
    }
  )

// plugins.sbt
// addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "2.1.5")
```

### การติดตั้ง Spark แบบ Local

```bash
# macOS
brew install apache-spark

# Ubuntu/Debian
wget https://dlcdn.apache.org/spark/spark-3.5.1/spark-3.5.1-bin-hadoop3.tgz
tar -xzf spark-3.5.1-bin-hadoop3.tgz
export SPARK_HOME=$HOME/spark-3.5.1-bin-hadoop3
export PATH=$PATH:$SPARK_HOME/bin

# ทดสอบ
spark-shell --version
# Welcome to
#       ____              __
#      / __/__  ___ _____/ /__
#     _\ \/ _ \/ _ `/ __/  '_/
#    /___/ .__/\_,_/_/ /_/\_\   version 3.5.1
#       /_/
```

---

## Step 703: SparkContext และ SparkSession

### SparkContext (Spark 1.x API - ยังใช้ได้)

```scala
// SparkContextExample.scala
import org.apache.spark.{SparkConf, SparkContext}

object SparkContextExample {
  def main(args: Array[String]): Unit = {
    // สร้าง SparkConf - กำหนด configuration
    val conf = new SparkConf()
      .setAppName("My First Spark App")  // ชื่อ application
      .setMaster("local[*]")             // local mode ใช้ทุก CPU cores
    
    // สร้าง SparkContext
    val sc = new SparkContext(conf)
    sc.setLogLevel("WARN") // ลด log noise
    
    try {
      // สร้าง RDD จาก collection
      val numbers = sc.parallelize(1 to 100)
      
      // คำนวณผลรวม
      val sum = numbers.reduce(_ + _)
      println(s"Sum of 1 to 100: $sum") // 5050
      
      // คำนวณค่าเฉลี่ย
      val avg = numbers.mean()
      println(s"Average: $avg") // 50.5
      
    } finally {
      sc.stop() // ต้อง stop เสมอ
    }
  }
}
```

### SparkSession (Spark 2.x+ ที่แนะนำ)

```scala
// SparkSessionExample.scala
import org.apache.spark.sql.{SparkSession, DataFrame}
import org.apache.spark.sql.functions._

object SparkSessionExample {
  
  // Helper function สร้าง SparkSession
  def createSparkSession(appName: String, master: String = "local[*]"): SparkSession = {
    SparkSession.builder()
      .appName(appName)
      .master(master)
      // Memory configuration
      .config("spark.driver.memory", "2g")
      .config("spark.executor.memory", "2g")
      .config("spark.executor.cores", "2")
      // Serialization
      .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
      // SQL settings
      .config("spark.sql.shuffle.partitions", "8") // default 200, ลดสำหรับ local
      .config("spark.sql.adaptive.enabled", "true") // Adaptive Query Execution
      .getOrCreate()
  }
  
  def main(args: Array[String]): Unit = {
    val spark = createSparkSession("SparkSession Demo")
    import spark.implicits._ // สำหรับ implicit conversions
    
    // SparkSession รวม SQLContext และ HiveContext
    println(s"Spark version: ${spark.version}")
    println(s"Scala version: ${util.Properties.versionString}")
    
    // สร้าง DataFrame จาก Seq
    val employeeData = Seq(
      (1, "สมชาย", "Engineering", 75000.0),
      (2, "สมหญิง", "Marketing",  65000.0),
      (3, "มานะ",   "Engineering", 85000.0),
      (4, "มานี",   "HR",          55000.0),
      (5, "วิชัย",  "Engineering", 90000.0)
    )
    
    val empDF = employeeData.toDF("id", "name", "department", "salary")
    
    // แสดงข้อมูล
    empDF.show()
    /*
    +---+--------+-----------+-------+
    | id|    name| department| salary|
    +---+--------+-----------+-------+
    |  1|  สมชาย|Engineering|75000.0|
    |  2| สมหญิง|  Marketing|65000.0|
    |  3|    มานะ|Engineering|85000.0|
    |  4|    มานี|         HR|55000.0|
    |  5|  วิชัย|Engineering|90000.0|
    +---+--------+-----------+-------+
    */
    
    // แสดง schema
    empDF.printSchema()
    /*
    root
     |-- id: integer (nullable = false)
     |-- name: string (nullable = true)
     |-- department: string (nullable = true)
     |-- salary: double (nullable = false)
    */
    
    // Query ด้วย SQL
    empDF.createOrReplaceTempView("employees")
    val avgSalaryByDept = spark.sql("""
      SELECT department, AVG(salary) as avg_salary, COUNT(*) as headcount
      FROM employees
      GROUP BY department
      ORDER BY avg_salary DESC
    """)
    
    avgSalaryByDept.show()
    
    spark.stop()
  }
}
```

---

## Step 704: Local Mode vs Cluster Mode

### Local Mode

```scala
// Local Mode - สำหรับ development และ testing
object LocalModeExample {
  def main(args: Array[String]): Unit = {
    
    // local - ใช้ 1 thread
    val spark1 = SparkSession.builder()
      .master("local")
      .appName("Single Thread")
      .getOrCreate()
    
    // local[N] - ใช้ N threads
    val spark2 = SparkSession.builder()
      .master("local[4]")
      .appName("4 Threads")
      .getOrCreate()
    
    // local[*] - ใช้ทุก CPU cores
    val spark3 = SparkSession.builder()
      .master("local[*]")
      .appName("All Cores")
      .getOrCreate()
    
    // local[N, M] - N threads, M max failures
    val spark4 = SparkSession.builder()
      .master("local[4, 3]")
      .appName("4 Threads with Retry")
      .getOrCreate()
    
    println(s"Available cores: ${Runtime.getRuntime.availableProcessors()}")
    spark4.stop()
    spark3.stop()
    spark2.stop()
    spark1.stop()
  }
}
```

### Cluster Mode Configuration

```scala
// ClusterModeConfig.scala
import org.apache.spark.sql.SparkSession

object ClusterModeConfig {
  
  def createProductionSession(): SparkSession = {
    SparkSession.builder()
      .appName("Production App")
      // master จะถูก set จาก spark-submit --master flag
      // ไม่ควร hardcode ใน production code
      
      // Resource configuration
      .config("spark.executor.instances", "10")
      .config("spark.executor.cores", "4")
      .config("spark.executor.memory", "8g")
      .config("spark.driver.memory", "4g")
      .config("spark.driver.maxResultSize", "2g")
      
      // Dynamic allocation
      .config("spark.dynamicAllocation.enabled", "true")
      .config("spark.dynamicAllocation.minExecutors", "2")
      .config("spark.dynamicAllocation.maxExecutors", "20")
      .config("spark.dynamicAllocation.initialExecutors", "5")
      
      // Shuffle service (required for dynamic allocation)
      .config("spark.shuffle.service.enabled", "true")
      
      // Serialization
      .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
      .config("spark.kryo.registrationRequired", "false")
      
      // Compression
      .config("spark.io.compression.codec", "snappy")
      .config("spark.rdd.compress", "true")
      
      // Network
      .config("spark.network.timeout", "120s")
      .config("spark.executor.heartbeatInterval", "10s")
      
      // SQL optimizations
      .config("spark.sql.adaptive.enabled", "true")
      .config("spark.sql.adaptive.coalescePartitions.enabled", "true")
      .config("spark.sql.adaptive.skewJoin.enabled", "true")
      
      .getOrCreate()
  }
  
  def main(args: Array[String]): Unit = {
    val spark = createProductionSession()
    
    // อ่าน config ที่ตั้งไว้
    println("=== Spark Configuration ===")
    spark.conf.getAll.foreach { case (k, v) =>
      if (k.startsWith("spark.executor") || k.startsWith("spark.driver")) {
        println(s"  $k = $v")
      }
    }
    
    spark.stop()
  }
}
```

---

## Step 705: spark-shell และ spark-submit

### spark-shell Commands

```bash
# เริ่ม spark-shell
spark-shell --master local[*] \
            --driver-memory 4g \
            --conf "spark.sql.shuffle.partitions=8"

# ใน spark-shell
scala> val data = 1 to 100
scala> val rdd = sc.parallelize(data)
scala> rdd.count()
res0: Long = 100

scala> rdd.filter(_ % 2 == 0).sum()
res1: Double = 2550.0

# สร้าง DataFrame
scala> val df = spark.range(10).toDF("id")
scala> df.show()

# อ่านไฟล์ CSV
scala> val csvDF = spark.read
         .option("header", "true")
         .option("inferSchema", "true")
         .csv("/path/to/file.csv")
scala> csvDF.printSchema()
scala> csvDF.count()

# ออกจาก spark-shell
scala> :quit
```

### spark-submit

```bash
# Submit local
spark-submit \
  --class com.example.MyApp \
  --master local[*] \
  --driver-memory 2g \
  target/scala-2.13/my-app-assembly-0.1.0.jar

# Submit to YARN cluster
spark-submit \
  --class com.example.MyApp \
  --master yarn \
  --deploy-mode cluster \
  --num-executors 5 \
  --executor-cores 4 \
  --executor-memory 8g \
  --driver-memory 4g \
  --conf "spark.sql.shuffle.partitions=200" \
  --conf "spark.serializer=org.apache.spark.serializer.KryoSerializer" \
  hdfs:///apps/my-app-assembly-0.1.0.jar \
  arg1 arg2

# Submit to Kubernetes
spark-submit \
  --master k8s://https://k8s-api-server:6443 \
  --deploy-mode cluster \
  --name my-spark-app \
  --class com.example.MyApp \
  --conf "spark.executor.instances=5" \
  --conf "spark.kubernetes.container.image=my-spark-image:latest" \
  local:///opt/spark/jars/my-app.jar
```

### SparkSubmit Configuration File

```scala
// สร้าง Application entry point ที่ดี
object ProductionApp {
  
  def main(args: Array[String]): Unit = {
    // Parse arguments
    if (args.length < 2) {
      System.err.println("Usage: ProductionApp <input-path> <output-path>")
      System.exit(1)
    }
    
    val inputPath  = args(0)
    val outputPath = args(1)
    
    val spark = SparkSession.builder()
      .appName("Production ETL Job")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    
    try {
      run(spark, inputPath, outputPath)
    } catch {
      case ex: Exception =>
        System.err.println(s"Job failed: ${ex.getMessage}")
        ex.printStackTrace()
        System.exit(1)
    } finally {
      spark.stop()
    }
  }
  
  def run(spark: SparkSession, inputPath: String, outputPath: String): Unit = {
    import spark.implicits._
    
    // อ่านข้อมูล
    val df = spark.read
      .option("header", "true")
      .option("inferSchema", "true")
      .csv(inputPath)
    
    println(s"Input records: ${df.count()}")
    
    // Process
    val result = df.filter($"value" > 0)
                   .groupBy($"category")
                   .agg(
                     count("*").as("count"),
                     sum("value").as("total")
                   )
    
    // เขียน output
    result.write
      .mode("overwrite")
      .parquet(outputPath)
    
    println(s"Output partitions written to: $outputPath")
  }
}
```

---

## Step 706: Spark Web UI

### การเข้าถึง Spark Web UI

```
Driver UI:     http://localhost:4040
History Server: http://localhost:18080

เมื่อ spark-shell รัน จะเห็น URL ใน console:
INFO SparkUI: Bound SparkUI to 0.0.0.0, and started at http://hostname:4040
```

### ข้อมูลใน Web UI

```
┌─────────────────────────────────────────┐
│            Spark Web UI Tabs            │
├──────────────┬──────────────────────────┤
│ Jobs         │ รายการ jobs ทั้งหมด       │
│ Stages       │ stages ในแต่ละ job        │
│ Storage      │ RDDs/DataFrames ที่ cache  │
│ Environment  │ Spark configs             │
│ Executors    │ executor stats, memory   │
│ SQL          │ SQL queries และ DAG       │
│ Streaming    │ streaming statistics     │
└──────────────┴──────────────────────────┘
```

### Programmatic Metrics

```scala
// MetricsExample.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.storage.StorageLevel

object MetricsExample {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Metrics Demo")
      .getOrCreate()
    
    val sc = spark.sparkContext
    
    // ดู executor info
    val executorInfos = sc.statusTracker.getExecutorInfos
    println(s"Number of executors: ${executorInfos.length}")
    
    // ดู active jobs
    val activeJobs = sc.statusTracker.getActiveJobIds()
    println(s"Active jobs: ${activeJobs.mkString(", ")}")
    
    // สร้าง RDD ที่ cache ไว้
    val rdd = sc.parallelize(1 to 1000000)
                .map(_ * 2)
                .persist(StorageLevel.MEMORY_AND_DISK)
    
    rdd.count() // trigger caching
    
    // ดู cached RDDs
    sc.getRDDStorageInfo.foreach { info =>
      println(s"RDD: ${info.name}, " +
              s"partitions: ${info.numPartitions}, " +
              s"memory: ${info.memSize} bytes")
    }
    
    spark.stop()
  }
}
```

---

## Step 707: Spark Data Lifecycle

### DAG (Directed Acyclic Graph)

```scala
// DAGExample.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._

object DAGExample {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("DAG Demo")
      .config("spark.sql.shuffle.partitions", "4")
      .getOrCreate()
    
    import spark.implicits._
    
    // สร้างข้อมูล
    val salesData = Seq(
      ("2024-01", "Electronics", 50000.0),
      ("2024-01", "Clothing",    30000.0),
      ("2024-02", "Electronics", 60000.0),
      ("2024-02", "Clothing",    35000.0),
      ("2024-03", "Electronics", 55000.0),
      ("2024-03", "Clothing",    40000.0)
    ).toDF("month", "category", "revenue")
    
    // สร้าง transformation chain
    // ทุก transformation เป็น lazy - ไม่ execute จนกว่าจะ action
    val step1 = salesData.filter($"revenue" > 40000)
    val step2 = step1.groupBy($"category")
                     .agg(sum($"revenue").as("total_revenue"))
    val step3 = step2.orderBy($"total_revenue".desc)
    
    // Action - ตอนนี้ DAG ถึงจะ execute
    println("=== DAG Execution ===")
    step3.explain(true) // แสดง Physical/Logical plan
    step3.show()
    
    // ดู query plan
    println("\n=== Query Plan ===")
    step3.explain("formatted")
    
    spark.stop()
  }
}
```

---

## Step 708: การจัดการ Errors และ Fault Tolerance

```scala
// FaultToleranceExample.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._

object FaultToleranceExample {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Fault Tolerance Demo")
      .config("spark.task.maxFailures", "4") // retry tasks 4 ครั้ง
      .getOrCreate()
    
    val sc = spark.sparkContext
    
    // Spark handle node/task failures อัตโนมัติผ่าน lineage
    // ถ้า partition หายไป จะ recompute จาก parent RDD
    
    // สร้าง RDD ที่มี checkpoint
    sc.setCheckpointDir("/tmp/spark-checkpoints")
    
    val data = sc.parallelize(1 to 1000, 10)
    val transformed = data
      .map(_ * 2)
      .filter(_ > 100)
    
    // Checkpoint - บันทึก RDD ลง disk ตัด lineage chain
    transformed.checkpoint()
    
    // Force checkpoint
    transformed.count()
    
    println(s"Checkpoint file: ${transformed.getCheckpointFile}")
    
    // Error handling ใน transformations
    import spark.implicits._
    
    val messyData = Seq("1", "2", "abc", "4", "five", "6").toDF("value")
    
    // วิธีที่ 1: ใช้ try_cast
    val cleaned1 = messyData
      .withColumn("number", col("value").cast("integer"))
      .filter(col("number").isNotNull)
    
    cleaned1.show()
    
    // วิธีที่ 2: ใช้ UDF พร้อม error handling
    val safeParseInt = udf((s: String) => 
      try { Some(s.toInt) }
      catch { case _: NumberFormatException => None }
    )
    
    val cleaned2 = messyData
      .withColumn("number", safeParseInt(col("value")))
      .filter(col("number").isNotNull)
    
    cleaned2.show()
    
    spark.stop()
  }
}
```

---

## Step 709: Spark Configuration Best Practices

```scala
// SparkConfigBestPractices.scala
import org.apache.spark.sql.SparkSession
import com.typesafe.config.ConfigFactory

object SparkConfigBestPractices {
  
  // อ่าน config จาก file
  def loadFromConfig(env: String): SparkSession = {
    val config = ConfigFactory.load(s"spark-$env.conf")
    
    val builder = SparkSession.builder()
      .appName(config.getString("spark.app.name"))
    
    // ตั้ง master ถ้าระบุใน config
    if (config.hasPath("spark.master")) {
      builder.master(config.getString("spark.master"))
    }
    
    // ตั้ง memory
    builder
      .config("spark.driver.memory",   config.getString("spark.driver.memory"))
      .config("spark.executor.memory", config.getString("spark.executor.memory"))
      .config("spark.executor.cores",  config.getString("spark.executor.cores"))
      .getOrCreate()
  }
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Config Best Practices")
      .getOrCreate()
    
    // Runtime config changes
    spark.conf.set("spark.sql.shuffle.partitions", "8")
    
    // อ่าน config
    println(spark.conf.get("spark.sql.shuffle.partitions")) // "8"
    
    // Check if config is modifiable
    try {
      spark.conf.set("spark.app.name", "New Name") // อาจ throw exception
    } catch {
      case ex: Exception => println(s"Cannot change: ${ex.getMessage}")
    }
    
    spark.stop()
  }
}
```

### application.conf สำหรับ Production

```hocon
# src/main/resources/spark-prod.conf
spark {
  app {
    name = "Production ETL"
  }
  driver {
    memory = "8g"
    maxResultSize = "4g"
  }
  executor {
    memory = "16g"
    cores = 4
    instances = 20
  }
  sql {
    shuffle.partitions = 400
    adaptive.enabled = true
    adaptive.coalescePartitions.enabled = true
  }
  serializer = "org.apache.spark.serializer.KryoSerializer"
}

# src/main/resources/spark-dev.conf
spark {
  app.name = "Dev ETL"
  master = "local[*]"
  driver.memory = "2g"
  executor.memory = "2g"
  executor.cores = 2
  sql.shuffle.partitions = 8
}
```

---

## Step 710: First Real Spark Application

```scala
// WordCountApp.scala - Classic Spark example
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._

object WordCountApp {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Word Count")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    
    import spark.implicits._
    
    // สร้างข้อมูล
    val text = Seq(
      "Apache Spark is fast",
      "Spark is a unified analytics engine",
      "Spark SQL provides a programming interface",
      "Spark Streaming enables processing of live data streams",
      "Apache Spark is written in Scala"
    ).toDF("line")
    
    // นับคำ ด้วย DataFrame API
    val wordCount = text
      .select(explode(split(lower($"line"), "\\s+")).as("word"))
      .filter($"word" =!= "")
      .groupBy($"word")
      .agg(count("*").as("count"))
      .orderBy($"count".desc)
    
    println("=== Word Count Results ===")
    wordCount.show(20)
    
    // นับคำ ด้วย RDD API (วิธีเก่า)
    val rddWordCount = spark.sparkContext
      .parallelize(Seq(
        "Apache Spark is fast",
        "Spark is great"
      ))
      .flatMap(_.toLowerCase.split("\\s+"))
      .filter(_.nonEmpty)
      .map((_, 1))
      .reduceByKey(_ + _)
      .sortBy(_._2, ascending = false)
    
    println("\n=== RDD Word Count ===")
    rddWordCount.collect().foreach { case (word, count) =>
      println(s"  $word: $count")
    }
    
    // บันทึก result
    wordCount
      .coalesce(1) // รวมเป็นไฟล์เดียว
      .write
      .mode("overwrite")
      .option("header", "true")
      .csv("/tmp/word-count-result")
    
    println("\nResults saved to /tmp/word-count-result")
    
    spark.stop()
  }
}
```

### E-Commerce Analytics Application

```scala
// ECommerceAnalytics.scala - Real-world example
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._

case class Order(
  orderId: String,
  customerId: String,
  productId: String,
  category: String,
  quantity: Int,
  unitPrice: Double,
  orderDate: String,
  status: String
)

object ECommerceAnalytics {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("E-Commerce Analytics")
      .config("spark.sql.shuffle.partitions", "8")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // สร้างข้อมูลจำลอง
    val orders = Seq(
      Order("ORD001", "C001", "P001", "Electronics", 2, 15000.0, "2024-01-15", "completed"),
      Order("ORD002", "C002", "P002", "Clothing",    3, 500.0,   "2024-01-16", "completed"),
      Order("ORD003", "C001", "P003", "Electronics", 1, 25000.0, "2024-02-01", "completed"),
      Order("ORD004", "C003", "P001", "Electronics", 1, 15000.0, "2024-02-05", "cancelled"),
      Order("ORD005", "C002", "P004", "Food",        10, 100.0,  "2024-02-10", "completed"),
      Order("ORD006", "C004", "P002", "Clothing",    2, 500.0,   "2024-03-01", "completed"),
      Order("ORD007", "C001", "P005", "Electronics", 1, 50000.0, "2024-03-15", "completed"),
      Order("ORD008", "C003", "P004", "Food",        5, 100.0,   "2024-03-20", "completed")
    ).toDS()
    
    // เพิ่ม computed columns
    val enrichedOrders = orders
      .filter($"status" === "completed")
      .withColumn("total_amount", $"quantity" * $"unitPrice")
      .withColumn("order_month", $"orderDate".substr(1, 7))
    
    println("=== Enriched Orders ===")
    enrichedOrders.show()
    
    // Revenue by category
    println("=== Revenue by Category ===")
    enrichedOrders
      .groupBy($"category")
      .agg(
        sum($"total_amount").as("total_revenue"),
        count("*").as("order_count"),
        avg($"total_amount").as("avg_order_value")
      )
      .orderBy($"total_revenue".desc)
      .show()
    
    // Monthly trend
    println("=== Monthly Revenue Trend ===")
    enrichedOrders
      .groupBy($"order_month")
      .agg(sum($"total_amount").as("monthly_revenue"))
      .orderBy($"order_month")
      .show()
    
    // Top customers
    println("=== Top Customers ===")
    enrichedOrders
      .groupBy($"customerId")
      .agg(
        sum($"total_amount").as("total_spend"),
        countDistinct($"orderId").as("order_count")
      )
      .orderBy($"total_spend".desc)
      .show()
    
    // Product performance
    println("=== Product Performance ===")
    enrichedOrders
      .groupBy($"productId", $"category")
      .agg(
        sum($"quantity").as("units_sold"),
        sum($"total_amount").as("revenue")
      )
      .orderBy($"revenue".desc)
      .show()
    
    spark.stop()
  }
}
```

---

## สรุป Part 71

| Concept | Description | ใช้เมื่อ |
|---------|-------------|----------|
| SparkContext | Low-level API, manages RDDs | Legacy code, fine-grained control |
| SparkSession | Unified API, manages DataFrames | Modern Spark development |
| Local Mode | Run on single machine | Development, testing |
| Cluster Mode | Run on distributed cluster | Production |
| DAG | Execution plan visualization | Debugging, optimization |
| Fault Tolerance | Lineage-based recovery | Auto, no manual action needed |
| spark-shell | Interactive REPL | Exploration, quick tests |
| spark-submit | Submit jobs to cluster | Production deployment |

### Key Points
1. **SparkSession** คือ entry point หลักสำหรับ Spark 2.x+
2. ใช้ `local[*]` สำหรับ development, cluster URL สำหรับ production
3. Spark ใช้ **lazy evaluation** - transformations จะ execute เมื่อ action ถูกเรียก
4. **DAG** ช่วยให้ Spark optimize execution plan อัตโนมัติ
5. Fault tolerance ทำงานผ่าน **lineage** - ไม่ต้อง checkpoint ทุก step

---

## แบบฝึกหัด Part 71

1. **Basic Setup**: ติดตั้ง Spark 3.5.x และสร้าง SparkSession แบบ local mode ลองรัน word count บน text file ของคุณ

2. **Configuration**: สร้าง SparkSession ที่อ่าน config จาก `application.conf` ให้รองรับ environment ต่างกัน (dev/staging/prod)

3. **DAG Analysis**: สร้าง transformation chain ที่ซับซ้อน (filter → join → groupBy → sort) แล้วใช้ `.explain("formatted")` ดู execution plan และอธิบายแต่ละ stage

4. **Error Handling**: สร้าง job ที่อ่าน CSV ที่มีข้อมูล invalid แล้วจัดการ error โดยไม่ให้ job fail ทั้งหมด (ใช้ try_cast หรือ UDF)

5. **Real Analytics**: ใช้ ECommerceAnalytics เป็น base เพิ่ม:
   - Customer Lifetime Value (CLV) calculation
   - Year-over-year growth comparison
   - Product recommendation based on co-purchase

---

## ไปต่อ: Part 72 — Spark RDD Deep Dive
ใน Part ถัดไปเราจะเจาะลึก RDD API, transformations, actions, partitioning strategy และ persistence levels

[→ Part 72: Spark RDD](./part-72-spark-rdd.md)
