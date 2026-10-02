# Part 79: Data Pipelines — Steps 781-790

## บทนำ: Data Pipeline Design

Data pipeline คือ series of data processing steps ที่ transform raw data เป็น actionable insights การออกแบบที่ดีทำให้ reliable, scalable และ maintainable

---

## Step 781: ETL Pipeline Design Patterns

```scala
// ETLPipelineDesign.scala
import org.apache.spark.sql.{SparkSession, DataFrame, Dataset}
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._
import scala.util.{Try, Success, Failure}

// Pipeline traits
trait DataSource[T] {
  def read(): Dataset[T]
}

trait DataTransformer[A, B] {
  def transform(input: Dataset[A]): Dataset[B]
}

trait DataSink[T] {
  def write(data: Dataset[T]): Unit
}

// Validation results
case class ValidationResult(
  isValid: Boolean,
  errorCount: Long,
  errors: List[String]
)

// Pipeline config
case class PipelineConfig(
  inputPath: String,
  outputPath: String,
  checkpointPath: String,
  batchDate: String,
  env: String
)

object ETLPipelineDesign {
  
  // ===== Pipeline Stage เป็น Pure Functions =====
  
  def extractOrders(spark: SparkSession, config: PipelineConfig): DataFrame = {
    import spark.implicits._
    
    spark.read
      .option("header", "true")
      .option("inferSchema", "true")
      .csv(config.inputPath)
      .withColumn("batch_date", lit(config.batchDate))
      .withColumn("ingested_at", current_timestamp())
  }
  
  def validateOrders(df: DataFrame): (DataFrame, DataFrame) = {
    import df.sparkSession.implicits._
    
    // Rules
    val validationRules = Map(
      "missing_order_id"    -> $"order_id".isNull,
      "missing_customer_id" -> $"customer_id".isNull,
      "invalid_amount"      -> ($"amount".isNull || $"amount" <= 0),
      "future_date"         -> ($"order_date" > current_date()),
      "invalid_status"      -> !$"status".isin("pending", "confirmed", "shipped", "delivered", "cancelled")
    )
    
    // Mark validation errors
    var validatedDF = df
    for ((ruleName, condition) <- validationRules) {
      validatedDF = validatedDF.withColumn(s"err_$ruleName", condition)
    }
    
    // Split valid/invalid
    val errorColumns = validationRules.keys.map(k => col(s"err_$k")).toSeq
    val hasError = errorColumns.reduce(_ || _)
    
    val valid   = validatedDF.filter(!hasError).drop(validationRules.keys.map(k => s"err_$k").toSeq: _*)
    val invalid = validatedDF.filter(hasError)
    
    (valid, invalid)
  }
  
  def transformOrders(df: DataFrame): DataFrame = {
    import df.sparkSession.implicits._
    
    df.withColumn("revenue", $"quantity" * $"unit_price")
      .withColumn("revenue_usd",
        when($"currency" === "THB", $"revenue" / 36.0)
          .when($"currency" === "JPY", $"revenue" / 150.0)
          .otherwise($"revenue"))
      .withColumn("order_year",  year($"order_date"))
      .withColumn("order_month", month($"order_date"))
      .withColumn("order_day",   dayofmonth($"order_date"))
      .withColumn("is_weekend",
        dayofweek($"order_date").isin(1, 7))
      .withColumn("price_tier",
        when($"unit_price" < 100, "budget")
          .when($"unit_price" < 1000, "mid-range")
          .otherwise("premium"))
  }
  
  def aggregateOrders(df: DataFrame): DataFrame = {
    import df.sparkSession.implicits._
    
    df.groupBy($"order_year", $"order_month", $"category", $"region")
      .agg(
        count("*").as("order_count"),
        sum("revenue_usd").as("total_revenue_usd"),
        avg("revenue_usd").as("avg_order_value"),
        countDistinct("customer_id").as("unique_customers"),
        sum(when($"status" === "delivered", 1).otherwise(0)).as("fulfilled_orders"),
        sum(when($"status" === "cancelled", 1).otherwise(0)).as("cancelled_orders")
      )
      .withColumn("fulfillment_rate",
        round($"fulfilled_orders" / $"order_count" * 100, 2))
      .withColumn("cancellation_rate",
        round($"cancelled_orders" / $"order_count" * 100, 2))
  }
  
  def loadToWarehouse(df: DataFrame, config: PipelineConfig): Unit = {
    df.write
      .mode("overwrite")
      .partitionBy("order_year", "order_month")
      .parquet(s"${config.outputPath}/order_aggregates")
  }
  
  // ===== Pipeline Orchestration =====
  def run(spark: SparkSession, config: PipelineConfig): Unit = {
    println(s"=== ETL Pipeline Starting: ${config.batchDate} ===")
    val startTime = System.currentTimeMillis()
    
    // Extract
    println("Step 1: Extracting data...")
    val raw = extractOrders(spark, config)
    println(s"  Records extracted: ${raw.count()}")
    
    // Validate
    println("Step 2: Validating data...")
    val (valid, invalid) = validateOrders(raw)
    println(s"  Valid records: ${valid.count()}")
    println(s"  Invalid records: ${invalid.count()}")
    
    // Write invalid records for review
    if (invalid.count() > 0) {
      invalid.write.mode("overwrite")
             .json(s"${config.outputPath}/invalid_records/${config.batchDate}")
    }
    
    // Transform
    println("Step 3: Transforming data...")
    val transformed = transformOrders(valid)
    
    // Aggregate
    println("Step 4: Aggregating data...")
    val aggregated = aggregateOrders(transformed)
    
    // Load
    println("Step 5: Loading to warehouse...")
    loadToWarehouse(aggregated, config)
    
    val elapsed = (System.currentTimeMillis() - startTime) / 1000.0
    println(s"=== Pipeline completed in ${elapsed}s ===")
  }
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("ETL Pipeline")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    
    val config = PipelineConfig(
      inputPath      = "/tmp/orders-input",
      outputPath     = "/tmp/orders-output",
      checkpointPath = "/tmp/orders-checkpoint",
      batchDate      = "2024-01-15",
      env            = "dev"
    )
    
    Try(run(spark, config)) match {
      case Success(_) => println("Pipeline succeeded")
      case Failure(ex) =>
        println(s"Pipeline FAILED: ${ex.getMessage}")
        ex.printStackTrace()
        System.exit(1)
    }
    
    spark.stop()
  }
}
```

