# Part 73: Spark DataFrames และ Dataset API — Steps 721-730

## บทนำ: DataFrame vs RDD

DataFrame คือ distributed collection of data organized into named columns เหมือน table ใน relational database หรือ pandas DataFrame ใน Python มี schema และ Catalyst optimizer ที่ทำให้เร็วกว่า RDD มาก

---

## Step 721: DataFrame Basics

```scala
// DataFrameBasics.scala
import org.apache.spark.sql.{SparkSession, DataFrame, Row}
import org.apache.spark.sql.types._
import org.apache.spark.sql.functions._

object DataFrameBasics {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("DataFrame Basics")
      .config("spark.sql.shuffle.partitions", "8")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== สร้าง DataFrame หลายวิธี =====
    
    // 1. จาก Seq[Tuple]
    val df1 = Seq(
      (1, "Alice", 30, "Engineering"),
      (2, "Bob",   25, "Marketing"),
      (3, "Carol", 35, "Engineering")
    ).toDF("id", "name", "age", "dept")
    
    // 2. จาก case class
    case class Employee(id: Int, name: String, age: Int, dept: String, salary: Double)
    
    val employees = Seq(
      Employee(1, "Alice", 30, "Engineering", 80000),
      Employee(2, "Bob",   25, "Marketing",   60000),
      Employee(3, "Carol", 35, "Engineering", 95000),
      Employee(4, "Dave",  28, "Marketing",   65000),
      Employee(5, "Eve",   32, "Engineering", 88000)
    ).toDS() // Dataset[Employee]
    
    val df2 = employees.toDF()
    
    // 3. จาก Row และ schema (explicit schema)
    val schema = StructType(List(
      StructField("id",     IntegerType,    nullable = false),
      StructField("name",   StringType,     nullable = true),
      StructField("age",    IntegerType,    nullable = true),
      StructField("dept",   StringType,     nullable = true),
      StructField("salary", DoubleType,     nullable = true)
    ))
    
    val rows = List(
      Row(1, "Alice", 30, "Engineering", 80000.0),
      Row(2, "Bob",   25, "Marketing",   60000.0)
    )
    
    val df3 = spark.createDataFrame(
      spark.sparkContext.parallelize(rows),
      schema
    )
    
    // 4. อ่านจากไฟล์
    // val df4 = spark.read
    //   .option("header", "true")
    //   .option("inferSchema", "true")
    //   .csv("/path/to/data.csv")
    
    // ===== DataFrame Operations =====
    println("=== Schema ===")
    df2.printSchema()
    
    println("=== Show Data ===")
    df2.show()
    
    // select: เลือก columns
    println("=== Select ===")
    df2.select("name", "dept", "salary").show()
    
    // select ด้วย Column expressions
    df2.select(
      $"name",
      $"salary",
      ($"salary" * 1.1).as("new_salary"),
      ($"age" + 1).as("next_age")
    ).show()
    
    // filter / where
    println("=== Filter ===")
    df2.filter($"dept" === "Engineering" && $"salary" > 80000).show()
    df2.where("dept = 'Engineering' AND salary > 80000").show() // SQL style
    
    // orderBy / sort
    println("=== Sort ===")
    df2.orderBy($"salary".desc, $"name".asc).show()
    
    // limit
    df2.limit(3).show()
    
    // distinct
    df2.select("dept").distinct().show()
    
    // withColumn: เพิ่ม/แก้ column
    val enriched = df2
      .withColumn("seniority",
        when($"age" < 28, "Junior")
          .when($"age" < 33, "Mid")
          .otherwise("Senior"))
      .withColumn("bonus", $"salary" * 0.1)
      .withColumn("total_comp", $"salary" + $"salary" * 0.1)
    
    println("=== Enriched ===")
    enriched.show()
    
    // drop column
    enriched.drop("bonus").show()
    
    // rename column
    df2.withColumnRenamed("dept", "department").show()
    
    spark.stop()
  }
}
```

---

## Step 722: Schema Inference vs Explicit Schema

