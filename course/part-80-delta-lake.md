# Part 80: Delta Lake — Steps 791-800

## บทนำ: Delta Lake

Delta Lake คือ open-source storage layer ที่ brings ACID transactions, scalable metadata handling, และ unification of streaming and batch data processing มาสู่ Data Lake ของคุณ

---

## Step 791: Delta Lake Setup

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.apache.spark"  %% "spark-sql"   % "3.5.1" % "provided",
  "io.delta"          %% "delta-spark" % "3.1.0",
  "io.delta"          %% "delta-core"  % "3.1.0"
)
```

```scala
// DeltaLakeSetup.scala
import org.apache.spark.sql.SparkSession
import io.delta.tables._
import org.apache.spark.sql.functions._

object DeltaLakeSetup {
  
  def createSparkWithDelta(): SparkSession = {
    SparkSession.builder()
      .master("local[*]")
      .appName("Delta Lake Demo")
      .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
      .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
      .getOrCreate()
  }
  
  def main(args: Array[String]): Unit = {
    val spark = createSparkWithDelta()
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== Create Delta Table =====
    val orders = Seq(
      (1L, "C001", "Electronics", 999.99, "confirmed",  "2024-01-15"),
      (2L, "C002", "Clothing",    49.99,  "pending",    "2024-01-16"),
      (3L, "C003", "Food",        9.99,   "shipped",    "2024-01-17"),
      (4L, "C004", "Electronics", 1299.99,"delivered",  "2024-01-18"),
      (5L, "C005", "Clothing",    79.99,  "confirmed",  "2024-01-19")
    ).toDF("order_id", "customer_id", "category", "amount", "status", "order_date")
    
    // Write เป็น Delta format
    orders.write
      .format("delta")
      .mode("overwrite")
      .save("/tmp/delta/orders")
    
    println("Delta table created")
    
    // Read Delta table
    val deltaDF = spark.read.format("delta").load("/tmp/delta/orders")
    deltaDF.show()
    
    // Describe table
    deltaDF.printSchema()
    
    spark.stop()
  }
}
```

---

## Step 792: ACID Transactions

```scala
// DeltaACID.scala
import org.apache.spark.sql.SparkSession
import io.delta.tables._
import org.apache.spark.sql.functions._

object DeltaACID {
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Delta ACID")
      .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
      .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val tablePath = "/tmp/delta/orders-acid"
    
    // Initial data
    Seq(
      (1L, "C001", 100.0, "pending"),
      (2L, "C002", 200.0, "pending"),
      (3L, "C003", 300.0, "pending")
    ).toDF("order_id", "customer_id", "amount", "status")
     .write.format("delta").mode("overwrite").save(tablePath)
    
    val deltaTable = DeltaTable.forPath(spark, tablePath)
    
    println("=== Initial State ===")
    deltaTable.toDF.show()
    
    // ===== UPDATE Operation =====
    println("=== After UPDATE (status = confirmed for order 1 & 2) ===")
    deltaTable.update(
      condition = $"order_id" <= 2,
      set = Map("status" -> lit("confirmed"))
    )
    deltaTable.toDF.show()
    
    // ===== DELETE Operation =====
    println("=== After DELETE (remove order 3) ===")
    deltaTable.delete(condition = $"order_id" === 3)
    deltaTable.toDF.show()
    
    // ===== INSERT (Append) =====
    println("=== After INSERT new orders ===")
    Seq(
      (4L, "C004", 400.0, "pending"),
      (5L, "C005", 500.0, "shipped")
    ).toDF("order_id", "customer_id", "amount", "status")
     .write.format("delta").mode("append").save(tablePath)
    
    deltaTable.toDF.show()
    
    // ===== MERGE (Upsert) =====
    println("=== After MERGE ===")
    val updates = Seq(
      (1L, "C001", 150.0, "shipped"),    // update existing
      (6L, "C006", 600.0, "pending")     // insert new
    ).toDF("order_id", "customer_id", "amount", "status")
    
    deltaTable.as("target")
      .merge(
        updates.as("source"),
        "target.order_id = source.order_id"
      )
      .whenMatched.updateAll()
      .whenNotMatched.insertAll()
      .execute()
    
    deltaTable.toDF.orderBy("order_id").show()
    
