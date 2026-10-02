# Part 03 — Control Flow: if/else, Loops, Match
## Steps 21–30: ควบคุมการทำงานของโปรแกรม

> **เป้าหมาย**: เข้าใจ if/else expressions, loops ทุกรูปแบบ, และ pattern matching เบื้องต้น

---

## Step 21 — if/else Expression

ใน Scala `if/else` เป็น **expression** (มีค่าส่งคืน) ไม่ใช่ statement

```scala
// แบบ Statement (Java style — ไม่แนะนำใน Scala)
var grade = ""
val score = 85
if (score >= 90) {
  grade = "A"
} else if (score >= 80) {
  grade = "B"
} else if (score >= 70) {
  grade = "C"
} else {
  grade = "F"
}

// แบบ Expression (Scala style — แนะนำ)
val score2 = 85
val grade2 = if (score2 >= 90) "A"
             else if (score2 >= 80) "B"
             else if (score2 >= 70) "C"
             else "F"

println(grade2)  // B
```

### if/else แบบ Scala 3

```scala
// Single expression
val max = if x > y then x else y

// Multiline Scala 3 (indentation-based)
val result =
  if score >= 90 then
    "Excellent!"
  else if score >= 70 then
    "Good"
  else
    "Needs improvement"

// กับ block
val description =
  if score >= 90 then
    val stars = "⭐" * 5
    s"$stars Excellent! Score: $score"
  else
    s"Score: $score. Keep studying!"

println(description)
```

### if/else กับ Unit

```scala
// ถ้า if ไม่มี else และ body เป็น Unit
val n = 10
if (n > 0) println("Positive")  // type = Unit

// ถ้า if/else ไม่ compatible type → Any
val result2 = if (n > 0) "positive" else 42
// result2: Any = "positive"
```

### Nested if/else

```scala
def classify(n: Int): String =
  if (n < 0)
    "negative"
  else if (n == 0)
    "zero"
  else if (n < 10)
    "small positive"
  else if (n < 100)
    "medium positive"
  else
    "large positive"

println(classify(-5))   // negative
println(classify(0))    // zero
println(classify(7))    // small positive
println(classify(42))   // medium positive
println(classify(999))  // large positive
```

---

## Step 22 — while Loop

```scala
// Basic while
var i = 0
while (i < 5) {
  println(s"i = $i")
  i += 1
}
// i = 0, 1, 2, 3, 4

// Scala 3 style (ไม่ต้องมี parentheses)
var j = 0
while j < 5 do
  println(s"j = $j")
  j += 1

// do-while (ทำงานอย่างน้อย 1 ครั้ง)
var k = 10
do {
  println(s"k = $k")
  k += 1
} while (k < 5)
// k = 10 (ทำงาน 1 ครั้ง แม้ condition เป็น false)
```

### while กับ Break (ไม่ควรใช้แต่รู้ไว้)

```scala
import scala.util.control.Breaks._

// Breakable block
breakable {
  var n = 0
  while (n < 100) {
    if (n == 5) break()  // ออกจาก loop
    println(n)
    n += 1
  }
}
// พิมพ์ 0, 1, 2, 3, 4

// แนะนำให้ใช้ recursive หรือ takeWhile แทน
val result = (0 until 100).takeWhile(_ < 5).toList
println(result)  // List(0, 1, 2, 3, 4)
```

---

## Step 23 — for Loop และ for Comprehension

### Basic for Loop

```scala
// Range loop
for (i <- 1 to 5) println(i)     // 1, 2, 3, 4, 5 (inclusive)
for (i <- 1 until 5) println(i)  // 1, 2, 3, 4 (exclusive)

// Scala 3 style
for i <- 1 to 5 do println(i)

// Step
for (i <- 1 to 10 by 2) println(i)  // 1, 3, 5, 7, 9
for (i <- 10 to 1 by -1) println(i) // 10, 9, ..., 1

// Iterate over collection
val fruits = List("apple", "banana", "cherry")
for (fruit <- fruits) println(fruit)

// With index
for ((fruit, index) <- fruits.zipWithIndex)
  println(s"${index + 1}. $fruit")
```

