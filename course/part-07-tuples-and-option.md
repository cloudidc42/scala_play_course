# Part 07 — Tuples, Option และ Either
## Steps 61–70: การจัดการค่าที่อาจไม่มี (Null Safety)

> **เป้าหมาย**: เขียน null-safe code ด้วย Option, Either, Try และ tuples

---

## Step 61 — Tuples

Tuples คือ ordered collection ของ values ที่มี fixed size และ fixed types

```scala
// Create tuples
val pair  = (1, "hello")                    // (Int, String)
val triple = (1, "hello", true)             // (Int, String, Boolean)
val quad  = (1, "hello", 3.14, true)        // (Int, String, Double, Boolean)

// Explicit type
val t: (Int, String, Boolean) = (42, "scala", true)

// Access elements (_1, _2, ...)
println(pair._1)    // 1
println(pair._2)    // hello
println(triple._3)  // true

// Destructuring
val (x, y) = pair
println(s"x=$x, y=$y")  // x=1, y=hello

val (a, b, c) = triple
println(s"a=$a, b=$b, c=$c")  // a=1, b=hello, c=true

// Ignore with _
val (num, _, flag) = triple
println(s"num=$num, flag=$flag")  // num=1, flag=true

// Tuple2 shorthand: ->
val kv = "name" -> "Alice"  // (String, String) = ("name", "Alice")
println(kv._1)  // name
println(kv._2)  // Alice
```

### Tuples กับ Collections

```scala
// List of tuples
val data = List(
  ("Alice", 30, 9.5),
  ("Bob",   25, 8.2),
  ("Charlie", 35, 7.8)
)

// Destructure in for
for ((name, age, score) <- data) {
  println(f"$name%-10s age=$age score=$score%.1f")
}

// Unzip
val names = data.map(_._1)
val ages  = data.map(_._2)
val (names2, rest) = data.map(t => (t._1, (t._2, t._3))).unzip

// zip creates List of tuples
val xs = List(1, 2, 3)
val ys = List("a", "b", "c")
val zipped = xs.zip(ys)          // List((1,a), (2,b), (3,c))
val (unzipped1, unzipped2) = zipped.unzip

// Tuple as Map entry
val map = Map("a" -> 1, "b" -> 2)  // Map[String, Int]
map.foreach { case (k, v) => println(s"$k -> $v") }

// toMap from List[(K, V)]
val pairs = List("x" -> 10, "y" -> 20, "z" -> 30)
val asMap = pairs.toMap  // Map(x -> 10, y -> 20, z -> 30)
```

---

## Step 62 — Option เบื้องต้น

Option แทน "ค่าที่อาจมีหรือไม่มี" — ใช้แทน null

```scala
// Option = Some(value) หรือ None
val maybeAge: Option[Int]    = Some(30)
val noAge: Option[Int]       = None
val maybeName: Option[String] = Some("Alice")
val noName: Option[String]   = None

// อย่าใช้ null! ใช้ Option แทน
// val name: String = null  // ❌ ทำไม่ได้ใน idiomatic Scala
// val name: Option[String] = None  // ✓

// Basic operations
println(maybeAge.isDefined)   // true
println(maybeAge.isEmpty)     // false
println(noAge.isDefined)      // false
println(noAge.isEmpty)        // true

// Get value
println(maybeAge.get)                    // 30
// println(noAge.get)                    // NoSuchElementException!

// Safe get
println(maybeAge.getOrElse(0))           // 30
println(noAge.getOrElse(0))              // 0 (default)
println(noAge.getOrElse(throw new RuntimeException("No age!")))

// orElse — alternative Option
println(noAge.orElse(Some(99)))          // Some(99)
println(maybeAge.orElse(Some(99)))       // Some(30) (original)
```

### Option Operations

