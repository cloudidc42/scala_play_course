# Part 02 — Variables, Data Types และ Type Inference
## Steps 11–20: ระบบ Type ของ Scala

> **เป้าหมาย**: เข้าใจ type system, ประกาศตัวแปร, และ type inference ของ Scala

---

## Step 11 — val และ var

Scala มีสองวิธีประกาศตัวแปร:

```scala
// val = immutable (ค่าคงที่ — แนะนำ)
val pi: Double = 3.14159
val name: String = "Scala"
val maxRetries: Int = 3

// var = mutable (เปลี่ยนค่าได้)
var count: Int = 0
var message: String = "initial"

// ทดสอบ
count = count + 1  // OK
count += 1         // OK
// pi = 3.14       // Error: reassignment to val

println(s"count = $count")    // count = 2
println(s"name = $name")      // name = Scala
```

### ทำไมต้องใช้ val?

```scala
// val ทำให้ code อ่านง่ายและ predictable
val total = 100
// ทุกที่ที่เห็น total รู้แน่ว่าคือ 100 เสมอ

// var ทำให้ต้อง track ว่าค่าเปลี่ยนตรงไหนบ้าง
var total2 = 100
total2 = 200  // ใครๆ อาจเปลี่ยนได้
total2 = 300

// ยิ่งใน concurrent code ยิ่งอันตราย
// var ทำให้เกิด race conditions
```

### Lazy val

```scala
// lazy val: คำนวณค่าแค่ครั้งแรกที่ใช้
lazy val expensiveComputation: Int = {
  println("Computing...")
  Thread.sleep(1000)  // จำลองการคำนวณที่ใช้เวลา
  42
}

// ตอนนี้ยังไม่คำนวณ
println("Before use")
println(expensiveComputation)  // Computing... 42 (คำนวณตอนนี้)
println(expensiveComputation)  // 42 (ใช้ค่าที่ cache ไว้แล้ว)
```

---

## Step 12 — Primitive Types ใน Scala

Scala มี type ที่ correspond กับ Java primitives:

```scala
// Numeric Types
val byteVal: Byte     = 127              // 8-bit: -128 to 127
val shortVal: Short   = 32767            // 16-bit: -32768 to 32767
val intVal: Int       = 2147483647       // 32-bit integer
val longVal: Long     = 9223372036854775807L  // 64-bit
val floatVal: Float   = 3.14f            // 32-bit floating point
val doubleVal: Double = 3.141592653589793    // 64-bit floating point

// Text
val charVal: Char     = 'A'             // Single character
val stringVal: String = "Hello Scala"   // String (immutable)

// Boolean
val boolVal: Boolean  = true

// Unit (เหมือน void ใน Java)
val unitVal: Unit = ()

// Nothing (bottom type — ไม่มี instance)
// def error(msg: String): Nothing = throw new RuntimeException(msg)

// Null (ไม่ควรใช้ใน Scala!)
// val nullVal: String = null  // อย่าทำแบบนี้!
```

### Type Sizes

```scala
println(s"Byte max: ${Byte.MaxValue}")       // 127
println(s"Short max: ${Short.MaxValue}")     // 32767
println(s"Int max: ${Int.MaxValue}")         // 2147483647
println(s"Long max: ${Long.MaxValue}")       // 9223372036854775807
println(s"Float max: ${Float.MaxValue}")     // 3.4028235E38
println(s"Double max: ${Double.MaxValue}")   // 1.7976931348623157E308
```

---

## Step 13 — Type Inference

Scala สามารถเดา type ได้เองโดยไม่ต้องระบุ:

```scala
// ระบุ type เอง (verbose)
val name: String = "Alice"
val age: Int = 30
val score: Double = 9.5
val active: Boolean = true

// ให้ Scala เดาเอง (concise — แนะนำ)
val name2 = "Alice"    // String
val age2 = 30          // Int
val score2 = 9.5       // Double
val active2 = true     // Boolean

// ตรวจสอบ type ใน REPL
// scala> :type name2
// String
```

