# Part 26: Implicits and Given

## Steps 251-260: Implicit Parameters, Implicit Conversions, Type Class Pattern, Given/Using (Scala 3 style)

---

## Step 251: Implicit Parameters พื้นฐาน

```scala
// Implicit parameters — compiler fills them automatically
object ImplicitParams extends App {
  
  // Simple implicit parameter
  def greet(name: String)(implicit prefix: String): String =
    s"$prefix, $name!"
  
  implicit val defaultPrefix: String = "Hello"
  
  println(greet("Alice"))          // Auto-fill: "Hello, Alice!"
  println(greet("Bob")("Hi"))      // Manual: "Hi, Bob!"
  println(greet("Carol")("Sawasdee"))
  
  // Implicit for configuration
  case class DbConfig(host: String, port: Int, dbName: String)
  case class EmailConfig(smtpHost: String, from: String)
  
  def connectToDb(implicit config: DbConfig): String =
    s"Connected to ${config.host}:${config.port}/${config.dbName}"
  
  def sendEmail(to: String, subject: String)(implicit config: EmailConfig): String =
    s"Email from ${config.from} to $to via ${config.smtpHost}: '$subject'"
  
  implicit val dbConfig: DbConfig = DbConfig("localhost", 5432, "myapp")
  implicit val emailConfig: EmailConfig = EmailConfig("smtp.example.com", "noreply@example.com")
  
  println("\n=== Implicit Config ===")
  println(connectToDb)
  println(sendEmail("user@example.com", "Welcome!"))
  
  // implicitly — retrieve implicit value
  def showConfig[A](implicit value: A): A = {
    val retrieved = implicitly[A]
    println(s"Config: $retrieved")
    retrieved
  }
  
  showConfig[DbConfig]
  
  // Multiple implicit parameters
  def processRequest(
    data: String
  )(implicit 
    db: DbConfig,
    email: EmailConfig
  ): String = s"Processing '$data' with db=${db.dbName}, smtp=${email.smtpHost}"
  
  println(processRequest("order-123"))
}
```

---

## Step 252: Implicit Conversions

```scala
object ImplicitConversions extends App {
  
  // Implicit conversion — automatic type transformation
  case class Celsius(value: Double) {
    def toFahrenheit: Fahrenheit = Fahrenheit(value * 9.0 / 5.0 + 32)
    override def toString: String = f"${value}%.1f°C"
  }
  
  case class Fahrenheit(value: Double) {
    def toCelsius: Celsius = Celsius((value - 32) * 5.0 / 9.0)
    override def toString: String = f"${value}%.1f°F"
  }
  
  implicit def celsiusToFahrenheit(c: Celsius): Fahrenheit = c.toFahrenheit
  implicit def fahrenheitToCelsius(f: Fahrenheit): Celsius = f.toCelsius
  
  def printTemp(f: Fahrenheit): Unit = println(s"  Temperature: $f")
  
  println("=== Implicit Conversions ===")
  printTemp(Celsius(100.0))   // Auto-convert
  printTemp(Celsius(0.0))
  printTemp(Celsius(37.0))
  
  // Implicit class (extension methods) — safest form of implicit conversion
  implicit class RichInt(val n: Int) extends AnyVal {
    def factorial: Long = if (n <= 1) 1L else n * (n - 1).factorial
    def times(f: => Unit): Unit = (1 to n).foreach(_ => f)
    def isEven: Boolean = n % 2 == 0
    def isOdd: Boolean = !isEven
    def digits: List[Int] = n.toString.map(_.asDigit).toList
    def reverseDigits: Int = digits.reverse.mkString.toInt
  }
  
  println("\n=== RichInt Extension Methods ===")
  println(s"5.factorial = ${5.factorial}")
  println(s"10.factorial = ${10.factorial}")
  println(s"42.isEven = ${42.isEven}")
  println(s"7.isOdd = ${7.isOdd}")
  println(s"12345.digits = ${12345.digits}")
  println(s"12345.reverseDigits = ${12345.reverseDigits}")
  
  print("3.times: ")
  3.times { print("hello ") }
  println()
  
  // Implicit class for strings
  implicit class RichString(val s: String) extends AnyVal {
    def toSnakeCase: String = s.replaceAll("([A-Z])", "_$1").toLowerCase.stripPrefix("_")
    def toCamelCase: String = s.split("_").zipWithIndex.map { case (w, 0) => w; case (w, _) => w.capitalize }.mkString
    def isPalindrome: Boolean = { val clean = s.toLowerCase.filter(_.isLetterOrDigit); clean == clean.reverse }
    def wordCount: Int = s.trim.split("\\s+").count(_.nonEmpty)
    def truncate(maxLen: Int, suffix: String = "..."): String =
      if (s.length <= maxLen) s else s.take(maxLen - suffix.length) + suffix
  }
  
  println("\n=== RichString ===")
  println(s"'HelloWorld'.toSnakeCase = ${"HelloWorld".toSnakeCase}")
  println(s"'hello_world'.toCamelCase = ${"hello_world".toCamelCase}")
  println(s"'racecar'.isPalindrome = ${"racecar".isPalindrome}")
  println(s"'hello world'.isPalindrome = ${"hello world".isPalindrome}")
  println(s"'hello world foo'.wordCount = ${"hello world foo".wordCount}")
  println(s"truncate: ${"This is a very long string".truncate(15)}")
}
```

