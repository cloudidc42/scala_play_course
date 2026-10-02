# Part 22: Map และ Set Operations

## Steps 211-220: HashMap, TreeMap, LinkedHashMap, Set Operations, SortedMap

---

## Step 211: Map — พื้นฐาน

```scala
object MapBasics extends App {
  
  // Creating maps
  val map1 = Map("alice" -> 30, "bob" -> 25, "carol" -> 35)
  val map2 = Map.empty[String, Int]
  val map3 = Map(1 -> "one", 2 -> "two", 3 -> "three")
  
  println("=== Map Basics ===")
  println(s"map1: $map1")
  
  // Access
  println(s"alice: ${map1("alice")}")        // Throws if missing
  println(s"safe: ${map1.get("alice")}")     // Option[Int]
  println(s"default: ${map1.getOrElse("dave", 0)}")
  println(s"contains: ${map1.contains("bob")}")
  
  // Operations - all return NEW maps
  val added = map1 + ("dave" -> 28)
  val removed = map1 - "bob"
  val updated = map1.updated("alice", 31)
  val merged = map1 ++ Map("eve" -> 22, "alice" -> 99)  // alice updated
  
  println(s"\nadded: $added")
  println(s"removed: $removed")
  println(s"updated: $updated")
  println(s"merged: $merged")
  println(s"original unchanged: $map1")
  
  // Transformation
  val doubled = map1.map { case (k, v) => k -> v * 2 }
  val filtered = map1.filter { case (_, age) => age >= 30 }
  val mapped = map1.mapValues(_ + 1)
  
  println(s"\ndoubled: $doubled")
  println(s"filtered: $filtered")
  println(s"mapValues: $mapped")
  
  // Fold over map
  val totalAge = map1.foldLeft(0) { case (acc, (_, age)) => acc + age }
  println(s"total age: $totalAge")
  
  // Grouping
  val people = List(
    ("Alice", "Engineering"),
    ("Bob", "Marketing"),
    ("Carol", "Engineering"),
    ("David", "HR"),
    ("Eve", "Marketing")
  )
  
  val byDept = people.groupBy(_._2).map { case (dept, members) =>
    dept -> members.map(_._1)
  }
  
  println(s"\nBy department: $byDept")
}
```

---

## Step 212: TreeMap, LinkedHashMap, SortedMap

```scala
import scala.collection.immutable.{TreeMap, SortedMap}
import scala.collection.mutable.{LinkedHashMap => MutableLinkedHashMap}

object MapVariants extends App {
  
  // HashMap (default) - no guaranteed order
  val hashMap = Map("c" -> 3, "a" -> 1, "b" -> 2)
  println(s"HashMap: $hashMap")  // Order may vary
  
  // TreeMap - sorted by key
  val treeMap = TreeMap("c" -> 3, "a" -> 1, "b" -> 2)
  println(s"TreeMap: $treeMap")  // Always: a->1, b->2, c->3
  
  // SortedMap with custom ordering
  val byLengthThenAlpha = Ordering.by((s: String) => (s.length, s))
  val sortedByLength = TreeMap("banana" -> 2, "apple" -> 1, "kiwi" -> 3, "fig" -> 4)(byLengthThenAlpha)
  println(s"Sorted by length: $sortedByLength")
  
  // LinkedHashMap (mutable) - insertion order
  val linked = MutableLinkedHashMap[String, Int]()
  linked("c") = 3
  linked("a") = 1
  linked("b") = 2
  println(s"LinkedHashMap: $linked")  // Insertion order: c,a,b
  
  // SortedMap range operations
  val scores = TreeMap(
    "Alice" -> 92,
    "Bob" -> 85,
    "Carol" -> 88,
    "David" -> 79,
    "Eve" -> 95,
    "Frank" -> 83
  )
  
  println(s"\n=== SortedMap Range Operations ===")
  println(s"From 'C': ${scores.from("C")}")
  println(s"To 'D': ${scores.to("D")}")  
  println(s"Range B-D: ${scores.range("B", "E")}")
  println(s"Head (first): ${scores.head}")
  println(s"Last: ${scores.last}")
  
  // Practical: Interval map (schedule)
  case class TimeSlot(hour: Int, description: String)
  
  val schedule = TreeMap(
    9  -> TimeSlot(9, "Stand-up meeting"),
    10 -> TimeSlot(10, "Development"),
    12 -> TimeSlot(12, "Lunch"),
    14 -> TimeSlot(14, "Code review"),
    16 -> TimeSlot(16, "Planning"),
    18 -> TimeSlot(18, "End of day")
  )
  
  def currentActivity(hour: Int): String = {
    schedule.to(hour).lastOption match {
      case Some((_, slot)) => slot.description
      case None => "Before work hours"
    }
  }
  
  println(s"\n=== Schedule ===")
  List(8, 9, 11, 13, 15, 17, 19).foreach { h =>
    println(s"  Hour $h: ${currentActivity(h)}")
  }
}
```