### Type Inference กับ Collections

```scala
val numbers = List(1, 2, 3)        // List[Int]
val names = List("Alice", "Bob")   // List[String]
val mixed = List(1, "hello")       // List[Any] — Scala รวม type

val tuple = (1, "hello", true)     // (Int, String, Boolean)

val map = Map("a" -> 1, "b" -> 2) // Map[String, Int]
```

### เมื่อไหร่ควรระบุ type เอง?

```scala
// 1. เมื่อ Scala เดาผิด
val x = 1      // Int (แต่เราอาจต้องการ Long)
val x2: Long = 1  // Long

// 2. Public API — ระบุ type ทำให้ clear
def calculateTax(income: Double): Double = income * 0.2

// 3. เมื่อ initialize เป็น null (อย่าทำ! ใช้ Option แทน)
var conn: java.sql.Connection = null  // ถ้าจำเป็น

// 4. เพื่อ document intent
val maxItems: Int = 100  // ชัดเจนว่าเป็น Int

// 5. Complex types — เพื่อ readability
val processor: PartialFunction[Any, String] = {
  case i: Int => s"Int: $i"
  case s: String => s"String: $s"
}
```

---

## Step 14 — String Types และ Operations

```scala
// String creation
val s1 = "Hello"
val s2 = "World"

// Concatenation
val s3 = s1 + " " + s2    // "Hello World"
val s4 = s1.concat(" ").concat(s2)  // เหมือนกัน

// String Interpolation (แนะนำ)
val name = "Alice"
val age = 30
val greeting = s"Hello, $name! You are $age years old."

// Expression interpolation
val price = 99.99
val msg = s"Price: ${price * 1.07} (with tax)"

// f-interpolator (printf style)
val formatted = f"Price: $price%.2f"    // Price: 99.99
val pi = 3.14159
val piStr = f"Pi = $pi%.4f"            // Pi = 3.1416

// raw-interpolator (ไม่ escape)
val path = raw"C:\Users\name\file.txt"  // ไม่ escape \
```

### String Methods ที่ใช้บ่อย

```scala
val text = "  Hello, Scala World!  "

// Case conversion
text.toUpperCase          // "  HELLO, SCALA WORLD!  "
text.toLowerCase          // "  hello, scala world!  "

// Trim
text.trim                 // "Hello, Scala World!"
text.strip                // Same as trim (Java 11+)

// Substring
val s = "Hello Scala"
s.substring(6)            // "Scala"
s.substring(0, 5)         // "Hello"
s.take(5)                 // "Hello"
s.drop(6)                 // "Scala"

// Search
s.contains("Scala")       // true
s.startsWith("Hello")     // true
s.endsWith("Scala")       // true
s.indexOf("Scala")        // 6
s.lastIndexOf("l")        // 7

// Replace
s.replace("Hello", "Hi")           // "Hi Scala"
s.replaceAll("[aeiou]", "*")        // "H*ll* Sc*l*" (regex)
s.replaceFirst("[A-Z]", "_")        // "_ello Scala"

// Split
"a,b,c,d".split(",")               // Array(a, b, c, d)
"Hello World Scala".split(" ")     // Array(Hello, World, Scala)

// Join
List("a", "b", "c").mkString(", ") // "a, b, c"
Array("x", "y", "z").mkString("-") // "x-y-z"

// Length and Chars
s.length              // 11
s.size                // 11 (alias)
s.charAt(0)           // 'H'
s(0)                  // 'H' (shorthand)
s.head                // 'H'
s.last                // 'a'
s.isEmpty             // false
s.nonEmpty            // true

// Reverse
s.reverse             // "alacS olleH"

// Repeat (Scala 3)
"ha".repeat(3)        // "hahaha"
"-" * 20              // "--------------------"
```

### Multiline Strings

