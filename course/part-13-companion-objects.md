# Part 13: Companion Objects

## Steps 121-130: Companion Objects, Factory Patterns, Apply/Unapply Deep Dive, Implicit Conversions Basics

---

## Step 121: Companion Object พื้นฐาน

Companion Object คือ `object` ที่มีชื่อเดียวกับ `class` และอยู่ในไฟล์เดียวกัน มีสิทธิ์เข้าถึง private members ของกัน

```scala
// class และ companion object
class Temperature private(val celsius: Double) {
  
  def toFahrenheit: Double = celsius * 9 / 5 + 32
  def toKelvin: Double = celsius + 273.15
  
  def +(other: Temperature): Temperature = Temperature(celsius + other.celsius)
  def -(other: Temperature): Temperature = Temperature(celsius - other.celsius)
  
  def isAboveFreeze: Boolean = celsius > 0
  def isBelowFreeze: Boolean = celsius < 0
  
  override def toString: String = f"$celsius%.1f°C"
  
  override def equals(other: Any): Boolean = other match {
    case t: Temperature => Math.abs(celsius - t.celsius) < 0.001
    case _ => false
  }
  
  override def hashCode: Int = celsius.hashCode
}

// Companion object
object Temperature {
  
  // Factory methods - ตั้งชื่อให้ชัดเจน
  def fromCelsius(c: Double): Temperature = new Temperature(c)
  def fromFahrenheit(f: Double): Temperature = new Temperature((f - 32) * 5 / 9)
  def fromKelvin(k: Double): Temperature = new Temperature(k - 273.15)
  
  // apply method - ใช้ syntax: Temperature(25.0)
  def apply(celsius: Double): Temperature = new Temperature(celsius)
  
  // Constants
  val AbsoluteZero: Temperature = fromKelvin(0)
  val Freezing: Temperature = new Temperature(0)
  val Boiling: Temperature = new Temperature(100)
  val BodyTemp: Temperature = new Temperature(37)
  
  // Implicit Ordering - ทำให้เรียงลำดับได้
  implicit val ordering: Ordering[Temperature] = 
    Ordering.by(_.celsius)
  
  // Utility
  def average(temps: List[Temperature]): Temperature = {
    if (temps.isEmpty) Temperature(0)
    else Temperature(temps.map(_.celsius).sum / temps.size)
  }
}

object CompanionObjectDemo extends App {
  // Using apply
  val t1 = Temperature(25.0)
  val t2 = Temperature.fromFahrenheit(98.6)
  val t3 = Temperature.fromKelvin(300)
  
  println("=== Temperature Factory ===")
  println(s"25°C = ${t1.toFahrenheit}°F")
  println(s"98.6°F = $t2")
  println(s"300K = $t3")
  
  // Using constants
  println(s"\nConstants:")
  println(s"Absolute Zero: ${Temperature.AbsoluteZero.toKelvin}K")
  println(s"Freezing: ${Temperature.Freezing}")
  println(s"Boiling: ${Temperature.Boiling}")
  println(s"Body Temp: ${Temperature.BodyTemp}")
  
  // Operations
  val sum = t1 + Temperature(10)
  println(s"\n$t1 + 10°C = $sum")
  
  // Sorting with implicit ordering
  val temps = List(Temperature(50), Temperature(20), Temperature(-10), Temperature(37), Temperature(0))
  val sorted = temps.sorted
  println(s"\nSorted: ${sorted.mkString(", ")}")
  
  // Average
  println(s"Average: ${Temperature.average(temps)}")
}
```

---

## Step 122: Apply Method — Syntactic Sugar