    // ===== Transaction Log =====
    println("=== Transaction History ===")
    deltaTable.history().select("version", "timestamp", "operation", "operationMetrics").show(truncate = false)
    
    spark.stop()
  }
}
```

---

## Step 793: Time Travel

```scala
// DeltaTimeTravel.scala
import org.apache.spark.sql.SparkSession
import io.delta.tables._
import org.apache.spark.sql.functions._

object DeltaTimeTravel {
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Delta Time Travel")
      .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
      .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val tablePath = "/tmp/delta/orders-timetravel"
    
    // Version 0: initial data
    Seq(
      (1L, "C001", 100.0, "v1"),
      (2L, "C002", 200.0, "v1")
    ).toDF("order_id", "customer_id", "amount", "version_tag")
     .write.format("delta").mode("overwrite").save(tablePath)
    
    // Version 1: update
    DeltaTable.forPath(spark, tablePath).update(
      $"order_id" === 1,
      Map("amount" -> lit(150.0), "version_tag" -> lit("v2"))
    )
    
    // Version 2: add new records
    Seq(
      (3L, "C003", 300.0, "v3"),
      (4L, "C004", 400.0, "v3")
    ).toDF("order_id", "customer_id", "amount", "version_tag")
     .write.format("delta").mode("append").save(tablePath)
    
    // Version 3: delete
    DeltaTable.forPath(spark, tablePath).delete($"order_id" === 2)
    
    // ===== Read by Version Number =====
    println("=== Current State (latest) ===")
    spark.read.format("delta").load(tablePath).orderBy("order_id").show()
    
    println("=== Version 0 (initial) ===")
    spark.read.format("delta").option("versionAsOf", 0).load(tablePath).show()
    
    println("=== Version 1 (after first update) ===")
    spark.read.format("delta").option("versionAsOf", 1).load(tablePath).show()
    
    println("=== Version 2 (after insert) ===")
    spark.read.format("delta").option("versionAsOf", 2).load(tablePath).show()
    
    // ===== Read by Timestamp =====
    // (ใช้ timestamp จริงได้ใน production)
    // spark.read.format("delta")
    //   .option("timestampAsOf", "2024-01-15 10:00:00")
    //   .load(tablePath)
    
    // ===== History =====
    println("=== Full Transaction History ===")
    val deltaTable = DeltaTable.forPath(spark, tablePath)
    deltaTable.history().show(truncate = false)
    
    // ===== Restore to Previous Version =====
    println("=== Restoring to Version 1 ===")
    deltaTable.restoreToVersion(1)
    spark.read.format("delta").load(tablePath).show()
    
    // ===== Compare Versions =====
    println("=== Records added between v0 and v2 ===")
    val v0 = spark.read.format("delta").option("versionAsOf", 0).load(tablePath)
    val v2 = spark.read.format("delta").option("versionAsOf", 2).load(tablePath)
    
    v2.join(v0, Seq("order_id"), "left_anti").show()
    
    spark.stop()
  }
}
```

---

## Step 794: Schema Evolution

```scala
// DeltaSchemaEvolution.scala
import org.apache.spark.sql.SparkSession
import io.delta.tables._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._

object DeltaSchemaEvolution {
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Delta Schema Evolution")
      .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
      .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val tablePath = "/tmp/delta/orders-schema"
    
    // Initial schema
    Seq(
      (1L, "C001", 100.0),
      (2L, "C002", 200.0)
    ).toDF("order_id", "customer_id", "amount")
     .write.format("delta").mode("overwrite").save(tablePath)
    
    println("=== Original Schema ===")
    spark.read.format("delta").load(tablePath).printSchema()
    
    // ===== Add New Column (Schema Evolution) =====
    // ต้องใช้ mergeSchema option
    Seq(
      (3L, "C003", 300.0, "Electronics"),
      (4L, "C004", 400.0, "Clothing")
    ).toDF("order_id", "customer_id", "amount", "category")
     .write
     .format("delta")
     .mode("append")
     .option("mergeSchema", "true")  // สำคัญ!
     .save(tablePath)
    
    println("=== After Adding 'category' Column ===")
    val evolved = spark.read.format("delta").load(tablePath)
    evolved.printSchema()
    evolved.show()
    