```scala
val age: Option[Int] = Some(25)

// map — transform if present
val nextYear = age.map(_ + 1)         // Some(26)
val ageStr = age.map(_.toString)      // Some("25")
val noResult = None.map(_ + 1)        // None (type: Option[Nothing])

// flatMap — returns Option
def toAdult(n: Int): Option[String] =
  if (n >= 18) Some("Adult") else None

val adult = age.flatMap(toAdult)      // Some("Adult")

// filter
val allowedAge = age.filter(_ >= 18)  // Some(25)
val blocked = age.filter(_ >= 30)     // None

// for comprehension (ดีมาก!)
val user: Option[String] = Some("Alice")
val email: Option[String] = Some("alice@example.com")
val score: Option[Double] = Some(9.5)
val noScore: Option[Double] = None

// ถ้าทุก Option มีค่า
val greeting = for {
  name  <- user
  mail  <- email
  grade <- score
} yield s"$name ($mail): $grade"
println(greeting)  // Some(Alice (alice@example.com): 9.5)

// ถ้า Option ใดเป็น None → ผลลัพธ์คือ None
val broken = for {
  name  <- user
  mail  <- email
  grade <- noScore  // None → short-circuit!
} yield s"$name ($mail): $grade"
println(broken)  // None

// fold: transform both cases
val result = age.fold("Unknown")(a => s"Age is $a")
println(result)  // Age is 25

val noneResult = noAdult.fold("Unknown")(a => s"Age is $a")
```

---

## Step 63 — Option Pattern Matching

```scala
val maybeScore: Option[Int] = Some(85)

// Pattern match on Option
maybeScore match {
  case Some(score) => println(s"Score: $score")
  case None        => println("No score")
}

// เพิ่ม guard
maybeScore match {
  case Some(score) if score >= 90 => println("Excellent!")
  case Some(score) if score >= 70 => println(s"Good: $score")
  case Some(score)                => println(s"Needs work: $score")
  case None                       => println("Not scored")
}

// Option กับ functions ที่ return null (Java)
val javaMap = new java.util.HashMap[String, String]()
javaMap.put("name", "Alice")

// Java: อาจ return null
// val v = javaMap.get("missing")  // null!

// Scala: wrap ด้วย Option
val safeGet = Option(javaMap.get("name"))     // Some("Alice")
val missing = Option(javaMap.get("missing"))  // None

// Convert nullable to Option
def findUser(id: Int): Option[String] = {
  val users = Map(1 -> "Alice", 2 -> "Bob")
  users.get(id)  // Map.get returns Option
}

println(findUser(1))   // Some(Alice)
println(findUser(99))  // None
```

---

## Step 64 — Option กับ Database / API

```scala
// Real-world patterns

case class User(id: Int, name: String, email: String)
case class Order(id: Int, userId: Int, total: Double)
case class Product(id: Int, name: String, price: Double)

// Repository pattern
object UserRepo {
  private val users = Map(
    1 -> User(1, "Alice", "alice@example.com"),
    2 -> User(2, "Bob",   "bob@example.com")
  )
  
  def findById(id: Int): Option[User] = users.get(id)
  
  def findByEmail(email: String): Option[User] =
    users.values.find(_.email == email)
}

object OrderRepo {
  private val orders = Map(
    101 -> Order(101, 1, 1500.00),
    102 -> Order(102, 1, 3200.00),
    103 -> Order(103, 2, 750.00)
  )
  
  def findById(id: Int): Option[Order] = orders.get(id)
  
  def findByUser(userId: Int): List[Order] =
    orders.values.filter(_.userId == userId).toList
}

// Chain operations safely
def getUserEmail(userId: Int): Option[String] =
  UserRepo.findById(userId).map(_.email)

def getUserOrders(userId: Int): List[Order] =
  UserRepo.findById(userId)
    .map(u => OrderRepo.findByUser(u.id))
    .getOrElse(List.empty)

def getTotalSpent(userId: Int): Option[Double] =
  UserRepo.findById(userId).map { user =>
    val orders = OrderRepo.findByUser(user.id)
    if (orders.isEmpty) 0.0
    else orders.map(_.total).sum
  }

// Test
println(getUserEmail(1))        // Some(alice@example.com)
println(getUserEmail(99))       // None
println(getUserOrders(1))       // List of orders
println(getTotalSpent(1))       // Some(4700.0)
println(getTotalSpent(99))      // None
```