```scala
// apply method - enables object(args) syntax
class Matrix(val rows: Int, val cols: Int) {
  private val data: Array[Array[Double]] = Array.ofDim[Double](rows, cols)
  
  // Apply for element access: matrix(row, col)
  def apply(row: Int, col: Int): Double = data(row)(col)
  
  // Update for element assignment: matrix(row, col) = value
  def update(row: Int, col: Int, value: Double): Unit = {
    data(row)(col) = value
  }
  
  def +(other: Matrix): Matrix = {
    require(rows == other.rows && cols == other.cols, "Matrix dimensions must match")
    val result = Matrix(rows, cols)
    for (i <- 0 until rows; j <- 0 until cols) {
      result(i, j) = this(i, j) + other(i, j)
    }
    result
  }
  
  def *(scalar: Double): Matrix = {
    val result = Matrix(rows, cols)
    for (i <- 0 until rows; j <- 0 until cols) {
      result(i, j) = this(i, j) * scalar
    }
    result
  }
  
  override def toString: String = {
    (0 until rows).map { i =>
      (0 until cols).map(j => f"${data(i)(j)}%6.2f").mkString("[", " ", "]")
    }.mkString("\n")
  }
}

object Matrix {
  // Factory via apply
  def apply(rows: Int, cols: Int): Matrix = new Matrix(rows, cols)
  
  // Create from 2D array
  def apply(data: Array[Array[Double]]): Matrix = {
    val m = new Matrix(data.length, data(0).length)
    for (i <- data.indices; j <- data(0).indices) {
      m(i, j) = data(i)(j)
    }
    m
  }
  
  // Identity matrix
  def identity(size: Int): Matrix = {
    val m = Matrix(size, size)
    for (i <- 0 until size) m(i, i) = 1.0
    m
  }
  
  // Zero matrix
  def zeros(rows: Int, cols: Int): Matrix = Matrix(rows, cols)
}

// Function-like objects using apply
class Multiplier(val factor: Double) {
  // This makes Multiplier instances callable as functions
  def apply(x: Double): Double = x * factor
}

object Multiplier {
  def apply(factor: Double): Multiplier = new Multiplier(factor)
}

object ApplyMethodDemo extends App {
  val m = Matrix(3, 3)
  m(0, 0) = 1.0; m(0, 1) = 2.0; m(0, 2) = 3.0
  m(1, 0) = 4.0; m(1, 1) = 5.0; m(1, 2) = 6.0
  m(2, 0) = 7.0; m(2, 1) = 8.0; m(2, 2) = 9.0
  
  println("=== Matrix ===")
  println(m)
  
  val identity = Matrix.identity(3)
  println("\nIdentity:")
  println(identity)
  
  val doubled = m * 2.0
  println("\nDoubled:")
  println(doubled)
  
  // Function-like object
  val double = Multiplier(2.0)
  val triple = Multiplier(3.0)
  
  println("\n=== Function-like Objects ===")
  println(s"double(5) = ${double(5)}")     // 10.0
  println(s"triple(7) = ${triple(7)}")     // 21.0
  
  val numbers = List(1.0, 2.0, 3.0, 4.0, 5.0)
  println(s"doubled: ${numbers.map(double(_))}")
}
```

---

## Step 123: Unapply Method — Pattern Matching Extractor

```scala
// unapply - makes classes work in pattern matching
class Email(val address: String) {
  require(address.contains("@"), "Invalid email address")
}

object Email {
  def apply(address: String): Email = new Email(address)
  
  // unapply - deconstructs an Email
  // Returns Option[String] - Some(value) if match, None if no match
  def unapply(email: Email): Option[String] = Some(email.address)
  
  // unapply for String - allows matching strings as emails
  def unapply(s: String): Option[String] = {
    if (s.matches("[^@]+@[^@]+\\.[^@]+")) Some(s)
    else None
  }
}

// More complex unapply
case class PersonName(first: String, last: String)

object FullName {
  def unapply(name: String): Option[(String, String)] = {
    name.split(" ") match {
      case Array(first, last) => Some((first, last))
      case Array(first, middle, last) => Some((s"$first $middle", last))
      case _ => None
    }
  }
}

// Unapply for Option-like pattern
class Positive(val value: Int) {
  require(value > 0, "Must be positive")
}

object Positive {
  def apply(n: Int): Option[Positive] = 
    if (n > 0) Some(new Positive(n)) else None
  
  def unapply(p: Positive): Option[Int] = Some(p.value)
}

object UnapplyDemo extends App {
  println("=== Unapply Demo ===")
  
  // Pattern matching with Email
  val emails = List("alice@example.com", "not-an-email", "bob@test.org")
  
  emails.foreach {
    case Email(addr) => println(s"Valid email: $addr")
    case invalid     => println(s"Invalid: $invalid")
  }
  
  println("\n=== Name Splitting ===")
  val names = List("Alice Chen", "Bob Smith Jones", "Madonna", "Grace Kelly")
  
  names.foreach {
    case FullName(first, last) => println(s"First: $first, Last: $last")
    case single                => println(s"Single name: $single")
  }
  
  println("\n=== Positive Numbers ===")
  val numbers = List(5, -3, 10, 0, 7)
  numbers.foreach { n =>
    Positive(n) match {
      case Some(Positive(v)) => println(s"$n is positive: $v")
      case None              => println(s"$n is not positive")
    }
  }
}
```