---

## Step 782: Batch vs Streaming

```scala
// BatchVsStreaming.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._
import org.apache.spark.sql.streaming._

object BatchVsStreaming {
  
  /*
  เมื่อไหร่ควรใช้ Batch vs Streaming:
  
  Batch Processing:
  ✓ ข้อมูลมาครั้งเดียว (daily, hourly)
  ✓ ต้องการ complete dataset (historical analysis)
  ✓ Complex aggregations
  ✓ Machine learning training
  ✗ Latency > minutes is acceptable
  
  Streaming Processing:
  ✓ Real-time decisions needed
  ✓ Continuous data flow
  ✓ Event-driven actions
  ✓ Low latency required (seconds)
  ✗ Stateless or simple stateful operations
  
  Hybrid (Lambda Architecture):
  ✓ ต้องการทั้ง real-time และ historical
  Speed Layer:  Streaming (last few hours)
  Batch Layer:  Full historical (daily batch)
  Serving Layer: Query combines both
  
  Kappa Architecture:
  ✓ ใช้ streaming เท่านั้น
  Stream: handles all data
  Replay: reprocess from Kafka if needed
  */
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Batch vs Streaming")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== Batch: Daily Sales Report =====
    def runDailyBatch(date: String): Unit = {
      println(s"Running batch for $date")
      
      // อ่านข้อมูลทั้งวัน
      val dailyOrders = spark.read
        .parquet(s"/tmp/orders/date=$date")
        .cache()
      
      // Complex analysis ที่ต้องการ full dataset
      val dailyReport = dailyOrders
        .groupBy($"category", $"region")
        .agg(
          count("*").as("orders"),
          sum("amount").as("revenue"),
          percentile_approx($"amount", 0.5).as("median_amount")
        )
      
      dailyReport.write
        .mode("overwrite")
        .parquet(s"/tmp/daily-reports/$date")
      
      println(s"Batch completed for $date")
      dailyOrders.unpersist()
    }
    
    // ===== Streaming: Real-time Dashboard =====
    def startRealtimeDashboard(): StreamingQuery = {
      spark.readStream
        .format("rate")
        .option("rowsPerSecond", 10)
        .load()
        .withColumn("amount", rand() * 1000)
        .withColumn("category",
          when($"value" % 3 === 0, "Electronics")
            .otherwise("Other"))
        .withWatermark("timestamp", "1 minute")
        .groupBy(window($"timestamp", "5 minutes"), $"category")
        .agg(count("*").as("orders"), sum("amount").as("revenue"))
        .writeStream
        .outputMode("update")
        .format("memory")
        .queryName("realtime_dashboard")
        .trigger(Trigger.ProcessingTime("30 seconds"))
        .start()
    }
    
    // ===== Hybrid: Lambda Pattern =====
    def hybridQuery(spark: SparkSession): Unit = {
      // Load batch results (pre-computed)
      val batchResults = spark.read
        .parquet("/tmp/batch-aggregates")
        .withColumn("source", lit("batch"))
      
      // Real-time results from memory sink
      val realtimeResults = spark.sql("SELECT * FROM realtime_dashboard")
        .withColumn("source", lit("streaming"))
      
      // Union และ aggregate
      val combined = batchResults.union(realtimeResults)
        .groupBy($"category")
        .agg(sum("orders").as("total_orders"), sum("revenue").as("total_revenue"))
      
      combined.show()
    }
    
    // Start streaming dashboard
    val streamQuery = startRealtimeDashboard()
    
    Thread.sleep(35000)
    
    // Query hybrid
    // hybridQuery(spark)
    
    streamQuery.stop()
    spark.stop()
  }
}
```

---

## Step 783: Data Quality Checks

