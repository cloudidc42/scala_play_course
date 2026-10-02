# Part 04 — Functions และ Methods
## Steps 31–40: เชี่ยวชาญการเขียน Functions

> **เป้าหมาย**: เขียน functions ทุกรูปแบบ, default parameters, named args, varargs, และ recursion

---

## Step 31 — Function Syntax พื้นฐาน

```scala
// Basic function definition
def greet(name: String): String = s"Hello, $name!"
println(greet("Alice"))  // Hello, Alice!

// Function with body block
def add(a: Int, b: Int): Int = {
  val sum = a + b
  sum  // last expression is return value
}
println(add(3, 4))  // 7

// ไม่ต้องมี return keyword
def multiply(a: Int, b: Int): Int =
  a * b

// Procedure (ไม่ส่งค่าคืน)
def printDivider(width: Int): Unit =
  println("-" * width)

// Single expression (Scala 3)
def square(x: Int) = x * x  // return type inferred
```

### Return Type Inference

```scala
// Scala เดา return type ได้
def double(x: Int) = x * 2          // Int
def hello(name: String) = s"Hi $name" // String
def always42() = 42                  // Int

// แต่ recursive function ต้องระบุ type
def factorial(n: Int): Int =  // ต้องระบุ!
  if (n <= 1) 1 else n * factorial(n - 1)

// แนะนำ: ระบุ return type เสมอสำหรับ public API
def getUserAge(userId: Int): Option[Int] = ???  // ??? = not implemented
```

---

## Step 32 — Multiple Parameter Lists

```scala
// Single parameter list
def add(a: Int, b: Int): Int = a + b

// Multiple parameter lists (currying)
def add2(a: Int)(b: Int): Int = a + b
val result = add2(3)(4)  // 7

// Partial application
val add10 = add2(10) _    // partial function
println(add10(5))         // 15
println(add10(20))        // 30

// Real-world use case: custom control structures
def repeat(times: Int)(block: => Unit): Unit =
  for (_ <- 1 to times) block

repeat(3) {
  println("Hello!")
}
// Hello!
// Hello!
// Hello!

// With type parameter
def twice[A](transform: A => A)(value: A): A =
  transform(transform(value))

val double = twice[Int](_ * 2) _
println(double(5))   // 20 (5 * 2 * 2)
```

---

## Step 33 — Default Parameters

```scala
// Default values
def connect(
  host: String = "localhost",
  port: Int = 8080,
  secure: Boolean = false
): String =
  val protocol = if (secure) "https" else "http"
  s"$protocol://$host:$port"

// เรียกแบบต่างๆ
println(connect())                           // http://localhost:8080
println(connect("example.com"))              // http://example.com:8080
println(connect("example.com", 443, true))   // https://example.com:443

// Named arguments (ข้ามลำดับได้)
println(connect(port = 443, secure = true))  // https://localhost:443
println(connect(secure = true, host = "api.example.com"))

// ดีมากสำหรับ config objects
case class Config(
  host: String     = "localhost",
  port: Int        = 5432,
  database: String = "mydb",
  maxConnections: Int = 10,
  timeout: Int     = 30000
)

val devConfig  = Config()
val prodConfig = Config(host = "db.prod.com", port = 5432, maxConnections = 100)
val testConfig = Config(database = "testdb")
```

---

## Step 34 — Named Arguments

```scala
// Named arguments ช่วยให้ code อ่านง่าย
def createUser(
  name: String,
  email: String,
  age: Int,
  role: String = "user",
  active: Boolean = true
): String =
  s"User($name, $email, $age, $role, active=$active)"

// ไม่ใช้ named args — ไม่ชัดเจน
createUser("Alice", "alice@example.com", 30, "admin", true)

// ใช้ named args — ชัดเจน
createUser(
  name   = "Alice",
  email  = "alice@example.com",
  age    = 30,
  role   = "admin",
  active = true
)

// ข้ามลำดับได้ถ้าใช้ named
createUser(
  email  = "bob@example.com",
  name   = "Bob",
  age    = 25
  // role = "user" (default)
  // active = true (default)
)
```

---

## Step 35 — Varargs (Variable Arguments)

