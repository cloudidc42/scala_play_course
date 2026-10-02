# Part 74: Spark SQL — Steps 731-740

## บทนำ: Spark SQL

Spark SQL ทำให้สามารถเขียน SQL queries บน distributed data ได้ รองรับ ANSI SQL, HiveQL และมี Catalyst optimizer ที่ powerful มาก

---

## Step 731: createTempView และ SQL Queries

```scala
// SparkSQLBasics.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._

object SparkSQLBasics {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Spark SQL Basics")
      .config("spark.sql.shuffle.partitions", "8")
      .enableHiveSupport() // ถ้าต้องการ Hive metastore
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // สร้าง DataFrames
    val employees = Seq(
      (1, "Alice",   "Engineering", 80000.0, 30, "2020-01-15"),
      (2, "Bob",     "Marketing",   60000.0, 25, "2022-06-01"),
      (3, "Carol",   "Engineering", 95000.0, 35, "2018-03-20"),
      (4, "Dave",    "Marketing",   65000.0, 28, "2021-09-10"),
      (5, "Eve",     "Engineering", 88000.0, 32, "2019-11-05"),
      (6, "Frank",   "HR",          55000.0, 40, "2015-07-22"),
      (7, "Grace",   "Engineering", 92000.0, 33, "2017-04-18"),
      (8, "Henry",   "HR",          58000.0, 38, "2016-12-01")
    ).toDF("id", "name", "dept", "salary", "age", "hire_date")
    
    val departments = Seq(
      ("Engineering", "Technology",  "San Francisco", 500000.0),
      ("Marketing",   "Business",    "New York",      300000.0),
      ("HR",          "Operations",  "Chicago",       200000.0),
      ("Finance",     "Business",    "New York",      350000.0)
    ).toDF("dept_name", "division", "office", "budget")
    
    val projects = Seq(
      (1, "Project Alpha",   1, "Lead"),
      (2, "Project Beta",    2, "Member"),
      (3, "Project Alpha",   3, "Lead"),
      (4, "Project Gamma",   1, "Member"),
      (5, "Project Beta",    5, "Lead"),
      (6, "Project Alpha",   7, "Member")
    ).toDF("project_id", "project_name", "emp_id", "role")
    
    // ===== createOrReplaceTempView =====
    // Temp view: ใช้ได้ใน session เดียว
    employees.createOrReplaceTempView("employees")
    departments.createOrReplaceTempView("departments")
    projects.createOrReplaceTempView("projects")
    
    // ===== Global Temp View =====
    // Global: ใช้ได้ข้าม sessions แต่ต้อง prefix ด้วย global_temp
    employees.createOrReplaceGlobalTempView("global_employees")
    // ใช้ด้วย: SELECT * FROM global_temp.global_employees
    
    // ===== Basic SQL Queries =====
    println("=== Basic SELECT ===")
    spark.sql("""
      SELECT id, name, dept, salary
      FROM employees
      WHERE dept = 'Engineering'
      ORDER BY salary DESC
    """).show()
    
    println("=== Aggregation ===")
    spark.sql("""
      SELECT 
        dept,
        COUNT(*) as headcount,
        AVG(salary) as avg_salary,
        MAX(salary) as max_salary,
        MIN(salary) as min_salary,
        SUM(salary) as total_salary
      FROM employees
      GROUP BY dept
      ORDER BY avg_salary DESC
    """).show()
    
    println("=== JOIN ===")
    spark.sql("""
      SELECT 
        e.name,
        e.dept,
        e.salary,
        d.division,
        d.office,
        d.budget
      FROM employees e
      INNER JOIN departments d ON e.dept = d.dept_name
      ORDER BY e.salary DESC
    """).show()
    
    println("=== LEFT JOIN =====")
    spark.sql("""
      SELECT 
        d.dept_name,
        d.budget,
        COUNT(e.id) as employee_count,
        AVG(e.salary) as avg_salary
      FROM departments d
      LEFT JOIN employees e ON d.dept_name = e.dept
      GROUP BY d.dept_name, d.budget
      ORDER BY d.budget DESC
    """).show()
    
    spark.stop()
  }
}
```

---

## Step 732: Complex SQL Queries