    // ===== Column Mapping (Rename/Drop) =====
    // ใช้ Delta Column Mapping feature
    spark.sql(s"""
      ALTER TABLE delta.`$tablePath`
      SET TBLPROPERTIES (
        'delta.columnMapping.mode' = 'name',
        'delta.minReaderVersion' = '2',
        'delta.minWriterVersion' = '5'
      )
    """)
    
    // ตอนนี้ rename column ได้
    // spark.sql(s"ALTER TABLE delta.`$tablePath` RENAME COLUMN amount TO total_amount")
    
    // ===== Schema Enforcement (Default) =====
    println("\n=== Schema Enforcement Example ===")
    println("Trying to write incompatible schema...")
    try {
      Seq(
        ("WRONG_TYPE", "C005", "not-a-number") // amount เป็น String แทน Double
      ).toDF("order_id", "customer_id", "amount")
       .write.format("delta").mode("append").save(tablePath)
      println("Should have failed!")
    } catch {
      case e: Exception =>
        println(s"Correctly rejected: ${e.getMessage.take(100)}")
    }
    
    // ===== Explicit Schema =====
    val explicitSchema = StructType(Seq(
      StructField("order_id",    LongType,   nullable = false),
      StructField("customer_id", StringType, nullable = false),
      StructField("amount",      DoubleType, nullable = true),
      StructField("category",    StringType, nullable = true),
      StructField("discount",    DoubleType, nullable = true) // new optional field
    ))
    
    println("\n=== Writing with explicit schema (mergeSchema) ===")
    spark.createDataFrame(
      spark.sparkContext.parallelize(Seq(
        org.apache.spark.sql.Row(5L, "C005", 500.0, "Electronics", 50.0)
      )),
      explicitSchema
    ).write.format("delta").mode("append").option("mergeSchema", "true").save(tablePath)
    
    spark.read.format("delta").load(tablePath).show()
    
    spark.stop()
  }
}
```

---

## Step 795: Merge และ Upsert

```scala
// DeltaMerge.scala
import org.apache.spark.sql.SparkSession
import io.delta.tables._
import org.apache.spark.sql.functions._

object DeltaMerge {
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Delta Merge")
      .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
      .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val tablePath = "/tmp/delta/customers"
    
    // Initial customers
    Seq(
      (1L, "Alice",   "alice@example.com",   "active",   1000.0),
      (2L, "Bob",     "bob@example.com",     "active",   2000.0),
      (3L, "Charlie", "charlie@example.com", "inactive", 0.0)
    ).toDF("customer_id", "name", "email", "status", "total_spent")
     .write.format("delta").mode("overwrite").save(tablePath)
    
    val deltaTable = DeltaTable.forPath(spark, tablePath)
    
    println("=== Initial Customers ===")
    deltaTable.toDF.show()
    
    // ===== Basic MERGE (upsert) =====
    val updates = Seq(
      (1L, "Alice Updated", "alice.new@example.com", "active",   1500.0), // update
      (2L, "Bob",           "bob@example.com",       "inactive", 2000.0), // update status
      (4L, "Diana",         "diana@example.com",     "active",   500.0)   // insert new
    ).toDF("customer_id", "name", "email", "status", "total_spent")
    
    deltaTable.as("target")
      .merge(updates.as("source"), "target.customer_id = source.customer_id")
      .whenMatched.updateAll()
      .whenNotMatched.insertAll()
      .execute()
    
    println("=== After Basic MERGE ===")
    deltaTable.toDF.show()
    
    // ===== MERGE with Conditional Logic =====
    val conditionalUpdates = Seq(
      (1L, 200.0),    // add to total_spent
      (2L, 500.0),    // add to total_spent
      (5L, 100.0)     // new customer - don't insert (threshold check)
    ).toDF("customer_id", "purchase_amount")
    
    deltaTable.as("target")
      .merge(conditionalUpdates.as("source"), "target.customer_id = source.customer_id")
      .whenMatched(
        // update เฉพาะ active customers
        $"target.status" === "active"
      ).update(Map(
        "total_spent" -> ($"target.total_spent" + $"source.purchase_amount")
      ))
      .whenNotMatched(
        // insert เฉพาะ เมื่อ purchase_amount > 150
        $"source.purchase_amount" > 150
      ).insert(Map(
        "customer_id"  -> $"source.customer_id",
        "name"         -> lit("Unknown"),
        "email"        -> lit("unknown@example.com"),
        "status"       -> lit("new"),
        "total_spent"  -> $"source.purchase_amount"
      ))
      .execute()
    
