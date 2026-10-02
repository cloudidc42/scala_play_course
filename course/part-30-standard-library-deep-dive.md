# Part 30: Standard Library Deep Dive

## Steps 291-300: Collections API, String Operations, Date/Time, Math, Regex, I/O, Concurrency Primitives

---

## Step 291: Collections Operations ลึก

```scala
object CollectionsDeepDive extends App {
  
  val nums = (1 to 20).toList
  
  println("=== Advanced Collection Operations ===")
  
  // partition
  val (evens, odds) = nums.partition(_ % 2 == 0)
  println(s"evens: $evens")
  println(s"odds: $odds")
  
  // span — split at first non-matching
  val (small, rest) = nums.span(_ < 10)
  println(s"small: $small")
  println(s"rest: $rest")
  
  // takeWhile / dropWhile
  val increasing = List(1, 2, 3, 5, 4, 6, 7)
  println(s"takeWhile <4: ${increasing.takeWhile(_ < 4)}")
  println(s"dropWhile <4: ${increasing.dropWhile(_ < 4)}")
  
  // groupBy
  val byMod3 = nums.groupBy(_ % 3)
  println(s"\ngroupBy mod3: $byMod3")
  
  // flatten vs flatMap
  val nested = List(List(1, 2), List(3, 4), List(5))
  println(s"\nnested.flatten: ${nested.flatten}")
  println(s"flatMap: ${nested.flatMap(l => l.map(_ * 2))}")
  
  // zip, unzip, zipWithIndex
  val names = List("Alice", "Bob", "Carol")
  val scores = List(95.0, 87.5, 92.0)
  
  val zipped = names.zip(scores)
  println(s"\nzipped: $zipped")
  
  val (ns, ss) = zipped.unzip
  println(s"unzipped names: $ns")
  println(s"unzipped scores: $ss")
  
  names.zipWithIndex.foreach { case (name, i) =>
    println(s"  $i: $name")
  }
  
  // zipAll — handles different lengths
  val short = List(1, 2, 3)
  val long2 = List("a", "b", "c", "d", "e")
  println(s"\nzipAll: ${short.zipAll(long2, 0, "-")}")
  
  // scan — like fold but keeps intermediates
  val running = nums.take(5).scanLeft(0)(_ + _)
  println(s"\nscanLeft (running sum): $running")
  
  // collect — map + filter in one
  val mixed: List[Any] = List(1, "two", 3, "four", 5, "six")
  val strings = mixed.collect { case s: String => s.toUpperCase }
  val ints = mixed.collect { case n: Int => n * 10 }
  println(s"\ncollect strings: $strings")
  println(s"collect ints: $ints")
  
  // foldLeft vs foldRight
  val words = List("Hello", "Functional", "World")
  val folded = words.foldLeft("")((acc, w) => if (acc.isEmpty) w else s"$acc $w")
  val folded2 = words.foldRight("")((w, acc) => if (acc.isEmpty) w else s"$w $acc")
  println(s"\nfoldLeft: $folded")
  println(s"foldRight: $folded2")
  
  // aggregate — parallel-friendly fold
  val sums = (1 to 100).par.aggregate(0)(_ + _, _ + _)
  println(s"\nparallel aggregate sum: $sums")
  
  // maxBy, minBy, sortBy
  case class Student(name: String, gpa: Double, year: Int)
  val students = List(
    Student("Alice", 3.9, 3), Student("Bob", 3.5, 2),
    Student("Carol", 3.7, 4), Student("Dave", 3.8, 1)
  )
  
  println(s"\n=== Student Operations ===")
  println(s"maxBy GPA: ${students.maxBy(_.gpa)}")
  println(s"minBy GPA: ${students.minBy(_.gpa)}")
  println(s"sortBy year: ${students.sortBy(_.year).map(_.name)}")
  println(s"sortBy (-gpa, name): ${students.sortBy(s => (-s.gpa, s.name)).map(_.name)}")
  
  // distinct, distinctBy
  val dupes = List(1, 2, 2, 3, 3, 3, 4)
  println(s"\ndistinct: ${dupes.distinct}")
  
  val students2 = List(
    Student("Alice", 3.9, 3), Student("Bob", 3.5, 3),
    Student("Carol", 3.7, 2), Student("Dave", 3.8, 2)
  )
  // Keep first student per year
  println(s"distinctBy year: ${students2.distinctBy(_.year).map(_.name)}")
}
```

---

## Step 292: String Operations