```scala
// SchemaManagement.scala
import org.apache.spark.sql.{SparkSession, DataFrame}
import org.apache.spark.sql.types._
import org.apache.spark.sql.functions._
import java.io.File
import java.io.PrintWriter

object SchemaManagement {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Schema Management")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    
    // สร้าง test CSV file
    val csvContent = """id,name,age,salary,hire_date,is_active
1,Alice,30,80000.50,2020-01-15,true
2,Bob,25,60000.00,2022-06-01,true
3,Carol,35,95000.75,2018-03-20,false
4,Dave,28,65000.00,2021-09-10,true
5,invalid_row,abc,not_a_number,bad_date,maybe"""
    
    val csvFile = "/tmp/employees.csv"
    new PrintWriter(csvFile) { write(csvContent); close() }
    
    // ===== Schema Inference (ช้าใน production) =====
    val inferredDF = spark.read
      .option("header", "true")
      .option("inferSchema", "true") // สแกนทั้ง file เพื่อ infer schema
      .csv(csvFile)
    
    println("=== Inferred Schema ===")
    inferredDF.printSchema()
    // อาจ infer ผิดสำหรับ column ที่มีข้อมูล invalid
    
    // ===== Explicit Schema (แนะนำสำหรับ production) =====
    val schema = StructType(Array(
      StructField("id",        IntegerType, nullable = true),
      StructField("name",      StringType,  nullable = true),
      StructField("age",       IntegerType, nullable = true),
      StructField("salary",    DoubleType,  nullable = true),
      StructField("hire_date", DateType,    nullable = true),
      StructField("is_active", BooleanType, nullable = true)
    ))
    
    val typedDF = spark.read
      .option("header", "true")
      .option("dateFormat", "yyyy-MM-dd")
      .schema(schema) // ไม่ต้อง scan ทั้งไฟล์
      .csv(csvFile)
    
    println("=== Explicit Schema ===")
    typedDF.printSchema()
    typedDF.show()
    
    // Row ที่ parse ไม่ได้จะเป็น null
    println("=== Null Rows ===")
    typedDF.filter($"salary".isNull || $"age".isNull).show()
    
    // ===== Schema Evolution =====
    // อ่าน schema จาก JSON
    val schemaJson = schema.json
    println(s"\nSchema as JSON:\n$schemaJson")
    
    // Restore schema จาก JSON
    val restoredSchema = DataType.fromJson(schemaJson).asInstanceOf[StructType]
    
    // ===== Complex Types =====
    import spark.implicits._
    
    // Array type
    val arrayData = Seq(
      (1, Array("spark", "scala", "kafka")),
      (2, Array("python", "pandas")),
      (3, Array("java", "spring", "hibernate"))
    ).toDF("id", "skills")
    
    arrayData.printSchema()
    
    // explode array
    arrayData.select($"id", explode($"skills").as("skill")).show()
    
    // Map type
    val mapData = Seq(
      (1, Map("math" -> 90, "science" -> 85, "english" -> 88)),
      (2, Map("math" -> 75, "science" -> 80, "english" -> 92))
    ).toDF("student_id", "scores")
    
    mapData.printSchema()
    
    // access map elements
    mapData.select(
      $"student_id",
      $"scores"("math").as("math_score"),
      map_keys($"scores").as("subjects"),
      map_values($"scores").as("all_scores")
    ).show()
    
    // Struct type (nested)
    case class Address(street: String, city: String, zipcode: String)
    case class Person(id: Int, name: String, address: Address)
    
    val personData = Seq(
      Person(1, "Alice", Address("123 Main St", "Bangkok",     "10110")),
      Person(2, "Bob",   Address("456 Oak Ave", "Chiang Mai", "50000"))
    ).toDF()
    
    personData.printSchema()
    
    // access nested fields
    personData.select(
      $"id",
      $"name",
      $"address.city".as("city"),
      $"address.zipcode".as("zip")
    ).show()
    
    spark.stop()
  }
}
```

---

## Step 723: Column Operations และ Functions

```scala
// ColumnOperations.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._
import org.apache.spark.sql.Column

object ColumnOperations {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Column Operations")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val data = Seq(
      (1, "Alice Smith",  30, 80000.0,  "2020-01-15", "Engineering"),
      (2, "Bob Jones",    25, 60000.0,  "2022-06-01", "Marketing"),
      (3, "Carol White",  35, 95000.0,  "2018-03-20", "Engineering"),
      (4, "Dave Brown",   28, 65000.0,  "2021-09-10", "Marketing"),
      (5, "Eve Davis",    null, 88000.0, "2019-11-05", "Engineering"),
      (6, "Frank Miller", 40, null,      "2015-07-22", "HR")
    ).toDF("id", "full_name", "age", "salary", "hire_date_str", "dept")
    
    // ===== String Functions =====
    println("=== String Functions ===")
    data.select(
      $"full_name",
      upper($"full_name").as("upper"),
      lower($"full_name").as("lower"),
      length($"full_name").as("length"),
      split($"full_name", " ")(0).as("first_name"),
      split($"full_name", " ")(1).as("last_name"),
      trim($"full_name").as("trimmed"),
      regexp_replace($"full_name", " ", "_").as("underscored"),
      regexp_extract($"full_name", "^(\\w+)", 1).as("first_word"),
      substring($"full_name", 1, 5).as("first_5_chars"),
      concat($"dept", lit(" - "), $"full_name").as("dept_name"),
      lpad($"id".cast("string"), 4, "0").as("padded_id"),
      initcap(lower($"full_name")).as("proper_case")
    ).show(truncate = false)
    
    // ===== Numeric Functions =====
    println("=== Numeric Functions ===")
    data.filter($"salary".isNotNull).select(
      $"full_name",
      $"salary",
      round($"salary", -3).as("rounded"),
      floor($"salary" / 10000).as("salary_bracket"),
      abs($"salary" - 75000.0).as("diff_from_avg"),
      pow($"salary" / 100000.0, 2).as("salary_squared"),
      greatest(lit(0), $"salary" - 80000.0).as("premium")
    ).show()
    
    // ===== Date Functions =====
    println("=== Date Functions ===")
    val withDate = data.withColumn("hire_date", to_date($"hire_date_str", "yyyy-MM-dd"))
    
    withDate.select(
      $"full_name",
      $"hire_date",
      year($"hire_date").as("year"),
      month($"hire_date").as("month"),
      dayofmonth($"hire_date").as("day"),
      dayofweek($"hire_date").as("day_of_week"),
      datediff(current_date(), $"hire_date").as("days_employed"),
      months_between(current_date(), $"hire_date").cast("int").as("months_employed"),
      date_add($"hire_date", 90).as("probation_end"),
      date_format($"hire_date", "MMMM dd, yyyy").as("formatted_date"),
      current_timestamp().as("now"),
      unix_timestamp($"hire_date_str", "yyyy-MM-dd").as("unix_ts")
    ).show(truncate = false)
    
    // ===== Conditional Functions =====
    println("=== Conditional Functions ===")
    data.select(
      $"full_name",
      $"age",
      $"salary",
      // when-otherwise
      when($"age" < 28, "Junior")
        .when($"age" < 33, "Mid-level")
        .when($"age" < 40, "Senior")
        .otherwise("Principal")
        .as("seniority"),
      
      // coalesce: first non-null value
      coalesce($"age", lit(0)).as("age_or_0"),
      coalesce($"salary", lit(50000.0)).as("salary_or_default"),
      
      // nullif: return null if values are equal
      nullif($"dept", lit("HR")).as("non_hr_dept"),
      
      // isnull / isnotnull
      $"age".isNull.as("age_missing"),
      $"salary".isNotNull.as("has_salary"),
      
      // if-else ด้วย Column API
      ($"dept" === "Engineering").as("is_engineer")
    ).show(truncate = false)
    
    // ===== Array Functions =====
    println("=== Array Functions ===")
    val arrayDF = Seq(
      (1, Array(1, 2, 3, 4, 5)),
      (2, Array(10, 20, 30)),
      (3, Array[Int]())
    ).toDF("id", "nums")
    
    arrayDF.select(
      $"id",
      $"nums",
      size($"nums").as("size"),
      array_contains($"nums", 3).as("contains_3"),
      sort_array($"nums", asc = false).as("sorted_desc"),
      slice($"nums", 1, 3).as("first_3"),
      array_distinct($"nums").as("distinct"),
      array_min($"nums").as("min"),
      array_max($"nums").as("max"),
      aggregate($"nums", lit(0), (acc, x) => acc + x).as("sum")
    ).show(truncate = false)
    
    spark.stop()
  }
}
```