---

## Step 124: Factory Pattern ด้วย Companion Objects

```scala
// Factory Pattern - สร้าง object ที่ซับซ้อน
sealed trait DatabaseType
case object PostgreSQL extends DatabaseType
case object MySQL extends DatabaseType
case object SQLite extends DatabaseType
case object H2 extends DatabaseType

class DatabaseConnection private(
  val dbType: DatabaseType,
  val host: String,
  val port: Int,
  val database: String,
  val username: String,
  private val password: String,
  val maxPoolSize: Int
) {
  private var isConnected = false
  
  def connect(): Unit = {
    println(s"Connecting to $dbType at $host:$port/$database...")
    isConnected = true
  }
  
  def disconnect(): Unit = {
    isConnected = false
    println(s"Disconnected from $host:$port/$database")
  }
  
  def isActive: Boolean = isConnected
  
  def jdbcUrl: String = dbType match {
    case PostgreSQL => s"jdbc:postgresql://$host:$port/$database"
    case MySQL      => s"jdbc:mysql://$host:$port/$database"
    case SQLite     => s"jdbc:sqlite:$database"
    case H2         => s"jdbc:h2:mem:$database"
  }
  
  override def toString: String = 
    s"DatabaseConnection($dbType, $host:$port/$database, pool=$maxPoolSize)"
}

object DatabaseConnection {
  
  // Named factory methods
  def postgres(
    host: String, 
    database: String,
    username: String,
    password: String,
    port: Int = 5432,
    maxPoolSize: Int = 10
  ): DatabaseConnection = new DatabaseConnection(
    PostgreSQL, host, port, database, username, password, maxPoolSize
  )
  
  def mysql(
    host: String,
    database: String,
    username: String,
    password: String,
    port: Int = 3306,
    maxPoolSize: Int = 10
  ): DatabaseConnection = new DatabaseConnection(
    MySQL, host, port, database, username, password, maxPoolSize
  )
  
  def sqlite(database: String): DatabaseConnection = new DatabaseConnection(
    SQLite, "localhost", 0, database, "", "", 1
  )
  
  def h2InMemory(database: String = "testdb"): DatabaseConnection = 
    new DatabaseConnection(H2, "localhost", 0, database, "sa", "", 5)
  
  // Config-based factory
  def fromConfig(config: Map[String, String]): Either[String, DatabaseConnection] = {
    val dbType = config.get("type").map(_.toLowerCase) match {
      case Some("postgresql") | Some("postgres") => Right(PostgreSQL)
      case Some("mysql")                          => Right(MySQL)
      case Some("sqlite")                         => Right(SQLite)
      case Some("h2")                             => Right(H2)
      case Some(unknown)                          => Left(s"Unknown DB type: $unknown")
      case None                                   => Left("Missing 'type' in config")
    }
    
    dbType.map { dt =>
      new DatabaseConnection(
        dbType = dt,
        host = config.getOrElse("host", "localhost"),
        port = config.get("port").map(_.toInt).getOrElse(5432),
        database = config.getOrElse("database", "default"),
        username = config.getOrElse("username", ""),
        password = config.getOrElse("password", ""),
        maxPoolSize = config.get("maxPoolSize").map(_.toInt).getOrElse(10)
      )
    }
  }
}

object FactoryPatternDemo extends App {
  println("=== Factory Pattern ===")
  
  val pgConn = DatabaseConnection.postgres(
    host = "db.production.com",
    database = "myapp",
    username = "app_user",
    password = "secret",
    maxPoolSize = 20
  )
  println(s"PostgreSQL: $pgConn")
  println(s"JDBC URL: ${pgConn.jdbcUrl}")
  
  val testConn = DatabaseConnection.h2InMemory("test")
  println(s"\nH2 Test DB: $testConn")
  println(s"JDBC URL: ${testConn.jdbcUrl}")
  
  // Config-based
  val config = Map(
    "type" -> "postgresql",
    "host" -> "config-db.example.com",
    "port" -> "5432",
    "database" -> "configured_db",
    "username" -> "config_user",
    "password" -> "config_pass",
    "maxPoolSize" -> "15"
  )
  
  DatabaseConnection.fromConfig(config) match {
    case Right(conn) => println(s"\nFrom config: $conn")
    case Left(err)   => println(s"Config error: $err")
  }
  
  DatabaseConnection.fromConfig(Map("type" -> "oracle")) match {
    case Right(conn) => println(s"From config: $conn")
    case Left(err)   => println(s"Expected error: $err")
  }
}
```

