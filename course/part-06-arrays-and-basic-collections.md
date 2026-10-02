# Part 06 — Arrays และ Basic Collections
## Steps 51–60: คอลเล็กชันพื้นฐานของ Scala

> **เป้าหมาย**: เข้าใจ Array, List, Vector, Range และ collection operations หลัก

---

## Step 51 — Arrays

Array ใน Scala เป็น mutable และ fixed-size (เหมือน Java array)

```scala
// Create arrays
val arr1 = Array(1, 2, 3, 4, 5)           // Array[Int]
val arr2 = Array("a", "b", "c")            // Array[String]
val arr3 = Array.fill(5)(0)                // Array(0, 0, 0, 0, 0)
val arr4 = Array.range(1, 6)               // Array(1, 2, 3, 4, 5)
val arr5 = Array.tabulate(5)(i => i * i)  // Array(0, 1, 4, 9, 16)

// With explicit type
val arr6: Array[Int] = new Array[Int](5)   // Array(0, 0, 0, 0, 0)
val arr7 = new Array[String](3)            // Array(null, null, null)

// 2D Array
val matrix = Array.ofDim[Int](3, 3)
val matrix2 = Array(
  Array(1, 2, 3),
  Array(4, 5, 6),
  Array(7, 8, 9)
)

// Access elements (0-indexed)
println(arr1(0))       // 1 (first)
println(arr1(4))       // 5 (last)
println(arr1.head)     // 1
println(arr1.last)     // 5

// Modify (mutable!)
val mutable = Array(1, 2, 3)
mutable(0) = 10
mutable(2) = 30
println(mutable.mkString(", "))  // 10, 2, 30

// Array info
println(arr1.length)    // 5
println(arr1.size)      // 5
println(arr1.isEmpty)   // false
println(arr1.nonEmpty)  // true
```

### Array Operations

```scala
val nums = Array(5, 3, 1, 4, 2)

// Sort (in-place for mutable, new for immutable collections)
val sorted = nums.sorted           // new Array(1, 2, 3, 4, 5)
val sortedDesc = nums.sortWith(_ > _)  // new Array(5, 4, 3, 2, 1)
println(nums.mkString(", "))       // 5, 3, 1, 4, 2 (unchanged)
println(sorted.mkString(", "))     // 1, 2, 3, 4, 5

// Search
println(sorted.contains(3))        // true
println(sorted.indexOf(3))         // 2
println(sorted.find(_ > 3))        // Some(4)
println(sorted.exists(_ > 10))     // false
println(sorted.forall(_ > 0))      // true

// Slice
println(sorted.slice(1, 4).mkString(", "))  // 2, 3, 4
println(sorted.take(3).mkString(", "))      // 1, 2, 3
println(sorted.drop(2).mkString(", "))      // 3, 4, 5

// Aggregate
println(sorted.sum)          // 15
println(sorted.product)      // 120
println(sorted.min)          // 1
println(sorted.max)          // 5

// Transform
println(sorted.map(_ * 2).mkString(", "))  // 2, 4, 6, 8, 10
println(sorted.filter(_ % 2 == 0).mkString(", "))  // 2, 4

// Flatten
val nested = Array(Array(1, 2), Array(3, 4), Array(5))
println(nested.flatten.mkString(", "))  // 1, 2, 3, 4, 5

// Concat
val a = Array(1, 2, 3)
val b = Array(4, 5, 6)
val c = a ++ b
println(c.mkString(", "))  // 1, 2, 3, 4, 5, 6
```

---

## Step 52 — List (Immutable)

List คือ linked list ที่ immutable — ใช้บ่อยที่สุดใน Scala

