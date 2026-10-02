# Part 14: Higher-Order Functions

## Steps 131-140: Higher-Order Functions, Function Composition, andThen, compose, Currying, Partial Application

---

## Step 131: Higher-Order Functions พื้นฐาน

Higher-Order Function (HOF) คือ function ที่รับ function อื่นเป็น parameter หรือ return function เป็น result

```scala
// Functions as values - function เป็น first-class citizen
object HOFBasics extends App {
  
  // Function type syntax: (InputType) => OutputType
  val double: Int => Int = x => x * 2
  val square: Int => Int = x => x * x
  val isEven: Int => Boolean = x => x % 2 == 0
  val toString2: Int => String = x => s"Number: $x"
  
  println("=== Functions as Values ===")
  println(s"double(5) = ${double(5)}")
  println(s"square(4) = ${square(4)}")
  println(s"isEven(6) = ${isEven(6)}")
  println(s"toString(7) = ${toString2(7)}")
  
  // Higher-order function - takes function as parameter
  def applyToAll(numbers: List[Int], f: Int => Int): List[Int] = 
    numbers.map(f)
  
  def applyAndFilter(numbers: List[Int], transform: Int => Int, predicate: Int => Boolean): List[Int] =
    numbers.map(transform).filter(predicate)
  
  val numbers = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
  
  println("\n=== Applying Functions ===")
  println(s"Doubled: ${applyToAll(numbers, double)}")
  println(s"Squared: ${applyToAll(numbers, square)}")
  println(s"Square then even: ${applyAndFilter(numbers, square, isEven)}")
  
  // Higher-order function that returns a function
  def multiplier(factor: Int): Int => Int = x => x * factor
  
  val times3 = multiplier(3)
  val times10 = multiplier(10)
  
  println("\n=== Functions Returning Functions ===")
  println(s"times3(7) = ${times3(7)}")
  println(s"times10(5) = ${times10(5)}")
  println(s"Applied: ${applyToAll(numbers, times3)}")
  
  // Function as return value - real-world example
  def validator(min: Int, max: Int): Int => Either[String, Int] = { n =>
    if (n < min) Left(s"$n is less than minimum $min")
    else if (n > max) Left(s"$n exceeds maximum $max")
    else Right(n)
  }
  
  val ageValidator = validator(0, 150)
  val scoreValidator = validator(0, 100)
  
  println("\n=== Validators ===")
  println(ageValidator(25))
  println(ageValidator(200))
  println(scoreValidator(85))
  println(scoreValidator(-5))
}
```

---

## Step 132: Function Types และ Type Aliases

```scala
// Function type aliases - ทำให้โค้ดอ่านง่ายขึ้น
object FunctionTypes extends App {
  
  // Type aliases for functions
  type Predicate[A] = A => Boolean
  type Transform[A, B] = A => B
  type Reducer[A] = (A, A) => A
  type Effect[A] = A => Unit
  
  // Using type aliases
  def filter[A](list: List[A], predicate: Predicate[A]): List[A] = 
    list.filter(predicate)
  
  def transform[A, B](list: List[A], f: Transform[A, B]): List[B] = 
    list.map(f)
  
  def reduce[A](list: List[A], f: Reducer[A]): A = 
    list.reduce(f)
  
  def forEach[A](list: List[A], effect: Effect[A]): Unit = 
    list.foreach(effect)
  
  val numbers = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
  
  val isEven: Predicate[Int] = _ % 2 == 0
  val doubleIt: Transform[Int, Int] = _ * 2
  val sum: Reducer[Int] = _ + _
  val printIt: Effect[Int] = n => print(s"$n ")
  
  println("=== Function Type Aliases ===")
  println(s"Even numbers: ${filter(numbers, isEven)}")
  println(s"Doubled: ${transform(numbers, doubleIt)}")
  println(s"Sum: ${reduce(numbers, sum)}")
  print("Print: ")
  forEach(numbers, printIt)
  println()
  
  // Multi-argument function types
  type BinaryOp[A] = (A, A) => A
  type Comparator[A] = (A, A) => Int
  
  val multiply: BinaryOp[Int] = _ * _
  val nameComparator: Comparator[String] = _.compareTo(_)
  
  println(s"\nmultiply(3, 7) = ${multiply(3, 7)}")
  
  val names = List("Charlie", "Alice", "Bob", "Diana")
  println(s"Sorted: ${names.sortWith((a, b) => nameComparator(a, b) < 0)}")
  
  // Function with multiple type parameters
  def zipWith[A, B, C](xs: List[A], ys: List[B])(f: (A, B) => C): List[C] =
    xs.zip(ys).map { case (x, y) => f(x, y) }
  
  val prices = List(100.0, 200.0, 150.0)
  val quantities = List(3, 1, 5)
  
  val totals = zipWith(prices, quantities)(_ * _)
  println(s"\nPrice x Qty = $totals")
  println(s"Grand total = ${totals.sum}")
}
```

