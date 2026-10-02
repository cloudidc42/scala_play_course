# Part 21: List และ Vector Deep Dive

## Steps 201-210: List, Vector, Performance, Immutable Operations, Structural Sharing

---

## Step 201: List — Linked List พื้นฐาน

```scala
object ListDeepDive extends App {
  
  // List = singly-linked list (head :: tail)
  val list1 = List(1, 2, 3, 4, 5)
  val list2 = 1 :: 2 :: 3 :: Nil  // Same as above
  val list3 = 0 :: list1           // Prepend O(1)
  
  println("=== List Basics ===")
  println(s"list1: $list1")
  println(s"list2: $list2")
  println(s"list3: $list3")
  println(s"head: ${list1.head}")
  println(s"tail: ${list1.tail}")
  println(s"init: ${list1.init}")  // All but last
  println(s"last: ${list1.last}")  // O(n)!
  
  // Pattern matching on List
  def describeList[A](list: List[A]): String = list match {
    case Nil         => "empty"
    case _ :: Nil    => "single element"
    case _ :: _ :: Nil => "two elements"
    case h :: t      => s"starts with $h, ${t.size} more"
  }
  
  println(s"\n${describeList(List.empty[Int])}")
  println(describeList(List(42)))
  println(describeList(List(1, 2)))
  println(describeList(list1))
  
  // Important: List operations complexity
  // prepend (::)        O(1)   ✓
  // head                O(1)   ✓
  // tail                O(1)   ✓
  // last                O(n)   slow!
  // append (:+)         O(n)   slow!
  // length              O(n)   slow!
  // apply/index         O(n)   slow!
  // reverse             O(n)
  
  // Building list efficiently (prepend then reverse)
  def buildList(n: Int): List[Int] = {
    var acc: List[Int] = Nil
    var i = 0
    while (i < n) {
      acc = i :: acc  // Prepend O(1)
      i += 1
    }
    acc.reverse  // O(n) at end
  }
  
  println(s"\nBuilt list: ${buildList(10)}")
  
  // List operations
  val numbers = List(3, 1, 4, 1, 5, 9, 2, 6, 5, 3)
  
  println(s"\n=== List Operations ===")
  println(s"distinct: ${numbers.distinct}")
  println(s"sorted: ${numbers.sorted}")
  println(s"sortedDesc: ${numbers.sorted.reverse}")
  println(s"grouped by 3: ${numbers.grouped(3).toList}")
  println(s"sliding(3): ${numbers.sliding(3).toList}")
  println(s"splitAt(5): ${numbers.splitAt(5)}")
  println(s"take(3): ${numbers.take(3)}")
  println(s"drop(7): ${numbers.drop(7)}")
  
  // Combinations and permutations
  val small = List(1, 2, 3)
  println(s"\ncombinations(2): ${small.combinations(2).toList}")
  println(s"permutations: ${small.permutations.toList}")
}
```

---

## Step 202: Vector — Efficient Random Access

