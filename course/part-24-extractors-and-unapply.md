# Part 24: Extractors and Unapply

## Steps 231-240: Custom Extractors, unapplySeq, Regex Extractors, Case Class Decomposition

---

## Step 231: Custom Extractors พื้นฐาน

```scala
// Extractor = object with unapply method
object IsAdult {
  def unapply(age: Int): Option[Int] = if (age >= 18) Some(age) else None
}

object IsMinor {
  def unapply(age: Int): Boolean = age < 18  // Boolean unapply
}

object Positive {
  def unapply(n: Int): Option[Int] = if (n > 0) Some(n) else None
}

object NonEmpty {
  def unapply[A](list: List[A]): Option[(A, List[A])] = list match {
    case Nil     => None
    case h :: t  => Some((h, t))
  }
}

object ExtractorsBasics extends App {
  
  val ages = List(15, 18, 21, 16, 25, 17, 30)
  
  println("=== Custom Extractors ===")
  ages.foreach {
    case IsAdult(age) => println(s"  $age is an adult")
    case IsMinor()    => println(s"  Minor (under 18)")
    case age          => println(s"  $age (other)")
  }
  
  println("\n=== Positive Extractor ===")
  List(-5, 0, 3, -1, 7).foreach {
    case Positive(n) => println(s"  $n is positive")
    case n           => println(s"  $n is not positive")
  }
  
  println("\n=== NonEmpty Extractor ===")
  List(List.empty[Int], List(1), List(1, 2, 3)).foreach {
    case NonEmpty(head, tail) => println(s"  head=$head, tail=$tail")
    case Nil                  => println("  empty list")
  }
  
  // Using extractors in variable binding
  val someList = List(1, 2, 3)
  val NonEmpty(first, rest) = someList
  println(s"\nfirst: $first, rest: $rest")
}
```

---

## Step 232: Regex Extractors

```scala
object RegexExtractors extends App {
  
  // Scala regex has built-in unapply support
  val datePattern = "(\\d{4})-(\\d{2})-(\\d{2})".r
  val emailPattern = "([^@]+)@([^@]+)\\.([a-zA-Z]{2,})".r
  val phonePattern = "\\+?(\\d{1,3})[-. ]?(\\d{3})[-. ]?(\\d{4,})".r
  val ipPattern = "(\\d{1,3})\\.(\\d{1,3})\\.(\\d{1,3})\\.(\\d{1,3})".r
  
  println("=== Regex Extractors ===")
  
  // Date parsing
  val dates = List("2024-01-15", "2024-12-31", "not-a-date", "2024-13-01")
  
  dates.foreach {
    case datePattern(year, month, day) if month.toInt in (1 to 12) =>
      println(s"  Date: $year/$month/$day")
    case datePattern(year, month, day) =>
      println(s"  Invalid date components: $year-$month-$day")
    case other =>
      println(s"  Not a date: $other")
  }
  
  // Email parsing
  println("\n--- Email ---")
  val emails = List("alice@example.com", "bob.smith@company.co.th", "not-an-email")
  
  emails.foreach {
    case emailPattern(user, domain, tld) =>
      println(s"  User: $user, Domain: $domain, TLD: $tld")
    case other =>
      println(s"  Invalid: $other")
  }
  
  // IP address
  println("\n--- IP Addresses ---")
  val ips = List("192.168.1.1", "10.0.0.1", "256.0.0.1", "not-an-ip")
  
  ips.foreach {
    case ipPattern(a, b, c, d) 
      if List(a,b,c,d).forall(n => n.toInt <= 255) =>
      println(s"  Valid IP: $a.$b.$c.$d")
    case ipPattern(a, b, c, d) =>
      println(s"  Invalid IP range: $a.$b.$c.$d")
    case other =>
      println(s"  Not an IP: $other")
  }
  
  // Complex URL parsing
  val urlPattern = "(https?)://([^/]+)(/[^?#]*)?(\\?[^#]*)?(#.*)?".r
  
  val urls = List(
    "https://api.example.com/users/123?page=1&limit=10#section",
    "http://localhost:8080/api/health",
    "not-a-url"
  )
  
  println("\n--- URL Parsing ---")
  urls.foreach {
    case urlPattern(scheme, host, path, query, fragment) =>
      println(s"  scheme=$scheme, host=$host, path=$path, query=$query, fragment=$fragment")
    case other =>
      println(s"  Not a URL: $other")
  }
}

// Extension method for in operator
implicit class IntInRange(n: Int) {
  def in(range: Range): Boolean = range.contains(n)
}
```

