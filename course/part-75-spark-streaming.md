# Part 75: Spark Structured Streaming — Steps 741-750

## บทนำ: Structured Streaming

Spark Structured Streaming คือ stream processing engine ที่ built บน Spark SQL engine มอง stream เป็น "unbounded table" ที่ grow ต่อเนื่อง ทำให้เขียน code แบบ batch แต่รันแบบ streaming ได้

---

## Step 741: Structured Streaming Concepts

```scala
// StreamingConcepts.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.functions._

object StreamingConcepts {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Streaming Concepts")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    
    /*
    Structured Streaming Model:
    
    Input Stream (unbounded table):
    +----+-----+----------+
    |key |value|timestamp |
    +----+-----+----------+
    |a   |1    |12:00:01  | ← new rows appended
    |b   |2    |12:00:02  |
    |a   |3    |12:00:03  |
    +----+-----+----------+
    
    Query runs continuously on new data
    
    Output Modes:
    - Complete: rewrite entire result table each trigger
    - Append: only add new rows to output
    - Update: only update changed rows
    */
    
    // ===== Rate Source - สร้าง data สำหรับ testing =====
    val rateStream = spark.readStream
      .format("rate")
      .option("rowsPerSecond", 10)  // 10 rows/second
      .option("numPartitions", 2)
      .load()
    
    // rateStream schema: timestamp TIMESTAMP, value BIGINT
    rateStream.printSchema()
    
    // Transform
    val processed = rateStream
      .withColumn("category", ($"value" % 3).cast("string"))
      .withColumn("amount", ($"value" * 10.0) + rand() * 100)
    
    // ===== Output Modes =====
    
    // Complete mode: เหมาะกับ aggregations
    val completeModeQuery = processed
      .groupBy($"category")
      .agg(count("*").as("count"), sum("amount").as("total"))
      .writeStream
      .outputMode(OutputMode.Complete())
      .format("console")
      .option("truncate", "false")
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    // Append mode: เหมาะกับ no aggregation หรือ windowed aggregation
    // val appendQuery = rateStream
    //   .writeStream
    //   .outputMode(OutputMode.Append())
    //   .format("console")
    //   .start()
    
    // Update mode: เหมาะกับ aggregations ที่ต้องการ update
    // val updateQuery = processed
    //   .groupBy($"category")
    //   .agg(count("*"))
    //   .writeStream
    //   .outputMode(OutputMode.Update())
    //   .format("console")
    //   .start()
    
    // รอ 15 วินาที แล้วหยุด
    completeModeQuery.awaitTermination(15000)
    completeModeQuery.stop()
    
    spark.stop()
  }
}
```

---

## Step 742: readStream จากหลาย Sources