```scala
// Varargs ด้วย *
def sum(nums: Int*): Int = nums.sum
def max(nums: Double*): Double = nums.max

println(sum(1, 2, 3))         // 6
println(sum(1, 2, 3, 4, 5))   // 15
println(max(3.0, 1.0, 4.0, 1.0, 5.0))  // 5.0

// Varargs เป็น Seq ข้างใน
def printAll(items: String*): Unit = {
  println(s"Count: ${items.length}")
  items.foreach(println)
}

printAll("apple", "banana", "cherry")

// Pass collection as varargs
val numbers = List(1, 2, 3, 4, 5)
println(sum(numbers*))    // Scala 3: spread operator
// println(sum(numbers: _*))  // Scala 2 style

// Combine fixed and varargs
def format(prefix: String, items: Any*): String =
  items.map(_.toString).mkString(s"$prefix[", ", ", "]")

println(format("nums: ", 1, 2, 3))      // nums: [1, 2, 3]
println(format("words: ", "a", "b"))    // words: [a, b]
```

---

## Step 36 — Higher-Order Functions (เบื้องต้น)

```scala
// Function as parameter
def applyTwice(f: Int => Int, x: Int): Int = f(f(x))

val double = (x: Int) => x * 2
val addOne = (x: Int) => x + 1

println(applyTwice(double, 3))   // 12 (3 * 2 * 2)
println(applyTwice(addOne, 5))   // 7  (5 + 1 + 1)

// Anonymous function (lambda)
println(applyTwice(x => x + 10, 5))  // 25

// Function as return value
def multiplier(factor: Int): Int => Int =
  (x: Int) => x * factor

val triple = multiplier(3)
val quadruple = multiplier(4)

println(triple(5))     // 15
println(quadruple(5))  // 20

// Functions stored in val
val operations: Map[String, (Int, Int) => Int] = Map(
  "add" -> (_ + _),
  "sub" -> (_ - _),
  "mul" -> (_ * _),
  "div" -> (_ / _)
)

println(operations("add")(10, 5))  // 15
println(operations("mul")(4, 7))   // 28
```

---

## Step 37 — Recursive Functions

```scala
// Basic recursion
def factorial(n: Int): Int =
  if (n <= 0) 1 else n * factorial(n - 1)

println(factorial(10))  // 3628800

// Tail recursion (ไม่ stackoverflow)
import scala.annotation.tailrec

def factorialTail(n: Int): Long = {
  @tailrec
  def loop(n: Int, acc: Long): Long =
    if (n <= 0) acc
    else loop(n - 1, n * acc)
  loop(n, 1L)
}

println(factorialTail(20))  // 2432902008176640000

// Fibonacci — naive (O(2^n))
def fib(n: Int): Int = n match {
  case 0 => 0
  case 1 => 1
  case n => fib(n - 1) + fib(n - 2)
}

// Fibonacci — tail recursive (O(n))
def fibTail(n: Int): Long = {
  @tailrec
  def loop(n: Int, a: Long, b: Long): Long =
    if (n == 0) a
    else loop(n - 1, b, a + b)
  loop(n, 0L, 1L)
}

println(fibTail(100))  // 354224848179261915075

// Power function
def pow(base: Double, exp: Int): Double = {
  @tailrec
  def loop(exp: Int, acc: Double): Double =
    if (exp == 0) acc
    else loop(exp - 1, acc * base)
  loop(exp, 1.0)
}

println(pow(2.0, 10))  // 1024.0

// Binary search (recursive)
def binarySearch(arr: Array[Int], target: Int): Int = {
  @tailrec
  def search(lo: Int, hi: Int): Int =
    if (lo > hi) -1
    else {
      val mid = (lo + hi) / 2
      if (arr(mid) == target) mid
      else if (arr(mid) < target) search(mid + 1, hi)
      else search(lo, mid - 1)
    }
  search(0, arr.length - 1)
}

val sorted = Array(1, 3, 5, 7, 9, 11, 13, 15)
println(binarySearch(sorted, 7))   // 3
println(binarySearch(sorted, 10))  // -1
```

---

## Step 38 — Function Types

```scala
// Function type syntax: (Input) => Output
val double: Int => Int = x => x * 2
val greet: String => String = name => s"Hello, $name!"
val add: (Int, Int) => Int = (a, b) => a + b
val noArgs: () => Int = () => 42

// Calling
println(double(5))      // 10
println(greet("Bob"))   // Hello, Bob!
println(add(3, 4))      // 7
println(noArgs())       // 42

// Function composition
val doubleAndAddOne = double.andThen(x => x + 1)
println(doubleAndAddOne(5))  // 11

val addOneThenDouble = double.compose(x => x + 1)
println(addOneThenDouble(5))  // 12

// Method vs Function
class Calculator {
  def multiply(a: Int, b: Int): Int = a * b
}

val calc = new Calculator()
val mulMethod = calc.multiply   // Method
val mulFunc: (Int, Int) => Int = calc.multiply  // Eta expansion → Function
println(mulFunc(3, 4))  // 12

// Eta expansion explicit
val mulFunc2 = calc.multiply _
```