    println("=== After Conditional MERGE ===")
    deltaTable.toDF.orderBy("customer_id").show()
    
    // ===== SCD Type 2 (Slowly Changing Dimension) =====
    println("\n=== SCD Type 2 Example ===")
    
    val scdPath = "/tmp/delta/customers-scd2"
    
    // SCD2 table with effective dates
    Seq(
      (1L, "Alice", "alice@old.com", "2023-01-01", "9999-12-31", true),
      (2L, "Bob",   "bob@old.com",   "2023-01-01", "9999-12-31", true)
    ).toDF("customer_id", "name", "email", "eff_from", "eff_to", "is_current")
     .write.format("delta").mode("overwrite").save(scdPath)
    
    val scdTable = DeltaTable.forPath(spark, scdPath)
    
    // New update for customer 1 (email changed)
    val changedCustomers = Seq(
      (1L, "Alice", "alice@new.com")
    ).toDF("customer_id", "name", "email")
    
    // SCD2 merge: expire old record and insert new
    scdTable.as("target")
      .merge(changedCustomers.as("source"), 
             "target.customer_id = source.customer_id AND target.is_current = true")
      .whenMatched(
        // expire old record when email changes
        $"target.email" =!= $"source.email"
      ).update(Map(
        "eff_to"     -> lit("2024-01-15"),
        "is_current" -> lit(false)
      ))
      .execute()
    
    // Insert new current record
    changedCustomers
      .filter($"customer_id" === 1)
      .withColumn("eff_from",   lit("2024-01-16"))
      .withColumn("eff_to",     lit("9999-12-31"))
      .withColumn("is_current", lit(true))
      .write.format("delta").mode("append").save(scdPath)
    
    println("SCD2 Result:")
    spark.read.format("delta").load(scdPath).orderBy("customer_id", "eff_from").show()
    
    spark.stop()
  }
}
```

---

## Step 796: Optimize และ Z-Order

```scala
// DeltaOptimize.scala
import org.apache.spark.sql.SparkSession
import io.delta.tables._
import org.apache.spark.sql.functions._

object DeltaOptimize {
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Delta Optimize")
      .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
      .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val tablePath = "/tmp/delta/orders-optimize"
    
    // สร้าง small files จำนวนมาก (small file problem)
    for (i <- 1 to 20) {
      Seq(
        (i * 10L, s"C00$i", (i * 100).toDouble, "Electronics", s"2024-01-${"0" + i takeRight 2}")
      ).toDF("order_id", "customer_id", "amount", "category", "order_date")
       .write.format("delta").mode("append").save(tablePath)
    }
    
    println("=== Before OPTIMIZE ===")
    println(s"Files before: ${new java.io.File(tablePath).listFiles().count(_.getName.endsWith(".parquet"))}")
    
    // ===== OPTIMIZE: compact small files =====
    val deltaTable = DeltaTable.forPath(spark, tablePath)
    deltaTable.optimize().executeCompaction()
    
    println("=== After OPTIMIZE ===")
    println(s"Files after compaction")
    
    // ===== Z-ORDER: co-locate related data =====
    // Z-ORDER by columns ที่ใช้ filter บ่อย
    deltaTable.optimize().executeZOrderBy("category", "customer_id")
    
    println("Z-ORDER by category, customer_id done")
    
    // ===== VACUUM: clean up old files =====
    // default retention = 7 days
    // CAUTION: ถ้า vacuum แล้วจะ time travel ย้อนหลังไม่ได้
    // deltaTable.vacuum()  // use default retention
    // deltaTable.vacuum(24) // 24 hours retention
    
    // For demo, disable retention check
    spark.conf.set("spark.databricks.delta.retentionDurationCheck.enabled", "false")
    deltaTable.vacuum(0) // ลบทันที (ห้ามทำใน production!)
    
    println("VACUUM completed")
    
    // ===== Query Performance with Z-Order =====
    // Spark จะ skip files ที่ไม่มี data เราต้องการ
    println("=== Filtered Query (benefits from Z-ORDER) ===")
    val result = spark.read.format("delta").load(tablePath)
      .filter($"category" === "Electronics")
      .filter($"customer_id" === "C001")
    
