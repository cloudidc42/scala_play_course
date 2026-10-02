# Part 05 — Strings, String Interpolation และ I/O
## Steps 41–50: การจัดการ Text และ Input/Output

> **เป้าหมาย**: master string manipulation, interpolation ทุกรูปแบบ, file I/O, และ console I/O

---

## Step 41 — String Interpolation ลึก

```scala
val name = "Alice"
val age = 30
val score = 98.756

// s-interpolator: ใส่ variable
val s1 = s"Name: $name, Age: $age"
println(s1)  // Name: Alice, Age: 30

// Expression ใน ${}
val s2 = s"Next year: ${age + 1}"
val s3 = s"Upper: ${name.toUpperCase}"
val s4 = s"Is adult: ${age >= 18}"
println(s2)  // Next year: 31
println(s3)  // Upper: ALICE
println(s4)  // Is adult: true

// f-interpolator: format numbers
val f1 = f"Score: $score%.2f"       // Score: 98.76
val f2 = f"Name: $name%-10s|"       // Name: Alice     |
val f3 = f"Age: $age%05d"           // Age: 00030
val f4 = f"Score: $score%e"         // Scientific: 9.875600e+01
println(f1); println(f2); println(f3); println(f4)

// raw-interpolator: ไม่ escape
val raw1 = raw"Tab:\t Newline:\n"    // Tab:\t Newline:\n (ไม่ escape)
val norm1 = s"Tab:\t Newline:\n"     // actual tab and newline
println(raw1)   // Tab:\t Newline:\n
println(norm1)  // Tab: (tab) Newline: (newline)

// Custom interpolator
implicit class RichStringContext(sc: StringContext) {
  def sql(args: Any*): String = {
    val parts = sc.parts.toList
    val argList = args.map {
      case s: String => s"'$s'"
      case n: Int    => n.toString
      case other     => other.toString
    }
    parts.zipAll(argList, "", "").map { case (p, a) => p + a }.mkString
  }
}

val tableName = "users"
val id = 42
val query = sql"SELECT * FROM $tableName WHERE id = $id"
println(query)  // SELECT * FROM 'users' WHERE id = 42
```

---

## Step 42 — String Formatting

```scala
// printf-style formatting
println(f"%-10s %5d %8.2f" format ("Alice", 30, 9876.54))
// Alice          30  9876.54

// String.format (Java style)
val formatted = String.format("%-10s %5d %8.2f", "Bob", 25, 1234.56)
println(formatted)

// printf directly
printf("Hello, %s! You are %d years old.\n", "Charlie", 28)

// Format table
case class Student(name: String, grade: String, score: Double)
val students = List(
  Student("Alice", "A", 95.5),
  Student("Bob", "B+", 87.3),
  Student("Charlie", "A-", 91.0),
  Student("Diana", "B", 82.7)
)

// Table header
println(f"\n${"Name"}%-12s ${"Grade"}%-8s ${"Score"}%6s")
println("-" * 30)
students.sortBy(-_.score).foreach { s =>
  println(f"${s.name}%-12s ${s.grade}%-8s ${s.score}%6.1f")
}
```

### Number Formatting

```scala
import java.text.{DecimalFormat, NumberFormat}
import java.util.Locale

// DecimalFormat
val df = new DecimalFormat("#,##0.00")
println(df.format(1234567.89))    // 1,234,567.89

// NumberFormat
val nf = NumberFormat.getCurrencyInstance(Locale.US)
println(nf.format(9999.99))       // $9,999.99

val thNF = NumberFormat.getCurrencyInstance(new Locale("th", "TH"))
println(thNF.format(1500.00))     // ฿1,500.00

// Percentage
val pf = NumberFormat.getPercentInstance
pf.setMinimumFractionDigits(2)
println(pf.format(0.8567))        // 85.67%

// Scientific notation
println(f"${1.23e-10}%.2e")       // 1.23e-10
println(f"${6.022e23}%.3e")       // 6.022e+23
```

---

## Step 43 — String Operations Advanced

