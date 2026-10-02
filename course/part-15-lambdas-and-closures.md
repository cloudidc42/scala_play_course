# Part 15: Lambdas และ Closures

## Steps 141-150: Lambda Expressions, Closures, Partial Functions, Function Literals, Eta Expansion

---

## Step 141: Lambda Expressions พื้นฐาน

Lambda คือ anonymous function ที่เขียนแบบกระชับโดยไม่ต้องตั้งชื่อ

```scala
object LambdaBasics extends App {
  
  // Full syntax vs shorthand
  val fullLambda: Int => Int = (x: Int) => x * 2
  val inferredLambda: Int => Int = x => x * 2
  val shortLambda: Int => Int = _ * 2  // _ เป็น placeholder สำหรับ single parameter
  
  println("=== Lambda Syntax ===")
  println(s"Full: ${fullLambda(5)}")
  println(s"Inferred: ${inferredLambda(5)}")
  println(s"Short: ${shortLambda(5)}")
  
  // Multi-parameter lambdas
  val add: (Int, Int) => Int = (x, y) => x + y
  val addShort: (Int, Int) => Int = _ + _
  
  val concat: (String, String) => String = (a, b) => a + " " + b
  
  println(s"\nadd(3, 4) = ${add(3, 4)}")
  println(s"concat: ${concat("Hello", "World")}")
  
  // Lambda as argument (inline)
  val numbers = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
  
  val evens = numbers.filter(_ % 2 == 0)
  val doubled = numbers.map(_ * 2)
  val sumOdd = numbers.filter(_ % 2 != 0).reduce(_ + _)
  
  println(s"\nEvens: $evens")
  println(s"Doubled: $doubled")
  println(s"Sum of odds: $sumOdd")
  
  // Multi-line lambda
  val complexTransform = numbers.map { n =>
    val doubled = n * 2
    val squared = doubled * doubled
    if (squared > 100) squared else doubled
  }
  println(s"Complex: $complexTransform")
  
  // Lambda with pattern matching
  val pairs = List((1, "one"), (2, "two"), (3, "three"))
  val formatted = pairs.map { case (num, name) => s"$num = $name" }
  println(s"\nPairs: $formatted")
  
  // Returning lambdas
  def adder(x: Int): Int => Int = y => x + y
  def multiplier(factor: Int): Int => Int = _ * factor
  
  val add5 = adder(5)
  val times3 = multiplier(3)
  
  println(s"\nadd5(10) = ${add5(10)}")
  println(s"times3(7) = ${times3(7)}")
}
```

---

## Step 142: Closures — Lambda ที่จับตัวแปรจาก Outer Scope

```scala
// Closures capture variables from their enclosing scope
object ClosureDemo extends App {
  
  // Simple closure
  val x = 10
  val addX: Int => Int = n => n + x  // Captures 'x' from outer scope
  
  println("=== Closures ===")
  println(s"addX(5) = ${addX(5)}")  // 15
  
  // Closure capturing mutable variable
  var counter = 0
  val increment: () => Int = () => {
    counter += 1
    counter
  }
  
  println(s"\ncounter: ${increment()}")  // 1
  println(s"counter: ${increment()}")    // 2
  println(s"counter: ${increment()}")    // 3
  println(s"counter value: $counter")    // 3 - shared state!
  
  // Closure in loop - common gotcha
  println("\n=== Loop Closures ===")
  
  // This captures 'i' by reference (mutable var in Scala is captured by value for local vars)
  val functions = for (i <- 1 to 5) yield {
    () => i * i  // Each closure captures its own 'i' value
  }
  
  functions.foreach(f => print(s"${f()} "))
  println()
  
  // Factory using closures
  def makeCounter(start: Int = 0, step: Int = 1): (() => Int, () => Unit) = {
    var current = start
    val next: () => Int = () => {
      val value = current
      current += step
      value
    }
    val reset: () => Unit = () => { current = start }
    (next, reset)
  }
  
  println("\n=== Counter Factory ===")
  val (next, reset) = makeCounter(10, 5)
  println(next())  // 10
  println(next())  // 15
  println(next())  // 20
  reset()
  println(next())  // 10 (reset!)
  
  // Closure for configuration
  def configureLogger(prefix: String, includeTimestamp: Boolean): String => Unit = {
    val timestampFn: () => String = 
      if (includeTimestamp) () => s"[${java.time.LocalTime.now()}] "
      else () => ""
    
    message => println(s"$prefix ${timestampFn()}$message")
  }
  
  println("\n=== Logger Closures ===")
  val appLogger = configureLogger("[APP]", includeTimestamp = true)
  val testLogger = configureLogger("[TEST]", includeTimestamp = false)
  
  appLogger("Application started")
  testLogger("Running test suite")
  appLogger("Processing request")
}
```