```scala
// Triple-quoted string
val poem = """
  |Roses are red,
  |Violets are blue,
  |Scala is awesome,
  |And so are you!
  """.stripMargin

println(poem)

// Strip leading whitespace with |
val sql = """
  |SELECT *
  |FROM users
  |WHERE active = true
  |ORDER BY name
  """.stripMargin.trim

println(sql)
```

---

## Step 15 — Numeric Operations

```scala
// Basic arithmetic
val a = 10
val b = 3

println(a + b)    // 13
println(a - b)    // 7
println(a * b)    // 30
println(a / b)    // 3 (integer division!)
println(a % b)    // 1 (modulus)

// Float division
println(a.toDouble / b)  // 3.3333...
println(a / b.toDouble)  // 3.3333...
println(10.0 / 3)        // 3.3333...

// Power (ไม่มี ** operator ใน Scala)
import scala.math._
println(pow(2, 10))    // 1024.0
println(2.0 ** 10)     // 1024.0 (Scala 3.x ใหม่)

// Math functions
println(abs(-42))       // 42
println(sqrt(16.0))     // 4.0
println(ceil(3.2))      // 4.0
println(floor(3.9))     // 3.0
println(round(3.5))     // 4
println(min(10, 20))    // 10
println(max(10, 20))    // 20

// BigInt และ BigDecimal สำหรับตัวเลขขนาดใหญ่
val bigNum = BigInt("123456789012345678901234567890")
val bigDec = BigDecimal("3.14159265358979323846264338327950288")

println(bigNum * 2)
println(bigDec.setScale(10, BigDecimal.RoundingMode.HALF_UP))
```

### Numeric Conversions

```scala
val i: Int = 42
val l: Long = i.toLong        // Int → Long
val d: Double = i.toDouble    // Int → Double
val f: Float = i.toFloat      // Int → Float
val s: String = i.toString    // Int → String
val c: Char = i.toChar        // Int → Char ('@' = 64)

// String → Numeric
val str = "123"
val num = str.toInt           // 123
val numL = str.toLong         // 123L
val numD = str.toDouble       // 123.0

// Safe conversion ด้วย Try
import scala.util.Try
val safe = Try("abc".toInt).getOrElse(0)  // 0 (ไม่ throw exception)
```

---

## Step 16 — Boolean Operations

```scala
val t = true
val f = false

// Logical operators
println(t && f)   // false (AND)
println(t || f)   // true  (OR)
println(!t)       // false (NOT)

// Short-circuit evaluation
def check(): Boolean = {
  println("check() called")
  true
}

val result1 = false && check()  // check() ไม่ถูกเรียก!
val result2 = true  || check()  // check() ไม่ถูกเรียก!
val result3 = true  && check()  // check() ถูกเรียก

// Comparison operators
val x = 10
val y = 20

println(x == y)   // false (equality)
println(x != y)   // true  (inequality)
println(x < y)    // true
println(x > y)    // false
println(x <= y)   // true
println(x >= y)   // false

// Reference equality
val s1 = new String("hello")
val s2 = new String("hello")
println(s1 == s2)    // true  (value equality in Scala)
println(s1 eq s2)    // false (reference equality)
println(s1 ne s2)    // true  (reference inequality)
```

---

## Step 17 — Char Type

```scala
val c: Char = 'A'

// Char operations
println(c.toLower)        // a
println(c.toUpper)        // A
println(c.isLetter)       // true
println(c.isDigit)        // false
println(c.isWhitespace)   // false
println(c.isUpper)        // true
println(c.isLower)        // false

// Char ↔ Int conversion
val charA = 'A'
val intA: Int = charA     // 65 (implicit)
val charFromInt = 66.toChar  // 'B'

// Iterate through alphabet
val alphabet = ('a' to 'z').mkString
println(alphabet)  // abcdefghijklmnopqrstuvwxyz

// ASCII art example
for (i <- 1 to 5) {
  val line = "*" * i
  println(line)
}
// *
// **
// ***
// ****
// *****
```

---

## Step 18 — Type Hierarchy ใน Scala