    result.show()
    
    // ===== Partition Pruning vs Z-Order =====
    /*
    Partitioning: ดีสำหรับ low cardinality (year, month, country)
    Z-Order: ดีสำหรับ high cardinality (customer_id, product_id)
    
    ใช้ร่วมกัน:
    - Partition by: date (year, month)
    - Z-Order by: customer_id, category
    */
    
    spark.stop()
  }
}
```

---

## Step 797: Delta Streaming

```scala
// DeltaStreaming.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._
import org.apache.spark.sql.streaming.Trigger
import io.delta.tables._

object DeltaStreaming {
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Delta Streaming")
      .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
      .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val sourcePath = "/tmp/delta/orders-source"
    val sinkPath   = "/tmp/delta/orders-aggregated"
    val checkPath  = "/tmp/delta/checkpoint"
    
    // Create source Delta table
    Seq(
      (1L, "Electronics", 999.99),
      (2L, "Clothing",    49.99),
      (3L, "Food",        9.99)
    ).toDF("order_id", "category", "amount")
     .write.format("delta").mode("overwrite").save(sourcePath)
    
    // ===== Read Stream from Delta =====
    val streamingDF = spark.readStream
      .format("delta")
      .load(sourcePath)
    
    // ===== Process and Write Stream to Delta =====
    val aggregated = streamingDF
      .withWatermark("timestamp", "5 minutes")  // ใน Delta ไม่ต้องมี timestamp
      .groupBy($"category")
      .agg(
        count("*").as("order_count"),
        sum("amount").as("total_revenue"),
        avg("amount").as("avg_order_value")
      )
    
    val query = aggregated.writeStream
      .format("delta")
      .outputMode("complete")
      .option("checkpointLocation", checkPath)
      .trigger(Trigger.ProcessingTime("10 seconds"))
      .start(sinkPath)
    
    // Add new data to source (triggers streaming update)
    Thread.sleep(5000)
    Seq(
      (4L, "Electronics", 1499.99),
      (5L, "Electronics", 299.99),
      (6L, "Food",        15.99)
    ).toDF("order_id", "category", "amount")
     .write.format("delta").mode("append").save(sourcePath)
    
    // Wait for processing
    Thread.sleep(15000)
    
    println("=== Streaming Results ===")
    spark.read.format("delta").load(sinkPath).show()
    
    query.stop()
    
    // ===== Change Data Feed (CDF) =====
    println("\n=== Change Data Feed ===")
    
    val cdfPath = "/tmp/delta/orders-cdf"
    
    // Enable CDF
    Seq(
      (1L, "C001", 100.0, "pending")
    ).toDF("order_id", "customer_id", "amount", "status")
     .write.format("delta")
     .option("delta.enableChangeDataFeed", "true")
     .mode("overwrite")
     .save(cdfPath)
    
    val cdfTable = DeltaTable.forPath(spark, cdfPath)
    
    // Make changes
    cdfTable.update($"order_id" === 1, Map("status" -> lit("confirmed")))
    
    Seq((2L, "C002", 200.0, "pending"))
      .toDF("order_id", "customer_id", "amount", "status")
      .write.format("delta").mode("append").save(cdfPath)
    
    cdfTable.delete($"order_id" === 1)
    
    // Read CDC changes
    println("All changes since version 0:")
    spark.read.format("delta")
      .option("readChangeFeed", "true")
      .option("startingVersion", 0)
      .load(cdfPath)
      .show(truncate = false)
    
    spark.stop()
  }
}
```

---

## Step 798: Delta Constraints

```scala
// DeltaConstraints.scala
import org.apache.spark.sql.SparkSession
import io.delta.tables._
import org.apache.spark.sql.functions._

object DeltaConstraints {
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Delta Constraints")
      .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
      .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val tablePath = "/tmp/delta/orders-constraints"
    
    // Create table
    Seq(
      (1L, "C001", 100.0, "pending")
    ).toDF("order_id", "customer_id", "amount", "status")
     .write.format("delta").mode("overwrite").save(tablePath)
    
    // ===== Add Check Constraints =====
    spark.sql(s"""
      ALTER TABLE delta.`$tablePath`
      ADD CONSTRAINT amount_positive CHECK (amount > 0)
    """)
    