```scala
// StreamingSources.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.types._
import org.apache.spark.sql.functions._
import java.io.{File, PrintWriter}
import java.util.concurrent.{Executors, TimeUnit}

object StreamingSources {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Streaming Sources")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== Source 1: File Stream =====
    // สร้าง directory สำหรับ incoming files
    val inputDir = "/tmp/streaming-input"
    new File(inputDir).mkdirs()
    
    // Schema สำหรับ incoming JSON files
    val orderSchema = StructType(Array(
      StructField("order_id",    StringType,  nullable = true),
      StructField("customer_id", StringType,  nullable = true),
      StructField("amount",      DoubleType,  nullable = true),
      StructField("category",    StringType,  nullable = true),
      StructField("timestamp",   LongType,    nullable = true)
    ))
    
    // อ่าน JSON files แบบ streaming
    val fileStream = spark.readStream
      .format("json")
      .schema(orderSchema)  // ต้องระบุ schema
      .option("maxFilesPerTrigger", 1)  // อ่านไม่เกิน 1 ไฟล์ต่อ trigger
      .load(inputDir)
    
    // Process file stream
    val fileQuery = fileStream
      .withColumn("order_time", to_timestamp($"timestamp".cast("string"), "yyyyMMddHHmmss"))
      .groupBy(
        window($"order_time", "10 seconds"),
        $"category"
      )
      .agg(
        count("*").as("order_count"),
        sum("amount").as("total_amount")
      )
      .writeStream
      .outputMode(OutputMode.Update())
      .format("console")
      .option("truncate", "false")
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    // Simulate incoming files
    val executor = Executors.newSingleThreadScheduledExecutor()
    var fileCounter = 0
    
    executor.scheduleAtFixedRate(() => {
      fileCounter += 1
      val filename = s"$inputDir/orders-${System.currentTimeMillis()}.json"
      val writer = new PrintWriter(filename)
      
      // สร้าง 5 orders ต่อไฟล์
      for (i <- 1 to 5) {
        val category = Seq("Electronics", "Clothing", "Food")(i % 3)
        val amount = 100 + math.random() * 900
        val ts = System.currentTimeMillis() / 1000
        writer.println(
          s"""{"order_id":"O${fileCounter}_$i","customer_id":"C${i % 10}","amount":$amount,"category":"$category","timestamp":$ts}"""
        )
      }
      writer.close()
      println(s"Created file $filename")
    }, 0, 3, TimeUnit.SECONDS)
    
    // ===== Source 2: Rate Source (testing) =====
    val rateStream = spark.readStream
      .format("rate")
      .option("rowsPerSecond", 5)
      .load()
    
    val rateQuery = rateStream
      .select(
        $"timestamp",
        $"value",
        ($"value" % 3).as("category_id")
      )
      .writeStream
      .outputMode(OutputMode.Append())
      .format("console")
      .option("numRows", 5)
      .trigger(Trigger.ProcessingTime("3 seconds"))
      .start()
    
    // รอ
    Thread.sleep(20000)
    
    executor.shutdown()
    fileQuery.stop()
    rateQuery.stop()
    
    spark.stop()
  }
}
```

---

## Step 743: writeStream ไปยัง Sinks

```scala
// StreamingSinks.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.{DataFrame, ForeachWriter, Row}

object StreamingSinks {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Streaming Sinks")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val source = spark.readStream
      .format("rate")
      .option("rowsPerSecond", 10)
      .load()
      .withColumn("category", ($"value" % 3).cast("string"))
      .withColumn("amount", $"value" * 10.0)
    
    // ===== Sink 1: Console (debugging) =====
    val consoleQuery = source
      .writeStream
      .outputMode("append")
      .format("console")
      .option("truncate", "false")
      .option("numRows", 5)
      .trigger(Trigger.ProcessingTime("3 seconds"))
      .start()
    
    // ===== Sink 2: Memory (testing) =====
    val memoryQuery = source
      .groupBy($"category")
      .agg(count("*").as("count"), sum("amount").as("total"))
      .writeStream
      .outputMode("complete")
      .format("memory")
      .queryName("category_stats") // table name ที่ใช้ query ได้
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    // Query จาก memory sink
    Thread.sleep(6000)
    spark.sql("SELECT * FROM category_stats ORDER BY category").show()
    
    // ===== Sink 3: File =====
    val fileQuery = source
      .writeStream
      .outputMode("append")
      .format("parquet")
      .option("path", "/tmp/streaming-output/parquet")
      .option("checkpointLocation", "/tmp/streaming-checkpoints/parquet")
      .trigger(Trigger.ProcessingTime("10 seconds"))
      .start()
    
    // ===== Sink 4: ForeachBatch (custom processing) =====
    val foreachBatchQuery = source
      .groupBy($"category")
      .agg(count("*").as("count"))
      .writeStream
      .outputMode("complete")
      .foreachBatch { (batchDF: DataFrame, batchId: Long) =>
        println(s"=== Batch $batchId ===")
        batchDF.show()
        
        // เขียนไปหลาย sinks
        batchDF.write.mode("overwrite").parquet(s"/tmp/batch-$batchId")
        
        // หรือเขียน JDBC, MongoDB, ฯลฯ
        // batchDF.write.jdbc(url, table, props)
      }
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    // ===== Sink 5: Foreach (per-row custom processing) =====
    class DatabaseWriter extends ForeachWriter[Row] {
      // เปิด connection ครั้งเดียวต่อ partition
      override def open(partitionId: Long, epochId: Long): Boolean = {
        // open DB connection
        println(s"Opening partition $partitionId, epoch $epochId")
        true
      }
      
      override def process(row: Row): Unit = {
        // เขียน row ไปยัง database
        val category = row.getAs[String]("category")
        val amount   = row.getAs[Double]("amount")
        // db.insert(category, amount)
      }
      
      override def close(errorOrNull: Throwable): Unit = {
        if (errorOrNull != null) {
          println(s"Error: ${errorOrNull.getMessage}")
        }
        // close connection
      }
    }
    
    // ใช้ Foreach sink (ระวัง: ทำงาน per-row ช้ากว่า foreachBatch)
    // source.writeStream.foreach(new DatabaseWriter()).start()
    
    // รอ queries
    Thread.sleep(15000)
    
    consoleQuery.stop()
    memoryQuery.stop()
    fileQuery.stop()
    foreachBatchQuery.stop()
    
    spark.stop()
  }
}
```