```scala
// DataQualityFramework.scala
import org.apache.spark.sql.{SparkSession, DataFrame}
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._

// Data quality rule
case class QualityRule(
  name: String,
  description: String,
  column: String,
  check: DataFrame => Long, // returns count of violations
  severity: String          // "error" | "warning"
)

// Quality result
case class QualityCheckResult(
  rule: String,
  severity: String,
  violationCount: Long,
  totalCount: Long,
  violationRate: Double,
  passed: Boolean
)

object DataQualityFramework {
  
  // ===== Define Quality Rules =====
  def createOrderRules(df: DataFrame): List[QualityRule] = {
    import df.sparkSession.implicits._
    
    List(
      QualityRule(
        name = "NOT_NULL_ORDER_ID",
        description = "order_id must not be null",
        column = "order_id",
        check = df => df.filter($"order_id".isNull).count(),
        severity = "error"
      ),
      QualityRule(
        name = "POSITIVE_AMOUNT",
        description = "amount must be positive",
        column = "amount",
        check = df => df.filter($"amount" <= 0 || $"amount".isNull).count(),
        severity = "error"
      ),
      QualityRule(
        name = "VALID_STATUS",
        description = "status must be in allowed values",
        column = "status",
        check = df => df.filter(!$"status".isin("pending","confirmed","shipped","delivered","cancelled")).count(),
        severity = "error"
      ),
      QualityRule(
        name = "UNIQUE_ORDER_ID",
        description = "order_id must be unique",
        column = "order_id",
        check = df => {
          val total = df.count()
          val unique = df.select("order_id").distinct().count()
          total - unique
        },
        severity = "error"
      ),
      QualityRule(
        name = "VALID_EMAIL",
        description = "customer_email should be valid format",
        column = "customer_email",
        check = df => df.filter(
          $"customer_email".isNotNull &&
          !$"customer_email".rlike("^[A-Za-z0-9+_.-]+@(.+)$")
        ).count(),
        severity = "warning"
      ),
      QualityRule(
        name = "RECENT_DATE",
        description = "order_date should be within last 2 years",
        column = "order_date",
        check = df => df.filter(
          $"order_date" < date_sub(current_date(), 730)
        ).count(),
        severity = "warning"
      ),
      QualityRule(
        name = "AMOUNT_RANGE",
        description = "amount should be between 1 and 1,000,000",
        column = "amount",
        check = df => df.filter($"amount" < 1 || $"amount" > 1000000).count(),
        severity = "warning"
      )
    )
  }
  
  // ===== Run Quality Checks =====
  def runQualityChecks(df: DataFrame): List[QualityCheckResult] = {
    val rules = createOrderRules(df)
    val totalCount = df.count()
    
    rules.map { rule =>
      val violationCount = rule.check(df)
      val violationRate = if (totalCount > 0) violationCount.toDouble / totalCount else 0.0
      val passed = rule.severity == "warning" || violationCount == 0
      
      QualityCheckResult(
        rule           = rule.name,
        severity       = rule.severity,
        violationCount = violationCount,
        totalCount     = totalCount,
        violationRate  = violationRate,
        passed         = passed
      )
    }
  }
  
  // ===== Quality Report =====
  def printQualityReport(results: List[QualityCheckResult]): Unit = {
    println("\n" + "=" * 80)
    println("DATA QUALITY REPORT")
    println("=" * 80)
    println(f"${"Rule"}%-30s ${"Severity"}%-10s ${"Violations"}%-12s ${"Rate"}%-8s ${"Status"}%-10s")
    println("-" * 80)
    
    results.foreach { r =>
      val status = if (r.passed) "✓ PASS" else "✗ FAIL"
      println(f"${r.rule}%-30s ${r.severity}%-10s ${r.violationCount}%-12d ${r.violationRate * 100}%-7.2f%% $status")
    }
    
    println("-" * 80)
    
    val errors   = results.count(r => !r.passed && r.severity == "error")
    val warnings = results.count(r => !r.passed && r.severity == "warning")
    
    println(f"Errors: $errors, Warnings: $warnings")
    println(s"Overall: ${if (errors == 0) "PASS" else "FAIL"}")
    println("=" * 80)
  }
  
  // ===== Column Statistics =====
  def computeColumnStats(df: DataFrame): Unit = {
    println("\n=== Column Statistics ===")
    
    val numericCols = df.schema.fields.filter { f =>
      f.dataType match {
        case _: NumericType => true
        case _              => false
      }
    }.map(_.name)
    
    if (numericCols.nonEmpty) {
      df.select(numericCols.map { col =>
        import df.sparkSession.implicits._
        struct(
          count(df(col)).as("count"),
          avg(df(col)).as("mean"),
          stddev(df(col)).as("stddev"),
          min(df(col)).as("min"),
          max(df(col)).as("max"),
          sum(when(df(col).isNull, 1).otherwise(0)).as("null_count")
        ).as(col)
      }: _*).show(truncate = false)
    }
    
    // String column stats
    val stringCols = df.schema.fields.filter(_.dataType == StringType).map(_.name)
    
    println("\n=== String Column Stats ===")
    stringCols.foreach { colName =>
      import df.sparkSession.implicits._
      val stats = df.agg(
        count(df(colName)).as("count"),
        sum(when(df(colName).isNull, 1).otherwise(0)).as("nulls"),
        countDistinct(df(colName)).as("cardinality"),
        avg(length(df(colName))).as("avg_length")
      ).first()
      
      println(f"  $colName%-20s: count=${stats.getAs[Long]("count")}, " +
              f"nulls=${stats.getAs[Long]("nulls")}, " +
              f"cardinality=${stats.getAs[Long]("cardinality")}")
    }
  }
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Data Quality")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // สร้างข้อมูล test ที่มีปัญหา
    val orders = Seq(
      ("O001", "C001", "Electronics", 999.99, "confirmed",  "2024-01-15", "customer1@email.com"),
      ("O002", "C002", "Clothing",    49.99,  "pending",    "2024-01-16", "customer2@email.com"),
      (null,   "C003", "Food",        9.99,   "shipped",    "2024-01-17", "customer3@email.com"),   // null order_id
      ("O004", "C004", "Electronics", -100.0, "confirmed",  "2024-01-18", "invalid-email"),           // negative amount + bad email
      ("O001", "C005", "Clothing",    29.99,  "delivered",  "2020-01-01", "customer5@email.com"),   // duplicate + old date
      ("O006", "C006", "Food",        5.99,   "INVALID",    "2024-01-20", "customer6@email.com"),   // invalid status
      ("O007", "C007", "Electronics", 1500.0, "confirmed",  "2024-01-21", "customer7@email.com")
    ).toDF("order_id", "customer_id", "category", "amount", "status", "order_date", "customer_email")
    
    println(s"Total records: ${orders.count()}")
    orders.show(truncate = false)
    
    // Run quality checks
    val results = runQualityChecks(orders)
    printQualityReport(results)
    computeColumnStats(orders)
    
    // ถ้ามี error ให้ fail pipeline
    val hasErrors = results.exists(r => !r.passed && r.severity == "error")
    if (hasErrors) {
      println("\n⚠️ PIPELINE BLOCKED: Data quality errors found!")
      // System.exit(1) // uncomment ใน production
    }
    
    spark.stop()
  }
}
```

