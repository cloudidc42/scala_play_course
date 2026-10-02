# Part 78: Spark + Kafka Integration — Steps 771-780

## บทนำ: Spark Structured Streaming + Kafka

การรวม Spark Structured Streaming กับ Kafka สร้าง powerful real-time analytics platform ที่รองรับ high-throughput, fault-tolerant stream processing

---

## Step 771: Setup

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.apache.spark" %% "spark-sql"              % "3.5.1" % "provided",
  "org.apache.spark" %% "spark-sql-kafka-0-10"   % "3.5.1",
  "org.apache.spark" %% "spark-streaming"        % "3.5.1" % "provided",
  "org.apache.kafka"  % "kafka-clients"           % "3.6.0",
  "io.delta"         %% "delta-spark"             % "3.1.0"
)
```

### SparkSession with Kafka

```scala
// SparkKafkaSetup.scala
import org.apache.spark.sql.SparkSession

object SparkKafkaSetup {
  def createSparkSession(): SparkSession = {
    SparkSession.builder()
      .master("local[*]")
      .appName("Spark-Kafka Integration")
      .config("spark.sql.shuffle.partitions", "8")
      .config("spark.sql.adaptive.enabled", "true")
      // Kafka settings ผ่าน Spark config
      .config("spark.kafka.consumer.cache.enabled", "true")
      .config("spark.kafka.consumer.cache.capacity", "64")
      .getOrCreate()
  }
}
```

---

## Step 772: readStream จาก Kafka

```scala
// ReadFromKafka.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._

object ReadFromKafka {
  def main(args: Array[String]): Unit = {
    val spark = SparkKafkaSetup.createSparkSession()
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== Basic Kafka Read =====
    val kafkaDF = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "orders")
      .option("startingOffsets", "latest")
      .option("maxOffsetsPerTrigger", "10000") // rate control
      .option("kafka.group.id", "spark-consumer") // optional group id
      .option("failOnDataLoss", "false")
      .load()
    
    // Kafka DataFrame schema:
    // key: binary, value: binary, topic: string,
    // partition: int, offset: long, timestamp: timestamp, timestampType: int
    
    kafkaDF.printSchema()
    
    // ===== Parse JSON Messages =====
    val orderSchema = StructType(Array(
      StructField("order_id",    StringType, true),
      StructField("customer_id", StringType, true),
      StructField("product_id",  StringType, true),
      StructField("amount",      DoubleType, true),
      StructField("category",    StringType, true),
      StructField("event_time",  TimestampType, true)
    ))
    
    val orders = kafkaDF
      .select(
        $"key".cast("string").as("message_key"),
        from_json($"value".cast("string"), orderSchema).as("order"),
        $"topic",
        $"partition",
        $"offset",
        $"timestamp".as("kafka_timestamp")
      )
      .select("message_key", "order.*", "topic", "partition", "offset", "kafka_timestamp")
      .filter($"order_id".isNotNull) // filter bad messages
    
    // ===== Multiple Topics =====
    val multiTopicDF = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "orders,payments,shipping") // comma-separated
      // .option("subscribePattern", "order.*")  // regex pattern
      .load()
    
    // แยก processing ตาม topic
    val topicRouted = multiTopicDF
      .select(
        $"topic",
        $"key".cast("string"),
        $"value".cast("string"),
        $"timestamp"
      )
    
    // ===== Specific Partitions =====
    val specificPartitionsDF = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("assign", """{"orders":[0,1,2]}""") // read only partitions 0,1,2
      .option("startingOffsets", """{"orders":{"0":100,"1":200,"2":0}}""") // specific offsets
      .load()
    
    // ===== Test: Write to Console =====
    val query = orders
      .writeStream
      .outputMode("append")
      .format("console")
      .option("truncate", "false")
      .option("numRows", 10)
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    query.awaitTermination(30000)
    query.stop()
    
    spark.stop()
  }
}
```

---

## Step 773: Real-time Aggregations จาก Kafka

```scala
// KafkaRealtimeAggregations.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._