---

## Step 724: Grouping และ Aggregations

```scala
// GroupingAggregations.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._
import org.apache.spark.sql.expressions.Window

object GroupingAggregations {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Grouping Aggregations")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val salesData = Seq(
      ("2024-01", "Electronics", "iPhone",  100, 999.99, "North"),
      ("2024-01", "Electronics", "Laptop",  50,  1299.99, "South"),
      ("2024-01", "Clothing",    "T-Shirt", 200, 29.99,   "North"),
      ("2024-02", "Electronics", "iPhone",  120, 999.99,  "East"),
      ("2024-02", "Electronics", "Laptop",  60,  1299.99, "North"),
      ("2024-02", "Clothing",    "Jeans",   150, 59.99,   "South"),
      ("2024-03", "Electronics", "Tablet",  80,  599.99,  "North"),
      ("2024-03", "Clothing",    "T-Shirt", 250, 29.99,   "East"),
      ("2024-03", "Food",        "Coffee",  500, 5.99,    "South")
    ).toDF("month", "category", "product", "units", "price", "region")
    
    val sales = salesData.withColumn("revenue", $"units" * $"price")
    
    // ===== Basic GroupBy =====
    println("=== Revenue by Category ===")
    sales.groupBy($"category")
      .agg(
        sum($"revenue").as("total_revenue"),
        sum($"units").as("total_units"),
        countDistinct($"product").as("products"),
        avg($"price").as("avg_price"),
        max($"price").as("max_price"),
        min($"price").as("min_price")
      )
      .orderBy($"total_revenue".desc)
      .show()
    
    // ===== Multi-level GroupBy =====
    println("=== Revenue by Month and Category ===")
    sales.groupBy($"month", $"category")
      .agg(sum($"revenue").as("revenue"))
      .orderBy($"month", $"category")
      .show()
    
    // ===== Pivot =====
    println("=== Revenue Pivot (month x category) ===")
    sales.groupBy($"month")
      .pivot($"category")
      .agg(round(sum($"revenue"), 0))
      .orderBy($"month")
      .show()
    
    // กำหนด values ที่ต้องการ pivot (เร็วกว่า เพราะไม่ต้อง scan หา distinct values)
    sales.groupBy($"month")
      .pivot($"category", Seq("Electronics", "Clothing", "Food"))
      .agg(sum($"units"))
      .orderBy($"month")
      .show()
    
    // ===== rollup และ cube =====
    println("=== Rollup ===")
    // rollup: สร้าง aggregations ที่ levels ต่างๆ (hierarchical)
    sales.rollup($"region", $"category")
      .agg(sum($"revenue").as("revenue"))
      .orderBy($"region".asc_nulls_last, $"category".asc_nulls_last)
      .show()
    
    println("=== Cube ===")
    // cube: สร้าง aggregations ทุก combination
    sales.cube($"region", $"category")
      .agg(sum($"revenue").as("revenue"))
      .orderBy($"region".asc_nulls_last, $"category".asc_nulls_last)
      .show()
    
    // ===== collect_list and collect_set =====
    println("=== Collect Operations ===")
    sales.groupBy($"category")
      .agg(
        collect_list($"product").as("all_products"),
        collect_set($"product").as("unique_products"),
        array_join(collect_set($"product"), ", ").as("products_str")
      )
      .show(truncate = false)
    
    // ===== Complex Aggregations =====
    println("=== Summary Statistics ===")
    sales.agg(
      count("*").as("total_rows"),
      countDistinct($"product").as("unique_products"),
      sum($"revenue").as("total_revenue"),
      avg($"revenue").as("avg_revenue"),
      stddev($"revenue").as("stddev_revenue"),
      percentile_approx($"revenue", 0.5).as("median_revenue"),
      percentile_approx($"revenue", array(lit(0.25), lit(0.75))).as("quartiles")
    ).show(truncate = false)
    
    spark.stop()
  }
}
```