---

## Step 784: Monitoring และ Alerting

```scala
// PipelineMonitoring.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._
import java.time.{LocalDateTime, Duration}
import scala.collection.mutable

// Pipeline metrics
case class PipelineMetrics(
  jobName: String,
  startTime: LocalDateTime,
  endTime: Option[LocalDateTime],
  inputRecords: Long,
  outputRecords: Long,
  errorRecords: Long,
  processingTimeMs: Long,
  status: String // running | success | failed
)

// Simple metrics store (ใน production ใช้ Prometheus, CloudWatch, ฯลฯ)
object MetricsStore {
  private val metrics = mutable.Map[String, PipelineMetrics]()
  
  def record(m: PipelineMetrics): Unit = metrics(m.jobName) = m
  def get(jobName: String): Option[PipelineMetrics] = metrics.get(jobName)
  def getAll: Map[String, PipelineMetrics] = metrics.toMap
}

// Alert rules
case class Alert(name: String, message: String, severity: String)

object PipelineMonitoring {
  
  def checkAlerts(metrics: PipelineMetrics): List[Alert] = {
    val alerts = mutable.ListBuffer[Alert]()
    
    // Check processing time
    if (metrics.processingTimeMs > 300000) { // > 5 minutes
      alerts += Alert(
        "SLOW_PIPELINE",
        s"${metrics.jobName} took ${metrics.processingTimeMs / 60000} minutes",
        "warning"
      )
    }
    
    // Check error rate
    val total = metrics.inputRecords
    if (total > 0) {
      val errorRate = metrics.errorRecords.toDouble / total
      if (errorRate > 0.05) { // > 5% errors
        alerts += Alert(
          "HIGH_ERROR_RATE",
          s"${metrics.jobName} error rate: ${(errorRate * 100).toInt}%",
          "error"
        )
      }
    }
    
    // Check output/input ratio
    if (metrics.inputRecords > 0 && metrics.outputRecords == 0) {
      alerts += Alert(
        "ZERO_OUTPUT",
        s"${metrics.jobName} produced 0 records from ${metrics.inputRecords} input",
        "error"
      )
    }
    
    // Check job failure
    if (metrics.status == "failed") {
      alerts += Alert(
        "JOB_FAILED",
        s"${metrics.jobName} failed",
        "critical"
      )
    }
    
    alerts.toList
  }
  
  def sendAlerts(alerts: List[Alert]): Unit = {
    alerts.foreach { alert =>
      val emoji = alert.severity match {
        case "critical" => "🔴"
        case "error"    => "🟠"
        case "warning"  => "🟡"
        case _          => "ℹ️"
      }
      println(s"$emoji [${alert.severity.toUpperCase}] ${alert.name}: ${alert.message}")
      // ใน production: ส่งไป Slack, PagerDuty, SNS ฯลฯ
    }
  }
  
  // Pipeline wrapper with monitoring
  def withMonitoring[T](jobName: String)(f: => T): T = {
    val start = LocalDateTime.now()
    val startMs = System.currentTimeMillis()
    
    try {
      val result = f
      val endMs = System.currentTimeMillis()
      
      println(s"✓ $jobName completed in ${endMs - startMs}ms")
      result
    } catch {
      case ex: Exception =>
        val endMs = System.currentTimeMillis()
        println(s"✗ $jobName failed after ${endMs - startMs}ms: ${ex.getMessage}")
        throw ex
    }
  }
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Pipeline Monitoring")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // Simulate pipeline run with monitoring
    val jobName = "daily-orders-etl"
    val startMs = System.currentTimeMillis()
    
    try {
      // Simulate processing
      val input = spark.range(10000)
        .withColumn("amount", rand() * 1000)
        .withColumn("category", ($"id" % 3).cast("string"))
      
      val output = withMonitoring("validation") {
        input.filter($"amount" > 0)
      }
      
      val aggregated = withMonitoring("aggregation") {
        output.groupBy($"category")
              .agg(count("*").as("cnt"), sum("amount").as("total"))
      }
      
      val recordCount = aggregated.count()
      
      // Record metrics
      val metrics = PipelineMetrics(
        jobName         = jobName,
        startTime       = LocalDateTime.now(),
        endTime         = Some(LocalDateTime.now()),
        inputRecords    = 10000,
        outputRecords   = recordCount,
        errorRecords    = 0,
        processingTimeMs = System.currentTimeMillis() - startMs,
        status          = "success"
      )
      
      MetricsStore.record(metrics)
      
      // Check alerts
      val alerts = checkAlerts(metrics)
      if (alerts.nonEmpty) {
        println("\n=== ALERTS ===")
        sendAlerts(alerts)
      } else {
        println(s"✓ All checks passed for $jobName")
      }
      
      // Print summary
      println(s"""
        |=== Pipeline Summary ===
        |Job: $jobName
        |Status: SUCCESS
        |Input:  ${metrics.inputRecords} records
        |Output: ${metrics.outputRecords} records
        |Errors: ${metrics.errorRecords} records
        |Time:   ${metrics.processingTimeMs}ms
      """.stripMargin)
      
    } catch {
      case ex: Exception =>
        val metrics = PipelineMetrics(
          jobName         = jobName,
          startTime       = LocalDateTime.now(),
          endTime         = Some(LocalDateTime.now()),
          inputRecords    = 0,
          outputRecords   = 0,
          errorRecords    = 0,
          processingTimeMs = System.currentTimeMillis() - startMs,
          status          = "failed"
        )
        
        MetricsStore.record(metrics)
        sendAlerts(checkAlerts(metrics))
        throw ex
    }
    
    spark.stop()
  }
}
```