---

## Step 133: Function Composition ด้วย compose และ andThen

```scala
// Function composition
object FunctionComposition extends App {
  
  // andThen: f andThen g = x => g(f(x))  (left to right)
  // compose: f compose g = x => f(g(x))  (right to left)
  
  val trim: String => String = _.trim
  val toLower: String => String = _.toLowerCase
  val removeSpaces: String => String = _.replaceAll("\\s+", "_")
  val addPrefix: String => String = "processed_" + _
  
  // Using andThen - natural reading order
  val normalizeUsername: String => String = 
    trim andThen toLower andThen removeSpaces
  
  val processUsername: String => String = 
    trim andThen toLower andThen removeSpaces andThen addPrefix
  
  println("=== Function Composition with andThen ===")
  println(normalizeUsername("  Alice Chen  "))   // alice_chen
  println(processUsername("  Bob Smith  "))      // processed_bob_smith
  
  // Using compose - reverse order
  val cleanString: String => String = removeSpaces compose toLower compose trim
  
  println(s"\ncompose: ${cleanString("  Hello World  ")}")  // hello_world
  
  // Practical pipeline
  case class RawData(text: String)
  case class ProcessedData(text: String, wordCount: Int, isClean: Boolean)
  
  val extractText: RawData => String = _.text
  val validateText: String => Either[String, String] = { text =>
    if (text.length < 5) Left("Text too short")
    else Right(text)
  }
  
  // Pipeline for data processing
  def processPipeline(raw: RawData): Either[String, ProcessedData] = {
    val cleaned = (trim andThen toLower)(extractText(raw))
    validateText(cleaned).map { text =>
      ProcessedData(
        text = text,
        wordCount = text.split("\\s+").length,
        isClean = !text.contains("badword")
      )
    }
  }
  
  val inputs = List(
    RawData("  Hello World How Are You  "),
    RawData("Hi"),  // Too short
    RawData("  Clean Text Here  ")
  )
  
  println("\n=== Data Processing Pipeline ===")
  inputs.foreach { input =>
    processPipeline(input) match {
      case Right(data) => println(s"OK: $data")
      case Left(err)   => println(s"Error: $err for '${input.text.trim}'")
    }
  }
  
  // Numeric function composition
  val addOne: Int => Int = _ + 1
  val timesTwo: Int => Int = _ * 2
  val square: Int => Int = x => x * x
  
  val f1 = addOne andThen timesTwo andThen square  // ((x+1)*2)^2
  val f2 = square compose timesTwo compose addOne   // same as f1
  
  println("\n=== Numeric Composition ===")
  (1 to 5).foreach { n =>
    println(s"f1($n) = ${f1(n)}, f2($n) = ${f2(n)}")
  }
}
```

---

## Step 134: Currying — แปลง Function หลาย Args เป็น Function Chains