```scala
// Create lists
val list1 = List(1, 2, 3, 4, 5)
val list2 = List("a", "b", "c")
val empty = List.empty[Int]         // หรือ Nil
val range = List.range(1, 6)        // List(1, 2, 3, 4, 5)
val filled = List.fill(5)("x")      // List(x, x, x, x, x)
val tabulate = List.tabulate(5)(n => n * n)  // List(0, 1, 4, 9, 16)

// Cons operator (สร้าง list)
val cons = 1 :: 2 :: 3 :: Nil      // List(1, 2, 3)
val prepend = 0 :: list1            // List(0, 1, 2, 3, 4, 5)

// Access
println(list1.head)      // 1
println(list1.tail)      // List(2, 3, 4, 5)
println(list1(2))        // 3 (O(n) — ไม่แนะนำ)
println(list1.last)      // 5 (O(n))
println(list1.init)      // List(1, 2, 3, 4)

// Length
println(list1.length)    // 5 (O(n))
println(list1.size)      // 5

// Pattern matching กับ List
list1 match {
  case Nil           => println("empty")
  case x :: Nil      => println(s"single: $x")
  case x :: y :: Nil => println(s"two: $x, $y")
  case x :: rest     => println(s"head: $x, rest length: ${rest.length}")
}
// head: 1, rest length: 4
```

### List Operations

```scala
val nums = List(3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5)

// Append/Prepend
val with0 = 0 :: nums         // prepend O(1)
val withEnd = nums :+ 99       // append O(n) — ช้า!
val concat = nums ++ List(10, 11)  // concat

// Sort
println(nums.sorted.mkString(", "))          // 1, 1, 2, 3, 3, 4, 5, 5, 5, 6, 9
println(nums.sortBy(-_).mkString(", "))      // 9, 6, 5, 5, 5, 4, 3, 3, 2, 1, 1

// Distinct
println(nums.distinct.mkString(", "))  // 3, 1, 4, 5, 9, 2, 6

// Group
println(nums.groupBy(identity))
// Map(3 -> List(3, 3), 1 -> List(1, 1), ...)

// Partition
val (evens, odds) = nums.partition(_ % 2 == 0)
println(evens)  // List(4, 2, 6)
println(odds)   // List(3, 1, 1, 5, 9, 5, 3, 5)

// Span (takeWhile + dropWhile combined)
val (before, after) = nums.span(_ < 5)
println(before)  // List(3, 1, 4, 1)
println(after)   // List(5, 9, 2, 6, 5, 3, 5)

// Zip
val a = List(1, 2, 3)
val b = List("x", "y", "z")
println(a.zip(b))       // List((1,x), (2,y), (3,z))
println(a.zipWithIndex) // List((1,0), (2,1), (3,2))

// Flatten and flatMap
val nested = List(List(1,2), List(3,4), List(5))
println(nested.flatten)          // List(1, 2, 3, 4, 5)
println(nested.flatMap(x => x))  // List(1, 2, 3, 4, 5)

// foldLeft vs foldRight
val sum = nums.foldLeft(0)(_ + _)    // 44
val product = List(1,2,3,4,5).foldLeft(1)(_ * _)  // 120

// scanLeft (running total)
val running = List(1,2,3,4,5).scanLeft(0)(_ + _)
println(running)  // List(0, 1, 3, 6, 10, 15)
```

---

## Step 53 — Vector

Vector คือ indexed sequence ที่ immutable และมี O(log n) สำหรับ random access

```scala
// Create
val v1 = Vector(1, 2, 3, 4, 5)
val v2 = Vector.fill(5)(0)
val v3 = (1 to 100).toVector

// Random access O(log n) — เร็วกว่า List
println(v1(3))      // 4

// Append/Prepend O(log n)
val v4 = v1 :+ 6         // append
val v5 = 0 +: v1          // prepend
val v6 = v1 ++ v2         // concat

// เมื่อไหร่ใช้ Vector vs List?
// List  → ถ้าต้องการ prepend บ่อยๆ (head/tail operations)
// Vector → ถ้าต้องการ random access หรือ append บ่อยๆ

// Performance comparison
val n = 1000000
val bigList = List.range(0, n)
val bigVector = Vector.range(0, n)

// List(n/2) is O(n), Vector(n/2) is O(log n)
```