---

## Step 125: Smart Constructor Pattern

```scala
// Smart constructors - validate before creating
case class Age private(value: Int) {
  def +(years: Int): Either[String, Age] = Age.create(value + years)
  def isAdult: Boolean = value >= 18
  override def toString: String = s"Age($value)"
}

object Age {
  def create(n: Int): Either[String, Age] = {
    if (n < 0)   Left(s"Age cannot be negative: $n")
    else if (n > 150) Left(s"Age too large: $n")
    else Right(new Age(n))
  }
  
  def unsafeCreate(n: Int): Age = 
    create(n).fold(err => throw new IllegalArgumentException(err), identity)
}

case class Email2 private(value: String) {
  def domain: String = value.split("@")(1)
  override def toString: String = value
}

object Email2 {
  private val EmailRegex = "^[^@\\s]+@[^@\\s]+\\.[^@\\s]{2,}$".r
  
  def create(s: String): Either[String, Email2] = {
    val normalized = s.trim.toLowerCase
    if (EmailRegex.matches(normalized)) Right(new Email2(normalized))
    else Left(s"Invalid email: $s")
  }
}

case class NonEmptyString private(value: String) {
  def length: Int = value.length
  def append(s: String): NonEmptyString = 
    new NonEmptyString(value + s)
}

object NonEmptyString {
  def create(s: String): Option[NonEmptyString] = {
    if (s.nonEmpty) Some(new NonEmptyString(s.trim))
    else None
  }
  
  def unapply(nes: NonEmptyString): Option[String] = Some(nes.value)
}

// Value objects with validation
case class MoneyAmount private(
  amount: BigDecimal,
  currency: String
) {
  def +(other: MoneyAmount): Either[String, MoneyAmount] = {
    if (currency != other.currency) Left(s"Currency mismatch: $currency != ${other.currency}")
    else Right(new MoneyAmount(amount + other.amount, currency))
  }
  
  def *(factor: Double): MoneyAmount = 
    new MoneyAmount(amount * factor, currency)
  
  override def toString: String = f"${amount}%.2f $currency"
}

object MoneyAmount {
  def create(amount: Double, currency: String): Either[String, MoneyAmount] = {
    val validCurrencies = Set("THB", "USD", "EUR", "GBP", "JPY")
    if (amount < 0) Left(s"Amount cannot be negative: $amount")
    else if (!validCurrencies.contains(currency.toUpperCase)) 
      Left(s"Invalid currency: $currency")
    else Right(new MoneyAmount(BigDecimal(amount).setScale(2, BigDecimal.RoundingMode.HALF_UP), currency.toUpperCase))
  }
  
  def thb(amount: Double): Either[String, MoneyAmount] = create(amount, "THB")
  def usd(amount: Double): Either[String, MoneyAmount] = create(amount, "USD")
}

object SmartConstructorDemo extends App {
  println("=== Smart Constructors ===")
  
  // Age validation
  println("Age:")
  Age.create(25).foreach(a => println(s"  Valid: $a, Adult: ${a.isAdult}"))
  Age.create(-5).left.foreach(e => println(s"  Error: $e"))
  Age.create(200).left.foreach(e => println(s"  Error: $e"))
  
  // Email validation
  println("\nEmail:")
  Email2.create("alice@example.com").foreach(e => println(s"  Valid: $e, Domain: ${e.domain}"))
  Email2.create("not-an-email").left.foreach(e => println(s"  Error: $e"))
  
  // Money amounts
  println("\nMoney:")
  val money1 = MoneyAmount.thb(1000.0)
  val money2 = MoneyAmount.thb(500.0)
  
  for {
    m1 <- money1
    m2 <- money2
    sum <- m1 + m2
  } yield println(s"  $m1 + $m2 = $sum")
  
  // Currency mismatch
  val usd = MoneyAmount.usd(50.0)
  for {
    t <- money1
    u <- usd
    result <- t + u
  } yield println(result) // Won't execute
  
  (for { t <- money1; u <- usd } yield t + u).foreach {
    case Right(sum) => println(s"  Sum: $sum")
    case Left(err)  => println(s"  Error: $err")
  }
}
```

---