---

## Step 785: Incremental Processing

```scala
// IncrementalProcessing.scala
import org.apache.spark.sql.{SparkSession, DataFrame}
import org.apache.spark.sql.functions._
import java.io.{File, PrintWriter}

object IncrementalProcessing {
  
  // State file สำหรับ tracking processed data
  case class ProcessingState(
    lastProcessedDate: String,
    lastProcessedOffset: Long,
    lastRunAt: String
  )
  
  def saveState(state: ProcessingState, statePath: String): Unit = {
    import spray.json._
    import DefaultJsonProtocol._
    implicit val stateFormat: RootJsonFormat[ProcessingState] = jsonFormat3(ProcessingState)
    
    val writer = new PrintWriter(statePath)
    writer.write(state.toJson.prettyPrint)
    writer.close()
  }
  
  def loadState(statePath: String): Option[ProcessingState] = {
    import spray.json._
    import DefaultJsonProtocol._
    implicit val stateFormat: RootJsonFormat[ProcessingState] = jsonFormat3(ProcessingState)
    
    val file = new File(statePath)
    if (file.exists()) {
      val content = scala.io.Source.fromFile(file).mkString
      Some(content.parseJson.convertTo[ProcessingState])
    } else {
      None
    }
  }
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Incremental Processing")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val statePath = "/tmp/incremental-state.json"
    
    // โหลด state จาก last run
    val lastState = loadState(statePath)
    val lastProcessedDate = lastState.map(_.lastProcessedDate).getOrElse("2024-01-01")
    
    println(s"Last processed date: $lastProcessedDate")
    
    // อ่านเฉพาะข้อมูลใหม่ (เพิ่มจาก last run)
    val today = java.time.LocalDate.now().toString
    
    // ===== Incremental Pattern 1: Date-based =====
    def readIncrementalByDate(fromDate: String, toDate: String): DataFrame = {
      // ใน production อ่านจาก database หรือ data lake
      // สำหรับ demo สร้าง fake data
      spark.range(100)
        .withColumn("order_date", date_add(lit(fromDate).cast("date"), ($"id" % 30).cast("int")))
        .withColumn("amount", rand() * 1000)
        .withColumn("category", ($"id" % 3).cast("string"))
        .filter($"order_date" >= fromDate && $"order_date" <= toDate)
    }
    
    val newData = readIncrementalByDate(lastProcessedDate, today)
    println(s"New records to process: ${newData.count()}")
    
    // Process new data
    val processed = newData
      .withColumn("processed_at", current_timestamp())
      .groupBy($"order_date", $"category")
      .agg(count("*").as("orders"), sum("amount").as("revenue"))
    
    // Upsert to output (merge new data with existing)
    // ใน production ใช้ Delta Lake merge
    processed.write.mode("append")
             .partitionBy("order_date")
             .parquet("/tmp/incremental-output")
    
    // อัปเดต state
    val newState = ProcessingState(
      lastProcessedDate = today,
      lastProcessedOffset = 0,
      lastRunAt = java.time.Instant.now().toString
    )
    
    saveState(newState, statePath)
    println(s"Updated state to: $today")
    
    // ===== Incremental Pattern 2: Watermark-based =====
    // อ่าน records ที่ updated_at > last watermark
    val lastWatermark = lastState.map(_.lastRunAt)
                                  .getOrElse("2024-01-01T00:00:00Z")
    
    println(s"Last watermark: $lastWatermark")
    
    // ===== Incremental Pattern 3: CDC (Change Data Capture) =====
    // Delta table สำหรับ CDC
    /*
    import io.delta.tables._
    
    val deltaTable = DeltaTable.forPath(spark, "/tmp/delta/orders")
    
    // Merge new data (upsert)
    deltaTable.as("existing")
      .merge(
        newData.as("updates"),
        "existing.order_id = updates.order_id"
      )
      .whenMatched.updateAll()  // update ถ้า match
      .whenNotMatched.insertAll() // insert ถ้าไม่ match
      .execute()
    */
    
    spark.stop()
  }
}
```