---

## Step 54 — Range

Range เป็น lazy sequence สำหรับตัวเลขต่อเนื่อง

```scala
// Create ranges
val r1 = 1 to 10        // Range(1, 2, ..., 10) inclusive
val r2 = 1 until 10     // Range(1, 2, ..., 9) exclusive
val r3 = 1 to 20 by 2   // Range(1, 3, 5, ..., 19)
val r4 = 20 to 1 by -1  // Range(20, 19, ..., 1)
val r5 = 'a' to 'z'     // Range of chars

// Char range
println(('a' to 'z').mkString)    // abcdefghijklmnopqrstuvwxyz
println(('A' to 'Z').mkString)    // ABCDEFGHIJKLMNOPQRSTUVWXYZ

// Range operations
println(r1.sum)         // 55
println(r1.min)         // 1
println(r1.max)         // 10
println(r1.contains(5)) // true

// Convert to collections
r1.toList               // List(1, 2, ..., 10)
r1.toArray              // Array(1, 2, ..., 10)
r1.toVector             // Vector(1, 2, ..., 10)
r1.toSet                // Set(1, 2, ..., 10)

// Range เป็น lazy — ไม่เปลือง memory
val bigRange = 1 to 1000000  // ไม่สร้าง array ขึ้นมา!
println(bigRange.sum)         // 500000500000

// Use with for
for (i <- 0 until 5; j <- 0 until 5)
  if (i + j == 4) println(s"($i,$j)")
// (0,4), (1,3), (2,2), (3,1), (4,0)
```

---

## Step 55 — Collection Methods หลัก

```scala
val nums = (1 to 10).toList

// ==============================
// Transformations
// ==============================

// map: transform each element
val doubled = nums.map(_ * 2)
// List(2, 4, 6, 8, 10, 12, 14, 16, 18, 20)

// flatMap: map + flatten
val pairs = nums.flatMap(n => List(n, -n))
// List(1, -1, 2, -2, ..., 10, -10)

// collect: map + filter using PartialFunction
val evenSquares = nums.collect {
  case n if n % 2 == 0 => n * n
}
// List(4, 16, 36, 64, 100)

// ==============================
// Filtering
// ==============================

// filter: keep matching
val evens = nums.filter(_ % 2 == 0)
// List(2, 4, 6, 8, 10)

// filterNot: remove matching
val odds = nums.filterNot(_ % 2 == 0)
// List(1, 3, 5, 7, 9)

// partition: split into two
val (small, big) = nums.partition(_ <= 5)
// small: List(1, 2, 3, 4, 5), big: List(6, 7, 8, 9, 10)

// ==============================
// Aggregation
// ==============================

println(nums.sum)            // 55
println(nums.product)        // 3628800
println(nums.min)            // 1
println(nums.max)            // 10
println(nums.count(_ > 5))   // 5

// reduce
println(nums.reduce(_ + _))           // 55 (same as sum)
println(nums.reduceLeft(_ max _))     // 10 (same as max)
println(nums.reduceRight(_ :: _))     // List(1, 2, ..., 10)

// fold
println(nums.foldLeft(0)(_ + _))      // 55
println(nums.foldLeft("")(_ + _.toString))  // "12345678910"

// ==============================
// Searching
// ==============================

println(nums.find(_ > 5))             // Some(6)
println(nums.exists(_ > 9))           // true
println(nums.forall(_ > 0))           // true
println(nums.count(_ % 3 == 0))       // 3

// ==============================
// Slicing
// ==============================

println(nums.take(3))                 // List(1, 2, 3)
println(nums.drop(7))                 // List(8, 9, 10)
println(nums.slice(2, 5))             // List(3, 4, 5)
println(nums.takeWhile(_ < 5))        // List(1, 2, 3, 4)
println(nums.dropWhile(_ < 5))        // List(5, 6, 7, 8, 9, 10)

// ==============================
// Grouping
// ==============================

val grouped = nums.groupBy(_ % 3)
// Map(0 -> List(3,6,9), 1 -> List(1,4,7,10), 2 -> List(2,5,8))

val sliding = nums.sliding(3, 2).toList
// List(List(1,2,3), List(3,4,5), List(5,6,7), List(7,8,9), List(9,10))

val chunked = nums.grouped(3).toList
// List(List(1,2,3), List(4,5,6), List(7,8,9), List(10))
```