## Step 126: Implicit Conversions ด้วย Companion Objects

```scala
// Implicit conversions (Scala 2 style)
// Note: ใน Scala 3 ใช้ given/using แทน แต่ยังรองรับ implicit

class Celsius(val value: Double) {
  override def toString: String = f"$value%.1f°C"
}

class Fahrenheit(val value: Double) {
  override def toString: String = f"$value%.1f°F"
}

object Celsius {
  implicit def fromFahrenheit(f: Fahrenheit): Celsius = 
    new Celsius((f.value - 32) * 5 / 9)
  
  implicit def fromDouble(d: Double): Celsius = new Celsius(d)
}

object Fahrenheit {
  implicit def fromCelsius(c: Celsius): Fahrenheit = 
    new Fahrenheit(c.value * 9 / 5 + 32)
}

// Enrichment via implicit class (Pimp My Library pattern)
object StringExtensions {
  implicit class RichString(val s: String) extends AnyVal {
    def toTitleCase: String = s.split(" ").map(_.capitalize).mkString(" ")
    def isPalindrome: Boolean = s.toLowerCase == s.toLowerCase.reverse
    def wordCount: Int = s.trim.split("\\s+").length
    def truncate(maxLength: Int, ellipsis: String = "..."): String = {
      if (s.length <= maxLength) s
      else s.take(maxLength - ellipsis.length) + ellipsis
    }
    def toSnakeCase: String = 
      "[A-Z]".r.replaceAllIn(s, m => "_" + m.group(0).toLowerCase)
        .stripPrefix("_")
  }
}

object IntExtensions {
  implicit class RichInt(val n: Int) extends AnyVal {
    def factorial: Long = (1L to n).product
    def isPrime: Boolean = n > 1 && (2 to Math.sqrt(n).toInt).forall(n % _ != 0)
    def times(f: => Unit): Unit = (1 to n).foreach(_ => f)
    def to(other: Int) = scala.collection.immutable.Range.inclusive(n, other)
  }
}

object ImplicitConversionDemo extends App {
  import StringExtensions._
  import IntExtensions._
  
  println("=== Implicit Conversions ===")
  
  // Temperature conversion
  val bodyTemp: Celsius = new Fahrenheit(98.6)  // Auto-converted
  println(s"98.6°F = $bodyTemp")
  
  val boilingF: Fahrenheit = new Celsius(100)  // Auto-converted
  println(s"100°C = $boilingF")
  
  // Implicit class enrichment
  println("\n=== String Extensions ===")
  println("hello world".toTitleCase)
  println("racecar".isPalindrome)
  println("not a palindrome".isPalindrome)
  println("The quick brown fox".wordCount)
  println("A very long string that needs truncating".truncate(20))
  println("CamelCaseString".toSnakeCase)
  
  println("\n=== Int Extensions ===")
  println(s"5! = ${5.factorial}")
  println(s"7! = ${7.factorial}")
  println(s"17 is prime: ${17.isPrime}")
  println(s"18 is prime: ${18.isPrime}")
  
  var count = 0
  3.times { count += 1 }
  println(s"count after 3.times: $count")
}
```

---

## Step 127: Companion Objects สำหรับ Enum-like Values