    spark.sql(s"""
      ALTER TABLE delta.`$tablePath`
      ADD CONSTRAINT valid_status CHECK (
        status IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled')
      )
    """)
    
    println("=== Constraints Added ===")
    
    // Valid data — passes
    println("Writing valid data...")
    Seq((2L, "C002", 200.0, "confirmed"))
      .toDF("order_id", "customer_id", "amount", "status")
      .write.format("delta").mode("append").save(tablePath)
    
    println("Success!")
    
    // Invalid amount — fails
    println("\nWriting invalid amount (negative)...")
    try {
      Seq((3L, "C003", -50.0, "pending"))
        .toDF("order_id", "customer_id", "amount", "status")
        .write.format("delta").mode("append").save(tablePath)
      println("Should have failed!")
    } catch {
      case e: Exception => println(s"Correctly rejected: ${e.getMessage.take(150)}")
    }
    
    // Invalid status — fails
    println("\nWriting invalid status...")
    try {
      Seq((4L, "C004", 50.0, "INVALID_STATUS"))
        .toDF("order_id", "customer_id", "amount", "status")
        .write.format("delta").mode("append").save(tablePath)
      println("Should have failed!")
    } catch {
      case e: Exception => println(s"Correctly rejected: ${e.getMessage.take(150)}")
    }
    
    // ===== NOT NULL Constraints =====
    spark.sql(s"""
      ALTER TABLE delta.`$tablePath`
      CHANGE COLUMN order_id SET NOT NULL
    """)
    
    println("\n=== Table Properties ===")
    spark.sql(s"DESCRIBE DETAIL delta.`$tablePath`").show(truncate = false)
    
    // ===== Remove Constraint =====
    spark.sql(s"""
      ALTER TABLE delta.`$tablePath`
      DROP CONSTRAINT amount_positive
    """)
    
    println("Constraint removed")
    
    spark.stop()
  }
}
```

---

## Step 799: Delta Performance Tuning

```scala
// DeltaPerformance.scala
import org.apache.spark.sql.SparkSession
import io.delta.tables._
import org.apache.spark.sql.functions._

object DeltaPerformance {
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Delta Performance")
      .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
      .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
      // Performance configs
      .config("spark.sql.shuffle.partitions", "50")
      .config("spark.delta.merge.optimizeMatchedFiles", "true")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val tablePath = "/tmp/delta/orders-perf"
    
    // ===== Partitioned Delta Table =====
    // Partition by date สำหรับ time-series queries
    spark.range(10000)
      .withColumn("order_date",  date_add(lit("2024-01-01").cast("date"), ($"id" % 90).cast("int")))
      .withColumn("category",    when($"id" % 3 === 0, "Electronics")
                                  .when($"id" % 3 === 1, "Clothing")
                                  .otherwise("Food"))
      .withColumn("customer_id", concat(lit("C"), ($"id" % 100).cast("string")))
      .withColumn("amount",      ($"id" % 1000 + 1).cast("double"))
      .withColumn("year",        year($"order_date"))
      .withColumn("month",       month($"order_date"))
      .write
      .format("delta")
      .partitionBy("year", "month")
      .mode("overwrite")
      .save(tablePath)
    
    println("=== Partitioned Delta Table Created ===")
    
    // ===== Partition Pruning =====
    println("\n=== Query with Partition Pruning ===")
    val t1 = System.currentTimeMillis()
    val result1 = spark.read.format("delta").load(tablePath)
      .filter($"year" === 2024 && $"month" === 3)
      .agg(count("*"), sum("amount"))
      .first()
    println(s"Time: ${System.currentTimeMillis() - t1}ms, Result: $result1")
    
    // ===== Data Skipping (Z-Order) =====
    println("\n=== Adding Z-ORDER index ===")
    val deltaTable = DeltaTable.forPath(spark, tablePath)
    // Z-order within partitions
    deltaTable.optimize().where("year = 2024 AND month = 1")
              .executeZOrderBy("category", "customer_id")
    
    // ===== Bloom Filter Index =====
    spark.sql(s"""
      ALTER TABLE delta.`$tablePath`
      SET TBLPROPERTIES (
        'delta.dataSkippingNumIndexedCols' = 10,
        'delta.bloomFilter.customer_id.enabled' = 'true',
        'delta.bloomFilter.customer_id.fpp' = '0.1'
      )
    """)
    