---

## Step 233: unapplySeq — Variable Length Patterns

```scala
object UnapplySeqDemo extends App {
  
  // unapplySeq for variable-length extraction
  object SplitBySpace {
    def unapplySeq(s: String): Option[List[String]] = {
      val parts = s.trim.split("\\s+").toList
      if (parts.isEmpty || parts == List("")) None
      else Some(parts)
    }
  }
  
  println("=== unapplySeq ===")
  val sentences = List("hello world", "one two three four", "single", "")
  
  sentences.foreach {
    case SplitBySpace(one)           => println(s"  One word: '$one'")
    case SplitBySpace(first, second) => println(s"  Two words: '$first', '$second'")
    case SplitBySpace(words @ _*)    => println(s"  ${words.size} words: ${words.mkString(", ")}")
    case _                           => println("  Empty string")
  }
  
  // CSV extractor
  object CSV {
    def unapplySeq(line: String): Option[List[String]] = {
      Some(line.split(",", -1).map(_.trim).toList)
    }
  }
  
  println("\n=== CSV Extractor ===")
  val csvLines = List(
    "Alice,30,Bangkok",
    "Bob,25,London,UK",
    "Carol",
    "Dave,35,Tokyo,Japan,Senior"
  )
  
  csvLines.foreach {
    case CSV(name, age, city) =>
      println(s"  3-column: name=$name, age=$age, city=$city")
    case CSV(name, age, city, country) =>
      println(s"  4-column: $name, $age, $city, $country")
    case CSV(fields @ _*) =>
      println(s"  ${fields.size} columns: ${fields.mkString(" | ")}")
  }
  
  // Version number extractor
  object VersionNumber {
    def unapplySeq(version: String): Option[List[Int]] = {
      val parts = version.split("\\.")
      try Some(parts.map(_.toInt).toList)
      catch { case _: NumberFormatException => None }
    }
  }
  
  println("\n=== Version Numbers ===")
  val versions = List("1.0.0", "2.5.3", "3", "1.0", "invalid", "1.0.0.alpha")
  
  versions.foreach {
    case VersionNumber(major, 0, 0) => 
      println(s"  Major release: v$major")
    case VersionNumber(major, minor, patch) => 
      println(s"  Full version: $major.$minor.$patch")
    case VersionNumber(major, minor) =>
      println(s"  No patch: $major.$minor")
    case VersionNumber(major) =>
      println(s"  Major only: $major")
    case other =>
      println(s"  Invalid: $other")
  }
}
```

---

## Step 234: Case Class Decomposition