object KafkaRealtimeAggregations {
  def main(args: Array[String]): Unit = {
    val spark = SparkKafkaSetup.createSparkSession()
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val orderSchema = StructType(Array(
      StructField("order_id",    StringType, true),
      StructField("customer_id", StringType, true),
      StructField("category",    StringType, true),
      StructField("amount",      DoubleType, true),
      StructField("region",      StringType, true),
      StructField("event_time",  LongType,   true) // unix timestamp in millis
    ))
    
    // อ่านจาก Kafka
    val orders = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "orders")
      .option("startingOffsets", "latest")
      .load()
      .select(from_json($"value".cast("string"), orderSchema).as("data"))
      .select($"data.*")
      .withColumn("event_time", to_timestamp(($"event_time" / 1000).cast("long")))
      .filter($"order_id".isNotNull)
    
    // ===== Running Aggregations (no window) =====
    val runningStats = orders
      .withWatermark("event_time", "1 minute")
      .groupBy($"category", $"region")
      .agg(
        count("*").as("order_count"),
        sum("amount").as("total_revenue"),
        avg("amount").as("avg_order_value"),
        max("amount").as("max_order")
      )
    
    val statsQuery = runningStats
      .writeStream
      .outputMode("update")
      .format("console")
      .option("truncate", "false")
      .option("checkpointLocation", "/tmp/checkpoints/running-stats")
      .trigger(Trigger.ProcessingTime("10 seconds"))
      .start()
    
    // ===== Windowed Aggregations =====
    val windowedRevenue = orders
      .withWatermark("event_time", "2 minutes") // allow 2 min late data
      .groupBy(
        window($"event_time", "1 minute", "30 seconds"), // 1min window, 30s slide
        $"category"
      )
      .agg(
        count("*").as("orders"),
        sum("amount").as("revenue"),
        approx_count_distinct("customer_id").as("unique_customers")
      )
      .select(
        $"window.start",
        $"window.end",
        $"category",
        $"orders",
        $"revenue",
        $"unique_customers"
      )
    
    val windowQuery = windowedRevenue
      .writeStream
      .outputMode("update")
      .format("memory")
      .queryName("windowed_revenue")
      .option("checkpointLocation", "/tmp/checkpoints/windowed-revenue")
      .trigger(Trigger.ProcessingTime("30 seconds"))
      .start()
    
    // Query from memory sink for testing
    Thread.sleep(35000)
    spark.sql("SELECT * FROM windowed_revenue ORDER BY start DESC LIMIT 10").show()
    
    statsQuery.stop()
    windowQuery.stop()
    
    spark.stop()
  }
}
```

---

## Step 774: writeStream ไปยัง Kafka

```scala
// WriteToKafka.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._