```scala
// ComplexSQL.scala
import org.apache.spark.sql.SparkSession

object ComplexSQL {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Complex SQL")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // Setup data
    val orders = Seq(
      ("O001", "C001", "P001", 2, 999.99,  "2024-01-15", "completed"),
      ("O002", "C002", "P002", 1, 49.99,   "2024-01-16", "completed"),
      ("O003", "C001", "P003", 1, 1299.99, "2024-02-01", "completed"),
      ("O004", "C003", "P001", 3, 999.99,  "2024-02-05", "cancelled"),
      ("O005", "C002", "P004", 5, 9.99,    "2024-02-10", "completed"),
      ("O006", "C004", "P002", 2, 49.99,   "2024-03-01", "completed"),
      ("O007", "C001", "P005", 1, 599.99,  "2024-03-15", "completed"),
      ("O008", "C003", "P003", 1, 1299.99, "2024-03-20", "completed"),
      ("O009", "C005", "P001", 1, 999.99,  "2024-04-01", "pending"),
      ("O010", "C002", "P004", 10, 9.99,   "2024-04-05", "completed")
    ).toDF("order_id", "customer_id", "product_id", "quantity", "unit_price", "order_date", "status")
    
    val customers = Seq(
      ("C001", "Alice Smith",   "Premium",  "2020-01-01"),
      ("C002", "Bob Jones",     "Standard", "2021-06-15"),
      ("C003", "Carol White",   "Premium",  "2019-03-20"),
      ("C004", "Dave Brown",    "Standard", "2023-01-10"),
      ("C005", "Eve Davis",     "VIP",      "2018-11-05")
    ).toDF("customer_id", "name", "tier", "member_since")
    
    val products = Seq(
      ("P001", "iPhone 15",    "Electronics", 999.99,  150),
      ("P002", "T-Shirt",      "Clothing",    49.99,   500),
      ("P003", "MacBook Pro",  "Electronics", 1299.99, 80),
      ("P004", "Coffee Blend", "Food",        9.99,    1000),
      ("P005", "iPad Air",     "Electronics", 599.99,  200)
    ).toDF("product_id", "product_name", "category", "list_price", "stock")
    
    orders.createOrReplaceTempView("orders")
    customers.createOrReplaceTempView("customers")
    products.createOrReplaceTempView("products")
    
    // ===== Subqueries =====
    println("=== Subquery: Customers above average spend ===")
    spark.sql("""
      SELECT 
        c.name,
        c.tier,
        customer_stats.total_spend
      FROM customers c
      JOIN (
        SELECT 
          customer_id,
          SUM(quantity * unit_price) as total_spend
        FROM orders
        WHERE status = 'completed'
        GROUP BY customer_id
      ) customer_stats ON c.customer_id = customer_stats.customer_id
      WHERE customer_stats.total_spend > (
        SELECT AVG(total_spend)
        FROM (
          SELECT customer_id, SUM(quantity * unit_price) as total_spend
          FROM orders
          WHERE status = 'completed'
          GROUP BY customer_id
        )
      )
      ORDER BY total_spend DESC
    """).show()
    
    // ===== CTEs (Common Table Expressions) =====
    println("=== CTE: Customer Lifetime Value ===")
    spark.sql("""
      WITH 
        completed_orders AS (
          SELECT *
          FROM orders
          WHERE status = 'completed'
        ),
        customer_stats AS (
          SELECT 
            customer_id,
            COUNT(DISTINCT order_id) as order_count,
            SUM(quantity * unit_price) as total_spend,
            AVG(quantity * unit_price) as avg_order_value,
            MIN(order_date) as first_order,
            MAX(order_date) as last_order,
            DATEDIFF(MAX(order_date), MIN(order_date)) as customer_tenure_days
          FROM completed_orders
          GROUP BY customer_id
        ),
        product_prefs AS (
          SELECT DISTINCT
            o.customer_id,
            FIRST_VALUE(p.category) OVER (
              PARTITION BY o.customer_id 
              ORDER BY SUM(o.quantity * o.unit_price) OVER (PARTITION BY o.customer_id, p.category) DESC
            ) as favorite_category
          FROM completed_orders o
          JOIN products p ON o.product_id = p.product_id
        )
      SELECT 
        c.name,
        c.tier,
        cs.order_count,
        ROUND(cs.total_spend, 2) as total_spend,
        ROUND(cs.avg_order_value, 2) as avg_order_value,
        cs.first_order,
        cs.last_order,
        cs.customer_tenure_days,
        pp.favorite_category
      FROM customers c
      LEFT JOIN customer_stats cs ON c.customer_id = cs.customer_id
      LEFT JOIN (
        SELECT DISTINCT customer_id, 
               FIRST_VALUE(category) OVER (
                 PARTITION BY customer_id 
                 ORDER BY cat_spend DESC
               ) as favorite_category
        FROM (
          SELECT o.customer_id, p.category,
                 SUM(o.quantity * o.unit_price) as cat_spend
          FROM completed_orders o
          JOIN products p ON o.product_id = p.product_id
          GROUP BY o.customer_id, p.category
        )
      ) pp ON c.customer_id = pp.customer_id
      ORDER BY total_spend DESC NULLS LAST
    """).show(truncate = false)
    
    // ===== HAVING Clause =====
    println("=== HAVING: High-value customers ===")
    spark.sql("""
      SELECT 
        customer_id,
        COUNT(*) as order_count,
        SUM(quantity * unit_price) as total_spend
      FROM orders
      WHERE status = 'completed'
      GROUP BY customer_id
      HAVING SUM(quantity * unit_price) > 1000
        AND COUNT(*) >= 2
      ORDER BY total_spend DESC
    """).show()
    
    // ===== EXISTS and IN =====
    println("=== EXISTS: Customers with Electronics purchases ===")
    spark.sql("""
      SELECT c.name, c.tier
      FROM customers c
      WHERE EXISTS (
        SELECT 1 
        FROM orders o
        JOIN products p ON o.product_id = p.product_id
        WHERE o.customer_id = c.customer_id
          AND p.category = 'Electronics'
          AND o.status = 'completed'
      )
    """).show()
    
    spark.stop()
  }
}
```