---

## Step 744: Watermarks

```scala
// WatermarkDemo.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.functions._
import java.sql.Timestamp

object WatermarkDemo {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Watermark Demo")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    /*
    Watermark ใช้จัดการ Late Data:
    
    Event time:  10:00  10:05  10:10  10:15  10:20  10:25
    Processing:  |      |      |      |      |      |
    
    ถ้า watermark = 10 minutes:
    - เมื่อ processing time = 10:20
    - watermark = 10:20 - 10 min = 10:10
    - Data ที่ event time < 10:10 จะถูก drop
    
    Window: [10:00, 10:10) จะ close เมื่อ watermark > 10:10
    */
    
    // สร้าง streaming data พร้อม late events
    case class ClickEvent(userId: String, page: String, eventTime: Timestamp)
    
    // สร้าง data ที่มี late events
    val events = Seq(
      ClickEvent("U001", "/home",    new Timestamp(System.currentTimeMillis())),
      ClickEvent("U002", "/product", new Timestamp(System.currentTimeMillis() - 5000)),  // 5s late
      ClickEvent("U001", "/cart",    new Timestamp(System.currentTimeMillis() - 15000)), // 15s late (will be dropped with 10s watermark)
      ClickEvent("U003", "/checkout",new Timestamp(System.currentTimeMillis()))
    )
    
    // จำลอง streaming source
    val staticDF = events.toDF("user_id", "page", "event_time")
    
    // ใน production จะอ่านจาก Kafka หรือ file stream
    val streamingDF = spark.readStream
      .format("rate")
      .option("rowsPerSecond", 5)
      .load()
      .select(
        current_timestamp().as("event_time"),
        ($"value" % 5).cast("string").as("user_id"),
        lit("/page").as("page")
      )
    
    // ===== Windowed Aggregation with Watermark =====
    val windowedCounts = streamingDF
      .withWatermark("event_time", "10 seconds") // ยอมรับ late data ไม่เกิน 10 วินาที
      .groupBy(
        window($"event_time", "10 seconds", "5 seconds"), // 10s window, 5s slide
        $"user_id"
      )
      .agg(
        count("*").as("page_views")
      )
    
    // Complete mode ต้องการ watermark เมื่อมี window
    val query = windowedCounts
      .writeStream
      .outputMode(OutputMode.Update())
      .format("console")
      .option("truncate", "false")
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    query.awaitTermination(30000)
    query.stop()
    
    spark.stop()
  }
}
```

---

## Step 745: Stateful Stream Processing