---

## Step 725: Window Functions

```scala
// WindowFunctions.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._
import org.apache.spark.sql.expressions.{Window, WindowSpec}

object WindowFunctions {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Window Functions")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val empSales = Seq(
      ("Alice", "Engineering", 2024, 1, 50000.0),
      ("Alice", "Engineering", 2024, 2, 55000.0),
      ("Alice", "Engineering", 2024, 3, 60000.0),
      ("Alice", "Engineering", 2024, 4, 58000.0),
      ("Bob",   "Engineering", 2024, 1, 45000.0),
      ("Bob",   "Engineering", 2024, 2, 48000.0),
      ("Bob",   "Engineering", 2024, 3, 52000.0),
      ("Bob",   "Engineering", 2024, 4, 55000.0),
      ("Carol", "Marketing",   2024, 1, 40000.0),
      ("Carol", "Marketing",   2024, 2, 42000.0),
      ("Carol", "Marketing",   2024, 3, 45000.0),
      ("Carol", "Marketing",   2024, 4, 44000.0),
      ("Dave",  "Marketing",   2024, 1, 38000.0),
      ("Dave",  "Marketing",   2024, 2, 40000.0),
      ("Dave",  "Marketing",   2024, 3, 43000.0),
      ("Dave",  "Marketing",   2024, 4, 41000.0)
    ).toDF("employee", "dept", "year", "quarter", "sales")
    
    // ===== Window Specs =====
    
    // Partition by dept, order by quarter
    val deptWindow = Window.partitionBy($"dept", $"year")
                           .orderBy($"quarter")
    
    // Partition by employee, full partition
    val empWindow = Window.partitionBy($"employee", $"year")
                          .orderBy($"quarter")
    
    // Unbounded window (whole partition)
    val deptFullWindow = Window.partitionBy($"dept", $"year")
    
    // ===== Ranking Functions =====
    println("=== Ranking Functions ===")
    empSales.withColumn("rank",       rank().over(deptWindow))
            .withColumn("dense_rank", dense_rank().over(deptWindow))
            .withColumn("row_number", row_number().over(deptWindow))
            .withColumn("ntile",      ntile(2).over(deptWindow))
            .show()
    
    // ===== Cumulative/Running Functions =====
    println("=== Running Totals ===")
    empSales.withColumn("running_total",  sum($"sales").over(empWindow))
            .withColumn("running_avg",    avg($"sales").over(empWindow))
            .withColumn("cumulative_max", max($"sales").over(empWindow))
            .withColumn("pct_of_annual",  
              $"sales" / sum($"sales").over(empWindow.rowsBetween(
                Window.unboundedPreceding, Window.unboundedFollowing
              )) * 100)
            .show()
    
    // ===== Lead และ Lag =====
    println("=== Lead and Lag ===")
    empSales.withColumn("prev_quarter_sales", lag($"sales", 1).over(empWindow))
            .withColumn("next_quarter_sales", lead($"sales", 1).over(empWindow))
            .withColumn("qoq_growth",
              ($"sales" - lag($"sales", 1).over(empWindow)) /
              lag($"sales", 1).over(empWindow) * 100)
            .show()
    
    // ===== Sliding Windows =====
    println("=== Moving Averages ===")
    // 2-quarter moving average
    val movingWindow = empWindow.rowsBetween(-1, 0) // previous row + current row
    
    empSales.withColumn("2q_moving_avg", avg($"sales").over(movingWindow))
            .show()
    
    // ===== Rankings and Percentiles =====
    println("=== Dept Rankings ===")
    empSales.withColumn("dept_rank",
              rank().over(Window.partitionBy($"dept", $"year", $"quarter")
                              .orderBy($"sales".desc)))
            .withColumn("pct_rank",
              percent_rank().over(Window.partitionBy($"dept", $"year")
                                       .orderBy($"sales")))
            .show()
    
    // ===== Top N per Group =====
    println("=== Top Earner per Dept per Quarter ===")
    empSales
      .withColumn("rank", rank().over(
        Window.partitionBy($"dept", $"quarter").orderBy($"sales".desc)
      ))
      .filter($"rank" === 1)
      .drop("rank")
      .show()
    
    spark.stop()
  }
}
```

---

## Step 726: Dataset[T] — Type-Safe API