object WriteToKafka {
  def main(args: Array[String]): Unit = {
    val spark = SparkKafkaSetup.createSparkSession()
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // Source: rate สำหรับ testing
    val source = spark.readStream
      .format("rate")
      .option("rowsPerSecond", 50)
      .load()
      .select(
        ($"value" % 5).cast("string").as("customer_id"),
        ($"value" % 3).cast("string").as("category"),
        ($"value" * 10.0 % 5000).as("amount"),
        $"timestamp"
      )
    
    // ===== ส่ง DataFrame ไป Kafka =====
    // ต้องมี column "value" (binary หรือ string)
    // optionally มี "key" และ "topic"
    
    // Simple: ทุก record ไป same topic
    val kafkaWriteQuery = source
      .select(
        to_json(struct("customer_id", "category", "amount")).as("value"),
        $"customer_id".as("key")  // optional: ใช้เป็น partition key
      )
      .writeStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("topic", "processed-orders")
      .option("checkpointLocation", "/tmp/checkpoints/kafka-write")
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    // ===== Dynamic Topic Routing =====
    val routedDF = source
      .withColumn("target_topic",
        when($"amount" > 3000, "high-value-orders")
          .when($"amount" > 1000, "medium-value-orders")
          .otherwise("regular-orders"))
      .select(
        $"target_topic".as("topic"),  // different topics per record!
        to_json(struct("customer_id", "category", "amount")).as("value"),
        $"customer_id".as("key")
      )
    
    val routedQuery = routedDF
      .writeStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("checkpointLocation", "/tmp/checkpoints/kafka-routed")
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    // ===== Kafka Headers =====
    // เพิ่ม headers ใน Kafka messages
    val withHeadersDF = source
      .select(
        to_json(struct("customer_id", "amount")).as("value"),
        $"customer_id".as("key"),
        // headers เป็น array of structs
        array(
          struct(lit("source").as("key"), lit("spark-app").cast("binary").as("value")),
          struct(lit("version").as("key"), lit("1.0").cast("binary").as("value"))
        ).as("headers")
      )
    
    withHeadersDF
      .writeStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("topic", "orders-with-headers")
      .option("checkpointLocation", "/tmp/checkpoints/kafka-headers")
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    Thread.sleep(20000)
    
    kafkaWriteQuery.stop()
    routedQuery.stop()
    
    spark.stop()
  }
}
```

---

## Step 775: Kafka-to-Kafka Pipeline

```scala
// KafkaPipeline.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._

object KafkaPipeline {
  
  val orderSchema = StructType(Array(
    StructField("order_id",    StringType, true),
    StructField("customer_id", StringType, true),
    StructField("product_id",  StringType, true),
    StructField("amount",      DoubleType, true),
    StructField("category",    StringType, true),
    StructField("region",      StringType, true),
    StructField("event_time",  LongType,   true)
  ))
  
  def main(args: Array[String]): Unit = {
    val spark = SparkKafkaSetup.createSparkSession()
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== Stage 1: Validation =====
    val rawOrders = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "raw-orders")
      .load()
      .select(from_json($"value".cast("string"), orderSchema).as("data"))
      .select($"data.*")
      .withColumn("event_time", to_timestamp(($"event_time" / 1000).cast("long")))
    
    // Validate and split into valid/invalid
    val validOrders = rawOrders.filter(
      $"order_id".isNotNull &&
      $"customer_id".isNotNull &&
      $"amount" > 0
    ).withColumn("validated_at", current_timestamp())
    
    val invalidOrders = rawOrders.filter(
      $"order_id".isNull ||
      $"customer_id".isNull ||
      !($"amount" > 0)
    ).withColumn("error_reason", 
      when($"order_id".isNull, "missing_order_id")
        .when($"customer_id".isNull, "missing_customer_id")
        .otherwise("invalid_amount"))
    