---

## Step 143: Partial Functions — Functions ที่ใช้ได้กับ Input บางส่วน

```scala
// PartialFunction[A, B] - defined for only some A values
object PartialFunctionDemo extends App {
  
  // Define a PartialFunction using case
  val divideByTwo: PartialFunction[Int, Double] = {
    case n if n != 0 => 100.0 / n
  }
  
  println("=== Partial Functions ===")
  println(s"100/5 = ${divideByTwo(5)}")
  println(s"isDefined(0) = ${divideByTwo.isDefinedAt(0)}")
  println(s"isDefined(5) = ${divideByTwo.isDefinedAt(5)}")
  
  // Safe application
  List(10, 0, 5, -2, 0, 4).foreach { n =>
    if (divideByTwo.isDefinedAt(n)) println(s"100/$n = ${divideByTwo(n)}")
    else println(s"Cannot divide by $n")
  }
  
  // collect - apply partial function to matching elements
  val numbers = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
  
  val squareOdd: PartialFunction[Int, String] = {
    case n if n % 2 != 0 => s"$n² = ${n * n}"
  }
  
  println(s"\nOdd squares: ${numbers.collect(squareOdd)}")
  
  // Combining partial functions with orElse
  val handlePositive: PartialFunction[Int, String] = {
    case n if n > 0 => s"Positive: $n"
  }
  
  val handleNegative: PartialFunction[Int, String] = {
    case n if n < 0 => s"Negative: $n"
  }
  
  val handleZero: PartialFunction[Int, String] = {
    case 0 => "Zero!"
  }
  
  // Chain with orElse
  val describe = handlePositive orElse handleNegative orElse handleZero
  
  println("\n=== orElse Chaining ===")
  List(-5, -1, 0, 1, 5).foreach(n => println(describe(n)))
  
  // Partial function with lift - convert to Option
  val safeDivide = divideByTwo.lift  // Int => Option[Double]
  
  println("\n=== Lifted Partial Function ===")
  List(10, 0, 5).foreach { n =>
    println(s"100/$n = ${safeDivide(n)}")
  }
  
  // Real-world: Event handler
  sealed trait AppEvent
  case class UserLogin(userId: String) extends AppEvent
  case class UserLogout(userId: String) extends AppEvent
  case class OrderPlaced(orderId: String, amount: Double) extends AppEvent
  case class SystemAlert(message: String) extends AppEvent
  
  type EventHandler = PartialFunction[AppEvent, Unit]
  
  val loginHandler: EventHandler = {
    case UserLogin(id) => println(s"  [Login] User $id logged in")
    case UserLogout(id) => println(s"  [Login] User $id logged out")
  }
  
  val orderHandler: EventHandler = {
    case OrderPlaced(id, amount) => 
      println(f"  [Order] Order $id placed: $amount%.2f THB")
  }
  
  val alertHandler: EventHandler = {
    case SystemAlert(msg) => println(s"  [ALERT] $msg")
  }
  
  // Combine handlers
  val allHandlers = loginHandler orElse orderHandler orElse alertHandler
  
  val events: List[AppEvent] = List(
    UserLogin("U001"),
    OrderPlaced("O001", 1500.0),
    UserLogin("U002"),
    SystemAlert("High CPU usage!"),
    OrderPlaced("O002", 2800.0),
    UserLogout("U001")
  )
  
  println("\n=== Event Handling ===")
  events.foreach { event =>
    if (allHandlers.isDefinedAt(event)) allHandlers(event)
    else println(s"  [Unhandled] $event")
  }
}
```