---

## Step 213: Set Operations

```scala
object SetOperations extends App {
  
  // Basic sets
  val s1 = Set(1, 2, 3, 4, 5)
  val s2 = Set(3, 4, 5, 6, 7)
  
  println("=== Set Operations ===")
  println(s"s1: $s1")
  println(s"s2: $s2")
  
  // Set algebra
  println(s"\nUnion (s1 | s2): ${s1 | s2}")      // 1,2,3,4,5,6,7
  println(s"Intersection (s1 & s2): ${s1 & s2}") // 3,4,5
  println(s"Difference (s1 &~ s2): ${s1 &~ s2}") // 1,2
  println(s"Diff (s2 &~ s1): ${s2 &~ s1}")       // 6,7
  
  // Subset/superset
  val small = Set(3, 4)
  println(s"\n$small subsetOf $s1: ${small.subsetOf(s1)}")
  println(s"$s1 supersetOf $small: ${s1.subsetOf(s1)}")
  
  // SortedSet
  import scala.collection.immutable.SortedSet
  val sorted = SortedSet(5, 2, 8, 1, 9, 3)
  println(s"\nSortedSet: $sorted")  // Always ordered
  
  // Practical: tag/permission system
  case class User(name: String, permissions: Set[String])
  
  val users = List(
    User("Alice", Set("read", "write", "delete", "admin")),
    User("Bob", Set("read", "write")),
    User("Carol", Set("read")),
    User("Dave", Set("read", "write", "approve"))
  )
  
  def canPerform(user: User, required: Set[String]): Boolean = 
    required.subsetOf(user.permissions)
  
  def hasAny(user: User, permissions: Set[String]): Boolean = 
    user.permissions.intersect(permissions).nonEmpty
  
  println("\n=== Permission System ===")
  val adminRequired = Set("read", "write", "admin")
  val writeRequired = Set("write")
  
  users.foreach { user =>
    val canAdmin = canPerform(user, adminRequired)
    val canWrite = canPerform(user, writeRequired)
    println(s"  ${user.name}: admin=$canAdmin, write=$canWrite")
  }
  
  // Find users with overlapping permissions
  def findUsersWithPermission(users: List[User], perm: String): List[User] = 
    users.filter(_.permissions.contains(perm))
  
  println(s"\nUsers who can 'write': ${findUsersWithPermission(users, "write").map(_.name)}")
  
  // Unique elements efficiently
  val items = List("apple", "banana", "apple", "cherry", "banana", "date")
  val unique = items.toSet
  val uniqueOrdered = items.foldLeft(Set.empty[String])(_ + _)  // Maintains some order
  println(s"\nUnique: $unique")
  println(s"Distinct (list order): ${items.distinct}")
}
```

---

## Step 214-220: Advanced Map Patterns