```scala
// StatefulStreaming.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.{DataFrame, Dataset}

object StatefulStreaming {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Stateful Streaming")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== Stateful Aggregation =====
    val rateStream = spark.readStream
      .format("rate")
      .option("rowsPerSecond", 10)
      .load()
      .withColumn("user_id", ($"value" % 5).cast("string"))
      .withColumn("event_type", 
        when($"value" % 3 === 0, "click")
          .when($"value" % 3 === 1, "view")
          .otherwise("purchase"))
    
    // Running total per user (stateful - state grows over time)
    val runningStats = rateStream
      .withWatermark("timestamp", "30 seconds")
      .groupBy($"user_id")
      .agg(
        count("*").as("total_events"),
        sum(when($"event_type" === "purchase", 1).otherwise(0)).as("purchases"),
        sum(when($"event_type" === "click", 1).otherwise(0)).as("clicks")
      )
    
    val statsQuery = runningStats
      .writeStream
      .outputMode(OutputMode.Update())
      .format("console")
      .option("truncate", "false")
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    // ===== mapGroupsWithState (custom state) =====
    // สำหรับ complex stateful operations
    
    case class UserEvent(userId: String, eventType: String, timestamp: Long)
    case class UserState(
      userId: String,
      totalEvents: Long,
      lastEventTime: Long,
      sessionCount: Long
    )
    
    // แปลง DataFrame เป็น Dataset[UserEvent]
    val eventStream = rateStream
      .select(
        $"user_id".as("userId"),
        $"event_type".as("eventType"),
        $"timestamp".cast("long").as("timestamp")
      )
      .as[UserEvent]
    
    // ===== Session Detection ด้วย flatMapGroupsWithState =====
    import org.apache.spark.sql.streaming.GroupStateTimeout
    
    case class SessionState(lastEventTime: Long, sessionEvents: Long)
    case class SessionOutput(userId: String, sessionEvents: Long, sessionEnded: Boolean)
    
    val SESSION_TIMEOUT_MS = 5000L // 5 seconds
    
    val sessionStream = eventStream
      .withWatermark("timestamp", "10 seconds")
      .as[UserEvent]
    
    // flatMapGroupsWithState ให้ control เต็มที่
    // val sessions = sessionStream
    //   .groupByKey(_.userId)
    //   .flatMapGroupsWithState(
    //     outputMode = OutputMode.Append(),
    //     timeoutConf = GroupStateTimeout.EventTimeTimeout()
    //   ) { (userId, events, state) =>
    //     // Handle timeout
    //     if (state.hasTimedOut) {
    //       val SessionState(_, sessionEvents) = state.get
    //       state.remove()
    //       Iterator(SessionOutput(userId, sessionEvents, sessionEnded = true))
    //     } else {
    //       val currentState = state.getOption.getOrElse(SessionState(0, 0))
    //       val eventsSeq = events.toSeq
    //       val maxTime = eventsSeq.map(_.timestamp).max
    //       val newState = SessionState(maxTime, currentState.sessionEvents + eventsSeq.size)
    //       state.update(newState)
    //       state.setTimeoutTimestamp(maxTime + SESSION_TIMEOUT_MS)
    //       Iterator.empty
    //     }
    //   }
    
    Thread.sleep(20000)
    statsQuery.stop()
    
    spark.stop()
  }
}
```

---

## Step 746: Triggers และ Checkpointing