---

## Step 786: Pipeline Scheduling

```scala
// PipelineScheduling.scala
// สำหรับ production ใช้ Apache Airflow, Prefect, หรือ Dagster
// นี่คือ simple scheduler ใน Scala

import java.util.{Timer, TimerTask}
import java.time.{LocalDateTime, LocalDate}
import scala.concurrent.{Future, ExecutionContext}
import scala.util.{Try, Success, Failure}

// Job definition
case class PipelineJob(
  name: String,
  schedule: String,  // cron-like: "daily", "hourly", "every5min"
  command: () => Unit
)

// Scheduler
class PipelineScheduler {
  private val timer = new Timer("pipeline-scheduler", true)
  
  def scheduleDaily(job: PipelineJob): Unit = {
    val task = new TimerTask {
      override def run(): Unit = {
        println(s"[${LocalDateTime.now()}] Running daily job: ${job.name}")
        try {
          job.command()
          println(s"[${LocalDateTime.now()}] Completed: ${job.name}")
        } catch {
          case ex: Exception =>
            println(s"[${LocalDateTime.now()}] FAILED: ${job.name} - ${ex.getMessage}")
        }
      }
    }
    
    // รันทุก 24 ชั่วโมง
    timer.schedule(task, 0L, 24 * 60 * 60 * 1000L)
  }
  
  def scheduleHourly(job: PipelineJob): Unit = {
    val task = new TimerTask {
      override def run(): Unit = {
        try { job.command() }
        catch { case ex: Exception => println(s"Job failed: ${ex.getMessage}") }
      }
    }
    
    timer.schedule(task, 0L, 60 * 60 * 1000L)
  }
  
  def stop(): Unit = timer.cancel()
}

object PipelineScheduling {
  def main(args: Array[String]): Unit = {
    val scheduler = new PipelineScheduler()
    
    // Define jobs
    val dailyETL = PipelineJob(
      name     = "daily-sales-etl",
      schedule = "daily",
      command  = () => {
        println(s"Running daily ETL for ${LocalDate.now()}")
        // Spark job ที่นี่
        Thread.sleep(1000) // simulate work
      }
    )
    
    val hourlyMetrics = PipelineJob(
      name     = "hourly-metrics",
      schedule = "hourly",
      command  = () => {
        println("Computing hourly metrics")
        Thread.sleep(500)
      }
    )
    
    // Schedule jobs
    println("Starting scheduler...")
    scheduler.scheduleHourly(hourlyMetrics)
    
    // รอ 5 วินาที แล้ว stop
    Thread.sleep(5000)
    scheduler.stop()
    println("Scheduler stopped")
    
    // ===== Apache Airflow DAG example (Python) =====
    println("""
      # Airflow DAG example (Python):
      
      from airflow import DAG
      from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator
      from datetime import datetime, timedelta
      
      dag = DAG(
          'daily_etl',
          schedule_interval='0 2 * * *',  # 2 AM daily
          start_date=datetime(2024, 1, 1),
          catchup=False,
          default_args={
              'retries': 3,
              'retry_delay': timedelta(minutes=5)
          }
      )
      
      extract_task = SparkSubmitOperator(
          task_id='extract',
          application='daily-etl.jar',
          conf={'spark.executor.memory': '8g'},
          dag=dag
      )
      
      transform_task = SparkSubmitOperator(...)
      load_task = SparkSubmitOperator(...)
      
      extract_task >> transform_task >> load_task
    """)
  }
}
```

---

## Step 787: Data Lineage