```scala
object CaseClassDecomposition extends App {
  
  // Case classes automatically generate unapply
  case class Color(r: Int, g: Int, b: Int)
  case class Point(x: Double, y: Double)
  case class Circle(center: Point, radius: Double)
  case class Rectangle(topLeft: Point, bottomRight: Point)
  
  sealed trait Shape
  case class CircleShape(c: Circle) extends Shape
  case class RectShape(r: Rectangle) extends Shape
  
  // Deep decomposition
  def describeShape(shape: Shape): String = shape match {
    case CircleShape(Circle(Point(0, 0), r)) =>
      s"Circle at origin, r=$r"
    case CircleShape(Circle(Point(x, y), r)) =>
      s"Circle at ($x,$y), r=$r"
    case RectShape(Rectangle(Point(x1, y1), Point(x2, y2))) =>
      s"Rect from ($x1,$y1) to ($x2,$y2)"
  }
  
  val shapes = List(
    CircleShape(Circle(Point(0, 0), 5.0)),
    CircleShape(Circle(Point(3, 4), 2.0)),
    RectShape(Rectangle(Point(0, 0), Point(10, 5)))
  )
  
  println("=== Case Class Decomposition ===")
  shapes.foreach(s => println(s"  ${describeShape(s)}"))
  
  // Extracting from nested structures
  case class Company(name: String, ceo: Option[Person2])
  case class Person2(name: String, email: String, salary: Double)
  
  def getCompanyCeoEmail(company: Company): Option[String] = company match {
    case Company(_, Some(Person2(_, email, _))) => Some(email)
    case _ => None
  }
  
  def isHighPaidCeo(company: Company, threshold: Double): Boolean = company match {
    case Company(_, Some(Person2(_, _, salary))) if salary > threshold => true
    case _ => false
  }
  
  val companies = List(
    Company("TechCorp", Some(Person2("Alice", "alice@tech.com", 500000.0))),
    Company("StartupX", Some(Person2("Bob", "bob@startup.com", 150000.0))),
    Company("SmallBiz", None)
  )
  
  println("\n=== Company Analysis ===")
  companies.foreach { company =>
    println(s"  ${company.name}:")
    println(s"    CEO email: ${getCompanyCeoEmail(company)}")
    println(s"    High-paid: ${isHighPaidCeo(company, 200000)}")
  }
}
```

---

## Step 235-240: Advanced Extractors