```scala
// Regex
import scala.util.matching.Regex

val emailRegex = """[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}""".r
val phoneRegex = """\d{3}[-.\s]?\d{3}[-.\s]?\d{4}""".r

val text = "Contact: john@example.com or 555-123-4567"

// findFirst
emailRegex.findFirstIn(text).foreach(e => println(s"Email: $e"))
phoneRegex.findFirstIn(text).foreach(p => println(s"Phone: $p"))

// findAll
val emails = emailRegex.findAllIn(text).toList
println(s"All emails: $emails")

// Match with groups
val dateRegex = """(\d{4})-(\d{2})-(\d{2})""".r
val dateStr = "Today is 2024-01-15 and tomorrow is 2024-01-16"

dateRegex.findAllMatchIn(dateStr).foreach { m =>
  println(s"Date: ${m.group(0)}, Year: ${m.group(1)}, Month: ${m.group(2)}, Day: ${m.group(3)}")
}

// Replace with regex
val cleaned = text.replaceAll("""\b\d{3}[-.\s]\d{3}[-.\s]\d{4}\b""", "[PHONE]")
println(cleaned)  // Contact: john@example.com or [PHONE]

// Split with regex
val csv = "Alice,30,\"New York\",Engineer"
// Simple split
csv.split(",").foreach(println)
```

### String Manipulation

```scala
// Padding and alignment
def padLeft(s: String, width: Int, char: Char = ' '): String =
  if (s.length >= width) s else char.toString * (width - s.length) + s

def padRight(s: String, width: Int, char: Char = ' '): String =
  if (s.length >= width) s else s + char.toString * (width - s.length)

def center(s: String, width: Int, char: Char = ' '): String = {
  val total = width - s.length
  val left = total / 2
  val right = total - left
  char.toString * left + s + char.toString * right
}

println(padLeft("42", 8, '0'))      // 00000042
println(padRight("hello", 10, '.')) // hello.....
println(center("Scala", 20, '='))   // =======Scala========

// Word wrap
def wordWrap(text: String, maxWidth: Int): List[String] = {
  val words = text.split("\\s+").toList
  words.foldLeft(List.empty[String]) { (lines, word) =>
    lines match {
      case Nil => List(word)
      case head :: tail =>
        if ((head + " " + word).length <= maxWidth)
          (head + " " + word) :: tail
        else
          word :: head :: tail
    }
  }.reverse
}

val longText = "The quick brown fox jumps over the lazy dog in the sunny afternoon"
wordWrap(longText, 30).foreach(println)

// Levenshtein distance
def levenshtein(s1: String, s2: String): Int = {
  val m = s1.length
  val n = s2.length
  val dp = Array.ofDim[Int](m + 1, n + 1)
  for (i <- 0 to m) dp(i)(0) = i
  for (j <- 0 to n) dp(0)(j) = j
  for (i <- 1 to m; j <- 1 to n) {
    if (s1(i - 1) == s2(j - 1)) dp(i)(j) = dp(i - 1)(j - 1)
    else dp(i)(j) = 1 + List(dp(i-1)(j), dp(i)(j-1), dp(i-1)(j-1)).min
  }
  dp(m)(n)
}

println(levenshtein("kitten", "sitting"))  // 3
println(levenshtein("saturday", "sunday")) // 3
```

---

## Step 44 — Console I/O

```scala
import scala.io.StdIn

// Basic output
println("Hello, World!")           // with newline
print("No newline: ")              // without newline
println()                          // just newline

// Multiple values
println(1, 2, 3)  // (1,2,3) — prints as tuple

// Console.out vs println
Console.out.println("To stdout")
Console.err.println("To stderr")

// Read line
val input = StdIn.readLine()       // blocks until Enter
val input2 = StdIn.readLine("Enter name: ")  // with prompt

// Read typed values
val n = StdIn.readInt()            // read Int
val d = StdIn.readDouble()         // read Double
val b = StdIn.readBoolean()        // read Boolean

// Safe reading with Try
import scala.util.Try

def safeReadInt(prompt: String): Option[Int] = {
  print(prompt)
  Try(StdIn.readLine().trim.toInt).toOption
}

// Interactive program example
@main def interactiveCalc(): Unit = {
  println("Simple Calculator (type 'quit' to exit)")
  
  var running = true
  while (running) {
    val line = StdIn.readLine("Enter expression (e.g., 5 + 3): ").trim
    line match {
      case "quit" | "exit" | "q" => running = false
      case expr =>
        val parts = expr.split("\\s+")
        if (parts.length == 3) {
          val result = for {
            a <- Try(parts(0).toDouble).toOption
            op = parts(1)
            b <- Try(parts(2).toDouble).toOption
          } yield op match {
            case "+" => a + b
            case "-" => a - b
            case "*" => a * b
            case "/" => if (b != 0) a / b else Double.NaN
            case _   => Double.NaN
          }
          result match {
            case Some(r) => println(f"= $r%.4f")
            case None    => println("Invalid expression")
          }
        } else {
          println("Format: number operator number")
        }
    }
  }
  println("Goodbye!")
}
```