### for กับ Guard (filter)

```scala
// for with guard
for (i <- 1 to 20 if i % 3 == 0)
  println(i)  // 3, 6, 9, 12, 15, 18

// Multiple guards
for (i <- 1 to 100
     if i % 2 == 0
     if i % 3 == 0)
  println(i)  // 6, 12, 18, 24, ...

// Nested for
for (i <- 1 to 3; j <- 1 to 3)
  println(s"($i, $j)")
```

### for Comprehension (ส่งคืน collection)

```scala
// for yield — สร้าง collection ใหม่
val doubled = for (i <- 1 to 5) yield i * 2
println(doubled)  // Vector(2, 4, 6, 8, 10)

// กับ filter
val evenSquares = for {
  i <- 1 to 10
  if i % 2 == 0
} yield i * i
println(evenSquares)  // Vector(4, 16, 36, 64, 100)

// Nested with yield
val pairs = for {
  i <- 1 to 3
  j <- 1 to 3
  if i != j
} yield (i, j)
println(pairs)
// Vector((1,2), (1,3), (2,1), (2,3), (3,1), (3,2))

// กับ List
val names = List("alice", "bob", "charlie")
val upper = for (name <- names) yield name.capitalize
println(upper)  // List(Alice, Bob, Charlie)
```

---

## Step 24 — for Comprehension กับ Option และ Collections

```scala
// for กับ Option (ดีมาก!)
val maybeAge: Option[Int] = Some(25)
val maybeScore: Option[Double] = Some(9.5)

val result = for {
  age   <- maybeAge
  score <- maybeScore
} yield s"Age: $age, Score: $score"

println(result)  // Some(Age: 25, Score: 9.5)

// ถ้า Option เป็น None
val noAge: Option[Int] = None
val result2 = for {
  age   <- noAge         // None → short-circuit
  score <- maybeScore
} yield s"Age: $age, Score: $score"

println(result2)  // None

// for กับ List (Cartesian product)
val xs = List(1, 2, 3)
val ys = List("a", "b")
val combos = for {
  x <- xs
  y <- ys
} yield s"$x$y"
println(combos)  // List(1a, 1b, 2a, 2b, 3a, 3b)
```

### for กับ Future (Preview)

```scala
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

// for กับ Future ทำงาน asynchronously
val futureResult = for {
  user  <- fetchUser(1)       // Future[User]
  order <- fetchOrder(user)   // Future[Order]
} yield (user, order)
// futureResult: Future[(User, Order)]
```

---

## Step 25 — Pattern Matching เบื้องต้น

Pattern Matching คือ switch statement แบบ supercharged

```scala
// Basic match
val day = 3
val dayName = day match {
  case 1 => "Monday"
  case 2 => "Tuesday"
  case 3 => "Wednesday"
  case 4 => "Thursday"
  case 5 => "Friday"
  case 6 => "Saturday"
  case 7 => "Sunday"
  case _ => "Invalid day"  // wildcard — ทุก case ที่เหลือ
}
println(dayName)  // Wednesday
```

### Match กับ Type

```scala
def describe(x: Any): String = x match {
  case i: Int    => s"Integer: $i"
  case d: Double => s"Double: $d"
  case s: String => s"String: '$s' (length=${s.length})"
  case b: Boolean => s"Boolean: $b"
  case l: List[?] => s"List with ${l.length} elements"
  case null      => "null value"
  case _         => s"Unknown: $x"
}

println(describe(42))          // Integer: 42
println(describe(3.14))        // Double: 3.14
println(describe("hello"))     // String: 'hello' (length=5)
println(describe(true))        // Boolean: true
println(describe(List(1,2,3))) // List with 3 elements
```

### Match กับ Guard