```scala
object AdvancedExtractors extends App {
  
  // Extractor with state
  class MultipleOf(n: Int) {
    def unapply(x: Int): Boolean = x % n == 0
  }
  
  val fizz = new MultipleOf(3)
  val buzz = new MultipleOf(5)
  
  println("=== FizzBuzz with Extractors ===")
  (1 to 20).foreach {
    case fizz() if buzz.unapply(1) => println("FizzBuzz")  // Won't work perfectly this way
    case n if n % 15 == 0 => println("FizzBuzz")
    case fizz() => println("Fizz")
    case buzz() => println("Buzz")
    case n => println(n)
  }
  
  // Parameterized extractor
  object DividesBy {
    def apply(n: Int) = new {
      def unapply(x: Int): Option[Int] = if (x % n == 0) Some(x / n) else None
    }
  }
  
  val by3 = DividesBy(3)
  val by5 = DividesBy(5)
  
  println("\n=== Parameterized Extractor ===")
  List(6, 10, 15, 7, 12).foreach {
    case by3(quotient) => println(s"  Divisible by 3: quotient=$quotient")
    case by5(quotient) => println(s"  Divisible by 5: quotient=$quotient")
    case n             => println(s"  $n: not divisible by 3 or 5")
  }
  
  // JSON-like extractor
  sealed trait JsonValue
  case class JsonNumber(value: Double) extends JsonValue
  case class JsonString(value: String) extends JsonValue
  case class JsonBool(value: Boolean) extends JsonValue
  case class JsonArray(elements: List[JsonValue]) extends JsonValue
  case class JsonObject(fields: Map[String, JsonValue]) extends JsonValue
  case object JsonNull extends JsonValue
  
  object JsonField {
    def apply(name: String) = new {
      def unapply(obj: JsonObject): Option[JsonValue] = obj.fields.get(name)
    }
  }
  
  val idField = JsonField("id")
  val nameField = JsonField("name")
  val ageField = JsonField("age")
  
  def parseUser(json: JsonValue): Option[(Int, String, Int)] = json match {
    case obj: JsonObject =>
      for {
        id   <- idField.unapply(obj).collect { case JsonNumber(n) => n.toInt }
        name <- nameField.unapply(obj).collect { case JsonString(s) => s }
        age  <- ageField.unapply(obj).collect { case JsonNumber(n) => n.toInt }
      } yield (id, name, age)
    case _ => None
  }
  
  val users: List[JsonValue] = List(
    JsonObject(Map(
      "id" -> JsonNumber(1),
      "name" -> JsonString("Alice"),
      "age" -> JsonNumber(30)
    )),
    JsonObject(Map(
      "id" -> JsonNumber(2),
      "name" -> JsonString("Bob")
      // missing age
    )),
    JsonString("not an object")
  )
  
  println("\n=== JSON Field Extractors ===")
  users.foreach { json =>
    parseUser(json) match {
      case Some((id, name, age)) => println(s"  User: id=$id, name=$name, age=$age")
      case None                  => println(s"  Failed to parse user")
    }
  }
  
  // Extractor composition: AND pattern
  object Both {
    class BothExtractor[A, B, C](e1: { def unapply(a: A): Option[B] }, e2: { def unapply(a: A): Option[C] }) {
      def unapply(a: A): Option[(B, C)] = for {
        b <- e1.unapply(a)
        c <- e2.unapply(a)
      } yield (b, c)
    }
    
    def apply[A, B, C](e1: { def unapply(a: A): Option[B] }, e2: { def unapply(a: A): Option[C] }) =
      new BothExtractor(e1, e2)
  }
  
  // Extractor as validator
  case class Email(address: String)
  
  object ValidEmail {
    private val pattern = "^[^@]+@[^@]+\\.[^@]{2,}$".r
    def unapply(s: String): Option[Email] = {
      if (pattern.matches(s.trim.toLowerCase)) Some(Email(s.trim.toLowerCase))
      else None
    }
  }
  
  object ValidAge {
    def unapply(n: Int): Option[Int] = if (n >= 0 && n <= 150) Some(n) else None
  }
  
  def registerUser(emailStr: String, age: Int): Either[String, (Email, Int)] = 
    (emailStr, age) match {
      case (ValidEmail(email), ValidAge(validAge)) => 
        Right((email, validAge))
      case (ValidEmail(_), _) => 
        Left(s"Invalid age: $age")
      case (_, ValidAge(_)) => 
        Left(s"Invalid email: $emailStr")
      case _ => 
        Left(s"Both email and age are invalid")
    }
  
  println("\n=== Validator Extractors ===")
  List(
    ("alice@example.com", 25),
    ("not-an-email", 30),
    ("bob@test.com", 200),
    ("", -5)
  ).foreach { case (email, age) =>
    registerUser(email, age) match {
      case Right((e, a)) => println(s"  OK: ${e.address}, age=$a")
      case Left(err)     => println(s"  Error: $err")
    }
  }
}
```

---

## สรุป Part 24

| Extractor Type | Method | คืนค่า | ตัวอย่าง |
|---------------|--------|--------|---------|
| Boolean | `unapply(a: A): Boolean` | match/no match | `IsPositive` |
| Option | `unapply(a: A): Option[B]` | Extract value | `Email(str)` |
| Tuple via Option | `unapply(a: A): Option[(B,C)]` | Multiple values | `FullName(first, last)` |
| Seq | `unapplySeq(a: A): Option[List[B]]` | Variable length | `CSV(fields@_*)` |
| Regex | `.r` creates extractor | Groups | `datePattern(y, m, d)` |

---

## แบบฝึกหัด Part 24

**ข้อ 1:** สร้าง `Range` extractor ที่ match ตัวเลขใน range: `case InRange(1, 100)(n) =>`

**ข้อ 2:** สร้าง Thai phone number extractor ที่แยก: country code, area code, number

**ข้อ 3:** สร้าง `PathMatcher` extractor สำหรับ URL paths: `/users/:id/orders/:orderId`

**ข้อ 4:** สร้าง `JsonPath` extractor: `case At("user", "address", "city")(city) =>`

**ข้อ 5:** Implement `case class` generator ที่ใช้ unapplySeq สำหรับ arbitrary arity

---

➡️ ต่อไป: [Part 25 — Generics and Variance](part-25-generics-and-variance.md)