---

## Step 39 — Partial Functions

```scala
import scala.{PartialFunction => PF}

// PartialFunction — defined only for some inputs
val divide: PF[Int, Int] = {
  case d if d != 0 => 100 / d
}

println(divide.isDefinedAt(0))   // false
println(divide.isDefinedAt(5))   // true
println(divide(5))               // 20

// orElse — chain partial functions
val handleZero: PF[Int, Int] = {
  case 0 => -1
}

val safeDivide = divide orElse handleZero
println(safeDivide(5))   // 20
println(safeDivide(0))   // -1

// collect — filter + map กับ PartialFunction
val numbers = List(1, 0, 2, 0, 3, 4, 0, 5)
val results = numbers.collect(divide)
println(results)  // List(100, 50, 33, 25, 20)

// Real use case: event handling
sealed trait Event
case class Click(x: Int, y: Int) extends Event
case class KeyPress(key: String) extends Event
case class Resize(w: Int, h: Int) extends Event

val handleClick: PF[Event, String] = {
  case Click(x, y) => s"Clicked at ($x, $y)"
}

val handleKey: PF[Event, String] = {
  case KeyPress(k) => s"Key pressed: $k"
}

val handleAll = handleClick orElse handleKey

val events: List[Event] = List(
  Click(100, 200),
  KeyPress("Enter"),
  Resize(1920, 1080),  // ไม่ handle
  Click(50, 75)
)

val handled = events.collect(handleAll)
println(handled)
// List(Clicked at (100, 200), Key pressed: Enter, Clicked at (50, 75))
```

---

## Step 40 — โปรแกรม Functions Complete