```scala
object StringOperations extends App {
  
  val s = "  Hello, Scala World!  "
  
  println("=== String Operations ===")
  println(s"trim: '${s.trim}'")
  println(s"toLowerCase: '${s.trim.toLowerCase}'")
  println(s"toUpperCase: '${s.trim.toUpperCase}'")
  println(s"length: ${s.trim.length}")
  println(s"reverse: '${s.trim.reverse}'")
  
  // Split and join
  val csv = "Alice,30,Bangkok,Thailand"
  val parts = csv.split(",")
  println(s"\nsplit: ${parts.toList}")
  println(s"join: ${parts.mkString(" | ")}")
  
  // Contains, startsWith, endsWith
  val str = "Hello World"
  println(s"\ncontains 'World': ${str.contains("World")}")
  println(s"startsWith 'Hello': ${str.startsWith("Hello")}")
  println(s"endsWith 'World': ${str.endsWith("World")}")
  
  // Replace
  println(s"\nreplace: ${str.replace("World", "Scala")}")
  println(s"replaceAll: ${"hello123world".replaceAll("[0-9]+", "#")}")
  println(s"replaceFirst: ${"aababab".replaceFirst("ab", "XX")}")
  
  // Index operations
  println(s"\nindexOf('o'): ${str.indexOf('o')}")
  println(s"lastIndexOf('o'): ${str.lastIndexOf('o')}")
  println(s"substring(6): ${str.substring(6)}")
  println(s"substring(6,11): ${str.substring(6, 11)}")
  
  // Char operations
  println(s"\nchars: ${str.toCharArray.take(5).mkString(", ")}")
  println(s"head: ${str.head}")
  println(s"last: ${str.last}")
  println(s"filter letters: ${str.filter(_.isLetter)}")
  println(s"count spaces: ${str.count(_ == ' ')}")
  
  // String interpolation
  val name = "Alice"
  val age = 30
  val pi = Math.PI
  println(s"\ns: $name is $age years old")
  println(f"f: pi = $pi%.4f")
  println(raw"raw: \n is newline")  // \n not processed
  
  // String builder
  val sb = new StringBuilder
  (1 to 5).foreach { i =>
    if (sb.nonEmpty) sb.append(", ")
    sb.append(s"item-$i")
  }
  println(s"\nStringBuilder: ${sb.toString}")
  
  // Useful string utilities
  def wordFrequency(text: String): Map[String, Int] = {
    text.toLowerCase
      .replaceAll("[^a-z\\s]", "")
      .split("\\s+")
      .filter(_.nonEmpty)
      .groupBy(identity)
      .map { case (word, occurrences) => word -> occurrences.length }
  }
  
  val text = "the quick brown fox jumps over the lazy dog the fox"
  val freq = wordFrequency(text)
  println(s"\n=== Word Frequency ===")
  freq.toList.sortBy(-_._2).take(5).foreach { case (w, n) => println(s"  '$w': $n") }
  
  // String comparison
  println(s"\n=== Comparison ===")
  println(s"'abc' compareTo 'abd': ${"abc" compareTo "abd"}")
  println(s"'ABC' equalsIgnoreCase 'abc': ${"ABC".equalsIgnoreCase("abc")}")
  println(s"'abc' matches '^[a-z]+$$': ${"abc".matches("^[a-z]+$")}")
}
```

---

## Step 293: Date and Time

```scala
import java.time._
import java.time.format.DateTimeFormatter
import java.time.temporal.ChronoUnit

object DateTimeOps extends App {
  
  // Modern Java time API — immutable, thread-safe
  val now = LocalDateTime.now()
  val today = LocalDate.now()
  val time = LocalTime.now()
  
  println("=== Date and Time ===")
  println(s"now: $now")
  println(s"today: $today")
  println(s"time: $time")
  
  // Formatting
  val formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm:ss")
  val thaiFormatter = DateTimeFormatter.ofPattern("dd MMMM yyyy", java.util.Locale.forLanguageTag("th"))
  
  println(s"\nFormatted: ${now.format(formatter)}")
  
  // Parsing
  val dateStr = "25/12/2024"
  val parsed = LocalDate.parse(dateStr, DateTimeFormatter.ofPattern("dd/MM/yyyy"))
  println(s"Parsed: $parsed")
  
  // Date arithmetic
  val nextWeek = today.plusDays(7)
  val lastMonth = today.minusMonths(1)
  val nextYear = today.plusYears(1)
  
  println(s"\nnextWeek: $nextWeek")
  println(s"lastMonth: $lastMonth")
  println(s"nextYear: $nextYear")
  
  // Differences
  val birthday = LocalDate.of(1990, 6, 15)
  val yearsOld = ChronoUnit.YEARS.between(birthday, today)
  val daysOld = ChronoUnit.DAYS.between(birthday, today)
  
  println(s"\n=== Age Calculation ===")
  println(s"Birthday: $birthday")
  println(s"Years old: $yearsOld")
  println(s"Days old: $daysOld")
  
  // Period and Duration
  val period = Period.between(birthday, today)
  println(s"Period: ${period.getYears}y ${period.getMonths}m ${period.getDays}d")
  
  val start = LocalDateTime.of(2024, 1, 1, 9, 0)
  val end = LocalDateTime.of(2024, 1, 1, 17, 30)
  val duration = Duration.between(start, end)
  println(s"\nWork duration: ${duration.toHours}h ${duration.toMinutesPart}m")
  
  // Comparison
  val d1 = LocalDate.of(2024, 3, 15)
  val d2 = LocalDate.of(2024, 6, 20)
  println(s"\n$d1 isBefore $d2: ${d1.isBefore(d2)}")
  println(s"$d1 isAfter $d2: ${d1.isAfter(d2)}")
  
  // Day of week, month name
  println(s"\nDay of week: ${today.getDayOfWeek}")
  println(s"Month: ${today.getMonth}")
  println(s"Quarter: ${today.get(java.time.temporal.IsoFields.QUARTER_OF_YEAR)}")
  
  // Time zones
  val bangkokZone = ZoneId.of("Asia/Bangkok")
  val londonZone = ZoneId.of("Europe/London")
  val nyZone = ZoneId.of("America/New_York")
  
  val bangkokTime = ZonedDateTime.now(bangkokZone)
  val londonTime = bangkokTime.withZoneSameInstant(londonZone)
  val nyTime = bangkokTime.withZoneSameInstant(nyZone)
  
  val tz = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm z")
  println(s"\n=== Time Zones ===")
  println(s"Bangkok: ${bangkokTime.format(tz)}")
  println(s"London:  ${londonTime.format(tz)}")
  println(s"New York: ${nyTime.format(tz)}")
}
```