```scala
// Currying - converting (a, b) => c to a => b => c
object CurryingDemo extends App {
  
  // Non-curried vs curried
  def addNormal(x: Int, y: Int): Int = x + y
  def addCurried(x: Int)(y: Int): Int = x + y  // Multi-parameter list
  
  // Using curried version
  val add5 = addCurried(5) _  // Partial application
  val add10 = addCurried(10) _
  
  println("=== Currying Basics ===")
  println(s"addNormal(3, 4) = ${addNormal(3, 4)}")
  println(s"addCurried(3)(4) = ${addCurried(3)(4)}")
  println(s"add5(7) = ${add5(7)}")
  println(s"add10(3) = ${add10(3)}")
  
  // Converting between curried and uncurried
  val curriedAdd = (addNormal _).curried
  val uncurriedAdd = Function.uncurried(addCurried _)
  
  println(s"\ncurriedAdd(3)(4) = ${curriedAdd(3)(4)}")
  println(s"uncurriedAdd(3, 4) = ${uncurriedAdd(3, 4)}")
  
  // Practical currying examples
  def filter[A](predicate: A => Boolean)(list: List[A]): List[A] = 
    list.filter(predicate)
  
  def map[A, B](f: A => B)(list: List[A]): List[B] = 
    list.map(f)
  
  // Create specialized functions
  val filterEven = filter((n: Int) => n % 2 == 0) _
  val filterPositive = filter((n: Int) => n > 0) _
  val doubleAll = map((n: Int) => n * 2) _
  
  val numbers = List(-3, -1, 0, 1, 2, 3, 4, 5, 6)
  
  println("\n=== Specialized Functions from Currying ===")
  println(s"Even: ${filterEven(numbers)}")
  println(s"Positive: ${filterPositive(numbers)}")
  println(s"Doubled: ${doubleAll(numbers)}")
  
  // Logger with currying
  def log(level: String)(message: String): Unit = 
    println(s"[$level] $message")
  
  val info = log("INFO") _
  val warn = log("WARN") _
  val error = log("ERROR") _
  
  println("\n=== Curried Logger ===")
  info("Application started")
  warn("Low memory detected")
  error("Connection failed!")
  
  // Database query builder using currying
  def query(table: String)(conditions: Map[String, Any])(columns: List[String]): String = {
    val cols = if (columns.isEmpty) "*" else columns.mkString(", ")
    val where = if (conditions.isEmpty) ""
    else " WHERE " + conditions.map { case (k, v) => s"$k = '$v'" }.mkString(" AND ")
    s"SELECT $cols FROM $table$where"
  }
  
  val queryUsers = query("users") _
  val activeUsers = queryUsers(Map("status" -> "active")) _
  
  println("\n=== Query Builder ===")
  println(queryUsers(Map())(List("id", "name", "email")))
  println(activeUsers(List("id", "name")))
  println(activeUsers(List()))  // SELECT * FROM users WHERE status = 'active'
}
```

---

## Step 135: Partial Application

