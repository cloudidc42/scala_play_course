# Part 98: Real-World Data Platform — Steps 971-980

## บทนำ: Real-Time Data Platform

Real-time data platform ด้วย Spark + Kafka + Delta Lake — E-commerce analytics pipeline ที่ handle events, compute metrics, และ serve dashboards

```scala
// build.sbt
name := "data-platform"
scalaVersion := "2.13.14"

libraryDependencies ++= Seq(
  "org.apache.spark" %% "spark-core"              % "3.5.1",
  "org.apache.spark" %% "spark-sql"               % "3.5.1",
  "org.apache.spark" %% "spark-streaming"          % "3.5.1",
  "org.apache.spark" %% "spark-sql-kafka-0-10"     % "3.5.1",
  "io.delta"         %% "delta-spark"             % "3.1.0",
  "org.apache.kafka"  % "kafka-clients"            % "3.6.1",
  "com.typesafe.akka" %% "akka-stream-kafka"       % "5.0.0"
)
```

---

## Step 971: Architecture Overview

```scala
// ArchitectureOverview.scala

/*
===== Real-Time E-commerce Data Platform =====

Data Sources:
  ┌─────────────────────────────────────────────────────────────┐
  │  Order Service   →  Kafka: orders.v1                        │
  │  Product Service →  Kafka: products.v1                      │
  │  User Service    →  Kafka: users.v1                         │
  │  Clickstream     →  Kafka: clickstream.v1                   │
  └─────────────────────────────────────────────────────────────┘
           ↓ Kafka Topics (3.6.x, 6 partitions each)
  
  ┌────────────────── Bronze Layer ─────────────────────────────┐
  │  Raw Ingest → Delta Lake s3://data/bronze/{topic}/          │
  │  Schema validation, deduplication, metadata enrichment       │
  └─────────────────────────────────────────────────────────────┘
           ↓ Spark Structured Streaming
  
  ┌────────────────── Silver Layer ─────────────────────────────┐
  │  Cleaned Data → Delta Lake s3://data/silver/{entity}/       │
  │  Joins, denormalization, SCD Type 2, data quality           │
  └─────────────────────────────────────────────────────────────┘
           ↓ Spark Batch (hourly) + Streaming
  
  ┌────────────────── Gold Layer ───────────────────────────────┐
  │  Aggregations → Delta Lake s3://data/gold/{metric}/         │
  │  Revenue, conversion rates, product rankings, user segments │
  └─────────────────────────────────────────────────────────────┘
           ↓ API Layer
  
  ┌────────────────── Serving Layer ────────────────────────────┐
  │  Play Framework API → Dashboard, Reports                    │
  │  Redis Cache → Real-time metrics                            │
  └─────────────────────────────────────────────────────────────┘
*/

case class PlatformConfig(
  kafkaBrokers:     String = "kafka:9092",
  s3Bucket:         String = "s3://mycompany-data",
  checkpointBucket: String = "s3://mycompany-checkpoints",
  redisUrl:         String = "redis://redis:6379"
)
```

---

## Step 972: Bronze Layer — Raw Ingest