---

## Step 144: Function Literals

```scala
// Function literals - different ways to write functions
object FunctionLiterals extends App {
  
  // Method vs Function
  def methodDouble(n: Int): Int = n * 2          // Method
  val functionDouble: Int => Int = n => n * 2    // Function value
  
  // Methods can be lifted to functions using eta expansion
  val liftedDouble = methodDouble _              // Explicit eta expansion
  val liftedDouble2: Int => Int = methodDouble   // Implicit eta expansion
  
  println("=== Method vs Function ===")
  println(s"method: ${methodDouble(5)}")
  println(s"function: ${functionDouble(5)}")
  println(s"lifted: ${liftedDouble(5)}")
  println(s"lifted2: ${liftedDouble2(5)}")
  
  // Function2, Function3, etc.
  val add: (Int, Int) => Int = (x, y) => x + y
  val add3: (Int, Int, Int) => Int = (x, y, z) => x + y + z
  
  // Explicit Function types
  val multiply = new Function2[Int, Int, Int] {
    def apply(x: Int, y: Int): Int = x * y
  }
  
  println(s"\nadd(3, 4) = ${add(3, 4)}")
  println(s"add3(1, 2, 3) = ${add3(1, 2, 3)}")
  println(s"multiply(5, 6) = ${multiply(5, 6)}")
  
  // Passing methods as function literals
  val numbers = List(1, 2, 3, 4, 5)
  
  def isEven(n: Int): Boolean = n % 2 == 0
  def format(n: Int): String = s"[${n}]"
  
  println(s"\nEvens: ${numbers.filter(isEven)}")
  println(s"Formatted: ${numbers.map(format)}")
  
  // Method references to class methods
  class Calculator {
    def add(x: Int, y: Int): Int = x + y
    def square(n: Int): Int = n * n
  }
  
  val calc = new Calculator()
  val calcAdd = calc.add _           // Bound method reference
  val calcSquare = calc.square _
  
  println(s"\ncalcAdd(3, 4) = ${calcAdd(3, 4)}")
  println(s"calcSquare(5) = ${calcSquare(5)}")
  
  // Anonymous class as function (old style)
  val upperCase = new (String => String) {
    def apply(s: String): String = s.toUpperCase
  }
  
  println(s"\nupperCase('hello') = ${upperCase("hello")}")
}
```

---

## Step 145: Eta Expansion — เปลี่ยน Method เป็น Function Value

```scala
// Eta expansion - converting methods to function values
object EtaExpansionDemo extends App {
  
  // Regular method
  def double(n: Int): Int = n * 2
  def add(x: Int, y: Int): Int = x + y
  def formatName(first: String, last: String): String = s"$last, $first"
  
  // Explicit eta expansion
  val doubleFn = double _          // Int => Int
  val addFn = add _                // (Int, Int) => Int
  val formatFn = formatName _      // (String, String) => String
  
  println("=== Eta Expansion ===")
  println(s"doubleFn(5) = ${doubleFn(5)}")
  println(s"addFn(3, 4) = ${addFn(3, 4)}")
  println(s"formatFn = ${formatFn("Alice", "Chen")}")
  
  // Implicit eta expansion (Scala 3 style, also works in Scala 2.13+)
  val numbers = List(1, 2, 3, 4, 5)
  
  println(s"\nImplicit eta expansion:")
  println(numbers.map(double))      // Method passed directly
  println(numbers.filter(n => n > 2))
  
  // Eta expansion for generic methods
  def identity[A](x: A): A = x
  
  // Can't directly eta expand generic methods without type specification
  val identityInt: Int => Int = identity[Int] _
  val identityStr: String => String = identity[String] _
  
  println(s"\nidentityInt(42) = ${identityInt(42)}")
  println(s"identityStr('hello') = ${identityStr("hello")}")
  
  // Practical use: method reference for callbacks
  class EventProcessor {
    def processLogin(userId: String): Unit = 
      println(s"Processing login for $userId")
    
    def processPayment(amount: Double): Unit = 
      println(f"Processing payment: $amount%.2f THB")
  }
  
  val processor = new EventProcessor()
  
  // Register callbacks using method references
  val loginCallback = processor.processLogin _
  val paymentCallback = processor.processPayment _
  
  println("\n=== Method as Callback ===")
  List("U001", "U002", "U003").foreach(loginCallback)
  List(500.0, 1200.0, 3000.0).foreach(paymentCallback)
  
  // Curried methods and eta expansion
  def logWithLevel(level: String)(message: String): Unit = 
    println(s"[$level] $message")
  
  val infoLog = logWithLevel("INFO") _    // String => Unit
  val errorLog = logWithLevel("ERROR") _
  
  println("\n=== Curried Method Eta Expansion ===")
  List("Started", "Running", "Completed").foreach(infoLog)
  List("File not found", "Connection refused").foreach(errorLog)
}
```