```scala
// DatasetAPI.scala
import org.apache.spark.sql.{SparkSession, Dataset, DataFrame}
import org.apache.spark.sql.functions._

// Domain model
case class Order(
  orderId: String,
  customerId: String,
  productId: String,
  category: String,
  quantity: Int,
  unitPrice: Double,
  orderDate: java.sql.Date,
  status: String
)

case class OrderSummary(
  customerId: String,
  totalOrders: Long,
  totalSpend: Double,
  avgOrderValue: Double,
  favoriteCategory: String
)

object DatasetAPI {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Dataset API")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== สร้าง Dataset[Order] =====
    val orders: Dataset[Order] = Seq(
      Order("O001", "C001", "P001", "Electronics", 2, 999.99,
            java.sql.Date.valueOf("2024-01-15"), "completed"),
      Order("O002", "C002", "P002", "Clothing",    1, 49.99,
            java.sql.Date.valueOf("2024-01-16"), "completed"),
      Order("O003", "C001", "P003", "Electronics", 1, 1299.99,
            java.sql.Date.valueOf("2024-02-01"), "completed"),
      Order("O004", "C003", "P001", "Electronics", 3, 999.99,
            java.sql.Date.valueOf("2024-02-05"), "cancelled"),
      Order("O005", "C002", "P004", "Food",        5, 9.99,
            java.sql.Date.valueOf("2024-02-10"), "completed")
    ).toDS()
    
    // ===== Type-Safe Operations =====
    // compile-time type checking!
    
    // map: แปลง type
    val revenueByOrder: Dataset[(String, Double)] = orders
      .filter(_.status == "completed")
      .map(order => (order.orderId, order.quantity * order.unitPrice))
    
    println("=== Revenue by Order ===")
    revenueByOrder.show()
    
    // flatMap
    val customerProductPairs: Dataset[(String, String)] = orders
      .flatMap(order => 
        if (order.status == "completed")
          Some((order.customerId, order.productId))
        else
          None
      )
    
    // groupByKey - type-safe grouping
    val ordersByCustomer = orders
      .filter(_.status == "completed")
      .groupByKey(_.customerId)
      .mapGroups { (customerId, ordersIter) =>
        val ordersList = ordersIter.toList
        val totalSpend = ordersList.map(o => o.quantity * o.unitPrice).sum
        val categories = ordersList.map(_.category)
        val favCat = categories.groupBy(identity).maxBy(_._2.size)._1
        
        OrderSummary(
          customerId    = customerId,
          totalOrders   = ordersList.size,
          totalSpend    = totalSpend,
          avgOrderValue = totalSpend / ordersList.size,
          favoriteCategory = favCat
        )
      }
    
    println("=== Customer Summary (type-safe) ===")
    ordersByCustomer.show()
    ordersByCustomer.printSchema()
    
    // ===== Dataset vs DataFrame =====
    // Dataset[T]: type-safe, slower serialization
    // DataFrame (= Dataset[Row]): untyped, faster (Catalyst optimizer)
    
    // แปลงระหว่างกัน
    val df: DataFrame = orders.toDF()           // Dataset -> DataFrame
    val ds: Dataset[Order] = df.as[Order]       // DataFrame -> Dataset
    
    // ===== Typed Actions =====
    orders.filter(_.status == "completed").foreach { order =>
      // type-safe access
      println(s"Order ${order.orderId}: ${order.quantity} x ${order.unitPrice}")
    }
    
    val completedOrders: Array[Order] = orders
      .filter(_.status == "completed")
      .collect()
    
    println(s"Completed orders: ${completedOrders.length}")
    
    // reduce - type-safe
    val totalRevenue = orders
      .filter(_.status == "completed")
      .map(o => o.quantity * o.unitPrice)
      .reduce(_ + _)
    
    println(f"Total Revenue: $$${totalRevenue}%.2f")
    
    // ===== Join Datasets =====
    case class Product(productId: String, name: String, brand: String)
    
    val products: Dataset[Product] = Seq(
      Product("P001", "iPhone 15", "Apple"),
      Product("P002", "T-Shirt",   "Brand A"),
      Product("P003", "MacBook",   "Apple"),
      Product("P004", "Coffee",    "Local")
    ).toDS()
    
    // joinWith: preserves type safety
    val ordersWithProducts = orders.joinWith(
      products,
      orders("productId") === products("productId"),
      "left"
    )
    
    ordersWithProducts.map { case (order, product) =>
      s"${order.orderId}: ${if (product != null) product.name else "Unknown"}"
    }.show()
    
    spark.stop()
  }
}
```

---

## Step 727: Reading และ Writing Data