---

## Step 65 — Either

Either แทน "success หรือ error" — `Right` คือ success, `Left` คือ error

```scala
// Either[Error, Success]
type Result[A] = Either[String, A]

def divide(a: Int, b: Int): Either[String, Int] =
  if (b == 0) Left("Division by zero")
  else Right(a / b)

println(divide(10, 2))   // Right(5)
println(divide(10, 0))   // Left(Division by zero)

// Pattern match
divide(10, 3) match {
  case Right(result) => println(s"Result: $result")
  case Left(error)   => println(s"Error: $error")
}

// map — transform Right only
val doubled = divide(10, 2).map(_ * 2)   // Right(10)
val failed  = divide(10, 0).map(_ * 2)   // Left(Division by zero)

// flatMap — chain operations
def sqrt(n: Int): Either[String, Double] =
  if (n < 0) Left("Cannot sqrt negative")
  else Right(math.sqrt(n))

val result = divide(100, 4).flatMap(n => sqrt(n))
println(result)  // Right(5.0)

val error = divide(100, 0).flatMap(n => sqrt(n))
println(error)   // Left(Division by zero)

// for comprehension
def processNumber(input: String): Either[String, Double] =
  for {
    str    <- if (input.nonEmpty) Right(input) else Left("Empty input")
    n      <- str.toIntOption.toRight(s"Not a number: $str")
    result <- divide(100, n)
    root   <- sqrt(result)
  } yield root

println(processNumber("4"))    // Right(5.0)
println(processNumber("0"))    // Left(Division by zero)
println(processNumber("abc"))  // Left(Not a number: abc)
println(processNumber(""))     // Left(Empty input)
```

---

## Step 66 — Either กับ Error Handling

```scala
// Typed errors with sealed trait
sealed trait AppError
case class ValidationError(field: String, message: String) extends AppError
case class DatabaseError(cause: String) extends AppError
case class NotFoundError(entity: String, id: Int) extends AppError
case class UnauthorizedError(userId: Int) extends AppError

type AppResult[A] = Either[AppError, A]

case class CreateUserRequest(name: String, email: String, age: Int)
case class User(id: Int, name: String, email: String, age: Int)

def validateName(name: String): AppResult[String] =
  if (name.length < 2) Left(ValidationError("name", "Too short (min 2 chars)"))
  else if (name.length > 50) Left(ValidationError("name", "Too long (max 50 chars)"))
  else Right(name.trim)

def validateEmail(email: String): AppResult[String] =
  if (!email.contains("@")) Left(ValidationError("email", "Invalid email format"))
  else if (email.length > 100) Left(ValidationError("email", "Email too long"))
  else Right(email.trim.toLowerCase)

def validateAge(age: Int): AppResult[Int] =
  if (age < 0) Left(ValidationError("age", "Age cannot be negative"))
  else if (age > 150) Left(ValidationError("age", "Age too large"))
  else Right(age)

def createUser(req: CreateUserRequest): AppResult[User] =
  for {
    name  <- validateName(req.name)
    email <- validateEmail(req.email)
    age   <- validateAge(req.age)
  } yield User(
    id    = scala.util.Random.nextInt(10000),
    name  = name,
    email = email,
    age   = age
  )

// Test
val requests = List(
  CreateUserRequest("Alice", "alice@example.com", 30),
  CreateUserRequest("A", "alice@example.com", 30),       // name too short
  CreateUserRequest("Bob", "not-an-email", 25),           // bad email
  CreateUserRequest("Charlie", "c@c.com", -5)            // negative age
)

requests.foreach { req =>
  createUser(req) match {
    case Right(user) => println(s"✓ Created: $user")
    case Left(ValidationError(field, msg)) => println(s"✗ Validation '$field': $msg")
    case Left(e) => println(s"✗ Error: $e")
  }
}
```

---

## Step 67 — Try