```scala
// TriggersAndCheckpoints.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.functions._

object TriggersAndCheckpoints {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Triggers and Checkpoints")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    
    val source = spark.readStream
      .format("rate")
      .option("rowsPerSecond", 10)
      .load()
    
    // ===== Trigger Types =====
    
    // 1. ProcessingTime: รัน ทุก N seconds
    val query1 = source
      .writeStream
      .format("console")
      .option("numRows", 3)
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    // 2. Once: รันครั้งเดียวแล้วหยุด (batch mode)
    // val query2 = source
    //   .writeStream
    //   .format("parquet")
    //   .option("path", "/tmp/once-output")
    //   .option("checkpointLocation", "/tmp/once-checkpoint")
    //   .trigger(Trigger.Once())
    //   .start()
    // query2.awaitTermination() // รอ until done
    
    // 3. AvailableNow: process all available data then stop (Spark 3.3+)
    // val query3 = source
    //   .writeStream
    //   .trigger(Trigger.AvailableNow())
    //   .start()
    
    // 4. Continuous: low-latency processing (experimental)
    // val query4 = source
    //   .writeStream
    //   .trigger(Trigger.Continuous("1 second"))
    //   .start()
    
    // ===== Checkpointing =====
    // Checkpoint บันทึก:
    // - Offsets อ่านมาแล้ว
    // - Progress ของ query
    // - State ของ stateful operations
    
    val checkpointedQuery = source
      .withColumn("category", ($"value" % 3).cast("string"))
      .groupBy($"category")
      .count()
      .writeStream
      .outputMode("complete")
      .format("console")
      .option("checkpointLocation", "/tmp/streaming-checkpoint") // ต้องมี checkpoint!
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    // Checkpoint ทำให้ restart query ได้โดยไม่ miss data
    // ถ้า query fail แล้ว restart จะ continue จาก checkpoint ล่าสุด
    
    // ===== Query Progress Monitoring =====
    Thread.sleep(10000)
    
    val progress = checkpointedQuery.lastProgress
    if (progress != null) {
      println(s"""
        |Query Progress:
        |  id: ${checkpointedQuery.id}
        |  runId: ${checkpointedQuery.runId}
        |  timestamp: ${progress.timestamp}
        |  inputRowsPerSecond: ${progress.inputRowsPerSecond}
        |  processedRowsPerSecond: ${progress.processedRowsPerSecond}
        |  batchId: ${progress.batchId}
        |  numInputRows: ${progress.numInputRows}
      """.stripMargin)
    }
    
    // Status
    val status = checkpointedQuery.status
    println(s"""
      |Query Status:
      |  message: ${status.message}
      |  isDataAvailable: ${status.isDataAvailable}
      |  isTriggerActive: ${status.isTriggerActive}
    """.stripMargin)
    
    // ===== StreamingQuery Management =====
    println(s"Active queries: ${spark.streams.active.map(_.name).mkString(", ")}")
    
    checkpointedQuery.stop()
    query1.stop()
    
    spark.stop()
  }
}
```

---

## Step 747: Kafka + Structured Streaming (Preview)

```scala
// KafkaStreaming.scala
// build.sbt ต้องมี:
// "org.apache.spark" %% "spark-sql-kafka-0-10" % "3.5.1"

import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._

object KafkaStreaming {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Kafka Streaming")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== อ่านจาก Kafka =====
    val kafkaStream = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "orders")           // single topic
      // .option("subscribePattern", "orders.*")  // regex pattern
      // .option("assign", """{"orders":[0,1]}""") // specific partitions
      .option("startingOffsets", "latest")  // latest | earliest | {"topic":{"0":23,"1":-1}}
      .option("maxOffsetsPerTrigger", 1000) // rate limiting
      .option("failOnDataLoss", "false")    // ไม่ fail ถ้า offset หาย
      .load()
    
    // Kafka message schema:
    // key: binary, value: binary, topic: string,
    // partition: int, offset: long, timestamp: timestamp,
    // timestampType: int
    
    // ===== Parse Kafka Messages =====
    val orderSchema = StructType(Array(
      StructField("order_id",    StringType,  true),
      StructField("customer_id", StringType,  true),
      StructField("product_id",  StringType,  true),
      StructField("amount",      DoubleType,  true),
      StructField("category",    StringType,  true),
      StructField("event_time",  LongType,    true)
    ))
    
    val orders = kafkaStream
      .select(
        $"key".cast("string").as("message_key"),
        from_json($"value".cast("string"), orderSchema).as("data"),
        $"timestamp".as("kafka_timestamp"),
        $"partition",
        $"offset"
      )
      .select("message_key", "data.*", "kafka_timestamp", "partition", "offset")
    
    // ===== Aggregations =====
    val orderStats = orders
      .withColumn("event_time", to_timestamp(($"event_time" / 1000).cast("long")))
      .withWatermark("event_time", "1 minute")
      .groupBy(
        window($"event_time", "1 minute"),
        $"category"
      )
      .agg(
        count("*").as("order_count"),
        sum("amount").as("total_amount"),
        avg("amount").as("avg_amount")
      )
    
    // ===== เขียนกลับไปยัง Kafka =====
    val kafkaOutput = orderStats
      .select(
        to_json(struct("*")).as("value"),
        $"category".as("key")
      )
      .writeStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("topic", "order-stats")
      .option("checkpointLocation", "/tmp/kafka-checkpoint")
      .outputMode("update")
      .trigger(Trigger.ProcessingTime("30 seconds"))
    // .start() // uncomment เมื่อมี Kafka จริง
    
    println("Kafka streaming query defined (not started - needs Kafka)")
    
    spark.stop()
  }
}
```