```scala
// Using companion object to create enum-like sealed hierarchy
sealed abstract class Color(val red: Int, val green: Int, val blue: Int) {
  def hex: String = f"#$red%02X$green%02X$blue%02X"
  def rgb: String = s"rgb($red, $green, $blue)"
  def isDark: Boolean = (red * 299 + green * 587 + blue * 114) / 1000 < 128
  def mix(other: Color): Color = Color.custom(
    (red + other.red) / 2,
    (green + other.green) / 2,
    (blue + other.blue) / 2
  )
  override def toString: String = s"Color(${hex})"
}

object Color {
  case object Red extends Color(255, 0, 0)
  case object Green extends Color(0, 255, 0)
  case object Blue extends Color(0, 0, 255)
  case object White extends Color(255, 255, 255)
  case object Black extends Color(0, 0, 0)
  case object Yellow extends Color(255, 255, 0)
  case object Cyan extends Color(0, 255, 255)
  case object Magenta extends Color(255, 0, 255)
  
  // Named colors
  val ThaiRed = Color.custom(200, 16, 46)
  val ThaiBlue = Color.custom(39, 69, 141)
  
  // Factory for custom colors
  def custom(r: Int, g: Int, b: Int): Color = {
    require(0 <= r && r <= 255, "Red out of range")
    require(0 <= g && g <= 255, "Green out of range")
    require(0 <= b && b <= 255, "Blue out of range")
    new Color(r, g, b) {}
  }
  
  // Parse hex color
  def fromHex(hex: String): Option[Color] = {
    val cleanHex = hex.stripPrefix("#")
    if (cleanHex.length != 6) None
    else try {
      Some(custom(
        Integer.parseInt(cleanHex.substring(0, 2), 16),
        Integer.parseInt(cleanHex.substring(2, 4), 16),
        Integer.parseInt(cleanHex.substring(4, 6), 16)
      ))
    } catch {
      case _: NumberFormatException => None
    }
  }
  
  val all: List[Color] = List(Red, Green, Blue, White, Black, Yellow, Cyan, Magenta)
}

object ColorDemo extends App {
  println("=== Colors ===")
  Color.all.foreach { c =>
    println(s"${c.toString.padTo(20, ' ')} ${c.hex}  ${c.rgb}  Dark: ${c.isDark}")
  }
  
  println("\n=== Color Operations ===")
  val mixed = Color.Red.mix(Color.Blue)
  println(s"Red + Blue = ${mixed.hex}")
  
  val custom = Color.custom(128, 64, 200)
  println(s"Custom: ${custom.hex}, Dark: ${custom.isDark}")
  
  println("\n=== Parse Hex ===")
  List("#FF5733", "#3498DB", "invalid", "#GGGGGG").foreach { hex =>
    Color.fromHex(hex) match {
      case Some(c) => println(s"  $hex -> ${c.rgb}")
      case None    => println(s"  $hex -> Invalid")
    }
  }
}
```

---

## Step 128: Builder Pattern ด้วย Companion Object

```scala
// Builder Pattern
case class HttpRequest(
  method: String,
  url: String,
  headers: Map[String, String],
  body: Option[String],
  timeout: Int,
  followRedirects: Boolean,
  auth: Option[(String, String)]
)

object HttpRequest {
  
  // Builder class
  class Builder(method: String, url: String) {
    private var headers: Map[String, String] = Map.empty
    private var body: Option[String] = None
    private var timeout: Int = 30000
    private var followRedirects: Boolean = true
    private var auth: Option[(String, String)] = None
    
    def withHeader(name: String, value: String): Builder = {
      headers = headers + (name -> value)
      this
    }
    
    def withHeaders(h: Map[String, String]): Builder = {
      headers = headers ++ h
      this
    }
    
    def withBody(b: String, contentType: String = "application/json"): Builder = {
      body = Some(b)
      headers = headers + ("Content-Type" -> contentType)
      this
    }
    
    def withTimeout(ms: Int): Builder = {
      timeout = ms
      this
    }
    
    def withAuth(username: String, password: String): Builder = {
      auth = Some((username, password))
      // Add Basic auth header
      val credentials = java.util.Base64.getEncoder.encodeToString(
        s"$username:$password".getBytes
      )
      headers = headers + ("Authorization" -> s"Basic $credentials")
      this
    }
    
    def withBearerToken(token: String): Builder = {
      headers = headers + ("Authorization" -> s"Bearer $token")
      this
    }
    
    def noRedirects(): Builder = {
      followRedirects = false
      this
    }
    
    def build(): HttpRequest = HttpRequest(
      method, url, headers, body, timeout, followRedirects, auth
    )
  }
  
  // Entry points
  def get(url: String): Builder = new Builder("GET", url)
  def post(url: String): Builder = new Builder("POST", url)
  def put(url: String): Builder = new Builder("PUT", url)
  def delete(url: String): Builder = new Builder("DELETE", url)
  def patch(url: String): Builder = new Builder("PATCH", url)
}

object BuilderPatternDemo extends App {
  // Build various HTTP requests
  val getRequest = HttpRequest
    .get("https://api.example.com/users")
    .withBearerToken("my-jwt-token")
    .withHeader("Accept", "application/json")
    .withTimeout(5000)
    .build()
  
  println("=== GET Request ===")
  println(s"Method: ${getRequest.method}")
  println(s"URL: ${getRequest.url}")
  println(s"Headers: ${getRequest.headers}")
  println(s"Timeout: ${getRequest.timeout}ms")
  
  val postRequest = HttpRequest
    .post("https://api.example.com/users")
    .withBearerToken("my-jwt-token")
    .withBody("""{"name":"Alice","email":"alice@example.com"}""")
    .withTimeout(10000)
    .build()
  
  println("\n=== POST Request ===")
  println(s"Method: ${postRequest.method}")
  println(s"Body: ${postRequest.body}")
  println(s"Content-Type: ${postRequest.headers.get("Content-Type")}")
  
  val authRequest = HttpRequest
    .get("https://api.example.com/protected")
    .withAuth("admin", "password123")
    .noRedirects()
    .build()
  
  println("\n=== Auth Request ===")
  println(s"Auth: ${authRequest.auth}")
  println(s"Follow redirects: ${authRequest.followRedirects}")
}
```