Try แทน "computation ที่อาจ throw exception"

```scala
import scala.util.{Try, Success, Failure}

// Try = Success(value) หรือ Failure(exception)
val t1: Try[Int] = Try(10 / 2)      // Success(5)
val t2: Try[Int] = Try(10 / 0)      // Failure(ArithmeticException)
val t3: Try[Int] = Try("abc".toInt) // Failure(NumberFormatException)

println(t1)  // Success(5)
println(t2)  // Failure(java.lang.ArithmeticException: / by zero)

// Pattern match
t1 match {
  case Success(value)   => println(s"Got: $value")
  case Failure(ex)      => println(s"Error: ${ex.getMessage}")
}

// Operations
val doubled = t1.map(_ * 2)           // Success(10)
val failed  = t2.map(_ * 2)           // Failure(...)
val safe    = t2.getOrElse(0)         // 0
val recover = t2.recover { case _: ArithmeticException => -1 }  // Success(-1)

// Convert
val option: Option[Int] = t1.toOption  // Some(5)
val either: Either[Throwable, Int] = t1.toEither  // Right(5)

// For comprehension
def parseInt(s: String): Try[Int] = Try(s.toInt)
def safeDivide(a: Int, b: Int): Try[Int] = Try(a / b)

val result = for {
  a <- parseInt("100")
  b <- parseInt("5")
  c <- safeDivide(a, b)
} yield c

println(result)  // Success(20)
```

### Try กับ Resource Management

```scala
import scala.util.{Try, Using}
import scala.io.Source
import java.io.{File, PrintWriter}

// Using — automatic resource management (Scala 2.13+)
def readFile(path: String): Try[String] =
  Using(Source.fromFile(path))(_.mkString)

def writeFile(path: String, content: String): Try[Unit] =
  Using(new PrintWriter(new File(path)))(_.write(content))

def countWords(path: String): Try[Int] =
  Using(Source.fromFile(path)) { source =>
    source.getLines()
      .flatMap(_.split("\\s+"))
      .count(_.nonEmpty)
  }

// Test
writeFile("/tmp/test.txt", "Hello Scala World\nThis is a test") match {
  case Success(_)  => println("Written!")
  case Failure(ex) => println(s"Write failed: ${ex.getMessage}")
}

readFile("/tmp/test.txt") match {
  case Success(content) => println(s"Content:\n$content")
  case Failure(ex)      => println(s"Read failed: ${ex.getMessage}")
}

countWords("/tmp/test.txt") match {
  case Success(n)  => println(s"Word count: $n")
  case Failure(ex) => println(s"Failed: ${ex.getMessage}")
}
```

---

## Step 68 — Combining Option, Either, Try

```scala
// Convert between them
val opt: Option[Int] = Some(42)
val eit: Either[String, Int] = Right(42)
val tri: Try[Int] = Success(42)

// Option → Either
val o2e1: Either[String, Int] = opt.toRight("No value")     // Right(42)
val o2e2: Either[String, Int] = None.toRight("No value")    // Left(No value)

// Option → Try
val o2t: Try[Int] = opt.fold[Try[Int]](Failure(new NoSuchElementException))(Success(_))

// Either → Option
val e2o: Option[Int] = eit.toOption  // Some(42)

// Try → Option
val t2o: Option[Int] = tri.toOption  // Some(42)

// Try → Either
val t2e: Either[Throwable, Int] = tri.toEither  // Right(42)

// Real-world: chain them
def getConfig(key: String): Option[String] =
  sys.env.get(key).orElse(Some("default"))

def parsePort(s: String): Try[Int] = Try(s.toInt)

def validatePort(port: Int): Either[String, Int] =
  if (port >= 1 && port <= 65535) Right(port)
  else Left(s"Invalid port: $port")

val port = for {
  portStr <- getConfig("PORT").toRight("PORT not configured")
  portInt <- parsePort(portStr).toEither.left.map(_.getMessage)
  valid   <- validatePort(portInt)
} yield valid

println(port)  // Right(default)... หรือ Left(error)
```