---

## Step 748: Stream-Static Join

```scala
// StreamStaticJoin.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.functions._

object StreamStaticJoin {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Stream-Static Join")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== Static Reference Data =====
    val productLookup = Seq(
      ("P001", "iPhone 15",   "Electronics", "Apple"),
      ("P002", "T-Shirt",     "Clothing",    "Brand A"),
      ("P003", "MacBook Pro", "Electronics", "Apple"),
      ("P004", "Coffee",      "Food",        "Local")
    ).toDF("product_id", "product_name", "category", "brand")
    
    val customerLookup = Seq(
      ("C001", "Alice",  "Premium",  "Bangkok"),
      ("C002", "Bob",    "Standard", "Chiang Mai"),
      ("C003", "Carol",  "VIP",      "Phuket")
    ).toDF("customer_id", "name", "tier", "city")
    
    // ===== Streaming Data =====
    val orderStream = spark.readStream
      .format("rate")
      .option("rowsPerSecond", 5)
      .load()
      .withColumn("order_id",    concat(lit("O"), $"value".cast("string")))
      .withColumn("customer_id", concat(lit("C00"), ($"value" % 3 + 1).cast("string")))
      .withColumn("product_id",  concat(lit("P00"), ($"value" % 4 + 1).cast("string")))
      .withColumn("amount",      ($"value" * 10.0) % 5000)
    
    // ===== Stream-Static Join =====
    // Spark reloads static table at each micro-batch (ถ้าเปิด streaming option)
    val enrichedOrders = orderStream
      .join(productLookup, "product_id")    // join กับ static DataFrame
      .join(customerLookup, "customer_id")  // join กับ static DataFrame
      .select(
        $"order_id",
        $"name".as("customer_name"),
        $"tier",
        $"city",
        $"product_name",
        $"category",
        $"brand",
        $"amount",
        $"timestamp"
      )
    
    val query = enrichedOrders
      .writeStream
      .outputMode("append")
      .format("console")
      .option("truncate", "false")
      .option("numRows", 5)
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    // ===== Alerts: Stream with filtering =====
    val alerts = enrichedOrders
      .filter($"amount" > 3000)
      .withColumn("alert_message",
        concat(lit("HIGH VALUE ORDER: "), $"order_id",
               lit(" from "), $"customer_name",
               lit(" - $"), $"amount"))
    
    val alertQuery = alerts
      .select($"alert_message")
      .writeStream
      .outputMode("append")
      .format("console")
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    Thread.sleep(20000)
    query.stop()
    alertQuery.stop()
    
    spark.stop()
  }
}
```

---

## Step 749: Error Handling และ Monitoring