```scala
// DataIO.scala
import org.apache.spark.sql.{SparkSession, SaveMode}
import org.apache.spark.sql.types._
import org.apache.spark.sql.functions._

object DataIO {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Data I/O")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // สร้างข้อมูล sample
    val df = Seq(
      (1, "Alice", "Engineering", 80000.0, "2020-01-15"),
      (2, "Bob",   "Marketing",   60000.0, "2022-06-01"),
      (3, "Carol", "Engineering", 95000.0, "2018-03-20")
    ).toDF("id", "name", "dept", "salary", "hire_date")
    
    // ===== CSV =====
    df.write
      .mode(SaveMode.Overwrite)
      .option("header", "true")
      .option("delimiter", ",")
      .csv("/tmp/output/csv")
    
    val csvDF = spark.read
      .option("header", "true")
      .option("inferSchema", "true")
      .csv("/tmp/output/csv")
    
    // ===== JSON =====
    df.write
      .mode(SaveMode.Overwrite)
      .json("/tmp/output/json")
    
    val jsonDF = spark.read
      .option("multiLine", "true")
      .json("/tmp/output/json")
    
    // ===== Parquet (แนะนำสำหรับ Spark) =====
    df.write
      .mode(SaveMode.Overwrite)
      .parquet("/tmp/output/parquet")
    
    val parquetDF = spark.read
      .parquet("/tmp/output/parquet")
    
    // Parquet ด้วย partitioning
    df.write
      .mode(SaveMode.Overwrite)
      .partitionBy("dept") // สร้าง directory structure: dept=Engineering, dept=Marketing
      .parquet("/tmp/output/parquet-partitioned")
    
    // อ่าน specific partition
    val engOnly = spark.read
      .parquet("/tmp/output/parquet-partitioned")
      .filter($"dept" === "Engineering") // Partition pruning!
    
    // ===== ORC =====
    df.write
      .mode(SaveMode.Overwrite)
      .orc("/tmp/output/orc")
    
    // ===== Avro (ต้องเพิ่ม dependency) =====
    // df.write.format("avro").save("/tmp/output/avro")
    
    // ===== Delta Lake (ต้องเพิ่ม dependency) =====
    // df.write.format("delta").save("/tmp/output/delta")
    
    // ===== JDBC =====
    // df.write
    //   .jdbc("jdbc:postgresql://localhost/mydb", "employees", jdbcProperties)
    
    // ===== SaveMode Options =====
    /*
    SaveMode.Overwrite    - เขียนทับ
    SaveMode.Append       - เพิ่มเข้า
    SaveMode.Ignore       - ไม่ทำอะไรถ้ามีอยู่แล้ว
    SaveMode.ErrorIfExists - throw error ถ้ามีอยู่แล้ว (default)
    */
    
    // ===== Format Comparison =====
    println("""
      Format Comparison:
      ┌──────────┬─────────┬───────────┬───────────┬────────────────┐
      │ Format   │ Size    │ Read Speed│ Write Speed│ Use Case       │
      ├──────────┼─────────┼───────────┼───────────┼────────────────┤
      │ CSV      │ Large   │ Slow      │ Medium    │ Exchange, debug │
      │ JSON     │ Large   │ Slow      │ Slow      │ API, nested    │
      │ Parquet  │ Small   │ Fast      │ Medium    │ Analytics      │
      │ ORC      │ Smaller │ Fast      │ Fast      │ Hive, Hadoop   │
      │ Avro     │ Medium  │ Medium    │ Fast      │ Kafka, schemas │
      │ Delta    │ Small   │ Fast      │ Fast      │ ACID, lake     │
      └──────────┴─────────┴───────────┴───────────┴────────────────┘
    """)
    
    spark.stop()
  }
}
```

---

## Step 728: UDFs — User Defined Functions

```scala
// UDFDemo.scala
import org.apache.spark.sql.{SparkSession, functions => F}
import org.apache.spark.sql.functions._
import org.apache.spark.sql.expressions.UserDefinedFunction

object UDFDemo {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("UDF Demo")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val data = Seq(
      (1, "สมชาย ใจดี",      "+66812345678", "100 Main St, Bangkok, 10110"),
      (2, "สมหญิง รักชาติ",   "0898765432",   "200 Oak Ave, Chiang Mai, 50000"),
      (3, "มานะ เก่งมาก",     "invalid",       "300 Pine Rd, Phuket, 83000")
    ).toDF("id", "name", "phone", "address")
    
    // ===== Simple UDF =====
    val normalizePhone = udf((phone: String) => {
      phone.replaceAll("[^0-9]", "") match {
        case p if p.startsWith("66") && p.length == 11 => s"0${p.substring(2)}"
        case p if p.length == 10 => p
        case _ => null
      }
    })
    
    // Register สำหรับ SQL
    spark.udf.register("normalize_phone", normalizePhone)
    
    data.withColumn("normalized_phone", normalizePhone($"phone")).show(truncate = false)
    
    // ===== UDF with Option (null-safe) =====
    val parseAddress = udf((address: String) => {
      Option(address).map { addr =>
        addr.split(",").map(_.trim) match {
          case Array(street, city, zip) => Map(
            "street" -> street,
            "city"   -> city,
            "zip"    -> zip
          )
          case _ => Map("raw" -> addr)
        }
      }.orNull
    })
    
    data.withColumn("parsed_address", parseAddress($"address"))
        .select($"id", $"parsed_address"("city").as("city"))
        .show()
    
    // ===== UDF returning Array =====
    val tokenize = udf((text: String) =>
      Option(text).map(_.split("\\s+").toSeq).getOrElse(Seq.empty[String])
    )
    
    data.withColumn("name_tokens", tokenize($"name"))
        .select($"id", $"name_tokens", size($"name_tokens").as("word_count"))
        .show(truncate = false)
    
    // ===== Pandas-style UDF (Vectorized UDF) - ต้องมี pandas =====
    // import org.apache.spark.sql.pandas.functions._
    // val vectorizedUDF = pandas_udf((s: pandas.Series) => s.str.upper(), StringType)
    
    // ===== เมื่อไหร่ควรใช้ UDF vs built-in functions =====
    /*
    ใช้ built-in functions (functions._) เมื่อ:
    - มี built-in ที่ทำงานที่ต้องการ
    - ต้องการ performance สูงสุด (Catalyst optimizable)
    
    ใช้ UDF เมื่อ:
    - ต้องการ logic ที่ซับซ้อน
    - ไม่มี built-in equivalent
    
    หลีกเลี่ยง UDF เมื่อ:
    - Hot path ใน large dataset
    - สามารถใช้ built-in ได้
    
    UDF ปิด Catalyst optimization ทำให้ช้ากว่า built-in
    */
    
    // เปรียบเทียบ performance
    val bigData = spark.range(1000000).toDF("value")
    
    // Built-in: Catalyst can optimize
    val builtinResult = bigData.withColumn("result", 
      when($"value" % 2 === 0, "even").otherwise("odd"))
    
    // UDF: no Catalyst optimization
    val evenOddUDF = udf((v: Long) => if (v % 2 == 0) "even" else "odd")
    val udfResult = bigData.withColumn("result", evenOddUDF($"value"))
    
    // Built-in เร็วกว่า UDF สำหรับ simple operations
    println("Built-in count: " + builtinResult.count())
    println("UDF count: " + udfResult.count())
    
    spark.stop()
  }
}
```