```scala
object AdvancedMapPatterns extends App {
  
  // Map as frequency counter
  def frequency[A](items: List[A]): Map[A, Int] = {
    items.foldLeft(Map.empty[A, Int]) { (acc, item) =>
      acc + (item -> (acc.getOrElse(item, 0) + 1))
    }
  }
  
  val words = "the quick brown fox jumps over the lazy dog the".split(" ").toList
  val wordFreq = frequency(words)
  
  println("=== Word Frequency ===")
  wordFreq.toList.sortBy(-_._2).take(5).foreach { case (word, count) =>
    println(s"  '$word': $count")
  }
  
  // Inverted index
  def buildInvertedIndex(documents: Map[String, String]): Map[String, Set[String]] = {
    documents.foldLeft(Map.empty[String, Set[String]]) { case (idx, (docId, content)) =>
      content.toLowerCase.split("\\W+").foldLeft(idx) { (acc, word) =>
        if (word.nonEmpty) acc + (word -> (acc.getOrElse(word, Set.empty) + docId))
        else acc
      }
    }
  }
  
  val documents = Map(
    "doc1" -> "Scala is a functional programming language",
    "doc2" -> "Java is an object-oriented language",
    "doc3" -> "Scala runs on the JVM like Java",
    "doc4" -> "Functional programming with Scala is powerful"
  )
  
  val index = buildInvertedIndex(documents)
  
  println("\n=== Inverted Index ===")
  List("scala", "java", "functional", "language").foreach { word =>
    println(s"  '$word' in: ${index.getOrElse(word, Set.empty)}")
  }
  
  // Multi-map pattern
  def multiMap[K, V](pairs: List[(K, V)]): Map[K, List[V]] = {
    pairs.foldLeft(Map.empty[K, List[V]]) { case (acc, (k, v)) =>
      acc + (k -> (acc.getOrElse(k, List.empty) :+ v))
    }
  }
  
  val orders = List(
    ("Alice", "O001"), ("Bob", "O002"), ("Alice", "O003"),
    ("Carol", "O004"), ("Bob", "O005"), ("Alice", "O006")
  )
  
  val userOrders = multiMap(orders)
  println(s"\n=== Multi-Map ===")
  userOrders.foreach { case (user, orders) =>
    println(s"  $user: ${orders.mkString(", ")}")
  }
  
  // Map merging strategies
  val mapA = Map("a" -> 1, "b" -> 2, "c" -> 3)
  val mapB = Map("b" -> 20, "c" -> 30, "d" -> 40)
  
  // Last wins
  val mergedLastWins = mapA ++ mapB
  // First wins
  val mergedFirstWins = mapB ++ mapA
  // Custom merge: sum values
  val mergedSum = (mapA.keySet ++ mapB.keySet).map { k =>
    k -> (mapA.getOrElse(k, 0) + mapB.getOrElse(k, 0))
  }.toMap
  
  println(s"\n=== Map Merging ===")
  println(s"Last wins: $mergedLastWins")
  println(s"First wins: $mergedFirstWins")
  println(s"Sum values: $mergedSum")
  
  // Pivot table
  case class Sale(region: String, product: String, amount: Double)
  
  val sales = List(
    Sale("North", "A", 100), Sale("North", "B", 200),
    Sale("South", "A", 150), Sale("South", "B", 250), Sale("South", "C", 100),
    Sale("East", "A", 80),   Sale("East", "C", 180),
    Sale("West", "B", 300),  Sale("West", "C", 220)
  )
  
  // Pivot: region -> product -> total
  val pivot = sales.groupBy(_.region).map { case (region, regionSales) =>
    region -> regionSales.groupBy(_.product).map { case (product, prodSales) =>
      product -> prodSales.map(_.amount).sum
    }
  }
  
  println("\n=== Pivot Table (Region x Product) ===")
  val products = sales.map(_.product).distinct.sorted
  val regions = pivot.keys.toList.sorted
  
  print(f"${"Region"}%-8s")
  products.foreach(p => print(f" $p%8s"))
  println(s" ${"Total"}%8s")
  println("-" * (8 + products.size * 9 + 9))
  
  regions.foreach { region =>
    val row = pivot(region)
    print(f"$region%-8s")
    var total = 0.0
    products.foreach { p =>
      val v = row.getOrElse(p, 0.0)
      total += v
      print(f" $v%8.0f")
    }
    println(f" $total%8.0f")
  }
}
```

---

## สรุป Part 22

| Type | Ordering | Lookup | Use When |
|------|----------|--------|----------|
| `Map[K,V]` | None (hash) | O(1) avg | General purpose |
| `TreeMap[K,V]` | Sorted | O(log n) | Need sorted keys |
| `SortedMap[K,V]` | Sorted | O(log n) | Range queries |
| `LinkedHashMap[K,V]` | Insertion | O(1) | Preserve order |
| `Set[A]` | None | O(1) avg | Membership test |
| `SortedSet[A]` | Sorted | O(log n) | Sorted unique |
| `TreeSet[A]` | Sorted | O(log n) | Efficient range |

---

## แบบฝึกหัด Part 22

**ข้อ 1:** สร้าง LRU Cache ด้วย `LinkedHashMap` ที่ evict oldest entry เมื่อ capacity เต็ม

**ข้อ 2:** Implement word concordance: สำหรับแต่ละคำใน text ให้แสดง list ของ line numbers ที่ปรากฏ

**ข้อ 3:** สร้าง multi-dimensional pivot table สำหรับ sales data: region × product × quarter

**ข้อ 4:** Implement trie data structure (prefix tree) ด้วย `Map[Char, TrieNode]`

**ข้อ 5:** สร้าง graph adjacency list representation ด้วย `Map[Node, Set[Node]]` พร้อม BFS/DFS traversal

---

➡️ ต่อไป: [Part 23 — Advanced Pattern Matching](part-23-pattern-matching-advanced.md)