---

## Step 56 — Sorting

```scala
case class Person(name: String, age: Int, score: Double)

val people = List(
  Person("Charlie", 25, 8.5),
  Person("Alice",   30, 9.2),
  Person("Bob",     25, 7.8),
  Person("Diana",   28, 9.2),
  Person("Eve",     22, 8.5)
)

// sortBy
val byAge = people.sortBy(_.age)
val byName = people.sortBy(_.name)
val byScoreDesc = people.sortBy(-_.score)

// Custom sort with Ordering
val byAgeNameScore = people.sortBy(p => (p.age, p.name, -p.score))

// sorted with custom Ordering
implicit val personOrdering: Ordering[Person] =
  Ordering.by((p: Person) => (-p.score, p.name))

val sortedPeople = people.sorted
println("Ranked players:")
sortedPeople.zipWithIndex.foreach { case (p, i) =>
  println(f"  ${i+1}. ${p.name}%-10s age=${p.age} score=${p.score}")
}

// sortWith (custom comparator)
val complex = people.sortWith { (p1, p2) =>
  if (p1.age != p2.age) p1.age < p2.age
  else if (p1.score != p2.score) p1.score > p2.score
  else p1.name < p2.name
}
```

---

## Step 57 — Collection Conversions

```scala
val list  = List(1, 2, 3, 4, 5)
val array = Array(1, 2, 3, 4, 5)
val set   = Set(1, 2, 3, 4, 5)
val vector = Vector(1, 2, 3, 4, 5)

// Between collections
val l2a = list.toArray       // List → Array
val a2l = array.toList       // Array → List
val l2v = list.toVector      // List → Vector
val l2s = list.toSet         // List → Set (unique)
val l2seq = list.toSeq       // List → Seq

// String ↔ Char collections
val str = "Hello Scala"
val chars = str.toList       // List[Char]
val back = chars.mkString    // "Hello Scala"

val charArray = str.toArray  // Array[Char]
val sorted = charArray.sorted.mkString  // " HScaalloe"

// Map operations
val pairs = List(("a", 1), ("b", 2), ("c", 3))
val map = pairs.toMap        // Map(a -> 1, b -> 2, c -> 3)
val backToPairs = map.toList // List((a,1), (b,2), (c,3))

// zip → Map
val keys = List("x", "y", "z")
val vals = List(1, 2, 3)
val zippedMap = keys.zip(vals).toMap  // Map(x -> 1, y -> 2, z -> 3)
```

---

## Step 58 — Lazy Collections (LazyList)

```scala
// LazyList — computed on demand
val naturals: LazyList[Int] = LazyList.from(1)
println(naturals.take(10).toList)  // List(1, 2, ..., 10)

// Fibonacci (infinite!)
val fibs: LazyList[BigInt] = {
  def fib(a: BigInt, b: BigInt): LazyList[BigInt] =
    a #:: fib(b, a + b)
  fib(0, 1)
}

println(fibs.take(15).toList)
// List(0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377)

// Primes (infinite)
def sieve(stream: LazyList[Int]): LazyList[Int] = {
  val prime = stream.head
  prime #:: sieve(stream.tail.filter(_ % prime != 0))
}

val primes = sieve(LazyList.from(2))
println(primes.take(20).toList)
// List(2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71)

// LazyList กับ Views
val result = (1 to 1000000)
  .view                      // lazy view
  .filter(_ % 2 == 0)
  .map(_ * _ )
  .take(5)
  .toList                    // evaluate here
println(result)  // List(4, 16, 36, 64, 100)
// ไม่สร้าง intermediate collections!
```