```scala
// Partial Application - applying some arguments now, rest later
object PartialApplicationDemo extends App {
  
  // Partial application with underscore placeholder
  def multiply(x: Int, y: Int): Int = x * y
  
  val double = multiply(2, _: Int)
  val triple = multiply(3, _: Int)
  val timesNegOne = multiply(-1, _: Int)
  
  println("=== Partial Application ===")
  println(s"double(5) = ${double(5)}")
  println(s"triple(4) = ${triple(4)}")
  
  val nums = List(1, 2, 3, 4, 5)
  println(s"Doubled: ${nums.map(double)}")
  println(s"Negated: ${nums.map(timesNegOne)}")
  
  // Partial application for API calls
  def sendRequest(
    baseUrl: String,
    apiKey: String,
    endpoint: String,
    params: Map[String, String]
  ): String = {
    val queryString = params.map { case (k, v) => s"$k=$v" }.mkString("&")
    s"GET $baseUrl/$endpoint?apiKey=$apiKey&$queryString"
  }
  
  // Partially apply base URL and API key
  val sendToProductionApi = sendRequest("https://api.production.com", "prod-key-123", _: String, _: Map[String, String])
  val sendToTestApi = sendRequest("https://api.test.com", "test-key-456", _: String, _: Map[String, String])
  
  println("\n=== API Calls ===")
  println(sendToProductionApi("users/list", Map("page" -> "1", "limit" -> "10")))
  println(sendToTestApi("users/create", Map("name" -> "Alice")))
  
  // Using Function.partial (alternative approach)
  def formatMessage(template: String, level: String, message: String): String =
    template.replace("{level}", level).replace("{msg}", message)
  
  val emailTemplate = formatMessage("[{level}] Email: {msg}", _: String, _: String)
  val slackTemplate = formatMessage(":bell: [{level}] {msg}", _: String, _: String)
  
  println("\n=== Message Formatting ===")
  println(emailTemplate("ERROR", "Payment failed"))
  println(slackTemplate("INFO", "New user registered"))
  
  // Real-world: Middleware pattern
  type Request = Map[String, String]
  type Response = String
  type Handler = Request => Response
  type Middleware = Handler => Handler
  
  def withLogging(handler: Handler): Handler = { req =>
    println(s"  [LOG] Request: $req")
    val response = handler(req)
    println(s"  [LOG] Response: $response")
    response
  }
  
  def withAuth(secretKey: String)(handler: Handler): Handler = { req =>
    if (req.get("token") == Some(secretKey)) handler(req)
    else "401 Unauthorized"
  }
  
  def withCaching(ttlSeconds: Int)(handler: Handler): Handler = {
    val cache = scala.collection.mutable.Map[String, String]()
    req =>
      val key = req.toString
      cache.getOrElseUpdate(key, {
        println(s"  [CACHE] Miss for $key, computing...")
        handler(req)
      })
  }
  
  val myHandler: Handler = req => s"200 OK: Hello, ${req.getOrElse("name", "World")}!"
  
  // Apply middleware
  val withAuth123 = withAuth("secret123") _
  val secured = withLogging(withAuth123(withCaching(60)(myHandler)))
  
  println("\n=== Middleware ===")
  println(secured(Map("name" -> "Alice", "token" -> "secret123")))
  println(secured(Map("name" -> "Bob", "token" -> "wrong")))
}
```

---

## Step 136: Function Composition Patterns

```scala
// Advanced function composition patterns
object AdvancedComposition extends App {
  
  // Kleisli composition: functions that return effects (Option, Either, Future)
  type Kleisli[F[_], A, B] = A => F[B]
  
  // Option Kleisli
  def divideOpt(x: Double)(y: Double): Option[Double] =
    if (y == 0) None else Some(x / y)
  
  def sqrtOpt(x: Double): Option[Double] =
    if (x < 0) None else Some(Math.sqrt(x))
  
  def logOpt(x: Double): Option[Double] =
    if (x <= 0) None else Some(Math.log(x))
  
  // Compose Option functions
  def composeOpt[A, B, C](f: A => Option[B], g: B => Option[C]): A => Option[C] =
    a => f(a).flatMap(g)
  
  val safeSqrtOfLog = composeOpt(logOpt, sqrtOpt)
  val safeResult = composeOpt(sqrtOpt, logOpt)
  
  println("=== Kleisli Composition ===")
  List(-1.0, 0.0, 1.0, 2.0, Math.E * Math.E).foreach { n =>
    println(s"safeSqrtOfLog($n) = ${safeSqrtOfLog(n)}")
  }
  
  // Pipeline with transformations
  case class Order(
    id: String,
    items: List[(String, Int, Double)],  // (name, qty, price)
    discount: Double,
    tax: Double
  )
  
  type OrderProcessor = Order => Order
  
  val applyDiscount: OrderProcessor = order => {
    val discountedItems = order.items.map { case (name, qty, price) =>
      (name, qty, price * (1 - order.discount))
    }
    order.copy(items = discountedItems)
  }
  
  val applyTax: Order => Map[String, Double] = { order =>
    val subtotal = order.items.map { case (_, qty, price) => qty * price }.sum
    val taxAmount = subtotal * order.tax
    Map(
      "subtotal" -> subtotal,
      "tax" -> taxAmount,
      "total" -> (subtotal + taxAmount)
    )
  }
  
  val processOrder: Order => Map[String, Double] = applyDiscount andThen applyTax
  
  val order = Order(
    id = "ORD-001",
    items = List(
      ("Laptop", 1, 45000.0),
      ("Mouse", 2, 890.0),
      ("Keyboard", 1, 1500.0)
    ),
    discount = 0.10,  // 10% discount
    tax = 0.07        // 7% VAT
  )
  
  println("\n=== Order Processing Pipeline ===")
  val result = processOrder(order)
  result.foreach { case (k, v) => println(f"  $k: $v%.2f THB") }
  
  // Function lifting - ยก function ธรรมดาให้ทำงานกับ Option
  def liftOption[A, B](f: A => B): Option[A] => Option[B] = _.map(f)
  
  val parseIntSafe: String => Option[Int] = s => scala.util.Try(s.toInt).toOption
  val doubleIt: Int => Int = _ * 2
  val doubleIfParseable = parseIntSafe andThen liftOption(doubleIt)
  
  println("\n=== Function Lifting ===")
  List("42", "abc", "100", "xyz").foreach { s =>
    println(s"""doubleIfParseable("$s") = ${doubleIfParseable(s)}""")
  }
}
```