```scala
// DataLineage.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._

object DataLineage {
  
  // Simple lineage tracker
  case class LineageNode(
    name: String,
    nodeType: String, // source, transform, sink
    inputs: List[String],
    description: String
  )
  
  class LineageTracker {
    private val nodes = scala.collection.mutable.Map[String, LineageNode]()
    
    def registerSource(name: String, location: String): Unit = {
      nodes(name) = LineageNode(
        name = name,
        nodeType = "source",
        inputs = List.empty,
        description = s"Source from: $location"
      )
    }
    
    def registerTransform(name: String, inputs: List[String], description: String): Unit = {
      nodes(name) = LineageNode(
        name = name,
        nodeType = "transform",
        inputs = inputs,
        description = description
      )
    }
    
    def registerSink(name: String, inputs: List[String], location: String): Unit = {
      nodes(name) = LineageNode(
        name = name,
        nodeType = "sink",
        inputs = inputs,
        description = s"Sink to: $location"
      )
    }
    
    def printLineage(): Unit = {
      println("\n=== DATA LINEAGE ===")
      nodes.values.foreach { node =>
        val inputStr = if (node.inputs.isEmpty) "none" else node.inputs.mkString(", ")
        println(s"[${node.nodeType.toUpperCase}] ${node.name}")
        println(s"  Inputs: $inputStr")
        println(s"  ${node.description}")
        println()
      }
    }
    
    def getUpstream(nodeName: String): Set[String] = {
      def getInputs(name: String): Set[String] = {
        nodes.get(name).map { node =>
          node.inputs.toSet ++ node.inputs.flatMap(getInputs).toSet
        }.getOrElse(Set.empty)
      }
      getInputs(nodeName)
    }
    
    def getDownstream(nodeName: String): Set[String] = {
      nodes.values.filter { n =>
        n.inputs.contains(nodeName) || 
        getUpstream(n.name).contains(nodeName)
      }.map(_.name).toSet
    }
  }
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Data Lineage")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    
    val lineage = new LineageTracker()
    
    // Register lineage
    lineage.registerSource("raw_orders",    "s3://data-lake/raw/orders/")
    lineage.registerSource("raw_customers", "s3://data-lake/raw/customers/")
    lineage.registerSource("raw_products",  "s3://data-lake/raw/products/")
    
    lineage.registerTransform("validated_orders",
      inputs = List("raw_orders"),
      description = "Filter invalid orders, add batch_date"
    )
    
    lineage.registerTransform("enriched_orders",
      inputs = List("validated_orders", "raw_customers", "raw_products"),
      description = "Join with customer and product data"
    )
    
    lineage.registerTransform("order_aggregates",
      inputs = List("enriched_orders"),
      description = "Aggregate by date, category, region"
    )
    
    lineage.registerSink("sales_warehouse",
      inputs = List("order_aggregates"),
      location = "s3://data-warehouse/sales/"
    )
    
    lineage.registerSink("customer_metrics",
      inputs = List("enriched_orders"),
      location = "s3://data-warehouse/customer-metrics/"
    )
    
    // Print lineage
    lineage.printLineage()
    
    // Find upstream of a node
    println("Upstream of order_aggregates:")
    lineage.getUpstream("order_aggregates").foreach(s => println(s"  - $s"))
    
    println("\nDownstream of raw_orders:")
    lineage.getDownstream("raw_orders").foreach(s => println(s"  - $s"))
    
    // Spark UI ก็แสดง lineage ใน SQL tab
    // สำหรับ production ใช้ Apache Atlas, OpenLineage, Marquez
    
    spark.stop()
  }
}
```

---

## Step 788: Testing Data Pipelines

```scala
// PipelineTests.scala
import org.apache.spark.sql.SparkSession
import org.scalatest.flatspec.AnyFlatSpec
import org.scalatest.matchers.should.Matchers
import org.scalatest.BeforeAndAfterAll

class DataPipelineTests extends AnyFlatSpec with Matchers with BeforeAndAfterAll {
  
  var spark: SparkSession = _
  
  override def beforeAll(): Unit = {
    spark = SparkSession.builder()
      .master("local[2]")
      .appName("Pipeline Tests")
      .getOrCreate()
    spark.sparkContext.setLogLevel("ERROR")
  }
  
  override def afterAll(): Unit = {
    spark.stop()
  }
  
  import spark.implicits._
  
  // ===== Unit Tests =====
  "validateOrders" should "filter out null order_ids" in {
    import spark.implicits._
    
    val input = Seq(
      ("O001", "C001", 100.0, "confirmed"),
      (null,   "C002", 200.0, "pending"),
      ("O003", "C003", 300.0, "shipped")
    ).toDF("order_id", "customer_id", "amount", "status")
    
    val (valid, invalid) = ETLPipelineDesign.validateOrders(input)
    
    valid.count() shouldBe 2
    invalid.count() shouldBe 1
  }
  
  "validateOrders" should "detect negative amounts" in {
    import spark.implicits._
    
    val input = Seq(
      ("O001", "C001", -100.0, "confirmed"),
      ("O002", "C002",  200.0, "pending")
    ).toDF("order_id", "customer_id", "amount", "status")
    
    val (valid, invalid) = ETLPipelineDesign.validateOrders(input)
    
    valid.count() shouldBe 1
    invalid.count() shouldBe 1
  }
  
  "transformOrders" should "calculate revenue correctly" in {
    import spark.implicits._
    
    val input = Seq(
      ("O001", 2, 100.0, "USD")
    ).toDF("order_id", "quantity", "unit_price", "currency")
    
    val result = ETLPipelineDesign.transformOrders(input)
    val row = result.select("revenue").first()
    
    row.getAs[Double]("revenue") shouldBe 200.0
  }
  
  // ===== Integration Tests =====
  "full pipeline" should "produce aggregated output" in {
    import spark.implicits._
    
    // Create test data
    val testData = Seq(
      ("O001", "C001", "Electronics", "confirmed", 2, 999.99, "USD", "2024-01-15", "North"),
      ("O002", "C002", "Clothing",    "pending",   1, 49.99,  "USD", "2024-01-15", "South"),
      ("O003", "C001", "Electronics", "shipped",   1, 1299.99,"USD", "2024-01-16", "North")
    ).toDF("order_id", "customer_id", "category", "status", "quantity", "unit_price", "currency", "order_date", "region")
    
    val config = PipelineConfig(
      inputPath = "", outputPath = "", checkpointPath = "",
      batchDate = "2024-01-15", env = "test"
    )
    
    val (valid, invalid) = ETLPipelineDesign.validateOrders(testData)
    val transformed = ETLPipelineDesign.transformOrders(valid)
    val aggregated = ETLPipelineDesign.aggregateOrders(transformed)
    
    aggregated.count() should be > 0L
    
    // Verify schema
    aggregated.schema.fieldNames should contain ("total_revenue_usd")
    aggregated.schema.fieldNames should contain ("order_count")
  }
  
  // ===== Data Quality Tests =====
  "quality checks" should "detect all violations" in {
    import spark.implicits._
    
    val badData = Seq(
      (null,   "C001", 100.0, "confirmed"),  // null order_id
      ("O002", "C002", -50.0, "pending")     // negative amount
    ).toDF("order_id", "customer_id", "amount", "status")
    
    val results = DataQualityFramework.runQualityChecks(badData)
    
    results.filter(r => !r.passed && r.severity == "error") should not be empty
  }
}
```