---

## Step 146: Function Identity และ Constant Functions

```scala
// Useful built-in function utilities
object FunctionUtilities extends App {
  
  // identity function
  val id = identity[Int] _
  println(s"identity(42) = ${id(42)}")
  
  // const function - always returns same value
  def const[A, B](a: A)(b: B): A = a
  val alwaysZero = const(0) _
  val alwaysTrue = const(true) _
  
  println("\nalwaysZero:")
  List("a", "b", "c").map(alwaysZero).foreach(print)
  println()
  
  // flip - swap argument order
  def flip[A, B, C](f: (A, B) => C): (B, A) => C = (b, a) => f(a, b)
  
  val divide = (a: Double, b: Double) => a / b
  val divisor = flip(divide)  // now: (b, a) => a / b
  
  println(s"\ndivide(10, 2) = ${divide(10, 2)}")
  println(s"divisor(2, 10) = ${divisor(2, 10)}")  // 10/2 = 5
  
  // on - apply function to both elements before combining
  def on[A, B, C](combine: (B, B) => C)(transform: A => B): (A, A) => C =
    (a1, a2) => combine(transform(a1), transform(a2))
  
  val compareByLength: (String, String) => Int = 
    on((a: Int, b: Int) => a - b)(_.length)
  
  val words = List("banana", "apple", "cherry", "date", "elderberry")
  println(s"\nSorted by length: ${words.sortWith(compareByLength(_, _) < 0)}")
  
  // tupled - converts (A, B) => C to ((A, B)) => C
  val addPair: ((Int, Int)) => Int = add.tupled
  def add(x: Int, y: Int): Int = x + y
  
  val pairs = List((1, 2), (3, 4), (5, 6))
  println(s"\nSum of pairs: ${pairs.map(addPair)}")
  
  // untupled - converts ((A, B)) => C to (A, B) => C
  val addUntupled: (Int, Int) => Int = Function.untupled(addPair)
  println(s"addUntupled(3, 4) = ${addUntupled(3, 4)}")
}
```

---

## Step 147: Recursive Lambdas

```scala
// Self-referential/recursive lambdas
object RecursiveLambdas extends App {
  
  // Using Y-combinator concept (simplified)
  // Regular recursive function
  def factorial(n: BigInt): BigInt = 
    if (n <= 0) 1 else n * factorial(n - 1)
  
  // Self-referential lambda using var (mutable reference)
  var factLambda: BigInt => BigInt = null
  factLambda = (n: BigInt) => if (n <= 0) 1 else n * factLambda(n - 1)
  
  println("=== Recursive Lambda ===")
  println(s"factorial(10) = ${factorial(10)}")
  println(s"factLambda(10) = ${factLambda(10)}")
  
  // Recursive lambda for tree traversal
  sealed trait Tree[+A]
  case class Leaf[A](value: A) extends Tree[A]
  case class Branch[A](left: Tree[A], right: Tree[A]) extends Tree[A]
  
  val sumTree: Tree[Int] => Int = {
    var fn: Tree[Int] => Int = null
    fn = {
      case Leaf(v)        => v
      case Branch(l, r)   => fn(l) + fn(r)
    }
    fn
  }
  
  val tree = Branch(
    Branch(Leaf(1), Leaf(2)),
    Branch(
      Leaf(3),
      Branch(Leaf(4), Leaf(5))
    )
  )
  
  println(s"\nTree sum: ${sumTree(tree)}")  // 15
  
  // Mutual recursion using def
  def isEven(n: Int): Boolean = if (n == 0) true else isOdd(n - 1)
  def isOdd(n: Int): Boolean = if (n == 0) false else isEven(n - 1)
  
  println("\n=== Mutual Recursion ===")
  (0 to 5).foreach(n => println(s"$n: even=${isEven(n)}, odd=${isOdd(n)}"))
}
```