```scala
// pipeline/BronzeIngest.scala
import org.apache.spark.sql._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.streaming.Trigger
import io.delta.tables.DeltaTable
import scala.concurrent.duration._

class BronzeIngest(spark: SparkSession, config: PlatformConfig) {
  import spark.implicits._
  
  // ===== Order events ingest =====
  def ingestOrders(): Unit = {
    spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", config.kafkaBrokers)
      .option("subscribe", "orders.v1")
      .option("startingOffsets", "earliest")
      .option("maxOffsetsPerTrigger", "100000")
      .load()
      
      // Extract metadata
      .select(
        col("key").cast("string").as("order_id"),
        col("value").cast("string").as("raw_payload"),
        col("topic"),
        col("partition"),
        col("offset"),
        col("timestamp").as("kafka_timestamp")
      )
      
      // Add ingestion metadata
      .withColumn("ingested_at",   current_timestamp())
      .withColumn("date_partition", to_date(col("kafka_timestamp")))
      .withColumn("source_system",  lit("order-service"))
      
      .writeStream
      .format("delta")
      .outputMode("append")
      .option("checkpointLocation", s"${config.checkpointBucket}/bronze/orders")
      .partitionBy("date_partition")
      .trigger(Trigger.ProcessingTime("1 minute"))
      .start(s"${config.s3Bucket}/bronze/orders")
  }
  
  // ===== Generic topic ingest =====
  def ingestTopic(topic: String, destPath: String): Unit = {
    spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", config.kafkaBrokers)
      .option("subscribe", topic)
      .option("startingOffsets", "latest")
      .load()
      .select(
        col("key").cast("string").as("event_key"),
        col("value").cast("string").as("payload"),
        col("timestamp").as("event_time"),
        current_timestamp().as("ingested_at"),
        to_date(col("timestamp")).as("date_partition")
      )
      .writeStream
      .format("delta")
      .outputMode("append")
      .option("checkpointLocation", s"${config.checkpointBucket}/bronze/$topic")
      .partitionBy("date_partition")
      .trigger(Trigger.ProcessingTime("30 seconds"))
      .start(destPath)
  }
  
  // ===== Schema validation =====
  val orderSchema = new org.apache.spark.sql.types.StructType()
    .add("orderId",    "long")
    .add("userId",     "long")
    .add("total",      "double")
    .add("items",      "array<struct<productId:long,quantity:int>>")
    .add("status",     "string")
    .add("createdAt",  "timestamp")
  
  def validateAndIngest(df: DataFrame): DataFrame = {
    df
      .withColumn("parsed", from_json(col("payload"), orderSchema))
      .withColumn("is_valid", col("parsed").isNotNull)
      .withColumn("validation_error", when(col("parsed").isNull, "JSON parse failed").otherwise(null))
  }
}
```

---

## Step 973: Silver Layer — Data Cleaning

```scala
// pipeline/SilverTransform.scala
import org.apache.spark.sql._
import org.apache.spark.sql.functions._
import io.delta.tables.DeltaTable

class SilverTransform(spark: SparkSession, config: PlatformConfig) {
  import spark.implicits._
  
  // ===== Clean order events → enriched orders =====
  def transformOrders(): Unit = {
    val bronze = spark.readStream
      .format("delta")
      .option("readChangeFeed", "true")
      .load(s"${config.s3Bucket}/bronze/orders")
      .where(col("is_valid") === true)
    
    val orderSchema = new org.apache.spark.sql.types.StructType()
      .add("orderId", "long").add("userId", "long").add("total", "double")
      .add("status", "string").add("createdAt", "timestamp")
    
    bronze
      .withColumn("order", from_json(col("payload"), orderSchema))
      .select(
        col("order.orderId").as("order_id"),
        col("order.userId").as("user_id"),
        col("order.total"),
        col("order.status"),
        col("order.createdAt").as("created_at"),
        col("ingested_at"),
        date_trunc("hour", col("order.createdAt")).as("hour_partition"),
        to_date(col("order.createdAt")).as("date_partition")
      )
      
      // Data quality filters
      .where(col("order_id").isNotNull && col("user_id").isNotNull)
      .where(col("total") >= 0)
      .where(col("status").isin("pending", "confirmed", "processing", "shipped", "delivered", "cancelled"))
      
      // Deduplicate
      .dropDuplicates("order_id", "status")
      
      .writeStream
      .format("delta")
      .outputMode("append")
      .option("checkpointLocation", s"${config.checkpointBucket}/silver/orders")
      .partitionBy("date_partition")
      .trigger(org.apache.spark.sql.streaming.Trigger.ProcessingTime("2 minutes"))
      .start(s"${config.s3Bucket}/silver/orders")
  }
  
  // ===== SCD Type 2 for product prices =====
  def updateProductPrices(): Unit = {
    val newPrices = spark.read
      .format("delta")
      .load(s"${config.s3Bucket}/bronze/products")
      .withColumn("product", from_json(col("payload"),
        new org.apache.spark.sql.types.StructType()
          .add("productId", "long").add("name", "string")
          .add("price", "double").add("updatedAt", "timestamp")))
      .select(
        col("product.productId").as("product_id"),
        col("product.name"),
        col("product.price"),
        col("product.updatedAt").as("effective_from"),
        lit(null).cast("timestamp").as("effective_to"),
        lit(true).as("is_current")
      )
    
    val silverProducts = DeltaTable.forPath(spark, s"${config.s3Bucket}/silver/products")
    
    silverProducts.alias("existing")
      .merge(newPrices.alias("new"), "existing.product_id = new.product_id AND existing.is_current = true")
      .whenMatched("existing.price != new.price")
        .updateExpr(Map(
          "effective_to" -> "new.effective_from",
          "is_current"   -> "false"
        ))
      .whenNotMatched()
        .insertAll()
      .execute()
    
    // Insert new price records for changed products
    newPrices.join(
      spark.read.format("delta").load(s"${config.s3Bucket}/silver/products")
        .where(col("is_current") === false && col("effective_to").isNotNull),
      "product_id"
    ).write.format("delta").mode("append").save(s"${config.s3Bucket}/silver/products")
  }
}
```