---

## Step 137: Memoization — Cache Function Results

```scala
// Memoization - cache expensive function results
object MemoizationDemo extends App {
  
  // Simple memoize
  def memoize[A, B](f: A => B): A => B = {
    val cache = scala.collection.mutable.Map[A, B]()
    a => cache.getOrElseUpdate(a, f(a))
  }
  
  // Fibonacci - classic example
  var fibCalls = 0
  
  def fibSlow(n: Int): Long = {
    fibCalls += 1
    if (n <= 1) n
    else fibSlow(n - 1) + fibSlow(n - 2)
  }
  
  // Memoized version
  lazy val fibMemo: Int => Long = memoize { n =>
    if (n <= 1) n.toLong
    else fibMemo(n - 1) + fibMemo(n - 2)
  }
  
  println("=== Memoization ===")
  
  val start1 = System.currentTimeMillis()
  println(s"fib(30) slow = ${fibSlow(30)}, calls = $fibCalls")
  println(s"Time: ${System.currentTimeMillis() - start1}ms")
  
  val start2 = System.currentTimeMillis()
  println(s"fib(50) memo = ${fibMemo(50)}")
  println(s"Time: ${System.currentTimeMillis() - start2}ms")
  
  // Memoize with TTL (time-to-live)
  import java.time.Instant
  
  def memoizeWithTTL[A, B](ttlMs: Long)(f: A => B): A => B = {
    val cache = scala.collection.mutable.Map[A, (B, Instant)]()
    a => {
      val now = Instant.now()
      cache.get(a) match {
        case Some((result, createdAt)) 
          if now.toEpochMilli - createdAt.toEpochMilli < ttlMs =>
          result  // Cache hit
        case _ =>
          val result = f(a)  // Cache miss or expired
          cache(a) = (result, now)
          result
      }
    }
  }
  
  // Simulating expensive API call
  var apiCallCount = 0
  def expensiveApiCall(userId: String): String = {
    apiCallCount += 1
    Thread.sleep(50)  // Simulate latency
    s"User data for $userId (fetched at ${Instant.now()})"
  }
  
  val cachedApiCall = memoizeWithTTL(1000)(expensiveApiCall)
  
  println("\n=== TTL Cache ===")
  val t1 = System.currentTimeMillis()
  println(cachedApiCall("user-001"))  // API call
  println(cachedApiCall("user-001"))  // Cache hit
  println(cachedApiCall("user-002"))  // API call
  println(cachedApiCall("user-001"))  // Cache hit
  println(s"API calls: $apiCallCount, time: ${System.currentTimeMillis() - t1}ms")
}
```

---

## Step 138: Higher-Order Functions สำหรับ Collections