---

## Step 148: Functions ใน Real-World Patterns

```scala
// Real-world functional patterns
object RealWorldFunctions extends App {
  
  // Strategy pattern using functions
  case class ShoppingCart(
    items: List[(String, Int, Double)],  // (name, qty, price)
    userId: String,
    couponCode: Option[String]
  ) {
    def subtotal: Double = items.map { case (_, qty, price) => qty * price }.sum
  }
  
  type DiscountStrategy = ShoppingCart => Double
  
  val noDiscount: DiscountStrategy = _ => 0.0
  
  val tenPercentOff: DiscountStrategy = cart => cart.subtotal * 0.10
  
  val bulkDiscount: DiscountStrategy = cart => {
    val total = cart.subtotal
    if (total > 5000) total * 0.15
    else if (total > 2000) total * 0.10
    else if (total > 1000) total * 0.05
    else 0.0
  }
  
  def couponDiscount(couponDb: Map[String, Double]): DiscountStrategy = cart => {
    cart.couponCode.flatMap(couponDb.get).getOrElse(0.0)
  }
  
  def maxDiscount(strategies: List[DiscountStrategy]): DiscountStrategy = cart =>
    strategies.map(_(cart)).max
  
  def totalDiscount(strategies: List[DiscountStrategy]): DiscountStrategy = cart =>
    strategies.map(_(cart)).sum
  
  // Checkout process
  def checkout(
    cart: ShoppingCart,
    discountStrategy: DiscountStrategy,
    taxRate: Double
  ): Map[String, Double] = {
    val subtotal = cart.subtotal
    val discount = discountStrategy(cart)
    val afterDiscount = subtotal - discount
    val tax = afterDiscount * taxRate
    val total = afterDiscount + tax
    
    Map(
      "subtotal" -> subtotal,
      "discount" -> discount,
      "afterDiscount" -> afterDiscount,
      "tax" -> tax,
      "total" -> total
    )
  }
  
  val cart = ShoppingCart(
    items = List(
      ("Laptop", 1, 35000.0),
      ("Mouse", 2, 850.0),
      ("Keyboard", 1, 1200.0)
    ),
    userId = "U001",
    couponCode = Some("SAVE500")
  )
  
  val coupons = Map("SAVE500" -> 500.0, "SAVE1000" -> 1000.0)
  
  println("=== Strategy Pattern with Functions ===")
  
  // Apply different strategies
  val strategies = Map(
    "No Discount" -> noDiscount,
    "10% Off" -> tenPercentOff,
    "Bulk Discount" -> bulkDiscount,
    "Coupon" -> couponDiscount(coupons),
    "Best Deal" -> maxDiscount(List(tenPercentOff, bulkDiscount, couponDiscount(coupons)))
  )
  
  strategies.foreach { case (name, strategy) =>
    val result = checkout(cart, strategy, 0.07)
    println(f"\n--- $name ---")
    result.foreach { case (k, v) => println(f"  $k: $v%.2f") }
  }
}
```

---

## Step 149: Lazy Evaluation ใน Functions