---

## Step 974: Gold Layer — Aggregations

```scala
// pipeline/GoldAggregations.scala
import org.apache.spark.sql._
import org.apache.spark.sql.functions._

class GoldAggregations(spark: SparkSession, config: PlatformConfig) {
  import spark.implicits._
  
  // ===== Daily revenue =====
  def computeDailyRevenue(date: String): Unit = {
    val orders = spark.read.format("delta")
      .load(s"${config.s3Bucket}/silver/orders")
      .where(col("date_partition") === date)
      .where(col("status") === "delivered")
    
    val revenue = orders
      .groupBy(
        col("date_partition").as("date"),
        hour(col("created_at")).as("hour")
      )
      .agg(
        count("*").as("order_count"),
        sum("total").as("revenue"),
        avg("total").as("avg_order_value"),
        countDistinct("user_id").as("unique_customers")
      )
      .withColumn("processed_at", current_timestamp())
    
    revenue.write.format("delta")
      .mode("overwrite")
      .option("replaceWhere", s"date = '$date'")
      .save(s"${config.s3Bucket}/gold/daily_revenue")
    
    println(s"Daily revenue computed for $date")
  }
  
  // ===== Product rankings =====
  def computeProductRankings(date: String): Unit = {
    spark.sql(s"""
      SELECT
        oi.product_id,
        oi.product_name,
        SUM(oi.quantity)         AS total_units_sold,
        SUM(oi.quantity * oi.unit_price) AS total_revenue,
        COUNT(DISTINCT o.order_id)       AS total_orders,
        RANK() OVER (ORDER BY SUM(oi.quantity * oi.unit_price) DESC) AS revenue_rank
      FROM delta.`${config.s3Bucket}/silver/order_items` oi
      JOIN delta.`${config.s3Bucket}/silver/orders` o
        ON oi.order_id = o.order_id
      WHERE o.date_partition = '$date'
        AND o.status = 'delivered'
      GROUP BY oi.product_id, oi.product_name
      ORDER BY total_revenue DESC
      LIMIT 1000
    """).write.format("delta")
      .mode("overwrite")
      .option("replaceWhere", s"date = '$date'")
      .save(s"${config.s3Bucket}/gold/product_rankings")
  }
  
  // ===== User segmentation =====
  def computeUserSegments(): Unit = {
    val userStats = spark.sql(s"""
      SELECT
        user_id,
        COUNT(*) AS total_orders,
        SUM(total) AS lifetime_value,
        MAX(created_at) AS last_order_date,
        DATEDIFF(CURRENT_DATE(), MAX(created_at)) AS days_since_last_order,
        CASE
          WHEN SUM(total) > 10000 THEN 'VIP'
          WHEN SUM(total) > 1000  THEN 'Premium'
          WHEN COUNT(*) > 10      THEN 'Loyal'
          ELSE 'Regular'
        END AS segment
      FROM delta.`${config.s3Bucket}/silver/orders`
      WHERE status IN ('delivered', 'shipped')
      GROUP BY user_id
    """)
    
    userStats.write.format("delta")
      .mode("overwrite")
      .save(s"${config.s3Bucket}/gold/user_segments")
  }
  
  // ===== Real-time streaming aggregation =====
  def streamingRevenue(): Unit = {
    spark.readStream
      .format("delta")
      .load(s"${config.s3Bucket}/silver/orders")
      .where(col("status") === "delivered")
      .withWatermark("created_at", "10 minutes")
      .groupBy(
        window(col("created_at"), "1 hour"),
        col("date_partition")
      )
      .agg(
        count("*").as("order_count"),
        sum("total").as("revenue")
      )
      .writeStream
      .format("delta")
      .outputMode("update")
      .option("checkpointLocation", s"${config.checkpointBucket}/gold/streaming_revenue")
      .trigger(org.apache.spark.sql.streaming.Trigger.ProcessingTime("5 minutes"))
      .start(s"${config.s3Bucket}/gold/streaming_revenue")
  }
}
```

---

## Step 975: Serving Layer — Play Framework API