```scala
def classify(n: Int): String = n match {
  case 0              => "zero"
  case n if n < 0    => s"negative ($n)"
  case n if n < 10   => s"small ($n)"
  case n if n < 100  => s"medium ($n)"
  case n             => s"large ($n)"
}

println(classify(0))    // zero
println(classify(-5))   // negative (-5)
println(classify(7))    // small (7)
println(classify(42))   // medium (42)
println(classify(999))  // large (999)
```

### Match กับ Tuple

```scala
def describe(point: (Int, Int)): String = point match {
  case (0, 0) => "Origin"
  case (x, 0) => s"On X-axis at x=$x"
  case (0, y) => s"On Y-axis at y=$y"
  case (x, y) if x == y => s"On diagonal at ($x,$y)"
  case (x, y) => s"Point at ($x,$y)"
}

println(describe((0, 0)))    // Origin
println(describe((5, 0)))    // On X-axis at x=5
println(describe((0, 3)))    // On Y-axis at y=3
println(describe((4, 4)))    // On diagonal at (4,4)
println(describe((2, 7)))    // Point at (2,7)
```

---

## Step 26 — Match กับ Case Classes

```scala
// Case class definitions
case class Point(x: Double, y: Double)
case class Circle(center: Point, radius: Double)
case class Rectangle(topLeft: Point, width: Double, height: Double)

// Sealed trait + case classes (ADT)
sealed trait Shape
case class Circle2(center: Point, radius: Double) extends Shape
case class Rectangle2(topLeft: Point, width: Double, height: Double) extends Shape
case class Triangle(a: Point, b: Point, c: Point) extends Shape

def area(shape: Shape): Double = shape match {
  case Circle2(_, r)             => math.Pi * r * r
  case Rectangle2(_, w, h)      => w * h
  case Triangle(a, b, c)        =>
    // Heron's formula
    val ab = math.sqrt(math.pow(b.x - a.x, 2) + math.pow(b.y - a.y, 2))
    val bc = math.sqrt(math.pow(c.x - b.x, 2) + math.pow(c.y - b.y, 2))
    val ca = math.sqrt(math.pow(a.x - c.x, 2) + math.pow(a.y - c.y, 2))
    val s = (ab + bc + ca) / 2
    math.sqrt(s * (s - ab) * (s - bc) * (s - ca))
}

val shapes: List[Shape] = List(
  Circle2(Point(0, 0), 5.0),
  Rectangle2(Point(0, 0), 4.0, 3.0),
  Triangle(Point(0, 0), Point(3, 0), Point(0, 4))
)

shapes.foreach { shape =>
  println(f"Area of ${shape.getClass.getSimpleName}: ${area(shape)}%.2f")
}
// Area of Circle2: 78.54
// Area of Rectangle2: 12.00
// Area of Triangle: 6.00
```

---

## Step 27 — Match กับ List Patterns

```scala
def describeList(list: List[Int]): String = list match {
  case Nil           => "empty list"
  case x :: Nil      => s"single element: $x"
  case x :: y :: Nil => s"two elements: $x and $y"
  case x :: rest     => s"starts with $x, has ${rest.length} more elements"
}

println(describeList(List()))          // empty list
println(describeList(List(1)))         // single element: 1
println(describeList(List(1, 2)))      // two elements: 1 and 2
println(describeList(List(1, 2, 3, 4))) // starts with 1, has 3 more elements

// Recursive sum with pattern matching
def sum(list: List[Int]): Int = list match {
  case Nil       => 0
  case x :: rest => x + sum(rest)
}

println(sum(List(1, 2, 3, 4, 5)))  // 15

// Fibonacci with match
def fib(n: Int): Int = n match {
  case 0 => 0
  case 1 => 1
  case n => fib(n - 1) + fib(n - 2)
}

(0 to 10).foreach(n => print(s"${fib(n)} "))
println()  // 0 1 1 2 3 5 8 13 21 34 55
```

---

## Step 28 — foreach และ Functional Loops