```scala
// StreamingMonitoring.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.functions._

object StreamingMonitoring {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Streaming Monitoring")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    
    // ===== Custom StreamingQueryListener =====
    spark.streams.addListener(new StreamingQueryListener() {
      override def onQueryStarted(event: StreamingQueryListener.QueryStartedEvent): Unit = {
        println(s"[MONITOR] Query started: id=${event.id}, name=${event.name}")
      }
      
      override def onQueryProgress(event: StreamingQueryListener.QueryProgressEvent): Unit = {
        val progress = event.progress
        println(s"""[MONITOR] Progress:
          |  query: ${progress.name}
          |  batchId: ${progress.batchId}
          |  inputRows: ${progress.numInputRows}
          |  inputRate: ${progress.inputRowsPerSecond} rows/s
          |  processRate: ${progress.processedRowsPerSecond} rows/s
          |  duration: ${progress.durationMs.get("triggerExecution")}ms
        """.stripMargin)
      }
      
      override def onQueryTerminated(event: StreamingQueryListener.QueryTerminatedEvent): Unit = {
        event.exception match {
          case Some(ex) => println(s"[MONITOR] Query FAILED: id=${event.id}, error=${ex}")
          case None     => println(s"[MONITOR] Query stopped normally: id=${event.id}")
        }
      }
    })
    
    val source = spark.readStream
      .format("rate")
      .option("rowsPerSecond", 20)
      .load()
    
    // ===== Error Recovery =====
    val query = source
      .withColumn("processed", {
        // จำลอง processing ที่อาจ fail
        when($"value" % 100 === 0, {
          // ปกติจะ throw exception แต่ใน Spark จัดการ gracefully
          $"value" // แทนที่จะ throw
        }).otherwise($"value" * 2)
      })
      .writeStream
      .outputMode("append")
      .format("console")
      .option("numRows", 3)
      .option("checkpointLocation", "/tmp/monitor-checkpoint")
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    // ===== Health Check Loop =====
    val checkInterval = 3000L
    var checks = 0
    
    while (query.isActive && checks < 5) {
      Thread.sleep(checkInterval)
      checks += 1
      
      val status = query.status
      val progress = query.lastProgress
      
      println(s"\n=== Health Check #$checks ===")
      println(s"  Active: ${query.isActive}")
      println(s"  Status: ${status.message}")
      
      if (progress != null) {
        println(s"  Batch: ${progress.batchId}")
        println(s"  Input rate: ${f"${progress.inputRowsPerSecond}%.1f"} rows/s")
        println(s"  Recent batches: ${query.recentProgress.length}")
      }
      
      // Check for exception
      query.exception match {
        case Some(ex) =>
          println(s"  ERROR: ${ex.getMessage}")
        case None =>
          println("  Status: HEALTHY")
      }
    }
    
    query.stop()
    
    spark.stop()
  }
}
```

---

## Step 750: Production Streaming Pipeline

```scala
// ProductionStreamPipeline.scala
import org.apache.spark.sql.{SparkSession, DataFrame}
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._

object ProductionStreamPipeline {
  
  // Schema definitions
  val clickEventSchema = StructType(Array(
    StructField("user_id",    StringType,    true),
    StructField("session_id", StringType,    true),
    StructField("page",       StringType,    true),
    StructField("action",     StringType,    true),
    StructField("event_time", TimestampType, true),
    StructField("device",     StringType,    true),
    StructField("country",    StringType,    true)
  ))
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Production Streaming Pipeline")
      .config("spark.sql.shuffle.partitions", "4")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== Simulate Event Stream =====
    val eventStream = spark.readStream
      .format("rate")
      .option("rowsPerSecond", 50)
      .load()
      .select(
        concat(lit("U"), ($"value" % 100).cast("string")).as("user_id"),
        concat(lit("S"), ($"value" % 20).cast("string")).as("session_id"),
        when($"value" % 4 === 0, "/home")
          .when($"value" % 4 === 1, "/product")
          .when($"value" % 4 === 2, "/cart")
          .otherwise("/checkout").as("page"),
        when($"value" % 3 === 0, "click")
          .when($"value" % 3 === 1, "view")
          .otherwise("purchase").as("action"),
        $"timestamp".as("event_time"),
        when($"value" % 2 === 0, "mobile").otherwise("desktop").as("device"),
        when($"value" % 5 === 0, "TH")
          .when($"value" % 5 === 1, "US")
          .when($"value" % 5 === 2, "SG")
          .when($"value" % 5 === 3, "JP")
          .otherwise("UK").as("country")
      )
    
    // ===== Real-time Dashboard Metrics =====
    val dashboardMetrics = eventStream
      .withWatermark("event_time", "30 seconds")
      .groupBy(
        window($"event_time", "1 minute"),
        $"country",
        $"device"
      )
      .agg(
        count("*").as("total_events"),
        countDistinct("user_id").as("unique_users"),
        countDistinct("session_id").as("unique_sessions"),
        sum(when($"action" === "purchase", 1).otherwise(0)).as("purchases"),
        sum(when($"action" === "view", 1).otherwise(0)).as("views"),
        sum(when($"action" === "click", 1).otherwise(0)).as("clicks")
      )
      .withColumn("conversion_rate",
        round($"purchases" / $"total_events" * 100, 2))
    
    // ===== Anomaly Detection =====
    val anomalyStream = eventStream
      .withWatermark("event_time", "10 seconds")
      .groupBy(
        window($"event_time", "10 seconds"),
        $"user_id"
      )
      .agg(count("*").as("events_per_10s"))
      .filter($"events_per_10s" > 20) // user ที่ทำ > 20 actions ใน 10 วินาที
      .select(
        $"window.start",
        $"user_id",
        $"events_per_10s",
        lit("SUSPICIOUS_ACTIVITY").as("alert_type")
      )
    
    // ===== Start Queries =====
    val metricsQuery = dashboardMetrics
      .writeStream
      .outputMode("update")
      .format("console")
      .option("truncate", "false")
      .option("numRows", 5)
      .option("checkpointLocation", "/tmp/metrics-checkpoint")
      .trigger(Trigger.ProcessingTime("10 seconds"))
      .start()
    
    val anomalyQuery = anomalyStream
      .writeStream
      .outputMode("append")
      .format("console")
      .option("truncate", "false")
      .option("checkpointLocation", "/tmp/anomaly-checkpoint")
      .trigger(Trigger.ProcessingTime("10 seconds"))
      .start()
    
    // ===== Graceful Shutdown =====
    Runtime.getRuntime.addShutdownHook(new Thread(() => {
      println("Shutting down streaming queries...")
      metricsQuery.stop()
      anomalyQuery.stop()
      spark.stop()
    }))
    
    // Run for 30 seconds
    metricsQuery.awaitTermination(30000)
    anomalyQuery.stop()
    
    spark.stop()
  }
}
```