```scala
// Practical HOF with collections
object CollectionHOF extends App {
  
  // Custom higher-order collection operations
  case class Student(
    name: String, 
    grade: Int, 
    scores: Map[String, Double],
    active: Boolean
  ) {
    def average: Double = 
      if (scores.isEmpty) 0.0 
      else scores.values.sum / scores.size
    
    def grade_: String = average match {
      case a if a >= 90 => "A"
      case a if a >= 80 => "B"
      case a if a >= 70 => "C"
      case a if a >= 60 => "D"
      case _            => "F"
    }
  }
  
  val students = List(
    Student("Alice", 11, Map("Math" -> 92.0, "Science" -> 88.0, "English" -> 95.0), true),
    Student("Bob", 11, Map("Math" -> 75.0, "Science" -> 70.0, "English" -> 68.0), true),
    Student("Carol", 12, Map("Math" -> 98.0, "Science" -> 96.0, "English" -> 94.0), true),
    Student("David", 12, Map("Math" -> 55.0, "Science" -> 60.0, "English" -> 58.0), false),
    Student("Eve", 11, Map("Math" -> 85.0, "Science" -> 90.0, "English" -> 82.0), true),
    Student("Frank", 12, Map("Math" -> 70.0, "Science" -> 72.0, "English" -> 75.0), true)
  )
  
  // HOF for student analysis
  def analyzeStudents(
    students: List[Student],
    filter: Student => Boolean,
    sortBy: Student => Double,
    transform: Student => String
  ): List[String] = {
    students
      .filter(filter)
      .sortBy(sortBy)
      .reverse
      .map(transform)
  }
  
  println("=== Student Analysis ===")
  
  // Top students by average
  val topStudents = analyzeStudents(
    students,
    filter = _.active,
    sortBy = _.average,
    transform = s => f"${s.name}: ${s.average}%.1f (${s.grade_})"
  )
  println("Top active students:")
  topStudents.foreach(s => println(s"  $s"))
  
  // Grade 11 students by Math score
  val grade11Math = analyzeStudents(
    students,
    filter = s => s.grade == 11,
    sortBy = s => s.scores.getOrElse("Math", 0.0),
    transform = s => s"${s.name}: Math=${s.scores.getOrElse("Math", 0.0)}"
  )
  println("\nGrade 11 by Math:")
  grade11Math.foreach(s => println(s"  $s"))
  
  // Aggregate functions
  def statisticsBy[T](
    students: List[Student],
    getValue: Student => T,
    aggregate: List[T] => Map[String, Any]
  ): Map[String, Any] = {
    aggregate(students.map(getValue))
  }
  
  def numericStats(values: List[Double]): Map[String, Any] = {
    if (values.isEmpty) Map()
    else Map(
      "count" -> values.size,
      "min" -> values.min,
      "max" -> values.max,
      "avg" -> values.sum / values.size,
      "median" -> values.sorted.apply(values.size / 2)
    )
  }
  
  val avgStats = statisticsBy(students.filter(_.active), _.average, 
    vs => numericStats(vs.map(_.asInstanceOf[Double])))
  
  println("\n=== Score Statistics ===")
  students.filter(_.active).map(_.average).pipe(numericStats).foreach { 
    case (k, v) => println(f"  $k: $v")
  }
  
  // GroupBy and aggregate
  val byGrade = students.groupBy(_.grade)
  println("\n=== By Grade ===")
  byGrade.toList.sortBy(_._1).foreach { case (grade, studs) =>
    val avg = studs.map(_.average).sum / studs.size
    println(f"  Grade $grade: ${studs.size} students, avg=$avg%.1f")
  }
}
```

---

## Step 139: Trampolining — ป้องกัน Stack Overflow