```scala
// Lazy evaluation and by-name parameters
object LazyFunctions extends App {
  
  // By-name parameter: evaluated each time it's used
  def ifTrue(condition: Boolean, action: => Unit): Unit = {
    if (condition) action  // action is only evaluated if condition is true
  }
  
  def tryExecute(operation: => String): Either[String, String] = {
    try Right(operation)
    catch { case e: Exception => Left(e.getMessage) }
  }
  
  println("=== By-Name Parameters ===")
  
  ifTrue(true, println("Condition was true!"))
  ifTrue(false, println("This won't print"))  // Not evaluated
  
  // By-name vs eager evaluation
  var sideEffectCount = 0
  
  def withSideEffect: Int = {
    sideEffectCount += 1
    42
  }
  
  // By-name - evaluated each access
  def useTwiceByName(x: => Int): Int = x + x
  // Eager - evaluated once
  def useTwiceEager(x: Int): Int = x + x
  
  sideEffectCount = 0
  println(s"\nBy-name result: ${useTwiceByName(withSideEffect)}")
  println(s"Side effects: $sideEffectCount")  // 2!
  
  sideEffectCount = 0
  println(s"Eager result: ${useTwiceEager(withSideEffect)}")
  println(s"Side effects: $sideEffectCount")  // 1
  
  // Lazy val - computed once, on first access
  println("\n=== Lazy Val ===")
  lazy val expensiveValue: Int = {
    println("Computing expensive value...")
    Thread.sleep(10)
    42
  }
  
  println("Before access")
  println(s"Value: $expensiveValue")  // Computed here
  println(s"Value again: $expensiveValue")  // Cached
  
  // Lazy list (Stream / LazyList)
  val naturals: LazyList[Int] = LazyList.from(1)
  val first10 = naturals.take(10).toList
  val first5Squares = naturals.map(n => n * n).take(5).toList
  
  println(s"\nFirst 10 naturals: $first10")
  println(s"First 5 squares: $first5Squares")
  
  // Infinite sequence of Fibonacci
  val fibs: LazyList[BigInt] = {
    def go(a: BigInt, b: BigInt): LazyList[BigInt] = a #:: go(b, a + b)
    go(0, 1)
  }
  
  println(s"First 15 Fibonacci: ${fibs.take(15).toList}")
  
  // Practical: lazy error messages
  def validate(n: Int, errorMsg: => String): Either[String, Int] = {
    if (n > 0) Right(n)
    else Left(errorMsg)  // Error message only computed if needed
  }
  
  println("\n=== Lazy Error Messages ===")
  validate(5, s"Failed at ${System.currentTimeMillis()}") match {
    case Right(n)  => println(s"Valid: $n")
    case Left(err) => println(s"Error: $err")
  }
  
  validate(-1, s"Failed at ${System.currentTimeMillis()}") match {
    case Right(n)  => println(s"Valid: $n")
    case Left(err) => println(s"Error: $err")
  }
}
```

---

## Step 150: Function Composition Best Practices