---

## Step 729: Joins

```scala
// JoinOperations.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._

object JoinOperations {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Join Operations")
      .config("spark.sql.shuffle.partitions", "4")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val customers = Seq(
      (1, "Alice",   "Premium"),
      (2, "Bob",     "Standard"),
      (3, "Carol",   "Premium"),
      (4, "Dave",    "Standard"),
      (5, "Eve",     "Premium")
    ).toDF("customer_id", "name", "tier")
    
    val orders = Seq(
      (101, 1, 500.0,  "2024-01"),
      (102, 2, 200.0,  "2024-01"),
      (103, 1, 800.0,  "2024-02"),
      (104, 3, 1200.0, "2024-02"),
      (105, 6, 300.0,  "2024-02"), // customer_id=6 ไม่มีใน customers
      (106, 2, 450.0,  "2024-03")
    ).toDF("order_id", "customer_id", "amount", "month")
    
    val products = Seq(
      (101, "iPhone",  "Electronics"),
      (102, "T-Shirt", "Clothing"),
      (103, "Laptop",  "Electronics"),
      (104, "Coffee",  "Food")
    ).toDF("order_id", "product", "category")
    
    // ===== Inner Join =====
    println("=== Inner Join ===")
    customers.join(orders, "customer_id")
             .show()
    
    // ===== Left Outer Join =====
    println("=== Left Outer Join (all customers) ===")
    customers.join(orders, Seq("customer_id"), "left_outer")
             .show()
    
    // ===== Right Outer Join =====
    println("=== Right Outer Join (all orders) ===")
    customers.join(orders, Seq("customer_id"), "right_outer")
             .show()
    
    // ===== Full Outer Join =====
    println("=== Full Outer Join ===")
    customers.join(orders, Seq("customer_id"), "full_outer")
             .show()
    
    // ===== Cross Join =====
    println("=== Cross Join (cartesian) ===")
    // customers.crossJoin(orders).show() // ระวัง: สร้าง N*M rows!
    
    // ===== Semi Join (filter based on existence) =====
    println("=== Left Semi Join (customers with orders) ===")
    customers.join(orders, Seq("customer_id"), "left_semi")
             .show()
    
    // ===== Anti Join (filter based on non-existence) =====
    println("=== Left Anti Join (customers without orders) ===")
    customers.join(orders, Seq("customer_id"), "left_anti")
             .show()
    
    // ===== Multi-table Join =====
    println("=== Multi-table Join ===")
    orders.join(customers, "customer_id")
          .join(products, "order_id")
          .select(
            $"order_id",
            $"name".as("customer"),
            $"tier",
            $"product",
            $"category",
            $"amount",
            $"month"
          )
          .show()
    
    // ===== Broadcast Join (Small table hint) =====
    println("=== Broadcast Join ===")
    // hint Spark ให้ broadcast small table
    orders.join(broadcast(customers), "customer_id")
          .explain() // ดู BroadcastHashJoin ใน plan
    
    // ===== Join Optimizations =====
    println("""
      Join Type Guide:
      ┌──────────────┬────────────────────────────────────────────┐
      │ Join Type    │ Use When                                   │
      ├──────────────┼────────────────────────────────────────────┤
      │ inner        │ Need matching rows from both sides         │
      │ left_outer   │ Keep all left + matching right             │
      │ right_outer  │ Keep all right + matching left             │
      │ full_outer   │ Keep all rows from both sides              │
      │ left_semi    │ Filter left by existence in right          │
      │ left_anti    │ Filter left by absence in right            │
      │ cross        │ All combinations (use carefully!)          │
      └──────────────┴────────────────────────────────────────────┘
      
      Performance Tips:
      - Broadcast join for small tables (< spark.sql.autoBroadcastJoinThreshold)
      - Filter before join to reduce data
      - Use same partitioning to avoid shuffle
    """)
    
    spark.stop()
  }
}
```