---

## Step 294-300: Math, Regex, I/O, Random, Collections Utilities

```scala
import scala.util.Random
import scala.util.matching.Regex

object StandardLibMisc extends App {
  
  // Math operations
  println("=== Math Operations ===")
  println(s"abs(-5): ${Math.abs(-5)}")
  println(s"ceil(3.2): ${Math.ceil(3.2)}")
  println(s"floor(3.9): ${Math.floor(3.9)}")
  println(s"round(3.5): ${Math.round(3.5)}")
  println(s"pow(2,10): ${Math.pow(2, 10)}")
  println(s"sqrt(144): ${Math.sqrt(144)}")
  println(s"log(E): ${Math.log(Math.E)}")
  println(s"log10(1000): ${Math.log10(1000)}")
  println(s"sin(PI/2): ${Math.sin(Math.PI / 2)}")
  
  // Min/Max/Clamp
  def clamp(value: Double, min: Double, max: Double): Double =
    Math.min(Math.max(value, min), max)
  
  println(s"clamp(150, 0, 100): ${clamp(150, 0, 100)}")
  println(s"clamp(-10, 0, 100): ${clamp(-10, 0, 100)}")
  println(s"clamp(50, 0, 100): ${clamp(50, 0, 100)}")
  
  // Random
  println("\n=== Random ===")
  val rng = new Random(42)  // Seeded for reproducibility
  
  println(s"nextInt(100): ${rng.nextInt(100)}")
  println(s"nextDouble: ${rng.nextDouble()}")
  println(s"nextBoolean: ${rng.nextBoolean()}")
  
  val shuffled = rng.shuffle(List(1, 2, 3, 4, 5))
  println(s"shuffle: $shuffled")
  
  val sample = rng.shuffle((1 to 100).toList).take(5)
  println(s"sample 5 from 1-100: $sample")
  
  // Weighted random choice
  def weightedChoice[A](choices: List[(A, Double)], rng: Random = new Random): A = {
    val total = choices.map(_._2).sum
    val threshold = rng.nextDouble() * total
    choices.foldLeft((threshold, choices.head._1)) { case ((remaining, current), (item, weight)) =>
      if (remaining <= 0) (remaining, current)
      else if (remaining <= weight) (0.0, item)
      else (remaining - weight, item)
    }._2
  }
  
  val rng2 = new Random(42)
  val choices = List(("common", 0.6), ("uncommon", 0.3), ("rare", 0.1))
  val counts = (1 to 1000).map(_ => weightedChoice(choices, rng2)).groupBy(identity).map { case (k, v) => k -> v.size }
  println(s"\nWeighted random (1000 trials): $counts")
  
  // Regex
  println("\n=== Regex ===")
  val ipRegex: Regex = "(\\d{1,3})\\.(\\d{1,3})\\.(\\d{1,3})\\.(\\d{1,3})".r
  val logLine = "2024-01-15 ERROR 192.168.1.100 Failed to connect"
  
  ipRegex.findFirstMatchIn(logLine) match {
    case Some(m) => println(s"Found IP: ${m.group(0)}")
    case None    => println("No IP found")
  }
  
  val text = "IPs: 192.168.1.1, 10.0.0.1, 172.16.0.5"
  val allIPs = ipRegex.findAllIn(text).toList
  println(s"All IPs: $allIPs")
  
  // Replace with regex
  val cleaned = "Hello   World  Scala".replaceAll("\\s+", " ")
  println(s"Cleaned: '$cleaned'")
  
  // Extract groups
  val dateRegex = "(\\d{4})-(\\d{2})-(\\d{2})".r
  val dates = List("2024-01-15", "2023-12-31", "not-a-date")
  dates.foreach {
    case dateRegex(year, month, day) => println(s"  Date: $year/$month/$day")
    case other => println(s"  Not a date: $other")
  }
  
  // Collections utility operations
  println("\n=== Collections Utilities ===")
  
  // View — lazy collection operations
  val lazyOps = (1 to 1000000).view
    .filter(_ % 2 == 0)
    .map(_ * 3)
    .take(5)
    .toList
  println(s"Lazy view: $lazyOps")  // Only computes 5 elements
  
  // Iterator operations
  val iter = Iterator.from(1).filter(_ % 7 == 0).take(5)
  println(s"Multiples of 7: ${iter.toList}")
  
  // Range operations
  val range = 1 to 100 by 7
  println(s"Range by 7: ${range.toList}")
  println(s"Sum 1-100: ${(1 to 100).sum}")
  
  // Mutable collections
  import scala.collection.mutable.{ArrayBuffer, HashMap => MutableMap, PriorityQueue}
  
  val buffer = ArrayBuffer(1, 2, 3)
  buffer += 4
  buffer ++= List(5, 6)
  buffer.remove(0)
  println(s"\nArrayBuffer: $buffer")
  
  val mMap = MutableMap[String, Int]()
  mMap("a") = 1; mMap("b") = 2; mMap("c") = 3
  mMap.update("a", 10)
  println(s"MutableMap: $mMap")
  
  // PriorityQueue (max heap by default)
  val pq = PriorityQueue(3, 1, 4, 1, 5, 9, 2, 6)
  println(s"PriorityQueue max: ${pq.dequeue()}")
  println(s"Next max: ${pq.dequeue()}")
  
  // Min heap
  val minPQ = PriorityQueue(3, 1, 4, 1, 5, 9)(Ordering[Int].reverse)
  println(s"Min PQ: ${minPQ.dequeue()}")
  
  // I/O
  println("\n=== I/O Operations ===")
  
  import java.io.{File, PrintWriter}
  import scala.io.Source
  
  // Write to temp file
  val tmpFile = File.createTempFile("scala-test", ".txt")
  tmpFile.deleteOnExit()
  
  val writer = new PrintWriter(tmpFile)
  try {
    writer.println("Line 1: Hello Scala")
    writer.println("Line 2: Functional Programming")
    writer.println("Line 3: Type System")
  } finally {
    writer.close()
  }
  
  // Read from file
  val lines = Source.fromFile(tmpFile).getLines().toList
  println(s"Read ${lines.size} lines:")
  lines.foreach(l => println(s"  $l"))
  
  // Read with auto-close
  val content = Using(Source.fromFile(tmpFile))(_.mkString).getOrElse("")
  println(s"\nContent length: ${content.length} chars")
}

// Using — auto-close resource
object Using {
  def apply[R <: java.io.Closeable, A](resource: R)(f: R => A): scala.util.Try[A] = {
    scala.util.Try {
      try f(resource) finally resource.close()
    }
  }
}
```