```scala
// FunctionsMasterDemo.scala

import scala.annotation.tailrec

@main def functionsDemo(): Unit =
  
  println("=== Functions Master Demo ===\n")
  
  // ==============================
  // 1. Function Pipeline
  // ==============================
  println("--- Function Pipeline ---")
  
  // Text processing pipeline
  type Transform = String => String
  
  def pipeline(transforms: Transform*): Transform =
    input => transforms.foldLeft(input)((s, f) => f(s))
  
  val processText = pipeline(
    _.trim,
    _.toLowerCase,
    _.replaceAll("[^a-z0-9 ]", ""),
    _.split(" ").distinct.sorted.mkString(" ")
  )
  
  val raw = "  Hello World! This is SCALA  this is AWESOME!  "
  println(s"Raw:       \"$raw\"")
  println(s"Processed: \"${processText(raw)}\"")
  
  // ==============================
  // 2. Memoization
  // ==============================
  println("\n--- Memoization ---")
  
  def memoize[A, B](f: A => B): A => B = {
    val cache = scala.collection.mutable.Map.empty[A, B]
    (a: A) => cache.getOrElseUpdate(a, f(a))
  }
  
  var callCount = 0
  val expensiveFib: Int => Long = memoize { n =>
    callCount += 1
    @tailrec
    def loop(n: Int, a: Long, b: Long): Long =
      if (n == 0) a else loop(n - 1, b, a + b)
    loop(n, 0L, 1L)
  }
  
  println(s"fib(40) = ${expensiveFib(40)}, calls: $callCount")
  println(s"fib(40) = ${expensiveFib(40)}, calls: $callCount")  // cached!
  println(s"fib(45) = ${expensiveFib(45)}, calls: $callCount")
  
  // ==============================
  // 3. Currying and Partial Application
  // ==============================
  println("\n--- Currying ---")
  
  def power(base: Double)(exp: Int): Double = {
    @tailrec
    def loop(e: Int, acc: Double): Double =
      if (e == 0) acc else loop(e - 1, acc * base)
    loop(exp, 1.0)
  }
  
  val square  = power(2.0)
  val cube    = power(3.0)
  val tenPow  = power(10.0)
  
  (0 to 5).foreach { n =>
    println(f"  2^$n = ${square(n)}%.0f, 3^$n = ${cube(n)}%.0f, 10^$n = ${tenPow(n)}%.0f")
  }
  
  // ==============================
  // 4. Function Composition
  // ==============================
  println("\n--- Function Composition ---")
  
  val parseNum: String => Option[Int] = s =>
    scala.util.Try(s.toInt).toOption
  
  val double: Int => Int = _ * 2
  val addHundred: Int => Int = _ + 100
  val toString: Int => String = n => s"Result: $n"
  
  def processInput(input: String): String =
    parseNum(input)
      .map(double)
      .map(addHundred)
      .map(toString)
      .getOrElse("Invalid input!")
  
  List("42", "abc", "10", "-5").foreach { input =>
    println(s"  processInput(\"$input\") = ${processInput(input)}")
  }
  
  // ==============================
  // 5. Recursive Data Structures
  // ==============================
  println("\n--- Binary Tree ---")
  
  sealed trait Tree[+A]
  case object Leaf extends Tree[Nothing]
  case class Node[A](value: A, left: Tree[A], right: Tree[A]) extends Tree[A]
  
  def insert(tree: Tree[Int], value: Int): Tree[Int] = tree match
    case Leaf => Node(value, Leaf, Leaf)
    case Node(v, l, r) =>
      if (value < v) Node(v, insert(l, value), r)
      else if (value > v) Node(v, l, insert(r, value))
      else tree
  
  def inorder(tree: Tree[Int]): List[Int] = tree match
    case Leaf => Nil
    case Node(v, l, r) => inorder(l) ++ List(v) ++ inorder(r)
  
  def height(tree: Tree[Int]): Int = tree match
    case Leaf => 0
    case Node(_, l, r) => 1 + math.max(height(l), height(r))
  
  val values = List(5, 3, 7, 1, 4, 6, 8, 2)
  val bst = values.foldLeft[Tree[Int]](Leaf)(insert)
  
  println(s"  Inserted: ${values.mkString(", ")}")
  println(s"  Inorder: ${inorder(bst).mkString(", ")}")
  println(s"  Height: ${height(bst)}")
  
  // ==============================
  // 6. Domain Modeling with Functions
  // ==============================
  println("\n--- Domain Modeling ---")
  
  type Validator[A] = A => Either[List[String], A]
  
  def minLength(min: Int): Validator[String] = s =>
    if (s.length >= min) Right(s)
    else Left(List(s"Must be at least $min characters"))
  
  def maxLength(max: Int): Validator[String] = s =>
    if (s.length <= max) Right(s)
    else Left(List(s"Must be at most $max characters"))
  
  def matches(pattern: String): Validator[String] = s =>
    if (s.matches(pattern)) Right(s)
    else Left(List(s"Must match pattern: $pattern"))
  
  def combine[A](v1: Validator[A], v2: Validator[A]): Validator[A] = a =>
    (v1(a), v2(a)) match
      case (Right(_), Right(_)) => Right(a)
      case (Left(e1), Left(e2)) => Left(e1 ++ e2)
      case (Left(e1), _)        => Left(e1)
      case (_, Left(e2))        => Left(e2)
  
  val emailValidator: Validator[String] = combine(
    combine(minLength(5), maxLength(100)),
    matches(".*@.*\\..*")
  )
  
  List("hi", "a" * 101, "notanemail", "user@example.com").foreach { email =>
    emailValidator(email) match
      case Right(e)     => println(s"  ✓ '$e' is valid")
      case Left(errors) => println(s"  ✗ '$email': ${errors.mkString(", ")}")
  }
  
  println("\n=== Functions Demo Complete ===")
```

---

## สรุป Part 04

| Step | สิ่งที่เรียน |
|------|-------------|
| 31 | Function syntax พื้นฐาน |
| 32 | Multiple parameter lists (currying) |
| 33 | Default parameters |
| 34 | Named arguments |
| 35 | Varargs |
| 36 | Higher-order functions |
| 37 | Recursive functions และ @tailrec |
| 38 | Function types และ eta expansion |
| 39 | Partial functions |
| 40 | โปรแกรม complete |

## แบบฝึกหัด

1. เขียน `compose` function ที่รับ `f: B => C` และ `g: A => B` แล้วส่งคืน `A => C`
2. สร้าง `retry(times: Int)(f: => A): Option[A]` ที่ลองทำ f ซ้ำถ้า exception
3. เขียน `zipWith(f: (A, B) => C)(as: List[A], bs: List[B]): List[C]`
4. implement `quicksort` ด้วย recursion
5. สร้าง `memoize` สำหรับ function ที่มี 2 parameters

## ต่อไป

**[Part 05 →](part-05-string-and-io.md)** — Strings, String Interpolation และ I/O