---

## Step 129: Object Singleton และ Global State

```scala
// Object as Singleton - เป็น instance เดียวใน JVM
object AppConfig {
  // Loaded from environment
  val environment: String = sys.env.getOrElse("APP_ENV", "development")
  val port: Int = sys.env.get("PORT").map(_.toInt).getOrElse(8080)
  val dbUrl: String = sys.env.getOrElse("DATABASE_URL", "postgresql://localhost/dev")
  val secretKey: String = sys.env.getOrElse("SECRET_KEY", "dev-secret-key")
  
  val isDevelopment: Boolean = environment == "development"
  val isProduction: Boolean = environment == "production"
  
  def summary: String = 
    s"AppConfig(env=$environment, port=$port, db=${dbUrl.take(20)}...)"
}

// Thread-safe counter using object
object RequestCounter {
  private var count: Long = 0L
  private val lock = new Object()
  
  def increment(): Long = lock.synchronized {
    count += 1
    count
  }
  
  def current: Long = lock.synchronized(count)
  
  def reset(): Unit = lock.synchronized { count = 0L }
}

// Registry pattern
object PluginRegistry {
  private var plugins: Map[String, () => Any] = Map.empty
  
  def register(name: String, factory: () => Any): Unit = {
    plugins = plugins.updated(name, factory)
    println(s"Registered plugin: $name")
  }
  
  def create(name: String): Option[Any] = 
    plugins.get(name).map(_())
  
  def list(): List[String] = plugins.keys.toList.sorted
}

object SingletonDemo extends App {
  println("=== App Config ===")
  println(AppConfig.summary)
  println(s"Is Dev: ${AppConfig.isDevelopment}")
  
  println("\n=== Request Counter ===")
  val ids = (1 to 5).map(_ => RequestCounter.increment())
  println(s"Request IDs: $ids")
  println(s"Current count: ${RequestCounter.current}")
  
  println("\n=== Plugin Registry ===")
  PluginRegistry.register("json-processor", () => "JSONProcessor instance")
  PluginRegistry.register("csv-processor", () => "CSVProcessor instance")
  PluginRegistry.register("xml-processor", () => "XMLProcessor instance")
  
  println(s"Registered: ${PluginRegistry.list()}")
  println(s"Create json: ${PluginRegistry.create("json-processor")}")
  println(s"Create missing: ${PluginRegistry.create("missing")}")
}
```

---

## Step 130: ตัวอย่าง Real-World — Event System