```scala
// Trampoline for stack-safe recursion
sealed trait Trampoline[+A] {
  def result: A = this match {
    case Done(a) => a
    case More(thunk) => thunk().result  // Tail-recursive via trampoline
  }
  
  def map[B](f: A => B): Trampoline[B] = flatMap(a => Done(f(a)))
  
  def flatMap[B](f: A => Trampoline[B]): Trampoline[B] = this match {
    case Done(a) => f(a)
    case More(thunk) => More(() => thunk().flatMap(f))
  }
}

case class Done[A](value: A) extends Trampoline[A]
case class More[A](thunk: () => Trampoline[A]) extends Trampoline[A]

object Trampoline {
  def run[A](t: Trampoline[A]): A = {
    var current: Trampoline[A] = t
    while (true) {
      current match {
        case Done(a) => return a
        case More(thunk) => current = thunk()
      }
    }
    throw new RuntimeException("Impossible")
  }
  
  // Factorial using trampoline
  def factorial(n: BigInt, acc: BigInt = 1): Trampoline[BigInt] = {
    if (n <= 0) Done(acc)
    else More(() => factorial(n - 1, n * acc))
  }
  
  // Fibonacci using trampoline
  def fibonacci(n: Int): Trampoline[BigInt] = {
    def fib(n: Int, a: BigInt, b: BigInt): Trampoline[BigInt] = {
      if (n == 0) Done(a)
      else More(() => fib(n - 1, b, a + b))
    }
    fib(n, 0, 1)
  }
}

object TrampolineDemo extends App {
  println("=== Trampoline (Stack-Safe Recursion) ===")
  
  // Large factorial - would cause StackOverflow without trampoline
  val fact20 = Trampoline.run(Trampoline.factorial(20))
  println(s"20! = $fact20")
  
  val fact100 = Trampoline.run(Trampoline.factorial(100))
  println(s"100! = $fact100")
  
  // Fibonacci
  (0 to 10).foreach { n =>
    val fib = Trampoline.run(Trampoline.fibonacci(n))
    print(s"$fib ")
  }
  println()
  
  val fib1000 = Trampoline.run(Trampoline.fibonacci(1000))
  println(s"fib(1000) = ${fib1000.toString.take(20)}...")
}
```

---

## Step 140: Real-World Pipeline — Data Processing