---

## สรุป Part 75

| Concept | Description | ใช้เมื่อ |
|---------|-------------|----------|
| readStream | อ่าน streaming source | Kafka, files, rate |
| writeStream | เขียน streaming output | Console, file, Kafka, custom |
| Trigger | กำหนด frequency ของ microbatch | ProcessingTime, Once |
| Watermark | จัดการ late data | Event-time windowed ops |
| OutputMode.Append | เพิ่มแถวใหม่ | No aggregation |
| OutputMode.Update | อัปเดตแถวที่เปลี่ยน | Aggregation |
| OutputMode.Complete | เขียน result table ทั้งหมด | Small aggregation |
| Checkpoint | รับประกัน exactly-once | Production queries |
| foreachBatch | Custom batch processing | Multi-sink, custom logic |
| mapGroupsWithState | Custom stateful logic | Session detection |

---

## แบบฝึกหัด Part 75

1. **Click Analytics**: สร้าง streaming pipeline ที่อ่าน click events แบบ rate source แล้วคำนวณ real-time metrics: page views per minute, unique users per session, conversion funnel

2. **Watermark Experiment**: สร้าง stream ที่มี late events (จำลองโดยใส่ timestamp เก่า) แล้วทดสอบ watermark ต่างๆ ดูว่า late events ถูก drop หรือไม่

3. **File Streaming ETL**: สร้าง pipeline ที่ watch directory สำหรับ incoming CSV files แล้ว parse, validate, transform, และเขียน clean data เป็น Parquet แบบ partitioned

4. **Multi-Query Dashboard**: สร้าง streaming application ที่รัน 3 queries พร้อมกัน: (1) real-time counts, (2) anomaly detection, (3) session aggregation พร้อม monitoring

5. **Exactly-Once Semantics**: implement idempotent write ด้วย foreachBatch ที่ตรวจสอบ batch ID ก่อนเขียน เพื่อ guarantee exactly-once ใน case ที่ query restart

---

## ไปต่อ: Part 76 — Kafka Basics
ใน Part ถัดไปจะเรียน Apache Kafka concepts, producers, consumers, topics และ partitions

[→ Part 76: Kafka Basics](./part-76-kafka-basics.md)