```scala
val numbers = List(1, 2, 3, 4, 5)

// forEach — side effect only
numbers.foreach(n => println(n))
numbers.foreach(println)  // อ่านง่ายกว่า

// map — transform each element
val doubled = numbers.map(n => n * 2)
val tripled = numbers.map(_ * 3)  // _ คือ parameter เดียว
println(doubled)  // List(2, 4, 6, 8, 10)

// filter — keep matching elements
val evens = numbers.filter(_ % 2 == 0)
println(evens)  // List(2, 4)

// foldLeft — accumulate
val sum = numbers.foldLeft(0)(_ + _)
println(sum)  // 15

val product = numbers.foldLeft(1)(_ * _)
println(product)  // 120

// reduce — same as fold but no initial value
val sum2 = numbers.reduce(_ + _)
println(sum2)  // 15

// Chain operations
val result = (1 to 100)
  .filter(_ % 2 == 0)    // เอาเฉพาะเลขคู่
  .map(_ * _ )            // ยกกำลังสอง
  .filter(_ < 1000)       // เอาแค่น้อยกว่า 1000
  .sum                    // หาผลรวม
println(result)  // 1240
```

---

## Step 29 — Loop Patterns ที่ใช้บ่อย

```scala
// 1. Process với index
val items = List("apple", "banana", "cherry")
items.zipWithIndex.foreach { case (item, i) =>
  println(s"${i + 1}. $item")
}

// 2. Group และ process
val nums = (1 to 20).toList
val groups = nums.grouped(5).toList
groups.foreach(group => println(group.mkString(", ")))
// 1, 2, 3, 4, 5
// 6, 7, 8, 9, 10
// ...

// 3. Sliding window
val data = List(1, 2, 3, 4, 5, 6)
val windows = data.sliding(3).toList
windows.foreach(w => println(w.sum))  // Moving sum of 3

// 4. zip สอง collections
val keys = List("name", "age", "city")
val values = List("Alice", "30", "Bangkok")
val zipped = keys.zip(values).toMap
println(zipped)  // Map(name -> Alice, age -> 30, city -> Bangkok)

// 5. flatMap (map + flatten)
val words = List("Hello World", "Scala Is", "Amazing")
val allWords = words.flatMap(_.split(" "))
println(allWords)  // List(Hello, World, Scala, Is, Amazing)

// 6. takeWhile / dropWhile
val seq = List(2, 4, 6, 1, 8, 10)
val before = seq.takeWhile(_ % 2 == 0)  // List(2, 4, 6)
val after  = seq.dropWhile(_ % 2 == 0)  // List(1, 8, 10)
```

---

## Step 30 — โปรแกรม Control Flow Complete