---

## Step 45 — File I/O

```scala
import scala.io.Source
import java.io.{File, FileWriter, PrintWriter, BufferedWriter}
import java.nio.file.{Files, Paths, Path}
import java.nio.charset.StandardCharsets

// ==============================
// Reading Files
// ==============================

// Read entire file
def readFile(path: String): String = {
  val source = Source.fromFile(path)
  try source.mkString
  finally source.close()
}

// Read lines
def readLines(path: String): List[String] = {
  val source = Source.fromFile(path)
  try source.getLines().toList
  finally source.close()
}

// Read with encoding
def readUTF8(path: String): String = {
  val source = Source.fromFile(path, "UTF-8")
  try source.mkString
  finally source.close()
}

// Using try-with-resources style
def readFileSafe(path: String): Either[Throwable, String] =
  scala.util.Try {
    val source = Source.fromFile(path)
    try source.mkString
    finally source.close()
  }.toEither

// ==============================
// Writing Files
// ==============================

def writeFile(path: String, content: String): Unit = {
  val writer = new PrintWriter(new File(path))
  try writer.write(content)
  finally writer.close()
}

def appendFile(path: String, content: String): Unit = {
  val writer = new FileWriter(path, true)  // true = append
  try writer.write(content)
  finally writer.close()
}

def writeLines(path: String, lines: Seq[String]): Unit = {
  val writer = new PrintWriter(new File(path))
  try lines.foreach(writer.println)
  finally writer.close()
}

// ==============================
// Using java.nio (Modern)
// ==============================

def readFileNio(path: String): String =
  new String(Files.readAllBytes(Paths.get(path)), StandardCharsets.UTF_8)

def writeFileNio(path: String, content: String): Unit =
  Files.write(Paths.get(path), content.getBytes(StandardCharsets.UTF_8))

def readLinesNio(path: String): List[String] =
  Files.readAllLines(Paths.get(path), StandardCharsets.UTF_8)
    .toArray.toList.map(_.toString)
```

---

## Step 46 — File Operations

```scala
import java.io.File
import java.nio.file.{Files, Paths, Path, StandardCopyOption}

// File operations
val file = new File("example.txt")

// Check
println(file.exists())      // true/false
println(file.isFile())      // true/false
println(file.isDirectory()) // true/false
println(file.canRead())     // true/false
println(file.canWrite())    // true/false
println(file.length())      // size in bytes

// Create
file.createNewFile()        // create empty file
new File("mydir").mkdir()   // create directory
new File("a/b/c").mkdirs()  // create nested directories

// Delete
file.delete()               // delete file/empty directory
def deleteRecursive(f: File): Boolean = {
  if (f.isDirectory) f.listFiles().foreach(deleteRecursive)
  f.delete()
}

// List files
val dir = new File(".")
dir.listFiles().foreach(f => println(f.getName))
dir.listFiles(_.isDirectory).foreach(f => println(s"DIR: ${f.getName}"))

// Filter files
val scalaFiles = dir.listFiles(f => f.getName.endsWith(".scala"))

// Rename/Move
val src = new File("old.txt")
val dst = new File("new.txt")
src.renameTo(dst)

// NIO copy
Files.copy(
  Paths.get("source.txt"),
  Paths.get("dest.txt"),
  StandardCopyOption.REPLACE_EXISTING
)

// Walk directory tree
import java.nio.file.{FileVisitResult, SimpleFileVisitor, FileVisitor}
import java.nio.file.attribute.BasicFileAttributes

Files.walkFileTree(Paths.get("."), new SimpleFileVisitor[Path] {
  override def visitFile(file: Path, attrs: BasicFileAttributes): FileVisitResult = {
    if (file.toString.endsWith(".scala"))
      println(file)
    FileVisitResult.CONTINUE
  }
})
```

---

## Step 47 — CSV Processing