---

## Step 253: Type Class Pattern with Implicits

```scala
// Full type class pattern
trait Show[A] {
  def show(value: A): String
}

object Show {
  // Summoner
  def apply[A](implicit s: Show[A]): Show[A] = s
  
  // Convenience function
  def show[A: Show](value: A): String = Show[A].show(value)
  
  // Instances for primitive types
  implicit val showInt: Show[Int] = (n: Int) => n.toString
  implicit val showDouble: Show[Double] = (d: Double) => f"$d%.4f"
  implicit val showString: Show[String] = (s: String) => s""""$s""""
  implicit val showBoolean: Show[Boolean] = (b: Boolean) => if (b) "true" else "false"
  
  // Derived instances
  implicit def showOption[A: Show]: Show[Option[A]] = {
    case None    => "None"
    case Some(v) => s"Some(${Show[A].show(v)})"
  }
  
  implicit def showList[A: Show]: Show[List[A]] = (list: List[A]) =>
    list.map(Show[A].show).mkString("[", ", ", "]")
    
  implicit def showTuple2[A: Show, B: Show]: Show[(A, B)] = {
    case (a, b) => s"(${Show[A].show(a)}, ${Show[B].show(b)})"
  }
  
  // Extension method via implicit class
  implicit class ShowOps[A: Show](val value: A) {
    def show: String = Show[A].show(value)
    def println(): Unit = scala.Predef.println(Show[A].show(value))
  }
}

// Custom type class instances
case class User(name: String, email: String, age: Int)
case class Product(id: String, name: String, price: Double)

object TypeClassPattern extends App {
  import Show._
  
  // Instances for our types
  implicit val showUser: Show[User] = (u: User) =>
    s"User{name=${u.name.show}, email=${u.email.show}, age=${u.age.show}}"
  
  implicit val showProduct: Show[Product] = (p: Product) =>
    s"Product{id=${p.id.show}, name=${p.name.show}, price=${p.price.show}}"
  
  println("=== Type Class Show ===")
  println(Show.show(42))
  println(Show.show("hello"))
  println(Show.show(List(1, 2, 3)))
  println(Show.show(Option(42)))
  println(Show.show(None: Option[Int]))
  println(Show.show(List(Option(1), None, Option(3))))
  
  val user = User("Alice", "alice@example.com", 30)
  val product = Product("P001", "Laptop", 49999.99)
  
  println(Show.show(user))
  println(Show.show(product))
  
  // Using extension methods
  user.println()
  product.println()
  List(user, user.copy(name = "Bob")).show.println()
  
  println("\n=== Derived Instances ===")
  val users = List(user, User("Bob", "bob@example.com", 25))
  println(Show.show(users))
  
  val optUser: Option[User] = Some(user)
  println(Show.show(optUser))
}
```

---

## Step 254-260: Advanced Implicit Patterns