```scala
// Real-world companion object usage: Event system
import scala.collection.mutable

sealed trait DomainEvent {
  def eventId: String
  def timestamp: java.time.Instant
  def aggregateId: String
}

case class UserCreated(
  userId: String,
  email: String,
  name: String
) extends DomainEvent {
  override val eventId: String = java.util.UUID.randomUUID().toString
  override val timestamp: java.time.Instant = java.time.Instant.now()
  override val aggregateId: String = userId
}

case class OrderPlaced(
  orderId: String,
  userId: String,
  items: List[String],
  totalAmount: Double
) extends DomainEvent {
  override val eventId: String = java.util.UUID.randomUUID().toString
  override val timestamp: java.time.Instant = java.time.Instant.now()
  override val aggregateId: String = orderId
}

case class PaymentProcessed(
  paymentId: String,
  orderId: String,
  amount: Double,
  status: String
) extends DomainEvent {
  override val eventId: String = java.util.UUID.randomUUID().toString
  override val timestamp: java.time.Instant = java.time.Instant.now()
  override val aggregateId: String = paymentId
}

// Event Bus
object EventBus {
  private type Handler = DomainEvent => Unit
  private val handlers = mutable.Map[Class[_], mutable.ListBuffer[Handler]]()
  private val eventLog = mutable.ListBuffer[DomainEvent]()
  
  def subscribe[T <: DomainEvent](handler: T => Unit)(implicit ct: reflect.ClassTag[T]): Unit = {
    val clazz = ct.runtimeClass
    handlers.getOrElseUpdate(clazz, mutable.ListBuffer.empty) += 
      (event => handler(event.asInstanceOf[T]))
    println(s"Subscribed to ${clazz.getSimpleName}")
  }
  
  def publish(event: DomainEvent): Unit = {
    eventLog += event
    val clazz = event.getClass
    handlers.get(clazz).foreach(_.foreach(h => h(event)))
    println(s"Published: ${clazz.getSimpleName}(${event.aggregateId})")
  }
  
  def eventHistory(): List[DomainEvent] = eventLog.toList
  
  def clear(): Unit = {
    handlers.clear()
    eventLog.clear()
  }
}

// Domain services subscribing to events
object NotificationHandler {
  def setup(): Unit = {
    EventBus.subscribe[UserCreated] { event =>
      println(s"  [Notification] Welcome email sent to ${event.email}")
    }
    
    EventBus.subscribe[OrderPlaced] { event =>
      println(s"  [Notification] Order confirmation sent for ${event.orderId}")
    }
    
    EventBus.subscribe[PaymentProcessed] { event =>
      println(s"  [Notification] Payment ${event.status} for order ${event.orderId}")
    }
  }
}

object AnalyticsHandler {
  def setup(): Unit = {
    EventBus.subscribe[UserCreated] { event =>
      println(s"  [Analytics] New user tracked: ${event.userId}")
    }
    
    EventBus.subscribe[OrderPlaced] { event =>
      println(f"  [Analytics] Revenue tracked: ${event.totalAmount}%.2f THB")
    }
  }
}

object EventSystemDemo extends App {
  // Setup handlers
  NotificationHandler.setup()
  AnalyticsHandler.setup()
  
  println("\n=== Publishing Events ===")
  
  EventBus.publish(UserCreated("U001", "alice@example.com", "Alice Chen"))
  println()
  
  EventBus.publish(OrderPlaced("O001", "U001", List("Laptop", "Mouse"), 46000.0))
  println()
  
  EventBus.publish(PaymentProcessed("P001", "O001", 46000.0, "SUCCESS"))
  
  println("\n=== Event History ===")
  EventBus.eventHistory().zipWithIndex.foreach { case (event, idx) =>
    println(s"  ${idx + 1}. ${event.getClass.getSimpleName} @ ${event.timestamp}")
  }
}
```

---

## สรุป Part 13

| Pattern | การใช้งาน | ตัวอย่าง |
|---------|-----------|---------|
| Companion Object | สร้าง factory methods, constants | `object User { def apply(...) }` |
| apply method | สร้าง instance ด้วย syntax พิเศษ | `User("Alice")` |
| unapply method | pattern matching | `case User(name) =>` |
| Smart Constructor | validate ก่อนสร้าง | `Email.create(str): Either` |
| Builder Pattern | สร้าง object ซับซ้อน | `HttpRequest.get(url).withAuth(...).build()` |
| Factory Method | named constructors | `DatabaseConnection.postgres(...)` |
| Singleton Object | global state/registry | `object AppConfig` |
| Implicit Class | extend existing types | `implicit class RichString(s: String)` |

---

## แบบฝึกหัด Part 13

**ข้อ 1:** สร้าง `class Money` ที่มี companion object พร้อม factory methods `Money.baht(amount)`, `Money.dollar(amount)`, `Money.fromString("100 THB")` และ smart constructor ที่ validate

**ข้อ 2:** สร้าง `class Config` ที่อ่านค่าจาก environment variables พร้อม companion object ที่มี pre-built instances สำหรับ `development`, `staging`, `production`

**ข้อ 3:** Implement Builder Pattern สำหรับ SQL Query: `SELECT.from("users").where("age > 18").orderBy("name").limit(10).build()`

**ข้อ 4:** สร้าง custom `unapply` สำหรับ Phone number ที่ parse เบอร์โทรศัพท์ไทย เช่น `"0812345678"` → `("08", "1234", "5678")`

**ข้อ 5:** สร้าง `EventStore` object ที่เก็บ event log ทั้งหมด พร้อม `replay()` method และ companion class ที่มี `subscribe/publish`

---

➡️ ต่อไป: [Part 14 — Higher-Order Functions](part-14-higher-order-functions.md)