```scala
// controllers/MetricsController.scala
import play.api.mvc._
import play.api.libs.json._
import javax.inject._
import scala.concurrent.{ExecutionContext, Future}

case class DailyRevenueMetric(date: String, revenue: Double, orderCount: Long, avgOrderValue: Double)
case class ProductRanking(productId: Long, name: String, revenue: Double, rank: Int)

@Singleton
class MetricsController @Inject()(
  cc: ControllerComponents,
  metricsService: MetricsService
)(implicit ec: ExecutionContext) extends AbstractController(cc) {
  
  implicit val revenueWrites: Writes[DailyRevenueMetric] = Json.writes[DailyRevenueMetric]
  implicit val rankingWrites: Writes[ProductRanking] = Json.writes[ProductRanking]
  
  def getDailyRevenue(from: String, to: String): Action[AnyContent] = Action.async {
    metricsService.getDailyRevenue(from, to).map { metrics =>
      Ok(Json.obj(
        "from"    -> from,
        "to"      -> to,
        "metrics" -> Json.toJson(metrics),
        "total"   -> metrics.map(_.revenue).sum
      ))
    }
  }
  
  def getTopProducts(date: String, limit: Int = 10): Action[AnyContent] = Action.async {
    metricsService.getTopProducts(date, limit).map { rankings =>
      Ok(Json.obj(
        "date"     -> date,
        "rankings" -> Json.toJson(rankings)
      ))
    }
  }
  
  def getRealtimeRevenue: Action[AnyContent] = Action.async {
    metricsService.getRealtimeRevenue.map { metric =>
      Ok(Json.obj(
        "currentHourRevenue"  -> metric.revenue,
        "currentHourOrders"   -> metric.orderCount,
        "updatedAt"           -> java.time.Instant.now().toString
      ))
    }
  }
  
  def getUserSegmentStats: Action[AnyContent] = Action.async {
    metricsService.getUserSegmentStats.map { stats =>
      Ok(Json.toJson(stats))
    }
  }
}
```

---

## Step 976: Redis Cache for Real-Time Metrics

```scala
// cache/MetricsCache.scala
import redis.RedisClient
import play.api.libs.json._
import scala.concurrent.{ExecutionContext, Future}
import scala.concurrent.duration._

class MetricsCache(redis: RedisClient)(implicit ec: ExecutionContext) {
  
  private val TTL = 60  // seconds
  
  def getCachedOrLoad[T: Reads: Writes](
    key: String,
    ttl: Int = TTL
  )(load: => Future[T]): Future[T] = {
    redis.get(key).flatMap {
      case Some(cached) =>
        Json.parse(cached).asOpt[T] match {
          case Some(v) => Future.successful(v)
          case None    => loadAndCache(key, ttl, load)
        }
      case None => loadAndCache(key, ttl, load)
    }
  }
  
  private def loadAndCache[T: Writes](key: String, ttl: Int, load: Future[T]): Future[T] = {
    load.flatMap { v =>
      redis.setex(key, ttl, Json.toJson(v).toString()).map(_ => v)
    }
  }
  
  def invalidate(keys: String*): Future[Unit] =
    Future.sequence(keys.map(redis.del(_))).map(_ => ())
  
  def invalidatePattern(pattern: String): Future[Unit] =
    redis.keys(pattern).flatMap { keys =>
      if (keys.nonEmpty) redis.del(keys.head, keys.tail: _*).map(_ => ())
      else Future.unit
    }
}
```

---

## Step 977: Data Quality Monitoring