```scala
// ControlFlowDemo.scala

@main def controlFlowDemo(): Unit =
  
  println("=== Control Flow Demo ===\n")
  
  // ==============================
  // 1. FizzBuzz Classic
  // ==============================
  println("--- FizzBuzz (1-20) ---")
  for (i <- 1 to 20) {
    val result = (i % 3, i % 5) match {
      case (0, 0) => "FizzBuzz"
      case (0, _) => "Fizz"
      case (_, 0) => "Buzz"
      case _      => i.toString
    }
    print(s"$result ")
  }
  println()
  
  // ==============================
  // 2. Number Pyramid
  // ==============================
  println("\n--- Number Pyramid ---")
  val height = 5
  for (row <- 1 to height) {
    val spaces = " " * (height - row)
    val nums = (1 to row).mkString(" ")
    println(s"$spaces$nums")
  }
  
  // ==============================
  // 3. Prime Numbers
  // ==============================
  println("\n--- Prime Numbers (2-50) ---")
  def isPrime(n: Int): Boolean =
    if (n < 2) false
    else if (n == 2) true
    else !(2 to math.sqrt(n).toInt).exists(n % _ == 0)
  
  val primes = (2 to 50).filter(isPrime)
  println(primes.mkString(", "))
  
  // ==============================
  // 4. Calculator with Match
  // ==============================
  println("\n--- Calculator ---")
  def calculate(a: Double, op: String, b: Double): Either[String, Double] =
    op match {
      case "+" => Right(a + b)
      case "-" => Right(a - b)
      case "*" => Right(a * b)
      case "/" =>
        if (b == 0) Left("Division by zero!")
        else Right(a / b)
      case "%" => Right(a % b)
      case op  => Left(s"Unknown operator: $op")
    }
  
  val operations = List(
    (10.0, "+", 5.0),
    (10.0, "-", 3.0),
    (6.0,  "*", 7.0),
    (10.0, "/", 3.0),
    (10.0, "/", 0.0),
    (10.0, "^", 2.0)
  )
  
  operations.foreach { case (a, op, b) =>
    calculate(a, op, b) match {
      case Right(result) => println(f"  $a $op $b = $result%.4f")
      case Left(error)   => println(s"  $a $op $b → Error: $error")
    }
  }
  
  // ==============================
  // 5. Word Frequency Counter
  // ==============================
  println("\n--- Word Frequency ---")
  val text = "scala is great scala is fast scala is fun"
  val wordFreq = text.split(" ")
    .groupBy(identity)
    .view
    .mapValues(_.length)
    .toSeq
    .sortBy(-_._2)
  
  wordFreq.foreach { case (word, count) =>
    val bar = "█" * count
    println(f"  $word%-10s $bar ($count)")
  }
  
  // ==============================
  // 6. Collatz Conjecture
  // ==============================
  println("\n--- Collatz Sequence for 27 ---")
  def collatz(n: Long): List[Long] =
    if (n == 1) List(1)
    else if (n % 2 == 0) n :: collatz(n / 2)
    else n :: collatz(3 * n + 1)
  
  val seq = collatz(27)
  println(s"Steps: ${seq.length}")
  println(s"Max: ${seq.max}")
  println(s"Sequence: ${seq.take(10).mkString(", ")}...")
  
  // ==============================
  // 7. Pattern Matching Complex
  // ==============================
  println("\n--- Pattern Matching ---")
  
  sealed trait Expr
  case class Num(value: Double) extends Expr
  case class Add(left: Expr, right: Expr) extends Expr
  case class Mul(left: Expr, right: Expr) extends Expr
  case class Neg(expr: Expr) extends Expr
  
  def eval(expr: Expr): Double = expr match
    case Num(v)    => v
    case Add(l, r) => eval(l) + eval(r)
    case Mul(l, r) => eval(l) * eval(r)
    case Neg(e)    => -eval(e)
  
  def show(expr: Expr): String = expr match
    case Num(v)    => v.toString
    case Add(l, r) => s"(${show(l)} + ${show(r)})"
    case Mul(l, r) => s"(${show(l)} * ${show(r)})"
    case Neg(e)    => s"-(${show(e)})"
  
  // (2 + 3) * -(4 + 1)
  val expr = Mul(
    Add(Num(2), Num(3)),
    Neg(Add(Num(4), Num(1)))
  )
  
  println(s"Expression: ${show(expr)}")
  println(s"Result: ${eval(expr)}")
  
  println("\n=== Control Flow Demo Complete ===")
```

---

## สรุป Part 03

| Step | สิ่งที่เรียน |
|------|-------------|
| 21 | if/else expression |
| 22 | while loop |
| 23 | for loop และ for comprehension |
| 24 | for กับ Option และ collection |
| 25 | Pattern matching เบื้องต้น |
| 26 | Match กับ case classes |
| 27 | Match กับ List patterns |
| 28 | forEach, map, filter (functional) |
| 29 | Loop patterns ที่ใช้บ่อย |
| 30 | โปรแกรม complete |

## แบบฝึกหัด

1. เขียน function หา factorial ด้วย recursion และ match
2. เขียน function ตรวจสอบว่า string เป็น palindrome หรือไม่
3. ใช้ for comprehension สร้าง multiplication table (1-10)
4. เขียน Bubble Sort ด้วย while loop
5. สร้าง simple interpreter สำหรับ expression (เพิ่ม Sub และ Div ใน Step 30)

## ต่อไป

**[Part 04 →](part-04-functions.md)** — Functions และ Methods แบบครบถ้วน