    println("Bloom filter index set")
    
    // ===== Cache Delta Table in Memory =====
    val cachedDF = spark.read.format("delta").load(tablePath).cache()
    cachedDF.count() // trigger caching
    
    val t2 = System.currentTimeMillis()
    cachedDF.filter($"category" === "Electronics").count()
    println(s"Cached query time: ${System.currentTimeMillis() - t2}ms")
    
    // ===== Statistics =====
    println("\n=== Table Statistics ===")
    spark.sql(s"DESCRIBE DETAIL delta.`$tablePath`")
      .select("numFiles", "sizeInBytes", "partitionColumns")
      .show(truncate = false)
    
    // ===== Optimize for Merge Performance =====
    /*
    Tips สำหรับ Merge performance:
    1. Partition target table appropriately
    2. Z-order by merge key columns
    3. Use merge predicate on partition columns
    4. Enable merge optimizations:
       spark.conf.set("spark.databricks.delta.merge.optimizeMatchedFiles", "true")
    */
    
    cachedDF.unpersist()
    spark.stop()
  }
}
```

---

## Step 800: Delta Production Architecture

```scala
// DeltaProductionArchitecture.scala
import org.apache.spark.sql.{SparkSession, DataFrame}
import io.delta.tables._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._

// Medallion Architecture: Bronze -> Silver -> Gold
object DeltaProductionArchitecture {
  
  val bronzePath = "/tmp/delta/bronze/orders"
  val silverPath = "/tmp/delta/silver/orders"
  val goldPath   = "/tmp/delta/gold/order-aggregates"
  
  // ===== Bronze Layer: Raw ingestion =====
  def ingestToBronze(spark: SparkSession, rawData: DataFrame): Unit = {
    println("=== Ingesting to Bronze ===")
    
    // Bronze: store raw data as-is with metadata
    rawData
      .withColumn("_ingested_at",   current_timestamp())
      .withColumn("_source",        lit("orders-api"))
      .withColumn("_batch_id",      lit(java.util.UUID.randomUUID().toString))
      .write
      .format("delta")
      .mode("append")
      .option("mergeSchema", "true") // Bronze ยืดหยุ่น
      .save(bronzePath)
    
    println(s"Bronze records: ${spark.read.format("delta").load(bronzePath).count()}")
  }
  
  // ===== Silver Layer: Cleaned & validated =====
  def processSilver(spark: SparkSession): Unit = {
    import spark.implicits._
    
    println("=== Processing to Silver ===")
    
    val bronze = spark.read.format("delta").load(bronzePath)
    
    // Clean and validate
    val silver = bronze
      .filter($"order_id".isNotNull)
      .filter($"amount" > 0)
      .filter($"status".isin("pending", "confirmed", "shipped", "delivered", "cancelled"))
      .withColumn("amount_usd",
        when($"currency" === "THB", $"amount" / 36.0)
          .otherwise($"amount"))
      .withColumn("order_date_parsed", $"order_date".cast("date"))
      .drop("_batch_id") // remove Bronze metadata
      .withColumn("_processed_at", current_timestamp())
    
    // Upsert to Silver (avoid duplicates)
    if (DeltaTable.isDeltaTable(spark, silverPath)) {
      val silverTable = DeltaTable.forPath(spark, silverPath)
      
      silverTable.as("target")
        .merge(silver.as("source"), "target.order_id = source.order_id")
        .whenMatched.updateAll()
        .whenNotMatched.insertAll()
        .execute()
    } else {
      silver.write.format("delta").mode("overwrite").save(silverPath)
    }
    
    println(s"Silver records: ${spark.read.format("delta").load(silverPath).count()}")
  }
  