---

## Step 733: Window Functions ใน SQL

```scala
// SQLWindowFunctions.scala
import org.apache.spark.sql.SparkSession

object SQLWindowFunctions {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("SQL Window Functions")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val sales = Seq(
      ("2024-01", "Electronics", "North", 100000.0),
      ("2024-01", "Electronics", "South", 80000.0),
      ("2024-01", "Clothing",    "North", 50000.0),
      ("2024-01", "Clothing",    "South", 40000.0),
      ("2024-02", "Electronics", "North", 120000.0),
      ("2024-02", "Electronics", "South", 90000.0),
      ("2024-02", "Clothing",    "North", 55000.0),
      ("2024-02", "Clothing",    "South", 45000.0),
      ("2024-03", "Electronics", "North", 110000.0),
      ("2024-03", "Electronics", "South", 95000.0),
      ("2024-03", "Clothing",    "North", 60000.0),
      ("2024-03", "Clothing",    "South", 50000.0)
    ).toDF("month", "category", "region", "revenue")
    
    sales.createOrReplaceTempView("monthly_sales")
    
    // ===== Ranking Functions =====
    println("=== Ranking ===")
    spark.sql("""
      SELECT 
        month,
        category,
        region,
        revenue,
        RANK()       OVER (PARTITION BY month ORDER BY revenue DESC) as rank_in_month,
        DENSE_RANK() OVER (PARTITION BY month ORDER BY revenue DESC) as dense_rank,
        ROW_NUMBER() OVER (PARTITION BY month ORDER BY revenue DESC) as row_num,
        NTILE(2)     OVER (PARTITION BY month ORDER BY revenue DESC) as half,
        PERCENT_RANK() OVER (PARTITION BY month ORDER BY revenue) as pct_rank
      FROM monthly_sales
      ORDER BY month, rank_in_month
    """).show()
    
    // ===== Running Aggregations =====
    println("=== Running Totals ===")
    spark.sql("""
      SELECT 
        month,
        category,
        region,
        revenue,
        SUM(revenue) OVER (
          PARTITION BY category 
          ORDER BY month 
          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) as cumulative_revenue,
        AVG(revenue) OVER (
          PARTITION BY category
          ORDER BY month
          ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
        ) as two_month_avg,
        SUM(revenue) OVER (PARTITION BY category) as category_total,
        ROUND(revenue / SUM(revenue) OVER (PARTITION BY category) * 100, 1) as pct_of_category_total
      FROM monthly_sales
      ORDER BY category, month
    """).show()
    
    // ===== LAG and LEAD =====
    println("=== Month-over-Month Growth ===")
    spark.sql("""
      SELECT 
        month,
        category,
        region,
        revenue,
        LAG(revenue, 1) OVER (
          PARTITION BY category, region 
          ORDER BY month
        ) as prev_month_revenue,
        LEAD(revenue, 1) OVER (
          PARTITION BY category, region 
          ORDER BY month
        ) as next_month_revenue,
        ROUND(
          (revenue - LAG(revenue, 1) OVER (PARTITION BY category, region ORDER BY month)) /
          LAG(revenue, 1) OVER (PARTITION BY category, region ORDER BY month) * 100, 
          1
        ) as mom_growth_pct
      FROM monthly_sales
      ORDER BY category, region, month
    """).show()
    
    // ===== FIRST_VALUE and LAST_VALUE =====
    println("=== First and Last Values ===")
    spark.sql("""
      SELECT 
        month,
        category,
        region,
        revenue,
        FIRST_VALUE(revenue) OVER (
          PARTITION BY category, region 
          ORDER BY month
          ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) as first_month_revenue,
        LAST_VALUE(revenue) OVER (
          PARTITION BY category, region 
          ORDER BY month
          ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) as last_month_revenue
      FROM monthly_sales
      ORDER BY category, region, month
    """).show()
    
    // ===== Complex Window: Moving Average =====
    println("=== 2-Month Moving Average ===")
    spark.sql("""
      SELECT 
        month,
        category,
        SUM(revenue) as monthly_total,
        AVG(SUM(revenue)) OVER (
          PARTITION BY category
          ORDER BY month
          ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
        ) as two_month_moving_avg
      FROM monthly_sales
      GROUP BY month, category
      ORDER BY category, month
    """).show()
    
    spark.stop()
  }
}
```

---

## Step 734: SQL UDFs