    // Write valid orders to next topic
    val validQuery = validOrders
      .select(
        $"order_id".as("key"),
        to_json(struct("order_id", "customer_id", "amount", "category", "region", "event_time", "validated_at")).as("value")
      )
      .writeStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("topic", "validated-orders")
      .option("checkpointLocation", "/tmp/checkpoints/valid-orders")
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    // Write invalid to DLQ
    val invalidQuery = invalidOrders
      .select(
        to_json(struct($"*")).as("value")
      )
      .writeStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("topic", "dead-letter-orders")
      .option("checkpointLocation", "/tmp/checkpoints/invalid-orders")
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    // ===== Stage 2: Enrichment =====
    // อ่าน validated orders แล้ว enrich
    val validatedOrders = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "validated-orders")
      .load()
      .select(from_json($"value".cast("string"), orderSchema).as("data"))
      .select($"data.*")
    
    // Static lookup for enrichment
    val categoryMetadata = spark.createDataFrame(Seq(
      ("Electronics", "Tech", 0.02),  // category, division, tax_rate
      ("Clothing",    "Fashion", 0.07),
      ("Food",        "FMCG", 0.0)
    )).toDF("category", "division", "tax_rate")
    
    val enrichedOrders = validatedOrders
      .join(categoryMetadata, "category")
      .withColumn("tax_amount", $"amount" * $"tax_rate")
      .withColumn("total_amount", $"amount" + $"tax_amount")
      .withColumn("enriched_at", current_timestamp())
    
    val enrichedQuery = enrichedOrders
      .select(
        $"order_id".as("key"),
        to_json(struct("order_id", "customer_id", "amount", "tax_amount", 
                      "total_amount", "category", "division", "region")).as("value")
      )
      .writeStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("topic", "enriched-orders")
      .option("checkpointLocation", "/tmp/checkpoints/enriched-orders")
      .trigger(Trigger.ProcessingTime("5 seconds"))
      .start()
    
    // ===== Stage 3: Aggregation =====
    val enrichedSchema = StructType(Array(
      StructField("order_id",      StringType, true),
      StructField("customer_id",   StringType, true),
      StructField("amount",        DoubleType, true),
      StructField("total_amount",  DoubleType, true),
      StructField("category",      StringType, true),
      StructField("division",      StringType, true),
      StructField("region",        StringType, true)
    ))
    
    val enrichedStream = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "enriched-orders")
      .load()
      .select(from_json($"value".cast("string"), enrichedSchema).as("data"))
      .select($"data.*")
    
    val realtimeDashboard = enrichedStream
      .withColumn("event_time", current_timestamp())
      .withWatermark("event_time", "1 minute")
      .groupBy(window($"event_time", "1 minute"), $"category", $"region")
      .agg(
        count("*").as("orders"),
        sum("total_amount").as("revenue"),
        avg("total_amount").as("avg_order_value")
      )
    
    val dashboardQuery = realtimeDashboard
      .select(
        concat($"category", lit("-"), $"region").as("key"),
        to_json(struct("window", "category", "region", "orders", "revenue", "avg_order_value")).as("value")
      )
      .writeStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("topic", "realtime-dashboard")
      .option("checkpointLocation", "/tmp/checkpoints/dashboard")
      .outputMode("update")
      .trigger(Trigger.ProcessingTime("30 seconds"))
      .start()
    
    // Monitor all queries
    println(s"Active streaming queries: ${spark.streams.active.length}")
    
    // Wait
    Thread.sleep(60000)
    
    validQuery.stop()
    invalidQuery.stop()
    enrichedQuery.stop()
    dashboardQuery.stop()
    
    spark.stop()
  }
}
```

---

## Step 776: Kafka Watermarks และ Late Data

```scala
// KafkaWatermarks.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._

object KafkaWatermarks {
  def main(args: Array[String]): Unit = {
    val spark = SparkKafkaSetup.createSparkSession()
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val eventSchema = StructType(Array(
      StructField("event_id",   StringType,    true),
      StructField("user_id",    StringType,    true),
      StructField("event_type", StringType,    true),
      StructField("value",      DoubleType,    true),
      StructField("event_time", TimestampType, true)  // event time from source
    ))
    
    val eventStream = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "user-events")
      .load()
      .select(from_json($"value".cast("string"), eventSchema).as("data"))
      .select($"data.*")
      .filter($"event_id".isNotNull)
    
    // ===== Event Time vs Processing Time =====
    val withBothTimes = eventStream
      .withColumn("processing_time", current_timestamp())
      .withColumn("latency_seconds", 
        unix_timestamp($"processing_time") - unix_timestamp($"event_time"))
    
    // ===== Watermark สำหรับ Late Data =====
    // ยอมรับ events ที่ delay ไม่เกิน 5 นาที
    val withWatermark = withBothTimes
      .withWatermark("event_time", "5 minutes")
    
    // Window aggregation ที่ handle late data
    val windowedMetrics = withWatermark
      .groupBy(
        window($"event_time", "1 minute"), // 1 minute windows
        $"event_type"
      )
      .agg(
        count("*").as("event_count"),
        sum("value").as("total_value"),
        avg("latency_seconds").as("avg_latency")
      )
    