---

## Step 730: DataFrame Performance Optimization

```scala
// DFOptimization.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._

object DFOptimization {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("DF Optimization")
      .config("spark.sql.adaptive.enabled", "true")
      .config("spark.sql.adaptive.coalescePartitions.enabled", "true")
      .config("spark.sql.adaptive.skewJoin.enabled", "true")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== Column Pruning =====
    // อ่านเฉพาะ columns ที่ต้องการ (Parquet รองรับ column pruning)
    
    // BAD: อ่านทุก column แล้ว filter
    // val bad = df.select("*").filter($"dept" === "Engineering").select("name")
    
    // GOOD: select เฉพาะที่ต้องการ
    // val good = df.select("name", "dept").filter($"dept" === "Engineering").select("name")
    
    // ===== Predicate Pushdown =====
    // Spark pushes filters down to data source ถ้า format รองรับ (Parquet, ORC, JDBC)
    
    // ===== Adaptive Query Execution (AQE) =====
    // Spark 3.x feature: optimize plan ระหว่าง execution
    println("AQE enabled: " + spark.conf.get("spark.sql.adaptive.enabled"))
    
    // AQE features:
    // 1. Coalescing post-shuffle partitions
    // 2. Converting sort-merge join to broadcast join
    // 3. Skew join optimization
    
    // ===== Caching Strategies =====
    val data = spark.range(1000000)
      .withColumn("value", rand())
      .withColumn("category", (rand() * 5).cast("int").cast("string"))
    
    // cache ถ้าใช้หลายครั้ง
    data.cache()
    data.count() // materialize
    
    // ใช้หลายครั้ง โดยไม่ recompute
    println(s"Count: ${data.count()}")
    println(s"Categories: ${data.select("category").distinct().count()}")
    
    data.unpersist()
    
    // ===== Explain Plan =====
    val complexQuery = data
      .filter($"value" > 0.5)
      .groupBy($"category")
      .agg(avg($"value").as("avg_value"), count("*").as("cnt"))
      .orderBy($"avg_value".desc)
    
    println("=== Physical Plan ===")
    complexQuery.explain("formatted")
    
    // ===== Hint API =====
    // Broadcast hint
    val smallDF = Seq(("A", 1), ("B", 2)).toDF("cat", "val")
    data.hint("broadcast")  // hint ให้ broadcast data
    data.join(smallDF.hint("broadcast"), $"category" === $"cat").explain()
    
    // Rebalance hint (Spark 3.4+)
    data.hint("rebalance")  // hint ให้ rebalance partitions
    
    // ===== Partition Control =====
    // coalesce: ลด partitions (ไม่ shuffle)
    val reduced = data.coalesce(2)
    println(s"Coalesced partitions: ${reduced.rdd.getNumPartitions}")
    
    // repartition: เปลี่ยน partitions (มี shuffle)
    val repartitioned = data.repartition(8, $"category")
    println(s"Repartitioned: ${repartitioned.rdd.getNumPartitions}")
    
    spark.stop()
  }
}
```

---

## สรุป Part 73

| API | Type Safety | Performance | Use Case |
|-----|-------------|-------------|----------|
| RDD | Runtime | Medium | Low-level, custom logic |
| DataFrame | None (Row) | High | SQL-like operations |
| Dataset[T] | Compile-time | High | Type-safe processing |

### DataFrame vs Dataset Trade-offs

| Feature | DataFrame | Dataset[T] |
|---------|-----------|------------|
| Type Safety | No | Yes |
| Catalyst Optimization | Yes | Yes |
| Serialization | Tungsten (fast) | Java/Kryo |
| Error Detection | Runtime | Compile-time |
| API Style | Column-based | Object-based |

---

## แบบฝึกหัด Part 73

1. **Schema Engineering**: อ่าน CSV ที่มีข้อมูล invalid แบบ production-ready โดยใช้ explicit schema, handle nulls, และเขียน clean data เป็น Parquet

2. **Window Analysis**: ใช้ sales data สร้าง report ที่มี: running total, month-over-month growth %, rank within category, moving 3-month average

3. **Complex Joins**: ออกแบบ multi-table join query สำหรับ e-commerce schema (customers, orders, products, reviews) แล้ว optimize ด้วย broadcast join

4. **UDF Performance**: สร้าง UDF แล้วเปรียบเทียบ performance กับ equivalent built-in functions บน dataset ขนาด 10M rows

5. **ETL Pipeline**: สร้าง complete ETL pipeline ที่อ่านจาก JSON → validate → transform → aggregate → เขียนเป็น Parquet แบบ partitioned

---

## ไปต่อ: Part 74 — Spark SQL
ใน Part ถัดไปจะเรียน Spark SQL อย่างละเอียด รวม complex queries, subqueries, และ UDFs ใน SQL

[→ Part 74: Spark SQL](./part-74-spark-sql.md)