```scala
// Implicit priority and disambiguation
trait Priority0 {
  implicit def lowPriority[A]: Show[A] = (a: A) => s"<$a>"
}

trait Priority1 extends Priority0 {
  implicit val medPriorityInt: Show[Int] = (n: Int) => s"#$n"
}

object Priority extends Priority1 {
  implicit val highPriorityInt: Show[Int] = (n: Int) => n.toString  // Wins
}

// Implicit evidence / type class coherence
trait CanEqual[A, B] {
  def equal(a: A, b: B): Boolean
}

object CanEqual {
  implicit def sameType[A]: CanEqual[A, A] = (a: A, b: A) => a == b
  implicit val intDouble: CanEqual[Int, Double] = (a: Int, b: Double) => a.toDouble == b
  implicit val doubleInt: CanEqual[Double, Int] = (a: Double, b: Int) => a == b.toDouble
}

def areEqual[A, B](a: A, b: B)(implicit ev: CanEqual[A, B]): Boolean = ev.equal(a, b)

// Implicit function types (Scala 3 style with Scala 2 workaround)
class Transaction(val id: String) {
  private var committed = false
  def commit(): Unit = committed = true
  def rollback(): Unit = committed = false
  def isCommitted: Boolean = committed
}

type WithTransaction[A] = Transaction => A

def withTransaction[A](block: WithTransaction[A]): A = {
  val tx = new Transaction(java.util.UUID.randomUUID().toString)
  try {
    val result = block(tx)
    tx.commit()
    result
  } catch {
    case e: Exception => tx.rollback(); throw e
  }
}

def saveUser(user: String)(implicit tx: Transaction): String = {
  println(s"  [tx:${tx.id.take(8)}] Saving user: $user")
  s"saved-$user"
}

def saveOrder(orderId: String)(implicit tx: Transaction): String = {
  println(s"  [tx:${tx.id.take(8)}] Saving order: $orderId")
  s"saved-order-$orderId"
}

// Magnet pattern — overload resolution via implicits
sealed trait IntOrString
case class IntMagnet(value: Int) extends IntOrString
case class StringMagnet(value: String) extends IntOrString

object IntOrString {
  implicit def fromInt(n: Int): IntOrString = IntMagnet(n)
  implicit def fromString(s: String): IntOrString = StringMagnet(s)
}

def process(magnet: IntOrString): String = magnet match {
  case IntMagnet(n)    => s"Processing int: $n"
  case StringMagnet(s) => s"Processing string: '$s'"
}

object AdvancedImplicitsDemo extends App {
  
  println("=== CanEqual ===")
  println(s"areEqual(1, 1): ${areEqual(1, 1)}")
  println(s"areEqual(1, 1.0): ${areEqual(1, 1.0)}")
  println(s"areEqual(3.14, 3): ${areEqual(3.14, 3)}")
  
  println("\n=== Transaction Pattern ===")
  val result = withTransaction { implicit tx =>
    val u = saveUser("alice@example.com")
    val o = saveOrder("ORD-001")
    s"$u, $o"
  }
  println(s"Result: $result")
  
  println("\n=== Magnet Pattern ===")
  println(process(42))         // implicit conversion
  println(process("hello"))    // implicit conversion
  
  println("\n=== Implicit in Collections ===")
  import scala.math.Ordering
  
  case class Student(name: String, gpa: Double, year: Int)
  
  implicit val byGPA: Ordering[Student] = Ordering.by(_.gpa).reverse
  
  val students = List(
    Student("Alice", 3.9, 3),
    Student("Bob", 3.5, 2),
    Student("Carol", 3.7, 4),
    Student("Dave", 3.8, 1)
  )
  
  println("By GPA (desc):")
  students.sorted.foreach(s => println(s"  ${s.name}: ${s.gpa}"))
  
  // Custom sort
  println("\nBy year, then GPA:")
  val byYearThenGPA = Ordering.by((s: Student) => (s.year, -s.gpa))
  students.sorted(byYearThenGPA).foreach(s => println(s"  year ${s.year}: ${s.name} (${s.gpa})"))
}
```

---

## สรุป Part 26

| Feature | Syntax | ใช้เมื่อ |
|---------|--------|---------|
| Implicit parameter | `def f(implicit x: T)` | Config, context injection |
| Implicit conversion | `implicit def toT(a: A): T` | Type widening, DSLs |
| Implicit class | `implicit class Ops(val x: T)` | Extension methods |
| Type class instance | `implicit val showInt: Show[Int]` | Polymorphism without inheritance |
| Context bound | `[A: Show]` | Shorthand for `(implicit s: Show[A])` |
| implicitly | `implicitly[Show[A]]` | Summon implicit value |
| Derived instance | `implicit def showList[A: Show]` | Automatic derivation |

---

## แบบฝึกหัด Part 26

**ข้อ 1:** Implement `Eq[A]` type class ที่มี `===` operator และ `=/=` operator

**ข้อ 2:** สร้าง `JsonEncoder[A]` type class พร้อม derived instances สำหรับ case classes

**ข้อ 3:** Implement `Monoid[A]` type class พร้อม `combineAll` ที่ใช้ implicit Monoid

**ข้อ 4:** สร้าง implicit conversion layer สำหรับ Java collections ↔ Scala collections

**ข้อ 5:** Implement `Ordering` instance สำหรับ version strings (e.g., "1.2.3" < "1.10.0" < "2.0.0")

---

➡️ ต่อไป: [Part 27 — Type Bounds and Constraints](part-27-type-bounds-and-constraints.md)