```scala
// SQLUDFs.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._

object SQLUDFs {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("SQL UDFs")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== Register UDFs สำหรับ SQL =====
    
    // UDF 1: แปลง Thai phone number
    spark.udf.register("format_thai_phone", (phone: String) => {
      Option(phone)
        .map(_.replaceAll("[^0-9]", ""))
        .filter(p => p.length == 10 || (p.length == 11 && p.startsWith("66")))
        .map { p =>
          if (p.startsWith("66")) s"0${p.substring(2)}"
          else p
        }
        .map { p => s"${p.substring(0,3)}-${p.substring(3,6)}-${p.substring(6)}" }
        .orNull
    })
    
    // UDF 2: คำนวณ discount ตาม tier
    spark.udf.register("calculate_discount", (price: Double, tier: String) => {
      tier match {
        case "VIP"      => price * 0.80  // 20% off
        case "Premium"  => price * 0.90  // 10% off
        case "Standard" => price * 0.95  // 5% off
        case _          => price
      }
    })
    
    // UDF 3: Categorize salary
    spark.udf.register("salary_band", (salary: Double) => {
      if (salary < 50000)      "Entry"
      else if (salary < 70000) "Mid"
      else if (salary < 90000) "Senior"
      else                     "Principal"
    })
    
    // UDF 4: Thai text processing
    spark.udf.register("count_thai_chars", (text: String) => {
      Option(text).map { t =>
        t.codePoints()
         .filter(cp => cp >= 0x0E00 && cp <= 0x0E7F)
         .count()
         .toInt
      }.getOrElse(0)
    })
    
    // ===== ใช้ UDFs ใน SQL =====
    val data = Seq(
      ("Alice",  "+66812345678", "Premium",  80000.0),
      ("Bob",    "0898765432",   "Standard", 60000.0),
      ("Carol",  "invalid",      "VIP",      120000.0),
      ("สมชาย",  "0812345678",   "Premium",  75000.0)
    ).toDF("name", "phone", "tier", "salary")
    
    data.createOrReplaceTempView("staff")
    
    spark.sql("""
      SELECT 
        name,
        phone,
        format_thai_phone(phone) as formatted_phone,
        tier,
        salary,
        calculate_discount(salary, tier) as discounted_salary,
        salary_band(salary) as band,
        count_thai_chars(name) as thai_chars
      FROM staff
    """).show(truncate = false)
    
    // ===== UDTFs (User Defined Table Functions) =====
    // สร้าง table-valued function
    import org.apache.spark.sql.api.java.UDF1
    
    spark.udf.register("generate_date_range", 
      (n: Int) => (0 until n).map(i => s"2024-${"%02d".format(i + 1)}").toArray
    )
    
    // ===== SQL Functions as DataFrame Column =====
    // สามารถใช้ registered UDFs ใน DataFrame API ด้วย
    val formatPhone = callUDF("format_thai_phone", col("phone"))
    val calcDiscount = callUDF("calculate_discount", col("salary"), col("tier"))
    
    data.select(
      $"name",
      formatPhone.as("formatted_phone"),
      calcDiscount.as("net_salary")
    ).show(truncate = false)
    
    spark.stop()
  }
}
```

---

## Step 735: Spark SQL Catalog

```scala
// SparkCatalog.scala
import org.apache.spark.sql.SparkSession

object SparkCatalog {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Spark Catalog")
      // เปิด Hive metastore สำหรับ persistent tables
      // .config("spark.sql.warehouse.dir", "/tmp/spark-warehouse")
      // .enableHiveSupport()
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // สร้าง temp views
    val df = Seq((1, "Alice"), (2, "Bob")).toDF("id", "name")
    df.createOrReplaceTempView("people")
    df.createOrReplaceGlobalTempView("global_people")
    
    // ===== Catalog API =====
    val catalog = spark.catalog
    
    // List databases
    println("=== Databases ===")
    catalog.listDatabases().show()
    
    // List tables ใน current database
    println("=== Tables ===")
    catalog.listTables().show()
    
    // List columns ของ table
    println("=== Columns of 'people' ===")
    catalog.listColumns("people").show()
    
    // List functions
    println("=== Built-in Functions ===")
    catalog.listFunctions().filter(_.className.contains("sql")).show(5)
    
    // Check if table/view exists
    println(s"Table 'people' exists: ${catalog.tableExists("people")}")
    println(s"Table 'unknown' exists: ${catalog.tableExists("unknown")}")
    
    // ===== Persistent Tables =====
    // สร้าง persistent table (ต้องการ Hive หรือ warehouse path)
    /*
    spark.sql("CREATE DATABASE IF NOT EXISTS mydb")
    spark.sql("USE mydb")
    
    df.write.mode("overwrite").saveAsTable("mydb.people")
    
    // อ่าน persistent table
    val persistedDF = spark.table("mydb.people")
    
    // หรือผ่าน SQL
    spark.sql("SELECT * FROM mydb.people").show()
    
    // Refresh table metadata
    catalog.refreshTable("mydb.people")
    
    // Drop table
    spark.sql("DROP TABLE IF EXISTS mydb.people")
    */
    
    // ===== SQL DDL Operations =====
    spark.sql("""
      CREATE OR REPLACE TEMP VIEW employee_summary AS
      SELECT dept, COUNT(*) as count, AVG(salary) as avg_salary
      FROM (
        SELECT 'Engineering' as dept, 85000.0 as salary UNION ALL
        SELECT 'Engineering' as dept, 90000.0 as salary UNION ALL
        SELECT 'Marketing'   as dept, 62000.0 as salary
      )
      GROUP BY dept
    """)
    
    spark.sql("SELECT * FROM employee_summary").show()
    
    // ===== Cache Table in Memory =====
    spark.sql("CACHE TABLE people")
    spark.sql("SELECT * FROM people WHERE id > 0").show()
    
    // Uncache
    spark.sql("UNCACHE TABLE people")
    
    // ===== Analyze Table (update statistics) =====
    // spark.sql("ANALYZE TABLE people COMPUTE STATISTICS")
    // spark.sql("ANALYZE TABLE people COMPUTE STATISTICS FOR COLUMNS id, name")
    
    spark.stop()
  }
}
```