---

## สรุป Part 30

| Category | Key Classes | ใช้เมื่อ |
|----------|------------|---------|
| Collections | List, Vector, Map, Set, LazyList | Data processing |
| Mutable | ArrayBuffer, HashMap, PriorityQueue | Performance-critical |
| String | String, StringBuilder, StringOps | Text processing |
| Date/Time | LocalDate, LocalDateTime, ZonedDateTime | Temporal data |
| Math | Math, BigDecimal, BigInt | Numeric computation |
| Random | Random, SecureRandom | Randomness |
| Regex | Regex, Pattern | Text matching |
| I/O | Source, File, PrintWriter | File operations |
| Concurrent | Future, Promise, Atomic | Async operations |

---

## แบบฝึกหัด Part 30

**ข้อ 1:** Implement `Statistics` class ด้วย `mean`, `median`, `mode`, `stddev`, `percentile`

**ข้อ 2:** สร้าง `DateRange` class ที่ support `contains`, `overlaps`, `union`, `intersection`

**ข้อ 3:** Implement log parser ที่ read structured log lines และสร้าง summary report

**ข้อ 4:** สร้าง `TemplateEngine` ที่ interpolate variables ใน string templates ด้วย Regex

**ข้อ 5:** Implement CSV reader/writer ที่ handle quoted fields, escape characters, different delimiters

---

➡️ ต่อไป: [Part 31 — Futures and Promises](part-31-futures-and-promises.md)