  // ===== Gold Layer: Business aggregates =====
  def processGold(spark: SparkSession): Unit = {
    import spark.implicits._
    
    println("=== Processing to Gold ===")
    
    val silver = spark.read.format("delta").load(silverPath)
    
    val gold = silver
      .withColumn("order_year",  year($"order_date_parsed"))
      .withColumn("order_month", month($"order_date_parsed"))
      .groupBy($"order_year", $"order_month", $"category", $"region")
      .agg(
        count("*").as("total_orders"),
        sum("amount_usd").as("total_revenue_usd"),
        avg("amount_usd").as("avg_order_value"),
        countDistinct("customer_id").as("unique_customers"),
        sum(when($"status" === "delivered", 1).otherwise(0)).as("fulfilled_count"),
        sum(when($"status" === "cancelled", 1).otherwise(0)).as("cancelled_count")
      )
      .withColumn("fulfillment_rate",
        round($"fulfilled_count" / $"total_orders" * 100, 2))
      .withColumn("_updated_at", current_timestamp())
    
    // Overwrite Gold partition
    gold.write
      .format("delta")
      .mode("overwrite")
      .option("replaceWhere", "order_year = 2024 AND order_month = 1")
      .save(goldPath)
    
    println("=== Gold Layer Results ===")
    spark.read.format("delta").load(goldPath).show()
  }
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Delta Medallion Architecture")
      .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
      .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // Raw data (เหมือนมาจาก Kafka หรือ API)
    val rawOrders = Seq(
      (1L, "C001", "Electronics", 999.99,  "confirmed", "USD", "2024-01-15", "North"),
      (2L, "C002", "Clothing",    49.99,   "pending",   "USD", "2024-01-15", "South"),
      (3L, "C003", "Food",        370.0,   "shipped",   "THB", "2024-01-16", "North"),
      (4L, null,   "Electronics", -100.0,  "invalid",   "USD", "2024-01-17", "East"),  // bad record
      (5L, "C005", "Clothing",    79.99,   "delivered", "USD", "2024-01-17", "West")
    ).toDF("order_id", "customer_id", "category", "amount", "status", "currency", "order_date", "region")
    
    // Run pipeline
    ingestToBronze(spark, rawOrders)
    processSilver(spark)
    processGold(spark)
    
    // ===== Cross-layer validation =====
    println("\n=== Layer Record Counts ===")
    println(s"Bronze: ${spark.read.format("delta").load(bronzePath).count()}")
    println(s"Silver: ${spark.read.format("delta").load(silverPath).count()}")
    println(s"Gold:   ${spark.read.format("delta").load(goldPath).count()}")
    
    // ===== Transaction Histories =====
    println("\n=== Silver History ===")
    DeltaTable.forPath(spark, silverPath).history(5)
      .select("version", "timestamp", "operation")
      .show(truncate = false)
    
    spark.stop()
  }
}
```

---

## สรุป Part 80: Delta Lake

| Feature | Description | Use Case |
|---------|-------------|----------|
| ACID | Atomic transactions | Concurrent writes |
| Time Travel | Read historical versions | Audit, rollback |
| Schema Evolution | Add columns seamlessly | Schema changes |
| MERGE | Upsert operations | CDC, SCD |
| OPTIMIZE | Compact small files | Performance |
| Z-ORDER | Co-locate related data | Query speed |
| CDF | Track row-level changes | Streaming CDC |
| Constraints | Data quality checks | Data integrity |
| Medallion | Bronze→Silver→Gold | Data lake org. |

### Delta vs Traditional Data Warehouse

| Aspect | Delta Lake | Traditional DW |
|--------|-----------|----------------|
| Cost | Pay for storage | Pay for compute+storage |
| Scale | Petabytes | Terabytes |
| Format | Open (Parquet) | Proprietary |
| Schema | Flexible | Rigid |
| Streaming | Native | ETL only |

---

## แบบฝึกหัด Part 80

1. **Time Travel Audit**: สร้าง Delta table สำหรับ financial transactions ที่ track ทุก change และ implement audit trail ที่สามารถ query state at any point in time

2. **SCD Type 2**: implement full Slowly Changing Dimension Type 2 ด้วย Delta merge โดยมี effective dates, current flags, และ history preservation

3. **Medallion Architecture**: สร้าง complete Medallion pipeline (Bronze→Silver→Gold) ที่ process clickstream data โดย Gold layer มี user behavior aggregates

4. **Delta Constraints**: implement data quality constraints บน Delta table ที่ cover not-null, range checks, และ referential integrity patterns

5. **CDC Stream**: ใช้ Change Data Feed อ่าน changes จาก Delta table เป็น stream และ sync ไปยัง downstream systems

---

## ไปต่อ: Part 81 — Microservices Design
ใน Part ถัดไปจะเรียน Microservices patterns, API design, และ service decomposition

[→ Part 81: Microservices Design](./part-81-microservices-design.md)