```
                   Any
                  /   \
               AnyVal  AnyRef
              /  | \    |
           Int Long .. String, List, etc.
                         |
                        Null
                    
           Nothing (ล่างสุด — subtype ของทุก type)
```

```scala
// Any — supertype ของทุกอย่าง
val anything: Any = 42
val anything2: Any = "hello"
val anything3: Any = List(1, 2, 3)

// AnyVal — supertype ของ value types
val v: AnyVal = 42
val v2: AnyVal = 3.14
val v3: AnyVal = 'A'

// AnyRef — supertype ของ reference types (= java.lang.Object)
val r: AnyRef = "hello"
val r2: AnyRef = List(1, 2, 3)

// Nothing — ไม่มี instance แต่เป็น subtype ของทุก type
def throwError(): Nothing = throw new RuntimeException("Error!")

// Null — มีแค่ null value (อย่าใช้ใน Scala!)
val nullStr: String = null  // ควรใช้ Option[String] แทน
```

### Type Casting

```scala
val any: Any = "Hello"

// Pattern matching (safe)
any match {
  case s: String => println(s"String: $s")
  case i: Int    => println(s"Int: $i")
  case _         => println("Unknown")
}

// isInstanceOf / asInstanceOf (Java style — ไม่แนะนำ)
if (any.isInstanceOf[String]) {
  val s = any.asInstanceOf[String]
  println(s.length)
}
```

---

## Step 19 — Unit Type

```scala
// Unit คล้าย void ใน Java
// ใช้กับ function ที่ไม่มีค่าส่งคืน

def printHello(): Unit = {
  println("Hello!")
  // ไม่มี return value (return Unit)
}

// ย่อแบบนี้ได้
def printWorld(): Unit = println("World!")

// Unit value คือ ()
val u: Unit = ()
println(u)  // ()

// Procedure syntax (เก่า — อย่าใช้)
// def oldStyle() {  // Scala 2 เท่านั้น
//   println("old")
// }

// ใน functional programming
// Unit ≈ "ทำ side effect แต่ไม่ส่งค่าคืน"
val result: Unit = printHello()
println(result)  // ()
```

---

## Step 20 — โปรแกรม Type System Complete