```scala
object VectorDeepDive extends App {
  
  // Vector = balanced tree, O(log n) for most operations
  // Random access: O(effectively constant) due to tree depth
  
  val vec1 = Vector(1, 2, 3, 4, 5)
  val vec2 = vec1 :+ 6       // Append O(effectively constant)
  val vec3 = 0 +: vec1       // Prepend O(effectively constant)
  val updated = vec1.updated(2, 99)  // Update at index O(log n)
  
  println("=== Vector Operations ===")
  println(s"vec1: $vec1")
  println(s"appended: $vec2")
  println(s"prepended: $vec3")
  println(s"updated index 2: $updated")
  println(s"vec1(2): ${vec1(2)}")  // O(effectively constant)
  
  // Vector vs List performance
  val N = 100000
  
  // List append is slow
  val listStart = System.nanoTime()
  var list: List[Int] = Nil
  for (i <- 1 to 1000) list = list :+ i  // O(n) each time!
  val listTime = System.nanoTime() - listStart
  
  // Vector append is fast
  val vecStart = System.nanoTime()
  var vec: Vector[Int] = Vector.empty
  for (i <- 1 to N) vec = vec :+ i  // O(log n) each time
  val vecTime = System.nanoTime() - vecStart
  
  println(s"\n=== Performance ===")
  println(f"List 1000 appends: ${listTime / 1000000.0}%.2f ms")
  println(f"Vector $N appends: ${vecTime / 1000000.0}%.2f ms")
  
  // Random access
  val bigVec = Vector.tabulate(N)(i => i * i)
  val bigList = bigVec.toList
  
  val vecAccessStart = System.nanoTime()
  val _ = bigVec(N / 2)  // O(effectively constant)
  val vecAccessTime = System.nanoTime() - vecAccessStart
  
  val listAccessStart = System.nanoTime()
  val _ = bigList(N / 2)  // O(n)
  val listAccessTime = System.nanoTime() - listAccessStart
  
  println(f"Vector random access: ${vecAccessTime}ns")
  println(f"List random access: ${listAccessTime}ns")
  
  // When to use what
  // List:   sequential processing, prepend-heavy, pattern matching
  // Vector: random access, append/update-heavy, indexed operations
  
  // Functional operations work the same on both
  val result = vec1
    .filter(_ % 2 != 0)
    .map(_ * 10)
    .take(2)
  
  println(s"\n=== Vector Functional Operations ===")
  println(s"filter odd, times 10, take 2: $result")
  
  // Converting between
  val fromList = List(1,2,3,4,5).toVector
  val fromVec = Vector(6,7,8).toList
  println(s"fromList: $fromList (${fromList.getClass.getSimpleName})")
  println(s"fromVec: $fromVec (${fromVec.getClass.getSimpleName})")
}
```

---

## Step 203: Structural Sharing

```scala
object StructuralSharing extends App {
  
  // List structural sharing - nodes are shared
  val tail = List(2, 3, 4, 5)
  val list1 = 1 :: tail  // Shares tail
  val list2 = 10 :: tail  // Also shares tail
  
  // Both list1 and list2 share the same tail object in memory
  println("=== Structural Sharing ===")
  println(s"list1: $list1")
  println(s"list2: $list2")
  println(s"tail identical: ${list1.tail eq list2.tail}")  // true - same object!
  
  // This is why prepend is O(1) and memory-efficient
  
  // Building a persistent data structure
  case class PersistentList[A](elements: List[A]) {
    def prepend(a: A): PersistentList[A] = PersistentList(a :: elements)
    def take(n: Int): PersistentList[A] = PersistentList(elements.take(n))
    def versions: Int = 0  // In real implementation, would track
    
    override def toString: String = s"PList(${elements.mkString(", ")})"
  }
  
  val v1 = PersistentList(List(3, 4, 5))
  val v2 = v1.prepend(2)  // Shares v1's list
  val v3 = v2.prepend(1)  // Shares v2's list
  
  println(s"\nv1: $v1")
  println(s"v2: $v2")
  println(s"v3: $v3")
  // v1 still exists and is unchanged - true persistence
  
  // Use case: history/undo
  class TextEditor {
    private var history: List[String] = List.empty
    private var current: String = ""
    
    def type_(char: Char): TextEditor = {
      history = current :: history  // Push current to history (O(1))
      current = current + char
      this
    }
    
    def undo(): TextEditor = history match {
      case Nil     => this
      case h :: t  => current = h; history = t; this
    }
    
    def text: String = current
    def historyDepth: Int = history.size
  }
  
  println("\n=== Text Editor with History ===")
  val editor = new TextEditor()
  editor.type_('H').type_('e').type_('l').type_('l').type_('o')
  println(s"Text: ${editor.text}")
  println(s"History depth: ${editor.historyDepth}")
  
  editor.undo().undo()
  println(s"After 2 undos: ${editor.text}")
}
```

---

## Step 204-210: Advanced List Operations and Real-World Patterns