---

## Step 59 — Collection Performance

```scala
// Performance characteristics:
// ┌──────────┬─────────┬──────────┬─────────┬──────────┐
// │ Coll     │ Prepend │ Append   │ Access  │ Update   │
// ├──────────┼─────────┼──────────┼─────────┼──────────┤
// │ List     │ O(1)    │ O(n)     │ O(n)    │ O(n)     │
// │ Vector   │ O(log n)│ O(log n) │ O(log n)│ O(log n) │
// │ Array    │ O(n)    │ O(n)     │ O(1)    │ O(1)     │
// │ ArrayBuf │ O(n)    │ O(1)*    │ O(1)    │ O(1)     │
// └──────────┴─────────┴──────────┴─────────┴──────────┘

// เมื่อไหร่ใช้อะไร?
// List  → head/tail operations, pattern matching
// Vector → random access, both ends operations
// Array → Java interop, mutable indexed, performance-critical
// ArrayBuffer → building large sequences element by element

import scala.collection.mutable.ArrayBuffer

// ArrayBuffer (mutable)
val buf = ArrayBuffer[Int]()
buf += 1          // append
buf += 2
buf += 3
buf.prepend(0)    // prepend
buf.append(4)     // append
buf.insert(2, 99) // insert at index
buf.remove(2)     // remove at index

println(buf.toList)  // List(0, 1, 2, 3, 4)

// Build large list efficiently
def buildList(n: Int): List[Int] = {
  // Bad: List append is O(n)
  // var result = List.empty[Int]
  // (1 to n).foreach(i => result = result :+ i)  // O(n²)
  
  // Good: prepend then reverse O(n)
  var result = List.empty[Int]
  (n to 1 by -1).foreach(i => result = i :: result)
  result
  
  // Better: use List.range or toList
  // List.range(1, n + 1)
}
```

---

## Step 60 — โปรแกรม Collections Complete