```scala
// TypeSystemDemo.scala — รวม type system ทั้งหมด

@main def typeSystemDemo(): Unit =
  
  println("=== Scala Type System Demo ===\n")
  
  // ==============================
  // 1. Basic Types
  // ==============================
  println("--- Basic Types ---")
  val byteVal: Byte     = 100
  val shortVal: Short   = 1000
  val intVal: Int       = 1_000_000    // _ เป็น separator (Scala 3)
  val longVal: Long     = 1_000_000_000L
  val floatVal: Float   = 3.14f
  val doubleVal: Double = 3.141592653589793
  val charVal: Char     = 'Σ'
  val strVal: String    = "Scala 3"
  val boolVal: Boolean  = true

  println(s"Byte:    $byteVal")
  println(s"Short:   $shortVal")
  println(s"Int:     $intVal")
  println(s"Long:    $longVal")
  println(s"Float:   $floatVal")
  println(s"Double:  $doubleVal")
  println(s"Char:    $charVal")
  println(s"String:  $strVal")
  println(s"Boolean: $boolVal")
  
  // ==============================
  // 2. Type Inference
  // ==============================
  println("\n--- Type Inference ---")
  val inferred1 = 42              // Int
  val inferred2 = 3.14            // Double
  val inferred3 = "hello"         // String
  val inferred4 = true            // Boolean
  val inferred5 = List(1, 2, 3)  // List[Int]
  val inferred6 = (1, "a", 3.0)  // (Int, String, Double)

  println(s"42        → ${inferred1.getClass.getSimpleName}")
  println(s"3.14      → ${inferred2.getClass.getSimpleName}")
  println(s"\"hello\" → ${inferred3.getClass.getSimpleName}")
  println(s"true      → ${inferred4.getClass.getSimpleName}")
  println(s"List(1,2) → ${inferred5.getClass.getSimpleName}")
  
  // ==============================
  // 3. Numeric Operations
  // ==============================
  println("\n--- Numeric Operations ---")
  
  case class Stats(
    min: Double,
    max: Double,
    sum: Double,
    avg: Double
  )
  
  def calcStats(nums: List[Double]): Stats =
    Stats(
      min = nums.min,
      max = nums.max,
      sum = nums.sum,
      avg = nums.sum / nums.length
    )
  
  val data = List(15.5, 23.1, 8.7, 42.0, 11.3)
  val stats = calcStats(data)
  
  println(f"Data: ${data.mkString(", ")}")
  println(f"Min:  ${stats.min}%.1f")
  println(f"Max:  ${stats.max}%.1f")
  println(f"Sum:  ${stats.sum}%.1f")
  println(f"Avg:  ${stats.avg}%.2f")
  
  // ==============================
  // 4. String Operations
  // ==============================
  println("\n--- String Operations ---")
  
  val sentence = "The quick brown fox jumps over the lazy dog"
  val words = sentence.split(" ")
  
  println(s"Sentence: $sentence")
  println(s"Words: ${words.length}")
  println(s"Longest: ${words.maxBy(_.length)}")
  println(s"Shortest: ${words.minBy(_.length)}")
  println(s"Uppercase: ${sentence.toUpperCase.take(20)}...")
  
  val wordFreq = words
    .groupBy(_.toLowerCase)
    .view
    .mapValues(_.length)
    .toMap
    .filter(_._2 > 1)
  
  if (wordFreq.nonEmpty)
    println(s"Repeated words: $wordFreq")
  else
    println("No repeated words")
  
  // ==============================
  // 5. Type Conversions
  // ==============================
  println("\n--- Type Conversions ---")
  
  val numStr = "42"
  val numInt = numStr.toInt
  val numDouble = numStr.toDouble
  val backToStr = numInt.toString
  
  println(s"\"$numStr\" → Int: $numInt")
  println(s"\"$numStr\" → Double: $numDouble")
  println(s"$numInt → String: \"$backToStr\"")
  println(s"Int → Char: ${65.toChar}")
  println(s"'A' → Int: ${'A'.toInt}")
  
  // ==============================
  // 6. Any Type
  // ==============================
  println("\n--- Any Type ---")
  
  val values: List[Any] = List(1, "hello", 3.14, true, List(1, 2))
  
  values.foreach { v =>
    val typeStr = v match
      case _: Int     => "Int"
      case _: String  => "String"
      case _: Double  => "Double"
      case _: Boolean => "Boolean"
      case _: List[?] => "List"
      case _          => "Unknown"
    println(s"  $v → $typeStr")
  }
  
  println("\n=== Type System Demo Complete ===")
```

### รัน

```bash
sbt run
```

---

## สรุป Part 02

| Step | สิ่งที่เรียน |
|------|-------------|
| 11 | val vs var และ lazy val |
| 12 | Primitive types ทั้งหมด |
| 13 | Type inference |
| 14 | String operations |
| 15 | Numeric operations และ Math |
| 16 | Boolean operations |
| 17 | Char type |
| 18 | Type hierarchy (Any, AnyVal, AnyRef) |
| 19 | Unit type |
| 20 | โปรแกรม complete |

## แบบฝึกหัด

1. สร้างตัวแปร 10 ตัวที่แตกต่าง type กัน และแสดงทั้งหมด
2. เขียน function รับ `String` และส่งคืนจำนวน vowels (a, e, i, o, u)
3. ทดลอง BigDecimal สำหรับการคำนวณเงิน (ไม่ใช้ Double เพราะ float error)
4. สร้าง function ที่แปลง temperature: Celsius → Fahrenheit → Kelvin
5. ทดสอบ type inference โดยไม่ระบุ type และใช้ `:type` ใน REPL ตรวจสอบ

## ต่อไป

**[Part 03 →](part-03-control-flow.md)** — Control Flow: if/else, loops, match