---

## Step 69 — Validated (Error Accumulation)

```scala
// Either short-circuits (fail fast)
// Validated accumulates all errors

// Simple Validated implementation
sealed trait Validated[+E, +A] {
  def map[B](f: A => B): Validated[E, B] = this match {
    case Valid(a)     => Valid(f(a))
    case Invalid(err) => Invalid(err)
  }
}

case class Valid[A](value: A) extends Validated[Nothing, A]
case class Invalid[E](errors: List[E]) extends Validated[E, Nothing]

object Validated {
  def valid[A](a: A): Validated[Nothing, A] = Valid(a)
  def invalid[E](e: E): Validated[E, Nothing] = Invalid(List(e))
  
  // mapN — combine multiple Validated
  def mapN[E, A, B, C](
    va: Validated[E, A],
    vb: Validated[E, B]
  )(f: (A, B) => C): Validated[E, C] = (va, vb) match {
    case (Valid(a), Valid(b))       => Valid(f(a, b))
    case (Invalid(e1), Invalid(e2)) => Invalid(e1 ++ e2)
    case (Invalid(e1), _)           => Invalid(e1)
    case (_, Invalid(e2))           => Invalid(e2)
  }
}

// Use cats.Validated in production:
// import cats.data.Validated
// import cats.data.Validated.{Valid, Invalid}

// Validation functions
def validateName(name: String): Validated[String, String] =
  if (name.length >= 2) Validated.valid(name)
  else Validated.invalid("Name too short")

def validateEmail(email: String): Validated[String, String] =
  if (email.contains("@")) Validated.valid(email)
  else Validated.invalid("Invalid email")

// Combine: accumulates errors!
val nameV = validateName("A")
val emailV = validateEmail("not-email")

val combined = Validated.mapN(nameV, emailV)((n, e) => s"$n - $e")
println(combined)
// Invalid(List(Name too short, Invalid email))
// Both errors! (Unlike Either which stops at first)
```

---

## Step 70 — โปรแกรม Option/Either Complete