```scala
import scala.io.Source

case class Product(
  id: Int,
  name: String,
  category: String,
  price: Double,
  stock: Int
)

object CSVProcessor {
  
  def parseCSV(path: String): List[Product] = {
    val source = Source.fromFile(path)
    try {
      source.getLines()
        .drop(1)  // skip header
        .map(parseLine)
        .toList
    } finally {
      source.close()
    }
  }
  
  private def parseLine(line: String): Product = {
    val cols = line.split(",").map(_.trim)
    Product(
      id       = cols(0).toInt,
      name     = cols(1),
      category = cols(2),
      price    = cols(3).toDouble,
      stock    = cols(4).toInt
    )
  }
  
  def writeCSV(path: String, products: List[Product]): Unit = {
    import java.io.PrintWriter
    val writer = new PrintWriter(path)
    try {
      writer.println("id,name,category,price,stock")
      products.foreach { p =>
        writer.println(s"${p.id},${p.name},${p.category},${p.price},${p.stock}")
      }
    } finally {
      writer.close()
    }
  }
}

// สร้าง test data
val products = List(
  Product(1, "Laptop", "Electronics", 45000.0, 50),
  Product(2, "Mouse", "Electronics", 500.0, 200),
  Product(3, "Desk", "Furniture", 8000.0, 30),
  Product(4, "Chair", "Furniture", 5000.0, 45),
  Product(5, "Monitor", "Electronics", 12000.0, 75)
)

// Write
CSVProcessor.writeCSV("/tmp/products.csv", products)

// Read back
val loaded = CSVProcessor.parseCSV("/tmp/products.csv")

// Analyze
println("=== Product Analysis ===")
println(f"Total products: ${loaded.length}")
println(f"Total value: ฿${loaded.map(p => p.price * p.stock).sum}%.2f")

val byCategory = loaded.groupBy(_.category)
byCategory.foreach { case (cat, prods) =>
  val avgPrice = prods.map(_.price).sum / prods.length
  println(f"  $cat: ${prods.length} items, avg price ฿$avgPrice%.2f")
}
```

---

## Step 48 — JSON-like String Processing

```scala
// Simple JSON builder
case class JsonValue(value: Any) {
  def toJson: String = value match {
    case null        => "null"
    case s: String   => s""""${s.replace("\"", "\\\"")}""""
    case n: Int      => n.toString
    case n: Double   => n.toString
    case b: Boolean  => b.toString
    case m: Map[?, ?] =>
      val pairs = m.map { case (k, v) => s""""$k": ${JsonValue(v).toJson}""" }
      s"{${pairs.mkString(", ")}}"
    case l: List[?]  =>
      s"[${l.map(i => JsonValue(i).toJson).mkString(", ")}]"
    case other       => s""""${other.toString}""""
  }
}

def toJson(obj: Map[String, Any]): String = JsonValue(obj).toJson

val user = Map[String, Any](
  "id"     -> 1,
  "name"   -> "Alice",
  "age"    -> 30,
  "active" -> true,
  "tags"   -> List("scala", "programming"),
  "address" -> Map[String, Any](
    "city"    -> "Bangkok",
    "country" -> "Thailand"
  )
)

println(toJson(user))
// {"id": 1, "name": "Alice", "age": 30, "active": true, "tags": ["scala", "programming"], "address": {"city": "Bangkok", "country": "Thailand"}}
```

---

## Step 49 — StringBuilder

```scala
// StringBuilder สำหรับ efficient string building
val sb = new StringBuilder()

sb.append("Hello")
sb.append(", ")
sb.append("World")
sb.append("!")

println(sb.toString())  // Hello, World!

// Method chaining
val result = new StringBuilder()
  .append("Scala")
  .append(" ")
  .append("is")
  .append(" ")
  .append("awesome!")
  .toString()

println(result)  // Scala is awesome!

// Building large strings efficiently
def buildReport(data: List[(String, Double)]): String = {
  val sb = new StringBuilder()
  sb.append("=== Report ===\n")
  sb.append(f"${"Item"}%-20s ${"Amount"}%10s\n")
  sb.append("-" * 32 + "\n")
  
  data.foreach { case (item, amount) =>
    sb.append(f"$item%-20s $amount%10.2f\n")
  }
  
  sb.append("-" * 32 + "\n")
  val total = data.map(_._2).sum
  sb.append(f"${"Total"}%-20s $total%10.2f\n")
  sb.toString()
}

val report = buildReport(List(
  ("Product A", 1500.00),
  ("Product B", 2750.50),
  ("Product C", 320.75),
  ("Product D", 8900.00)
))

println(report)
```

---

## Step 50 — โปรแกรม String & I/O Complete