```scala
// Real-world data processing pipeline using HOF
import scala.util.{Try, Success, Failure}

object DataProcessingPipeline extends App {
  
  // Domain model
  case class RawRecord(
    id: String,
    timestamp: String,
    data: Map[String, String]
  )
  
  case class ValidatedRecord(
    id: String,
    timestamp: java.time.Instant,
    data: Map[String, Any]
  )
  
  case class EnrichedRecord(
    id: String,
    timestamp: java.time.Instant,
    data: Map[String, Any],
    metadata: Map[String, Any]
  )
  
  // Pipeline stages as functions
  type Stage[A, B] = A => Either[String, B]
  
  def parseTimestamp(raw: RawRecord): Either[String, RawRecord] = {
    Try(java.time.Instant.parse(raw.timestamp)) match {
      case Success(_) => Right(raw)
      case Failure(_) => Left(s"Invalid timestamp: ${raw.timestamp} in record ${raw.id}")
    }
  }
  
  def validateRequiredFields(required: List[String])(raw: RawRecord): Either[String, RawRecord] = {
    val missing = required.filterNot(raw.data.contains)
    if (missing.isEmpty) Right(raw)
    else Left(s"Missing required fields: ${missing.mkString(", ")} in ${raw.id}")
  }
  
  def convertTypes(raw: RawRecord): Either[String, ValidatedRecord] = {
    val convertedData: Map[String, Any] = raw.data.map { case (key, value) =>
      key -> (Try(value.toLong).toOption
               .orElse(Try(value.toDouble).toOption)
               .orElse(Some(value)))
               .get
    }
    Right(ValidatedRecord(
      id = raw.id,
      timestamp = java.time.Instant.parse(raw.timestamp),
      data = convertedData
    ))
  }
  
  def enrichWithMetadata(validated: ValidatedRecord): Either[String, EnrichedRecord] = {
    Right(EnrichedRecord(
      id = validated.id,
      timestamp = validated.timestamp,
      data = validated.data,
      metadata = Map(
        "processedAt" -> java.time.Instant.now().toString,
        "fieldCount" -> validated.data.size,
        "version" -> "1.0"
      )
    ))
  }
  
  // Compose pipeline stages
  def pipeline(raw: RawRecord): Either[String, EnrichedRecord] = {
    for {
      timestampValid <- parseTimestamp(raw)
      fieldsValid    <- validateRequiredFields(List("userId", "action"))(timestampValid)
      validated      <- convertTypes(fieldsValid)
      enriched       <- enrichWithMetadata(validated)
    } yield enriched
  }
  
  // Batch processing
  def processBatch(
    records: List[RawRecord],
    process: RawRecord => Either[String, EnrichedRecord]
  ): (List[EnrichedRecord], List[(RawRecord, String)]) = {
    val results = records.map(r => r -> process(r))
    val successes = results.collect { case (_, Right(e)) => e }
    val failures = results.collect { case (r, Left(err)) => r -> err }
    (successes, failures)
  }
  
  // Test data
  val records = List(
    RawRecord("R001", "2024-01-15T10:30:00Z", Map("userId" -> "U001", "action" -> "login", "ip" -> "192.168.1.1")),
    RawRecord("R002", "invalid-timestamp", Map("userId" -> "U002", "action" -> "purchase")),
    RawRecord("R003", "2024-01-15T11:00:00Z", Map("action" -> "logout")), // missing userId
    RawRecord("R004", "2024-01-15T11:30:00Z", Map("userId" -> "U003", "action" -> "purchase", "amount" -> "1500")),
    RawRecord("R005", "2024-01-15T12:00:00Z", Map("userId" -> "U004", "action" -> "view", "pageId" -> "42"))
  )
  
  println("=== Data Processing Pipeline ===")
  val (successes, failures) = processBatch(records, pipeline)
  
  println(s"\nSuccessfully processed: ${successes.size}")
  successes.foreach { e =>
    println(s"  ${e.id}: ${e.data}")
  }
  
  println(s"\nFailed: ${failures.size}")
  failures.foreach { case (r, err) =>
    println(s"  ${r.id}: $err")
  }
  
  // Statistical summary using HOF
  println("\n=== Processing Summary ===")
  val total = records.size
  val successRate = successes.size.toDouble / total * 100
  println(f"Total: $total, Success: ${successes.size} ($successRate%.1f%%), Failed: ${failures.size}")
}
```

---

## สรุป Part 14

| แนวคิด | Syntax | ตัวอย่าง |
|--------|--------|---------|
| Function type | `A => B` | `val f: Int => String` |
| HOF parameter | `def foo(f: A => B)` | รับ function เป็น param |
| Return function | `def foo(): A => B` | return function |
| andThen | `f andThen g` | f แล้วค่อย g |
| compose | `f compose g` | g แล้วค่อย f |
| Currying | `def f(x: Int)(y: Int)` | Multi-param list |
| Partial apply | `f(x, _: Int)` | Fix some args |
| Memoize | cache wrapper | เก็บ result ไว้ |
| Trampoline | `Done/More` | Stack-safe recursion |

---

## แบบฝึกหัด Part 14

**ข้อ 1:** สร้าง `Pipeline[A, B]` class ที่ chain operations ได้: `Pipeline(input).map(f).filter(p).fold(initial)(combine)` โดยใช้ HOF ทั้งหมด

**ข้อ 2:** Implement `retry` HOF ที่ retry function ที่ return `Try[T]` ตาม max attempts และมี exponential backoff delay

**ข้อ 3:** สร้าง function composition library เล็กๆ ที่มี: `andThen`, `compose`, `pipe`, `tap` (execute side effect without changing value)

**ข้อ 4:** Implement memoization ด้วย LRU (Least Recently Used) cache ที่มีขนาด limit

**ข้อ 5:** สร้าง data validation pipeline ที่ compose validators หลายตัว และ collect errors ทั้งหมดแทนที่จะหยุดที่ error แรก (Applicative style)

---

➡️ ต่อไป: [Part 15 — Lambdas and Closures](part-15-lambdas-and-closures.md)