---

## Step 736: Hive Integration

```scala
// HiveIntegration.scala
import org.apache.spark.sql.SparkSession

object HiveIntegration {
  def main(args: Array[String]): Unit = {
    // เปิด Hive support
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Hive Integration")
      .config("spark.sql.warehouse.dir", "/tmp/spark-warehouse")
      .config("hive.metastore.warehouse.dir", "/tmp/spark-warehouse")
      // ถ้ามี Hive metastore:
      // .config("hive.metastore.uris", "thrift://localhost:9083")
      .enableHiveSupport()
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== Hive DDL =====
    spark.sql("CREATE DATABASE IF NOT EXISTS ecommerce")
    spark.sql("USE ecommerce")
    
    // สร้าง managed table
    spark.sql("""
      CREATE TABLE IF NOT EXISTS products (
        product_id STRING,
        name       STRING,
        category   STRING,
        price      DOUBLE,
        stock      INT
      )
      STORED AS PARQUET
    """)
    
    // Insert data
    spark.sql("""
      INSERT INTO products VALUES
        ('P001', 'iPhone 15',   'Electronics', 999.99,  150),
        ('P002', 'T-Shirt',     'Clothing',    49.99,   500),
        ('P003', 'MacBook Pro', 'Electronics', 1299.99, 80)
    """)
    
    spark.sql("SELECT * FROM products").show()
    
    // ===== External Table =====
    spark.sql("""
      CREATE EXTERNAL TABLE IF NOT EXISTS external_orders (
        order_id    STRING,
        customer_id STRING,
        amount      DOUBLE,
        order_date  STRING
      )
      ROW FORMAT DELIMITED
      FIELDS TERMINATED BY ','
      LINES TERMINATED BY '\n'
      STORED AS TEXTFILE
      LOCATION '/tmp/orders-data'
    """)
    
    // ===== Partitioned Table =====
    spark.sql("""
      CREATE TABLE IF NOT EXISTS sales_partitioned (
        sale_id    STRING,
        product_id STRING,
        amount     DOUBLE
      )
      PARTITIONED BY (year INT, month INT)
      STORED AS PARQUET
    """)
    
    // เพิ่ม partition
    spark.sql("""
      INSERT INTO sales_partitioned PARTITION (year=2024, month=1)
      SELECT 'S001', 'P001', 999.99
    """)
    
    // ===== Hive Functions =====
    // Hive UDFs สามารถใช้ใน Spark SQL ได้
    // spark.sql("ADD JAR /path/to/my-udf.jar")
    // spark.sql("CREATE TEMPORARY FUNCTION my_func AS 'com.example.MyUDF'")
    
    // ===== Hive SerDe =====
    // อ่านข้อมูลจาก Hive JSON SerDe
    /*
    spark.sql("""
      CREATE EXTERNAL TABLE json_table (
        id INT,
        name STRING
      )
      ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
      STORED AS TEXTFILE
      LOCATION '/path/to/json'
    """)
    */
    
    // cleanup
    spark.sql("DROP TABLE IF EXISTS products")
    spark.sql("DROP TABLE IF EXISTS sales_partitioned")
    
    spark.stop()
  }
}
```

---

## Step 737: Query Optimization