```scala
// TextAnalyzer.scala — Text analysis tool

import scala.io.Source
import java.io.{File, PrintWriter}

object TextAnalyzer {
  
  case class Analysis(
    charCount: Int,
    wordCount: Int,
    lineCount: Int,
    uniqueWords: Int,
    avgWordLength: Double,
    longestWord: String,
    mostFrequentWord: String,
    wordFrequency: Map[String, Int]
  )
  
  def analyze(text: String): Analysis = {
    val lines = text.split("\n")
    val words = text.toLowerCase
      .replaceAll("[^a-z\\s]", "")
      .split("\\s+")
      .filter(_.nonEmpty)
    
    val freq = words.groupBy(identity).view.mapValues(_.length).toMap
    val topWord = freq.maxBy(_._2)._1
    
    Analysis(
      charCount       = text.length,
      wordCount       = words.length,
      lineCount       = lines.length,
      uniqueWords     = freq.size,
      avgWordLength   = words.map(_.length).sum.toDouble / words.length,
      longestWord     = words.maxBy(_.length),
      mostFrequentWord = topWord,
      wordFrequency   = freq
    )
  }
  
  def formatReport(analysis: Analysis): String = {
    val sb = new StringBuilder()
    sb.append("=" * 50 + "\n")
    sb.append(s"${center("TEXT ANALYSIS REPORT", 50)}\n")
    sb.append("=" * 50 + "\n\n")
    sb.append(f"Characters:        ${analysis.charCount}%10d\n")
    sb.append(f"Words:             ${analysis.wordCount}%10d\n")
    sb.append(f"Lines:             ${analysis.lineCount}%10d\n")
    sb.append(f"Unique words:      ${analysis.uniqueWords}%10d\n")
    sb.append(f"Avg word length:   ${analysis.avgWordLength}%10.2f\n")
    sb.append(f"Longest word:      ${analysis.longestWord}%10s\n")
    sb.append(f"Most frequent:     ${analysis.mostFrequentWord}%10s\n\n")
    
    sb.append("Top 10 words:\n")
    analysis.wordFrequency.toSeq
      .sortBy(-_._2)
      .take(10)
      .foreach { case (word, count) =>
        val bar = "▓" * math.min(count, 20)
        sb.append(f"  ${word}%-15s ${count}%3d $bar\n")
      }
    
    sb.toString()
  }
  
  private def center(s: String, width: Int): String = {
    val padding = (width - s.length) / 2
    " " * padding + s
  }
}

@main def textAnalyzerDemo(): Unit = {
  val sampleText = """
    |Scala is a strong statically typed high-level general-purpose programming language
    |that supports both object-oriented programming and functional programming.
    |Designed to be concise, many of Scala's design decisions were made to address
    |criticisms of Java. Scala source code can be compiled to Java bytecode and run
    |on a Java virtual machine. Scala provides language interoperability with Java,
    |so that libraries written in either language may be referenced directly in Scala
    |or Java code. Like Java, Scala is object-oriented, and uses a curly-brace syntax
    |reminiscent of the C programming language. Unlike Java, Scala has many features
    |of functional programming languages like Scheme, Standard ML, and Haskell.
    """.stripMargin.trim
  
  val analysis = TextAnalyzer.analyze(sampleText)
  val report = TextAnalyzer.formatReport(analysis)
  
  println(report)
  
  // Save to file
  val writer = new PrintWriter(new File("/tmp/text-analysis.txt"))
  try writer.write(report)
  finally writer.close()
  
  println("Report saved to /tmp/text-analysis.txt")
}
```

---

## สรุป Part 05

| Step | สิ่งที่เรียน |
|------|-------------|
| 41 | String interpolation ทุกรูปแบบ |
| 42 | String formatting |
| 43 | Regex และ string operations |
| 44 | Console I/O |
| 45 | File I/O (read/write) |
| 46 | File operations |
| 47 | CSV processing |
| 48 | String-based data format |
| 49 | StringBuilder |
| 50 | Text analyzer โปรแกรม complete |

## แบบฝึกหัด

1. สร้าง CSV reader/writer generic สำหรับ case class ใดๆ
2. เขียน `diff` function เปรียบเทียบสอง text files
3. สร้าง simple template engine: `render("Hello, {{name}}!", Map("name" -> "Alice"))`
4. เขียน function อ่าน properties file (`key=value`) แล้ว return `Map[String, String]`
5. สร้าง log rotator ที่ rename ไฟล์ตาม timestamp

## ต่อไป

**[Part 06 →](part-06-arrays-and-basic-collections.md)** — Arrays และ Basic Collections