```scala
// quality/DataQualityMonitor.scala
import org.apache.spark.sql._
import org.apache.spark.sql.functions._

case class DataQualityResult(
  table: String,
  date: String,
  totalRows: Long,
  nullRows: Long,
  duplicates: Long,
  violations: List[String]
)

class DataQualityMonitor(spark: SparkSession, config: PlatformConfig) {
  
  def checkOrders(date: String): DataQualityResult = {
    val df = spark.read.format("delta")
      .load(s"${config.s3Bucket}/silver/orders")
      .where(col("date_partition") === date)
    
    val totalRows = df.count()
    
    // Null checks
    val nullRows = df.where(
      col("order_id").isNull || col("user_id").isNull || col("total").isNull
    ).count()
    
    // Duplicate checks
    val duplicates = totalRows - df.dropDuplicates("order_id", "status").count()
    
    // Business rule violations
    val violations = scala.collection.mutable.ListBuffer.empty[String]
    
    val negativeTotal = df.where(col("total") < 0).count()
    if (negativeTotal > 0) violations += s"$negativeTotal orders with negative total"
    
    val invalidStatus = df.where(!col("status").isin(
      "pending", "confirmed", "processing", "shipped", "delivered", "cancelled"
    )).count()
    if (invalidStatus > 0) violations += s"$invalidStatus orders with invalid status"
    
    val futureDates = df.where(col("created_at") > current_timestamp()).count()
    if (futureDates > 0) violations += s"$futureDates orders with future dates"
    
    val result = DataQualityResult("silver.orders", date, totalRows, nullRows, duplicates, violations.toList)
    
    // Publish metrics to Prometheus
    if (nullRows > totalRows * 0.01) {
      println(s"ALERT: High null rate in orders: ${nullRows}/${totalRows}")
    }
    
    if (violations.nonEmpty) {
      println(s"ALERT: Data quality violations: ${violations.mkString(", ")}")
    }
    
    result
  }
  
  // ===== Great Expectations-style checks =====
  def expectColumnValues(df: DataFrame, column: String, expectations: List[ColumnExpectation]): Boolean = {
    expectations.forall {
      case NotNull =>
        val nullCount = df.where(col(column).isNull).count()
        if (nullCount > 0) println(s"FAIL: $column has $nullCount nulls"); nullCount == 0
      
      case InSet(values) =>
        val invalidCount = df.where(!col(column).isin(values: _*)).count()
        if (invalidCount > 0) println(s"FAIL: $column has $invalidCount invalid values"); invalidCount == 0
      
      case PositiveValues =>
        val negCount = df.where(col(column) <= 0).count()
        if (negCount > 0) println(s"FAIL: $column has $negCount non-positive values"); negCount == 0
    }
  }
  
  sealed trait ColumnExpectation
  case object NotNull extends ColumnExpectation
  case class InSet(values: List[String]) extends ColumnExpectation
  case object PositiveValues extends ColumnExpectation
}
```

---

## Step 978: Pipeline Orchestration

```python
# airflow/dags/data_platform_dag.py
from airflow import DAG
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-platform',
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'email_on_failure': True,
    'email': ['data-team@mycompany.com']
}

with DAG(
    'ecommerce_data_pipeline',
    default_args=default_args,
    schedule_interval='@hourly',
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=['data-platform', 'ecommerce']
) as dag:
    
    ingest_bronze = SparkSubmitOperator(
        task_id='ingest_bronze',
        application='s3://jobs/bronze_ingest.jar',
        application_args=['--date', '{{ ds }}'],
        conf={'spark.executor.memory': '4g', 'spark.executor.cores': '4'}
    )
    
    transform_silver = SparkSubmitOperator(
        task_id='transform_silver',
        application='s3://jobs/silver_transform.jar',
        application_args=['--date', '{{ ds }}'],
        conf={'spark.executor.memory': '8g'}
    )
    
    compute_gold = SparkSubmitOperator(
        task_id='compute_gold',
        application='s3://jobs/gold_aggregations.jar',
        application_args=['--date', '{{ ds }}']
    )
    
    quality_check = PythonOperator(
        task_id='quality_check',
        python_callable=lambda **ctx: print(f"Quality check for {ctx['ds']}")
    )
    
    invalidate_cache = PythonOperator(
        task_id='invalidate_cache',
        python_callable=lambda **ctx: print("Cache invalidated")
    )
    
    # Pipeline: bronze → silver → gold → quality → cache
    ingest_bronze >> transform_silver >> compute_gold >> quality_check >> invalidate_cache
```

---

## Step 979: Monitoring & Alerting