```scala
object AdvancedListOps extends App {
  
  // Sliding window analysis
  case class StockPrice(date: String, price: Double)
  
  val prices = List(
    StockPrice("2024-01-01", 100.0),
    StockPrice("2024-01-02", 105.0),
    StockPrice("2024-01-03", 103.0),
    StockPrice("2024-01-04", 108.0),
    StockPrice("2024-01-05", 112.0),
    StockPrice("2024-01-06", 110.0),
    StockPrice("2024-01-07", 115.0),
    StockPrice("2024-01-08", 113.0),
    StockPrice("2024-01-09", 118.0),
    StockPrice("2024-01-10", 120.0)
  )
  
  // Moving average
  def movingAverage(data: List[Double], window: Int): List[Double] = {
    data.sliding(window).map(w => w.sum / w.size).toList
  }
  
  // Price differences
  def dailyReturns(data: List[Double]): List[Double] = {
    data.zip(data.tail).map { case (prev, curr) => (curr - prev) / prev * 100 }
  }
  
  val priceValues = prices.map(_.price)
  val ma3 = movingAverage(priceValues, 3)
  val returns = dailyReturns(priceValues)
  
  println("=== Stock Analysis ===")
  println(s"3-day MA: ${ma3.map(v => f"$v%.2f")}")
  println(s"Daily returns %: ${returns.map(v => f"$v%.2f%")}")
  println(f"Avg return: ${returns.sum / returns.size}%.2f%%")
  
  // Group consecutive elements
  def groupConsecutive[A](list: List[A])(pred: (A, A) => Boolean): List[List[A]] = {
    list.foldRight(List.empty[List[A]]) { (elem, acc) =>
      acc match {
        case Nil => List(List(elem))
        case head :: tail if head.nonEmpty && pred(elem, head.head) =>
          (elem :: head) :: tail
        case _ => List(elem) :: acc
      }
    }
  }
  
  val nums = List(1, 2, 3, 5, 6, 7, 10, 11, 15)
  val consecutive = groupConsecutive(nums)((a, b) => b - a == 1)
  println(s"\nConsecutive groups: $consecutive")
  
  // Transpose
  val matrix = List(
    List(1, 2, 3),
    List(4, 5, 6),
    List(7, 8, 9)
  )
  println(s"\nOriginal: $matrix")
  println(s"Transposed: ${matrix.transpose}")
  
  // Interleave two lists
  def interleave[A](xs: List[A], ys: List[A]): List[A] = (xs, ys) match {
    case (Nil, _) => ys
    case (_, Nil) => xs
    case (h1 :: t1, h2 :: t2) => h1 :: h2 :: interleave(t1, t2)
  }
  
  println(s"\nInterleave: ${interleave(List(1,3,5), List(2,4,6))}")
  
  // Chunk list into groups of n
  val chunked = (1 to 10).toList.grouped(3).toList
  println(s"Chunked by 3: $chunked")
  
  // Real-world: batch processing
  case class ApiItem(id: Int, data: String)
  
  def processItems(items: List[ApiItem], batchSize: Int): List[List[ApiItem]] = {
    items.grouped(batchSize).toList
  }
  
  val items = (1 to 25).map(i => ApiItem(i, s"data-$i")).toList
  val batches = processItems(items, 5)
  
  println(s"\n=== Batch Processing ===")
  println(s"Total items: ${items.size}")
  println(s"Batches: ${batches.size}")
  batches.zipWithIndex.foreach { case (batch, idx) =>
    println(s"  Batch ${idx + 1}: IDs ${batch.map(_.id).mkString(", ")}")
  }
}
```

---

## สรุป Part 21

| Collection | Access | Prepend | Append | Use When |
|-----------|--------|---------|--------|----------|
| `List[A]` | O(n) | O(1) | O(n) | Sequential processing |
| `Vector[A]` | O(log n)* | O(log n)* | O(log n)* | Random access, balanced ops |
| `LazyList[A]` | lazy | O(1) | lazy | Infinite sequences |
| `Array[A]` | O(1) | O(n) | O(n) | Mutable, Java interop |

*Vector operations are effectively O(1) due to small constant factor from 32-way branching

---

## แบบฝึกหัด Part 21

**ข้อ 1:** Implement `groupBy` ด้วย `foldLeft` ที่ return `Map[K, List[A]]`

**ข้อ 2:** สร้าง pagination function: `paginate(items, pageSize, pageNum): (List[A], totalPages)`

**ข้อ 3:** Implement `windowed` function ที่สร้าง all overlapping sub-windows ของ size n

**ข้อ 4:** สร้าง efficient ring buffer ด้วย Vector ที่ keep last N items

**ข้อ 5:** Implement merge sort ด้วย structural sharing ที่ reuses input list nodes where possible

---

➡️ ต่อไป: [Part 22 — Map and Set](part-22-map-and-set.md)