```scala
// Complete example: Order processing with functional patterns
object OrderProcessingFunctional extends App {
  
  import scala.util.{Try, Success, Failure}
  
  // Domain
  case class OrderItem(productId: String, name: String, price: Double, quantity: Int) {
    def total: Double = price * quantity
  }
  
  case class Order(
    id: String,
    customerId: String,
    items: List[OrderItem],
    coupon: Option[String] = None
  ) {
    def subtotal: Double = items.map(_.total).sum
  }
  
  case class ProcessedOrder(
    order: Order,
    discount: Double,
    tax: Double,
    total: Double,
    timestamp: java.time.Instant
  )
  
  // Function types
  type Validator[T] = T => Either[List[String], T]
  type Processor[A, B] = A => Either[String, B]
  
  // Validators (collect all errors)
  def validateNotEmpty[T](field: String)(getValue: T => List[_]): Validator[T] = item => {
    if (getValue(item).nonEmpty) Right(item)
    else Left(List(s"$field cannot be empty"))
  }
  
  def validateOrder: Validator[Order] = order => {
    val errors = scala.collection.mutable.ListBuffer[String]()
    if (order.items.isEmpty) errors += "Order must have at least one item"
    if (order.customerId.isEmpty) errors += "Customer ID is required"
    order.items.foreach { item =>
      if (item.quantity <= 0) errors += s"Quantity for ${item.name} must be positive"
      if (item.price <= 0) errors += s"Price for ${item.name} must be positive"
    }
    if (errors.isEmpty) Right(order)
    else Left(errors.toList)
  }
  
  // Processors
  def applyDiscount(coupons: Map[String, Double]): Processor[Order, (Order, Double)] = order => {
    val discount = order.coupon
      .flatMap(coupons.get)
      .getOrElse {
        val subtotal = order.subtotal
        if (subtotal > 10000) subtotal * 0.15
        else if (subtotal > 5000) subtotal * 0.10
        else 0.0
      }
    Right((order, discount))
  }
  
  def calculateTax(rate: Double): Processor[(Order, Double), ProcessedOrder] = { case (order, discount) =>
    val afterDiscount = order.subtotal - discount
    val tax = afterDiscount * rate
    Right(ProcessedOrder(
      order = order,
      discount = discount,
      tax = tax,
      total = afterDiscount + tax,
      timestamp = java.time.Instant.now()
    ))
  }
  
  // Processing pipeline
  val coupons = Map("SAVE10" -> 100.0, "VIP20" -> 200.0, "FLASH30" -> 300.0)
  
  def processOrder(order: Order): Either[List[String], ProcessedOrder] = {
    validateOrder(order) match {
      case Left(errors) => Left(errors)
      case Right(validOrder) =>
        (applyDiscount(coupons) andThen {
          case Right((o, d)) => calculateTax(0.07)((o, d))
          case Left(err) => Left(err)
        })(validOrder).left.map(List(_))
    }
  }
  
  // Test orders
  val orders = List(
    Order("O001", "C001", List(
      OrderItem("P001", "Laptop", 35000.0, 1),
      OrderItem("P002", "Mouse", 850.0, 2)
    ), Some("SAVE10")),
    
    Order("O002", "C002", List(
      OrderItem("P003", "Phone", 15000.0, 1)
    )),
    
    Order("O003", "", List()), // Invalid
    
    Order("O004", "C004", List(
      OrderItem("P004", "Monitor", 8000.0, 1),
      OrderItem("P005", "Keyboard", 1500.0, 1),
      OrderItem("P006", "Webcam", 2000.0, -1) // Invalid quantity
    ))
  )
  
  println("=== Order Processing Pipeline ===")
  
  orders.foreach { order =>
    processOrder(order) match {
      case Right(processed) =>
        println(s"\nOrder ${order.id} - SUCCESS")
        println(f"  Subtotal: ${order.subtotal}%.2f")
        println(f"  Discount: ${processed.discount}%.2f")
        println(f"  Tax:      ${processed.tax}%.2f")
        println(f"  Total:    ${processed.total}%.2f")
        
      case Left(errors) =>
        println(s"\nOrder ${order.id} - FAILED")
        errors.foreach(e => println(s"  - $e"))
    }
  }
}
```

---

## สรุป Part 15

| แนวคิด | Syntax | การใช้งาน |
|--------|--------|----------|
| Lambda | `x => x * 2` | Anonymous function |
| Shorthand | `_ * 2` | Single param placeholder |
| Closure | จับตัวแปรจาก outer scope | เก็บ state |
| PartialFunction | `{ case ... }` | Defined for some inputs |
| `orElse` | `pf1 orElse pf2` | Chain partial functions |
| `lift` | `pf.lift` | PartialFunction → Option |
| `collect` | `list.collect(pf)` | Filter + transform |
| Eta expansion | `method _` | Method → Function |
| By-name param | `=> T` | Lazy evaluation |
| Lazy val | `lazy val` | Compute once, on demand |

---

## แบบฝึกหัด Part 15

**ข้อ 1:** สร้าง closure-based state machine สำหรับ traffic light: สถานะ Red → Green → Yellow → Red โดยใช้ mutable var ใน closure

**ข้อ 2:** เขียน `PartialFunction` chains สำหรับ JSON-like data processing: handle String, Number, Boolean, Array, Null แบบ exhaustive

**ข้อ 3:** Implement `memoize` ด้วย by-name parameter ที่ compute value lazily และเก็บ cache

**ข้อ 4:** สร้าง lazy infinite sequence ของ prime numbers โดยใช้ `LazyList` และ `filter`

**ข้อ 5:** Rewrite order processing pipeline ใน Step 150 ให้ collect errors ทั้งหมด (ไม่หยุดที่ error แรก) โดยใช้ `Validated` concept

---

➡️ ต่อไป: [Part 16 — Map, Filter, Reduce](part-16-map-filter-reduce.md)