    // ===== Session Windows =====
    // Session: ช่วงเวลาที่ user active (gap ระหว่าง events < 10 minutes)
    val sessionMetrics = withWatermark
      .groupBy(
        session_window($"event_time", "10 minutes"), // session gap
        $"user_id"
      )
      .agg(
        count("*").as("events_in_session"),
        min("event_time").as("session_start"),
        max("event_time").as("session_end")
      )
    
    // ===== Monitor Watermark Progress =====
    val metricsQuery = windowedMetrics
      .writeStream
      .outputMode("update")
      .format("console")
      .option("truncate", "false")
      .option("checkpointLocation", "/tmp/checkpoints/watermark-metrics")
      .trigger(Trigger.ProcessingTime("30 seconds"))
      .start()
    
    // Monitor watermark progress
    Thread.sleep(10000)
    val progress = metricsQuery.lastProgress
    if (progress != null) {
      println(s"""
        |Watermark Info:
        |  Event time watermark: ${progress.eventTime.get("watermark")}
        |  Max event time: ${progress.eventTime.get("max")}
        |  Min event time: ${progress.eventTime.get("min")}
      """.stripMargin)
    }
    
    metricsQuery.stop()
    spark.stop()
  }
}
```

---

## Step 777: Kafka + Spark Batch for Historical Analysis

```scala
// SparkKafkaBatch.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._

object SparkKafkaBatch {
  def main(args: Array[String]): Unit = {
    val spark = SparkKafkaSetup.createSparkSession()
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== Batch Read จาก Kafka =====
    // ใช้ read (ไม่ใช่ readStream) สำหรับ one-time batch processing
    val historicalData = spark.read
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "orders")
      .option("startingOffsets", "earliest")
      .option("endingOffsets", "latest")
      .load()
    
    val orderSchema = StructType(Array(
      StructField("order_id",    StringType, true),
      StructField("customer_id", StringType, true),
      StructField("amount",      DoubleType, true),
      StructField("category",    StringType, true),
      StructField("event_time",  LongType,   true)
    ))
    
    val orders = historicalData
      .select(from_json($"value".cast("string"), orderSchema).as("data"))
      .select($"data.*")
      .filter($"order_id".isNotNull)
      .withColumn("event_time", to_timestamp(($"event_time" / 1000).cast("long")))
    
    println(s"Total historical orders: ${orders.count()}")
    
    // ===== Historical Analysis =====
    println("=== Revenue by Category ===")
    orders.groupBy($"category")
          .agg(
            count("*").as("orders"),
            sum("amount").as("revenue")
          )
          .orderBy($"revenue".desc)
          .show()
    
    // ===== Read ช่วง Offset ที่ต้องการ =====
    val specificRange = spark.read
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "orders")
      .option("startingOffsets", """{"orders":{"0":0,"1":0,"2":0}}""")
      .option("endingOffsets",   """{"orders":{"0":100,"1":100,"2":100}}""")
      .load()
    
    println(s"Specific range records: ${specificRange.count()}")
    
    // ===== ผสม Streaming + Batch (Lambda Architecture) =====
    // Speed Layer: real-time streaming
    val speedLayer = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "orders")
      .option("startingOffsets", "latest")
      .load()
    
    // Batch Layer: historical analysis
    // val batchLayer = orders.groupBy($"category").agg(sum("amount"))
    
    // Serving Layer: combine ทั้งสอง
    // Real implementation จะใช้ Delta Lake หรือ database
    
    spark.stop()
  }
}
```

---

## Step 778: Schema Evolution

```scala
// SchemaEvolution.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._