```scala
// QueryOptimization.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._

object QueryOptimization {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Query Optimization")
      .config("spark.sql.shuffle.partitions", "8")
      .config("spark.sql.adaptive.enabled", "true")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val orders = spark.range(100000)
      .withColumn("customer_id", ($"id" % 1000).cast("string"))
      .withColumn("product_id",  ($"id" % 100).cast("string"))
      .withColumn("amount", rand() * 1000)
      .withColumn("category", 
        when($"id" % 3 === 0, "Electronics")
          .when($"id" % 3 === 1, "Clothing")
          .otherwise("Food"))
    
    orders.createOrReplaceTempView("orders")
    
    // ===== Explain Plan =====
    println("=== Explain (simple) ===")
    spark.sql("""
      SELECT category, SUM(amount) as total
      FROM orders
      GROUP BY category
    """).explain()
    
    println("=== Explain (formatted) ===")
    spark.sql("""
      SELECT category, SUM(amount) as total
      FROM orders
      GROUP BY category
    """).explain("formatted")
    
    // ===== Cost-Based Optimizer =====
    // เปิด CBO (Cost-Based Optimizer)
    spark.sql("SET spark.sql.cbo.enabled = true")
    spark.sql("SET spark.sql.cbo.joinReorder.enabled = true")
    
    // ===== Hints =====
    println("=== Join Hints ===")
    
    // BROADCAST hint - force broadcast join
    spark.sql("""
      SELECT /*+ BROADCAST(small_table) */ 
        o.customer_id, 
        s.category_name
      FROM orders o
      JOIN (SELECT 'Electronics' as id, 'Tech' as category_name UNION ALL
            SELECT 'Clothing',          'Fashion'               UNION ALL
            SELECT 'Food',              'Grocery') small_table
        ON o.category = small_table.id
    """).explain()
    
    // MERGE hint - force sort-merge join
    spark.sql("""
      SELECT /*+ MERGE(orders) */ 
        category, SUM(amount)
      FROM orders
      GROUP BY category
    """).explain()
    
    // REPARTITION hint
    spark.sql("""
      SELECT /*+ REPARTITION(10) */ *
      FROM orders
      WHERE amount > 500
    """).explain()
    
    // ===== Partition Pruning =====
    val partitionedData = spark.range(100000)
      .withColumn("year",  ($"id" % 3 + 2022).cast("int"))
      .withColumn("month", ($"id" % 12 + 1).cast("int"))
      .withColumn("amount", rand() * 1000)
    
    partitionedData.write
      .mode("overwrite")
      .partitionBy("year", "month")
      .parquet("/tmp/partitioned-orders")
    
    // อ่าน specific partitions - partition pruning!
    val specific = spark.read
      .parquet("/tmp/partitioned-orders")
      .filter("year = 2024 AND month = 1") // ไม่ต้อง scan ทุก partition
    
    specific.explain() // ดู PartitionFilters ใน plan
    
    spark.stop()
  }
}
```

---

## Step 738: Spark SQL Advanced Features

```scala
// SparkSQLAdvanced.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._

object SparkSQLAdvanced {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Spark SQL Advanced")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    val data = Seq(
      (1, "Alice", Array("Scala", "Spark", "Kafka"), Map("skill" -> 90, "experience" -> 5)),
      (2, "Bob",   Array("Python", "TensorFlow"),     Map("skill" -> 85, "experience" -> 3)),
      (3, "Carol", Array("Java", "Spring", "Docker"), Map("skill" -> 88, "experience" -> 7))
    ).toDF("id", "name", "skills", "attributes")
    
    data.createOrReplaceTempView("staff")
    
    // ===== LATERAL VIEW explode =====
    println("=== LATERAL VIEW explode ===")
    spark.sql("""
      SELECT id, name, skill
      FROM staff
      LATERAL VIEW explode(skills) skills_table AS skill
    """).show()
    
    // ===== LATERAL VIEW posexplode =====
    println("=== LATERAL VIEW posexplode (with position) ===")
    spark.sql("""
      SELECT id, name, pos, skill
      FROM staff
      LATERAL VIEW posexplode(skills) skills_table AS pos, skill
    """).show()
    
    // ===== Explode Map =====
    println("=== Explode Map ===")
    spark.sql("""
      SELECT id, name, key, value
      FROM staff
      LATERAL VIEW explode(attributes) attr_table AS key, value
    """).show()
    
    // ===== GROUPING SETS =====
    val sales = Seq(
      ("2024-Q1", "Electronics", "North", 100000.0),
      ("2024-Q1", "Clothing",    "South", 50000.0),
      ("2024-Q2", "Electronics", "East",  120000.0),
      ("2024-Q2", "Clothing",    "North", 60000.0)
    ).toDF("quarter", "category", "region", "revenue")
    
    sales.createOrReplaceTempView("quarterly_sales")
    
    println("=== GROUPING SETS ===")
    spark.sql("""
      SELECT 
        quarter,
        category,
        region,
        SUM(revenue) as total_revenue
      FROM quarterly_sales
      GROUP BY GROUPING SETS (
        (quarter, category, region),  -- most granular
        (quarter, category),           -- no region
        (category),                    -- only category
        ()                            -- grand total
      )
      ORDER BY quarter NULLS LAST, category NULLS LAST, region NULLS LAST
    """).show()
    
    // ===== ROLLUP =====
    println("=== ROLLUP ===")
    spark.sql("""
      SELECT 
        COALESCE(quarter, 'ALL QUARTERS') as quarter,
        COALESCE(category, 'ALL CATEGORIES') as category,
        SUM(revenue) as total_revenue
      FROM quarterly_sales
      GROUP BY ROLLUP (quarter, category)
      ORDER BY quarter, category
    """).show()
    
    // ===== CUBE =====
    println("=== CUBE ===")
    spark.sql("""
      SELECT 
        COALESCE(quarter,  'TOTAL') as quarter,
        COALESCE(category, 'TOTAL') as category,
        COALESCE(region,   'TOTAL') as region,
        SUM(revenue) as total_revenue
      FROM quarterly_sales
      GROUP BY CUBE (quarter, category, region)
      ORDER BY quarter, category, region
    """).show()
    
    // ===== TRANSFORM (map function) =====
    println("=== TRANSFORM ===")
    spark.sql("""
      SELECT 
        id,
        name,
        TRANSFORM(skills, s -> UPPER(s)) as upper_skills,
        FILTER(skills, s -> LENGTH(s) > 4) as long_skills,
        AGGREGATE(skills, 0, (acc, s) -> acc + LENGTH(s)) as total_skill_chars,
        EXISTS(skills, s -> s = 'Spark') as knows_spark
      FROM staff
    """).show(truncate = false)
    
    // ===== STRUCT Functions =====
    println("=== STRUCT ===")
    spark.sql("""
      SELECT 
        id,
        STRUCT(name, attributes['skill'] as skill_score) as profile
      FROM staff
    """).show(truncate = false)
    
    spark.stop()
  }
}
```