---

## Step 789: Pipeline Documentation

```markdown
# Data Pipeline Documentation Template

## Pipeline: daily-orders-etl

### Overview
Daily ETL pipeline ที่ process orders จาก raw S3 bucket ไปยัง data warehouse

### Schedule
- Frequency: Daily
- Time: 02:00 UTC
- SLA: ต้องเสร็จภายใน 4 ชั่วโมง

### Data Flow
```
S3 raw/orders → Extract → Validate → Transform → Aggregate → DW
                              ↓
                         DLQ (invalid records)
```

### Dependencies
- Upstream: Order service (produces to S3)
- Downstream: BI dashboards, ML models

### Configuration
| Parameter | Value | Description |
|-----------|-------|-------------|
| input_path | s3://data-lake/raw/orders/ | Input location |
| output_path | s3://data-warehouse/orders/ | Output location |
| partitioning | date, category | Partition columns |

### Data Quality Rules
| Rule | Severity | Action |
|------|----------|--------|
| NOT_NULL_ORDER_ID | Error | Block pipeline |
| POSITIVE_AMOUNT | Error | Block pipeline |
| VALID_STATUS | Warning | Log only |

### Monitoring
- Metrics: records processed, error rate, duration
- Alerts: Slack #data-alerts channel
- Dashboard: Grafana > Data Pipelines

### Runbook
1. ถ้า pipeline fail: ดู Airflow log
2. ถ้า error rate > 5%: ตรวจสอบ upstream data quality
3. ถ้า SLA breach: scale up Spark cluster
```

---

## Step 790: Production Deployment

```bash
#!/bin/bash
# deploy-pipeline.sh

set -e  # exit on error

APP_NAME="daily-orders-etl"
VERSION="1.2.3"
SPARK_HOME="/opt/spark"
HDFS_JAR_PATH="hdfs:///apps/jars"

echo "Deploying $APP_NAME version $VERSION"

# 1. Build JAR
echo "Building..."
sbt clean assembly
JAR="target/scala-2.13/$APP_NAME-assembly-$VERSION.jar"

# 2. Upload to HDFS
echo "Uploading to HDFS..."
hdfs dfs -put -f $JAR $HDFS_JAR_PATH/

# 3. Run Spark job
echo "Submitting Spark job..."
$SPARK_HOME/bin/spark-submit \
  --class com.example.ETLPipelineDesign \
  --master yarn \
  --deploy-mode cluster \
  --num-executors 10 \
  --executor-cores 4 \
  --executor-memory 8g \
  --driver-memory 4g \
  --conf "spark.serializer=org.apache.spark.serializer.KryoSerializer" \
  --conf "spark.sql.shuffle.partitions=200" \
  --conf "spark.sql.adaptive.enabled=true" \
  --conf "spark.dynamicAllocation.enabled=true" \
  --conf "spark.dynamicAllocation.maxExecutors=20" \
  --files "src/main/resources/application-prod.conf#application.conf" \
  $HDFS_JAR_PATH/$APP_NAME-assembly-$VERSION.jar \
  --date $(date +%Y-%m-%d) \
  --env prod

echo "Deployment completed"
```

---

## สรุป Part 79

| Pattern | When to Use | Example |
|---------|-------------|---------|
| Batch ETL | Daily/hourly data | Sales aggregation |
| Streaming | Real-time needs | Fraud detection |
| Lambda | Both needed | Analytics platform |
| Incremental | Large data | Daily delta loads |
| CDC | Database sync | Replica tables |

### Pipeline Health Metrics
1. **Freshness**: data age, update delay
2. **Completeness**: % records processed vs expected
3. **Accuracy**: validation pass rate
4. **Consistency**: cross-system checks

---

## แบบฝึกหัด Part 79

1. **Pipeline Framework**: สร้าง reusable framework ที่ implement stages เป็น composable functions แล้วเขียน 3 pipelines ต่างๆ โดยใช้ framework เดียวกัน

2. **Data Quality**: implement data quality framework ที่มี rules catalog, automated checks, และ quality report ที่ส่ง email เมื่อ quality drops

3. **Incremental Load**: implement incremental ETL ด้วย watermark-based approach ที่ handle late arrivals และ restarts ได้อย่าง idempotent

4. **Pipeline Testing**: เขียน comprehensive test suite สำหรับ data pipeline ที่ cover unit tests, integration tests, และ data quality tests

5. **Monitoring Dashboard**: สร้าง pipeline monitoring dashboard ที่แสดง throughput, latency, error rates และ alert history

---

## ไปต่อ: Part 80 — Delta Lake
ใน Part ถัดไปจะเรียน Delta Lake, ACID transactions, time travel, และ merge/upsert

[→ Part 80: Delta Lake](./part-80-delta-lake.md)