object SchemaEvolution {
  def main(args: Array[String]): Unit = {
    val spark = SparkKafkaSetup.createSparkSession()
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // Schema v1 (old)
    val schemaV1 = StructType(Array(
      StructField("order_id",    StringType, true),
      StructField("customer_id", StringType, true),
      StructField("amount",      DoubleType, true)
    ))
    
    // Schema v2 (new - added fields)
    val schemaV2 = StructType(Array(
      StructField("order_id",    StringType, true),
      StructField("customer_id", StringType, true),
      StructField("amount",      DoubleType, true),
      StructField("category",    StringType, true),  // new field
      StructField("region",      StringType, true)   // new field
    ))
    
    // ===== Handle Schema Evolution =====
    // อ่านด้วย schema ที่ flexible (nullable fields)
    val flexibleSchema = StructType(Array(
      StructField("order_id",    StringType, nullable = true),
      StructField("customer_id", StringType, nullable = true),
      StructField("amount",      DoubleType, nullable = true),
      StructField("category",    StringType, nullable = true),  // optional
      StructField("region",      StringType, nullable = true)   // optional
    ))
    
    val orders = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "orders-v2")
      .load()
      .select(from_json($"value".cast("string"), flexibleSchema).as("data"))
      .select($"data.*")
      // handle null ที่มาจาก old schema
      .withColumn("category", coalesce($"category", lit("unknown")))
      .withColumn("region",   coalesce($"region",   lit("unknown")))
    
    // ===== Schema Registry Integration =====
    // ใน production ควรใช้ Confluent Schema Registry
    // จะ auto-handle schema evolution
    
    /*
    // ด้วย schema registry:
    spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "orders")
      .option("schema.registry.url", "http://localhost:8081")
      .load()
    */
    
    println("Schema evolution handling configured")
    
    spark.stop()
  }
}
```

---

## Step 779: Monitoring และ Alerting

```scala
// KafkaSparkMonitoring.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.functions._

object KafkaSparkMonitoring {
  def main(args: Array[String]): Unit = {
    val spark = SparkKafkaSetup.createSparkSession()
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== Streaming Query Listener =====
    spark.streams.addListener(new StreamingQueryListener() {
      override def onQueryStarted(event: StreamingQueryListener.QueryStartedEvent): Unit = {
        println(s"[KAFKA-MONITOR] Query started: ${event.name}")
      }
      
      override def onQueryProgress(event: StreamingQueryListener.QueryProgressEvent): Unit = {
        val p = event.progress
        
        // Alert ถ้า processing lag มาก
        val kafkaSources = Option(p.sources).map(_.filter(_.description.contains("kafka"))).getOrElse(Array.empty)
        kafkaSources.foreach { source =>
          println(s"""[KAFKA-MONITOR] ${p.name}:
            |  Input rate: ${p.inputRowsPerSecond} rows/s
            |  Process rate: ${p.processedRowsPerSecond} rows/s  
            |  Batch: ${p.batchId}
            |  Duration: ${p.durationMs.getOrDefault("triggerExecution", -1)}ms
          """.stripMargin)
          
          // Alert if falling behind
          if (p.inputRowsPerSecond > p.processedRowsPerSecond * 1.5) {
            println(s"[ALERT] ${p.name}: Processing lag detected!")
          }
        }
      }
      
      override def onQueryTerminated(event: StreamingQueryListener.QueryTerminatedEvent): Unit = {
        event.exception match {
          case Some(ex) => println(s"[KAFKA-MONITOR] Query FAILED: ${ex}")
          case None     => println(s"[KAFKA-MONITOR] Query stopped normally")
        }
      }
    })
    
    // ===== Real-time Anomaly Detection =====
    val eventSchema = org.apache.spark.sql.types.StructType(Array(
      org.apache.spark.sql.types.StructField("user_id",     org.apache.spark.sql.types.StringType, true),
      org.apache.spark.sql.types.StructField("amount",      org.apache.spark.sql.types.DoubleType, true),
      org.apache.spark.sql.types.StructField("event_time",  org.apache.spark.sql.types.LongType,   true)
    ))
    
    val events = spark.readStream
      .format("rate")
      .option("rowsPerSecond", 100)
      .load()
      .select(
        ($"value" % 100).cast("string").as("user_id"),
        ($"value" * 10.0 % 10000).as("amount"),
        ($"timestamp".cast("long") * 1000).as("event_time"),
        $"timestamp"
      )
    
    // Detect high frequency users (potential bot)
    val highFreqUsers = events
      .withWatermark("timestamp", "1 minute")
      .groupBy(window($"timestamp", "10 seconds"), $"user_id")
      .agg(count("*").as("event_count"))
      .filter($"event_count" > 50)
      .select(
        $"user_id",
        $"event_count",
        lit("HIGH_FREQUENCY").as("alert_type"),
        current_timestamp().as("detected_at")
      )
    
    // Detect abnormal amounts
    val largeTransactions = events
      .filter($"amount" > 9000)
      .select(
        $"user_id",
        $"amount",
        lit("LARGE_TRANSACTION").as("alert_type"),
        current_timestamp().as("detected_at")
      )
    
    // Combine alerts
    val alerts = highFreqUsers.union(largeTransactions)
    
    val alertQuery = alerts
      .writeStream
      .outputMode("update")
      .format("console")
      .option("truncate", "false")
      .option("checkpointLocation", "/tmp/checkpoints/alerts")
      .trigger(Trigger.ProcessingTime("10 seconds"))
      .start()
    
    alertQuery.awaitTermination(30000)
    alertQuery.stop()
    
    spark.stop()
  }
}
```

---

## Step 780: Complete Real-time Analytics System

```scala
// RealtimeAnalyticsSystem.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._