---

## Step 739: Performance Tuning for SQL

```scala
// SQLPerformanceTuning.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._

object SQLPerformanceTuning {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("SQL Performance")
      .config("spark.sql.adaptive.enabled", "true")
      .config("spark.sql.adaptive.coalescePartitions.enabled", "true")
      .config("spark.sql.adaptive.skewJoin.enabled", "true")
      .config("spark.sql.autoBroadcastJoinThreshold", "10485760") // 10MB
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // ===== Broadcast Join Threshold =====
    println("Auto broadcast threshold: " + 
      spark.conf.get("spark.sql.autoBroadcastJoinThreshold"))
    
    // เปลี่ยน threshold
    spark.sql("SET spark.sql.autoBroadcastJoinThreshold = 52428800") // 50MB
    
    // ===== Bucketing =====
    // Bucket tables เพื่อหลีกเลี่ยง shuffle ใน joins
    val orders = spark.range(100000)
      .withColumn("customer_id", ($"id" % 1000).cast("string"))
      .withColumn("amount", rand() * 1000)
    
    // สร้าง bucketed table
    orders.write
      .mode("overwrite")
      .bucketBy(8, "customer_id")  // 8 buckets, bucketed by customer_id
      .sortBy("customer_id")
      .saveAsTable("bucketed_orders")
    
    val customers = spark.range(1000)
      .withColumn("customer_id", $"id".cast("string"))
      .withColumn("name", concat(lit("Customer"), $"id"))
    
    customers.write
      .mode("overwrite")
      .bucketBy(8, "customer_id")
      .sortBy("customer_id")
      .saveAsTable("bucketed_customers")
    
    // Join บน bucketed tables ไม่ต้อง shuffle!
    val result = spark.sql("""
      SELECT c.name, SUM(o.amount) as total
      FROM bucketed_orders o
      JOIN bucketed_customers c ON o.customer_id = c.customer_id
      GROUP BY c.name
    """)
    
    result.explain() // ดู SortMergeJoin ไม่มี Exchange (shuffle)
    
    // ===== Statistics Collection =====
    spark.sql("ANALYZE TABLE bucketed_orders COMPUTE STATISTICS")
    spark.sql("ANALYZE TABLE bucketed_orders COMPUTE STATISTICS FOR COLUMNS customer_id, amount")
    
    // ===== Query Caching =====
    spark.sql("CACHE TABLE bucketed_orders")
    spark.sql("CACHE TABLE bucketed_customers")
    
    val cachedResult = spark.sql("""
      SELECT customer_id, COUNT(*) as order_count, SUM(amount) as total
      FROM bucketed_orders
      GROUP BY customer_id
      ORDER BY total DESC
      LIMIT 10
    """)
    
    cachedResult.show()
    
    spark.sql("UNCACHE TABLE bucketed_orders")
    spark.sql("UNCACHE TABLE bucketed_customers")
    spark.sql("DROP TABLE IF EXISTS bucketed_orders")
    spark.sql("DROP TABLE IF EXISTS bucketed_customers")
    
    spark.stop()
  }
}
```

---

## Step 740: Production SQL Patterns