```scala
// monitoring/PipelineMonitor.scala
import io.prometheus.client._

object PipelineMetrics {
  
  val rowsIngested: Counter = Counter.build()
    .name("pipeline_rows_ingested_total")
    .help("Total rows ingested")
    .labelNames("layer", "table")
    .register()
  
  val processingDuration: Histogram = Histogram.build()
    .name("pipeline_processing_duration_seconds")
    .help("Processing duration per stage")
    .labelNames("stage")
    .buckets(1, 5, 30, 60, 300, 600, 1800)
    .register()
  
  val qualityErrors: Counter = Counter.build()
    .name("pipeline_quality_errors_total")
    .help("Data quality errors")
    .labelNames("table", "error_type")
    .register()
  
  val lastSuccessfulRun: Gauge = Gauge.build()
    .name("pipeline_last_successful_run_timestamp")
    .help("Timestamp of last successful run")
    .labelNames("stage")
    .register()
  
  def recordStageSuccess(stage: String, rows: Long, durationSec: Double): Unit = {
    rowsIngested.labels("pipeline", stage).inc(rows)
    processingDuration.labels(stage).observe(durationSec)
    lastSuccessfulRun.labels(stage).setToCurrentTime()
  }
}

// Alerting rules (Prometheus)
/*
groups:
  - name: data_platform
    rules:
      - alert: PipelineStaleness
        expr: time() - pipeline_last_successful_run_timestamp{stage="silver_orders"} > 3600
        for: 5m
        annotations:
          summary: "Silver orders pipeline hasn't run in 1+ hour"
      
      - alert: HighQualityErrors
        expr: rate(pipeline_quality_errors_total[1h]) > 100
        annotations:
          summary: "High data quality error rate"
      
      - alert: SlowPipeline
        expr: pipeline_processing_duration_seconds{quantile="0.99"} > 1800
        annotations:
          summary: "Pipeline P99 latency > 30 minutes"
*/
```

---

## Step 980: Delta Lake Optimization

```scala
// optimization/DeltaOptimizer.scala
import org.apache.spark.sql.SparkSession
import io.delta.tables.DeltaTable

class DeltaOptimizer(spark: SparkSession, config: PlatformConfig) {
  
  // ===== Run nightly optimization =====
  def optimizeAll(): Unit = {
    val tables = List(
      s"${config.s3Bucket}/bronze/orders",
      s"${config.s3Bucket}/silver/orders",
      s"${config.s3Bucket}/gold/daily_revenue"
    )
    
    tables.foreach { path =>
      println(s"Optimizing: $path")
      
      val dt = DeltaTable.forPath(spark, path)
      
      // OPTIMIZE: compact small files into 1GB files
      spark.sql(s"OPTIMIZE delta.`$path`")
      
      // Z-ORDER for common query patterns
      if (path.contains("orders")) {
        spark.sql(s"OPTIMIZE delta.`$path` ZORDER BY (user_id, created_at)")
      }
      if (path.contains("daily_revenue")) {
        spark.sql(s"OPTIMIZE delta.`$path` ZORDER BY (date)")
      }
      
      // VACUUM: remove old data files (keep 7 days for time travel)
      dt.vacuum(168)  // hours = 7 days
      
      println(s"Optimization complete: $path")
    }
  }
  
  // ===== Show table statistics =====
  def showStats(path: String): Unit = {
    val dt = DeltaTable.forPath(spark, path)
    dt.detail().show(truncate = false)
    dt.history(10).show(truncate = false)
  }
  
  // ===== Auto-optimize configuration =====
  def configureAutoOptimize(path: String): Unit = {
    spark.sql(s"""
      ALTER TABLE delta.`$path` SET TBLPROPERTIES (
        'delta.autoOptimize.optimizeWrite' = 'true',
        'delta.autoOptimize.autoCompact'   = 'true',
        'delta.dataSkippingNumIndexedCols' = '5'
      )
    """)
  }
}
```

---

## สรุป Part 98: Real-World Data Platform

| Layer | Technology | Latency | Storage |
|-------|------------|---------|---------|
| Bronze | Spark Streaming + Delta | 1-2 min | Raw |
| Silver | Spark Streaming + Delta | 2-5 min | Cleaned |
| Gold | Spark Batch (hourly) + Delta | 1 hour | Aggregated |
| Serving | Play + Redis | < 10ms | Cached |

---

## แบบฝึกหัด Part 98

1. **Bronze Ingest**: implement Kafka → Delta Lake pipeline ที่ ingest order events ด้วย schema validation, deduplication, และ partition by date

2. **Silver Transform**: implement Bronze → Silver transformation ที่ parse JSON, clean nulls, deduplicate, และ enrich ด้วย product names

3. **Gold Revenue**: compute hourly revenue aggregation จาก Silver orders และ write ไปยัง Gold layer ด้วย proper partitioning

4. **Data Quality**: implement `DataQualityMonitor` ที่ check null rates, duplicate rates, invalid status values และ publish metrics ไปยัง Prometheus

5. **Real-time Dashboard**: implement Play Framework API endpoints ที่ serve metrics จาก Gold layer ด้วย Redis caching (TTL 60s)

---

## ไปต่อ: Part 99 — Open Source Contributions
[→ Part 99: Open Source Contributions](./part-99-open-source-contributions.md)