```scala
// SafeDataProcessor.scala

import scala.util.{Try, Success, Failure}

@main def safeDataDemo(): Unit =
  
  println("=== Safe Data Processing ===\n")
  
  // ==============================
  // 1. Safe Configuration
  // ==============================
  println("--- Configuration Loading ---")
  
  case class DbConfig(host: String, port: Int, name: String, user: String)
  
  def getEnv(key: String, default: Option[String] = None): Option[String] =
    sys.env.get(key).orElse(default)
  
  def loadDbConfig: Either[String, DbConfig] =
    for {
      host <- getEnv("DB_HOST", Some("localhost")).toRight("DB_HOST required")
      port <- getEnv("DB_PORT", Some("5432"))
                .toRight("DB_PORT required")
                .flatMap(s => Try(s.toInt).toEither.left.map(_.getMessage))
                .flatMap { p =>
                  if (p > 0 && p < 65536) Right(p) else Left(s"Invalid port: $p")
                }
      name <- getEnv("DB_NAME", Some("myapp")).toRight("DB_NAME required")
      user <- getEnv("DB_USER", Some("postgres")).toRight("DB_USER required")
    } yield DbConfig(host, port, name, user)
  
  loadDbConfig match {
    case Right(config) => println(s"Connected to: ${config.host}:${config.port}/${config.name}")
    case Left(error)   => println(s"Config error: $error")
  }
  
  // ==============================
  // 2. Safe Parsing
  // ==============================
  println("\n--- Safe Parsing ---")
  
  case class DataRecord(id: Int, value: Double, label: String)
  
  def parseRecord(line: String): Either[String, DataRecord] = {
    val parts = line.split(",").map(_.trim)
    if (parts.length != 3)
      Left(s"Expected 3 fields, got ${parts.length}: '$line'")
    else
      for {
        id    <- Try(parts(0).toInt).toEither.left.map(_ => s"Invalid id: '${parts(0)}'")
        value <- Try(parts(1).toDouble).toEither.left.map(_ => s"Invalid value: '${parts(1)}'")
        label = parts(2)
        _     <- if (label.nonEmpty) Right(()) else Left("Label cannot be empty")
      } yield DataRecord(id, value, label)
  }
  
  val lines = List(
    "1, 3.14, alpha",
    "2, 2.71, beta",
    "3, abc, gamma",       // invalid value
    "4, 1.41",             // missing label
    "5, 0.99, epsilon"
  )
  
  val (errors, records) = lines.map(parseRecord).partitionMap(identity)
  
  println(s"Parsed ${records.length} records, ${errors.length} errors:")
  records.foreach(r => println(f"  Record ${r.id}: ${r.value}%.2f (${r.label})"))
  errors.foreach(e => println(s"  Error: $e"))
  
  // ==============================
  // 3. Safe Computation Chain
  // ==============================
  println("\n--- Computation Chain ---")
  
  case class Stats(count: Int, sum: Double, min: Double, max: Double)
  
  def computeStats(records: List[DataRecord]): Option[Stats] = {
    if (records.isEmpty) None
    else Some(Stats(
      count = records.length,
      sum   = records.map(_.value).sum,
      min   = records.map(_.value).min,
      max   = records.map(_.value).max
    ))
  }
  
  def formatStats(stats: Stats): String =
    f"Count=${stats.count} Sum=${stats.sum}%.2f Min=${stats.min}%.2f Max=${stats.max}%.2f"
  
  computeStats(records)
    .map(formatStats)
    .foreach(println)
  
  // ==============================
  // 4. Option Chaining
  // ==============================
  println("\n--- Option Chaining ---")
  
  val inventory = Map(
    "widget" -> 100,
    "gadget" -> 50,
    "doohickey" -> 0
  )
  
  val prices = Map(
    "widget" -> 9.99,
    "gadget" -> 24.99
  )
  
  def checkAvailable(item: String): Option[Int] =
    inventory.get(item).filter(_ > 0)
  
  def getPrice(item: String): Option[Double] =
    prices.get(item)
  
  def calculateCost(item: String, qty: Int): Option[Double] =
    for {
      available <- checkAvailable(item)
      _         <- if (qty <= available) Some(()) else None
      price     <- getPrice(item)
    } yield price * qty
  
  val tests = List(
    ("widget", 10),
    ("gadget", 60),    // insufficient stock
    ("doohickey", 5),  // out of stock
    ("unknown", 1)     // not in inventory
  )
  
  tests.foreach { case (item, qty) =>
    calculateCost(item, qty) match {
      case Some(cost) => println(f"  ✓ $qty × $item = $$$cost%.2f")
      case None       => println(s"  ✗ Cannot order $qty × $item")
    }
  }
  
  println("\n=== Safe Data Processing Complete ===")
```

---

## สรุป Part 07

| Step | สิ่งที่เรียน |
|------|-------------|
| 61 | Tuples — creation, destructuring |
| 62 | Option เบื้องต้น — Some/None |
| 63 | Option pattern matching |
| 64 | Option กับ database/API patterns |
| 65 | Either — Right/Left |
| 66 | Either กับ typed errors |
| 67 | Try — Success/Failure |
| 68 | Combining Option/Either/Try |
| 69 | Validated (error accumulation) |
| 70 | โปรแกรม complete |

## แบบฝึกหัด

1. สร้าง `safeHead[A](list: List[A]): Option[A]` และ `safeLast`
2. เขียน parser สำหรับ phone number ที่ส่งคืน `Either[String, String]`
3. สร้าง `sequence[A](opts: List[Option[A]]): Option[List[A]]` ที่ fail ถ้า Option ใดเป็น None
4. implement `traverse[A, B](list: List[A])(f: A => Either[String, B]): Either[String, List[B]]`
5. ใช้ `Try` สร้าง retry logic สำหรับ HTTP request simulation

## ต่อไป

**[Part 08 →](part-08-classes-and-objects.md)** — Classes และ Objects เบื้องต้น