```scala
// ProductionSQLPatterns.scala
import org.apache.spark.sql.{SparkSession, DataFrame}
import org.apache.spark.sql.functions._

object ProductionSQLPatterns {
  
  // Pattern 1: SQL in companion object สำหรับ maintainability
  object Queries {
    val customerLTV = """
      WITH orders_summary AS (
        SELECT 
          customer_id,
          COUNT(DISTINCT order_id) as total_orders,
          SUM(quantity * unit_price) as total_revenue,
          MIN(order_date) as first_order_date,
          MAX(order_date) as last_order_date
        FROM orders
        WHERE status = 'completed'
        GROUP BY customer_id
      )
      SELECT 
        c.customer_id,
        c.name,
        c.tier,
        os.total_orders,
        os.total_revenue,
        os.total_revenue / GREATEST(DATEDIFF(os.last_order_date, os.first_order_date), 1) * 365 as annual_ltv
      FROM customers c
      JOIN orders_summary os ON c.customer_id = os.customer_id
    """
    
    val productPerformance = """
      SELECT 
        p.category,
        p.product_name,
        SUM(o.quantity) as units_sold,
        SUM(o.quantity * o.unit_price) as revenue,
        RANK() OVER (PARTITION BY p.category ORDER BY SUM(o.quantity * o.unit_price) DESC) as rank_in_category
      FROM orders o
      JOIN products p ON o.product_id = p.product_id
      WHERE o.status = 'completed'
      GROUP BY p.category, p.product_name
    """
  }
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName("Production SQL Patterns")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    
    // Setup
    val orders = Seq(
      ("O001", "C001", "P001", 2, 999.99,  "2024-01-15", "completed"),
      ("O002", "C002", "P002", 1, 49.99,   "2024-01-16", "completed"),
      ("O003", "C001", "P003", 1, 1299.99, "2024-02-01", "completed")
    ).toDF("order_id", "customer_id", "product_id", "quantity", "unit_price", "order_date", "status")
    
    val customers = Seq(
      ("C001", "Alice",  "Premium"),
      ("C002", "Bob",    "Standard")
    ).toDF("customer_id", "name", "tier")
    
    val products = Seq(
      ("P001", "iPhone",  "Electronics"),
      ("P002", "T-Shirt", "Clothing"),
      ("P003", "MacBook", "Electronics")
    ).toDF("product_id", "product_name", "category")
    
    orders.createOrReplaceTempView("orders")
    customers.createOrReplaceTempView("customers")
    products.createOrReplaceTempView("products")
    
    // Pattern 2: Parameterized Queries
    def queryByDateRange(startDate: String, endDate: String): DataFrame = {
      spark.sql(s"""
        SELECT customer_id, SUM(quantity * unit_price) as revenue
        FROM orders
        WHERE order_date BETWEEN '$startDate' AND '$endDate'
          AND status = 'completed'
        GROUP BY customer_id
      """)
    }
    
    queryByDateRange("2024-01-01", "2024-01-31").show()
    
    // Pattern 3: ใช้ stored results
    val ltvResult = spark.sql(Queries.customerLTV)
    ltvResult.cache()
    ltvResult.show()
    
    // Pattern 4: Dynamic SQL generation
    val metricsToCompute = List("SUM(quantity * unit_price)", "COUNT(*)", "AVG(unit_price)")
    val metricSQL = metricsToCompute.zipWithIndex.map { case (metric, i) =>
      s"$metric as metric_$i"
    }.mkString(", ")
    
    spark.sql(s"""
      SELECT category, $metricSQL
      FROM orders o
      JOIN products p ON o.product_id = p.product_id
      GROUP BY category
    """).show()
    
    ltvResult.unpersist()
    spark.stop()
  }
}
```

---

## สรุป Part 74

| Feature | SQL Syntax | DataFrame API |
|---------|-----------|---------------|
| Filter | `WHERE` | `.filter()` |
| Group | `GROUP BY` | `.groupBy()` |
| Sort | `ORDER BY` | `.orderBy()` |
| Join | `JOIN ... ON` | `.join()` |
| Window | `OVER (PARTITION BY ... ORDER BY ...)` | `Window.partitionBy().orderBy()` |
| CTE | `WITH ... AS (...)` | Chained transforms |
| Pivot | Manual CASE WHEN | `.pivot()` |

### SQL Optimization Checklist
1. ✅ ใช้ partition pruning (filter บน partitioned columns)
2. ✅ Broadcast small tables
3. ✅ เก็บ statistics ด้วย ANALYZE TABLE
4. ✅ Bucket tables ที่ join บ่อย
5. ✅ เปิด AQE (Adaptive Query Execution)
6. ✅ ใช้ CTE แทน nested subqueries เพื่อ readability

---

## แบบฝึกหัด Part 74

1. **Complex Analytics**: เขียน SQL query ที่คำนวณ customer cohort analysis (grouping customers by first purchase month) พร้อม retention rate ต่อเดือน

2. **Window Functions**: เขียน SQL ที่หา top 3 products ต่อ category ต่อ quarter โดยใช้ window functions และ CTEs

3. **SQL Optimization**: มี slow query ให้ analyze ด้วย EXPLAIN แล้วปรับปรุงโดยใช้ hints, partitioning, หรือ bucketing

4. **UDF Creation**: สร้าง SQL UDF ที่คำนวณ shipping cost ตาม weight, distance, และ shipping_tier แล้ว register และ ใช้ใน complex query

5. **Dynamic Report**: สร้าง function ที่รับ parameters (date range, categories, regions) แล้ว generate SQL query แบบ dynamic และ return DataFrame

---

## ไปต่อ: Part 75 — Spark Structured Streaming
ใน Part ถัดไปจะเรียน real-time data processing ด้วย Structured Streaming

[→ Part 75: Spark Structured Streaming](./part-75-spark-streaming.md)