```scala
// CollectionsMasterDemo.scala

@main def collectionsDemo(): Unit =
  
  println("=== Collections Master Demo ===\n")
  
  // ==============================
  // 1. Matrix Operations
  // ==============================
  println("--- Matrix Operations ---")
  
  type Matrix = Array[Array[Int]]
  
  def createMatrix(rows: Int, cols: Int)(f: (Int, Int) => Int): Matrix =
    Array.tabulate(rows, cols)(f)
  
  def transpose(m: Matrix): Matrix = {
    val rows = m.length
    val cols = m(0).length
    Array.tabulate(cols, rows)((i, j) => m(j)(i))
  }
  
  def multiply(a: Matrix, b: Matrix): Matrix = {
    val n = a.length
    val m = b(0).length
    val k = b.length
    Array.tabulate(n, m) { (i, j) =>
      (0 until k).map(p => a(i)(p) * b(p)(j)).sum
    }
  }
  
  def printMatrix(m: Matrix): Unit = {
    m.foreach(row => println("  " + row.mkString(" ")))
  }
  
  val identity = createMatrix(3, 3)((i, j) => if (i == j) 1 else 0)
  val numbers  = createMatrix(3, 3)((i, j) => i * 3 + j + 1)
  
  println("Numbers matrix:")
  printMatrix(numbers)
  println("Transposed:")
  printMatrix(transpose(numbers))
  println("I × Numbers:")
  printMatrix(multiply(identity, numbers))
  
  // ==============================
  // 2. Statistical Analysis
  // ==============================
  println("\n--- Statistical Analysis ---")
  
  val data = List(
    4.2, 7.8, 2.1, 9.5, 3.6, 8.1, 5.4, 6.7, 1.9, 7.3,
    4.8, 6.2, 3.1, 8.9, 5.7, 2.4, 9.1, 4.5, 7.6, 3.8
  )
  
  val n = data.length
  val mean = data.sum / n
  val variance = data.map(x => math.pow(x - mean, 2)).sum / n
  val stddev = math.sqrt(variance)
  val sorted = data.sorted
  val median = if (n % 2 == 0)
    (sorted(n/2 - 1) + sorted(n/2)) / 2.0
  else sorted(n/2)
  
  val q1 = sorted(n/4)
  val q3 = sorted(3*n/4)
  val iqr = q3 - q1
  
  println(f"Count:    $n")
  println(f"Mean:     $mean%.3f")
  println(f"Median:   $median%.3f")
  println(f"Std Dev:  $stddev%.3f")
  println(f"Min:      ${sorted.head}%.3f")
  println(f"Q1:       $q1%.3f")
  println(f"Q3:       $q3%.3f")
  println(f"Max:      ${sorted.last}%.3f")
  println(f"IQR:      $iqr%.3f")
  
  // Histogram
  val bins = 5
  val binWidth = (sorted.last - sorted.head) / bins
  println("\nHistogram:")
  (0 until bins).foreach { i =>
    val lo = sorted.head + i * binWidth
    val hi = lo + binWidth
    val count = data.count(x => x >= lo && (i == bins - 1 || x < hi))
    val bar = "█" * count
    println(f"  [$lo%.1f-$hi%.1f) $bar ($count)")
  }
  
  // ==============================
  // 3. Graph as Adjacency List
  // ==============================
  println("\n--- Graph BFS/DFS ---")
  
  val graph = Map(
    "A" -> List("B", "C"),
    "B" -> List("A", "D", "E"),
    "C" -> List("A", "F"),
    "D" -> List("B"),
    "E" -> List("B", "F"),
    "F" -> List("C", "E")
  )
  
  def bfs(graph: Map[String, List[String]], start: String): List[String] = {
    var visited = Set.empty[String]
    var queue = List(start)
    var result = List.empty[String]
    
    while (queue.nonEmpty) {
      val node = queue.head
      queue = queue.tail
      if (!visited.contains(node)) {
        visited = visited + node
        result = result :+ node
        val neighbors = graph.getOrElse(node, Nil).filterNot(visited.contains)
        queue = queue ++ neighbors
      }
    }
    result
  }
  
  def dfs(graph: Map[String, List[String]], start: String): List[String] = {
    var visited = Set.empty[String]
    var stack = List(start)
    var result = List.empty[String]
    
    while (stack.nonEmpty) {
      val node = stack.head
      stack = stack.tail
      if (!visited.contains(node)) {
        visited = visited + node
        result = result :+ node
        val neighbors = graph.getOrElse(node, Nil).filterNot(visited.contains)
        stack = neighbors ++ stack
      }
    }
    result
  }
  
  println(s"BFS from A: ${bfs(graph, "A").mkString(" → ")}")
  println(s"DFS from A: ${dfs(graph, "A").mkString(" → ")}")
  
  println("\n=== Collections Demo Complete ===")
```

---

## สรุป Part 06

| Step | สิ่งที่เรียน |
|------|-------------|
| 51 | Arrays — creation, access, operations |
| 52 | List — immutable linked list |
| 53 | Vector — indexed immutable |
| 54 | Range — lazy number sequences |
| 55 | Collection methods หลัก |
| 56 | Sorting |
| 57 | Collection conversions |
| 58 | LazyList |
| 59 | Performance characteristics |
| 60 | โปรแกรม complete (matrix, stats, graph) |

## แบบฝึกหัด

1. implement `mergeSort` ด้วย List
2. สร้าง `sliding average` function สำหรับ time series data
3. เขียน `topN(n: Int, items: List[T], score: T => Int)` ที่ return top N items
4. สร้าง `histogram(data: List[Double], bins: Int): Map[Range, Int]`
5. implement Dijkstra's shortest path algorithm บน graph

## ต่อไป

**[Part 07 →](part-07-tuples-and-option.md)** — Tuples, Option และ Either