object RealtimeAnalyticsSystem {
  def main(args: Array[String]): Unit = {
    val spark = SparkKafkaSetup.createSparkSession()
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== Input: Simulated order stream =====
    val orders = spark.readStream
      .format("rate")
      .option("rowsPerSecond", 100)
      .load()
      .select(
        concat(lit("O"), $"value".cast("string")).as("order_id"),
        concat(lit("C"), ($"value" % 1000).cast("string")).as("customer_id"),
        concat(lit("P"), ($"value" % 100).cast("string")).as("product_id"),
        when($"value" % 4 === 0, "Electronics")
          .when($"value" % 4 === 1, "Clothing")
          .when($"value" % 4 === 2, "Food")
          .otherwise("Other").as("category"),
        when($"value" % 3 === 0, "North")
          .when($"value" % 3 === 1, "South")
          .otherwise("East").as("region"),
        ($"value" * 7.77 % 5000 + 100).as("amount"),
        $"timestamp"
      )
    
    // ===== Dashboard: Real-time KPIs =====
    val kpis = orders
      .withWatermark("timestamp", "1 minute")
      .groupBy(window($"timestamp", "1 minute"))
      .agg(
        count("*").as("total_orders"),
        sum("amount").as("total_revenue"),
        avg("amount").as("avg_order_value"),
        countDistinct("customer_id").as("unique_customers"),
        countDistinct("product_id").as("unique_products")
      )
      .withColumn("revenue_per_customer",
        $"total_revenue" / $"unique_customers")
    
    // ===== Product Performance =====
    val productPerf = orders
      .withWatermark("timestamp", "2 minutes")
      .groupBy(window($"timestamp", "5 minutes"), $"category")
      .agg(
        count("*").as("orders"),
        sum("amount").as("revenue"),
        avg("amount").as("avg_price")
      )
    
    // ===== Regional Breakdown =====
    val regional = orders
      .withWatermark("timestamp", "2 minutes")
      .groupBy(window($"timestamp", "5 minutes"), $"region")
      .agg(
        count("*").as("orders"),
        sum("amount").as("revenue")
      )
    
    // ===== Write Queries =====
    val kpiQuery = kpis
      .writeStream
      .outputMode("update")
      .format("memory")
      .queryName("kpis")
      .option("checkpointLocation", "/tmp/checkpoints/kpis")
      .trigger(Trigger.ProcessingTime("30 seconds"))
      .start()
    
    val productQuery = productPerf
      .writeStream
      .outputMode("update")
      .format("memory")
      .queryName("product_perf")
      .option("checkpointLocation", "/tmp/checkpoints/product-perf")
      .trigger(Trigger.ProcessingTime("30 seconds"))
      .start()
    
    val regionalQuery = regional
      .writeStream
      .outputMode("update")
      .format("memory")
      .queryName("regional")
      .option("checkpointLocation", "/tmp/checkpoints/regional")
      .trigger(Trigger.ProcessingTime("30 seconds"))
      .start()
    
    // ===== Dashboard Loop =====
    println("Real-time analytics started. Refreshing every 35 seconds...")
    
    for (i <- 1 to 3) {
      Thread.sleep(35000)
      
      println(s"\n=== DASHBOARD UPDATE ${i} ===")
      
      println("\n--- KPIs (last 1 minute) ---")
      spark.sql("""
        SELECT 
          window.start as period,
          total_orders,
          ROUND(total_revenue, 0) as revenue,
          ROUND(avg_order_value, 2) as avg_order,
          unique_customers
        FROM kpis
        ORDER BY period DESC
        LIMIT 3
      """).show(truncate = false)
      
      println("\n--- Category Performance (last 5 minutes) ---")
      spark.sql("""
        SELECT category, 
               SUM(orders) as orders, 
               ROUND(SUM(revenue), 0) as revenue
        FROM product_perf
        GROUP BY category
        ORDER BY revenue DESC
      """).show()
      
      println("\n--- Regional (last 5 minutes) ---")
      spark.sql("""
        SELECT region,
               SUM(orders) as orders,
               ROUND(SUM(revenue), 0) as revenue
        FROM regional
        GROUP BY region
        ORDER BY revenue DESC
      """).show()
    }
    
    kpiQuery.stop()
    productQuery.stop()
    regionalQuery.stop()
    
    spark.stop()
  }
}
```

---

## สรุป Part 78

| Component | Role | Configuration |
|-----------|------|---------------|
| Kafka Source | Input stream | bootstrap.servers, subscribe |
| Watermark | Handle late data | withWatermark(column, delay) |
| Window | Time-based grouping | window(time, size, slide) |
| Trigger | Micro-batch timing | ProcessingTime, Once |
| Checkpoint | Fault tolerance | checkpointLocation |
| OutputMode | How to write | append, update, complete |

### Kafka-Spark Integration Checklist
1. ✅ ใช้ `startingOffsets=latest` สำหรับ production
2. ✅ ตั้ง `checkpointLocation` เสมอ
3. ✅ ใช้ `withWatermark` ก่อน windowed aggregations
4. ✅ ตั้ง `maxOffsetsPerTrigger` เพื่อควบคุม rate
5. ✅ Monitor input/output rates ผ่าน StreamingQueryListener

---

## แบบฝึกหัด Part 78

1. **Multi-Topic Pipeline**: สร้าง Spark pipeline ที่อ่านจาก 3 topics พร้อมกัน (orders, payments, shipping) แล้ว join และ produce enriched events ไป Kafka

2. **Exactly-Once with Delta**: implement exactly-once pipeline ด้วย Spark Structured Streaming + Delta Lake (foreachBatch + merge)

3. **Late Data Analysis**: สร้าง stream ที่มี late events (จำลองด้วย timestamp เก่า) แล้วทดสอบว่า watermark handle late data ถูกต้อง

4. **Real-time Dashboard**: สร้าง complete real-time dashboard ที่แสดง metrics อัปเดตทุก 30 วินาที รวม anomaly detection

5. **Schema Evolution**: ทดสอบ schema evolution โดย produce messages ด้วย 2 versions ของ schema แล้ว Spark consumer handle ได้ทั้ง 2

---

## ไปต่อ: Part 79 — Data Pipelines
ใน Part ถัดไปจะเรียน ETL pipeline design, batch vs streaming, data quality, monitoring

[→ Part 79: Data Pipelines](./part-79-data-pipelines.md)
