# Part 08 — Classes และ Objects เบื้องต้น
## Steps 71–80: Object-Oriented Programming ใน Scala

> **เป้าหมาย**: สร้าง classes, objects, constructors, fields, methods, และ companion objects

---

## Step 71 — Classes เบื้องต้น

```scala
// Basic class
class Person(val name: String, val age: Int) {
  
  // Method
  def greet(): String = s"Hi, I'm $name and I'm $age years old."
  
  // Computed property
  def isAdult: Boolean = age >= 18
  
  // Override toString
  override def toString: String = s"Person($name, $age)"
}

// Create instance
val alice = new Person("Alice", 30)
val bob   = new Person("Bob", 16)

println(alice.name)        // Alice
println(alice.age)         // 30
println(alice.greet())     // Hi, I'm Alice and I'm 30 years old.
println(alice.isAdult)     // true
println(bob.isAdult)       // false
println(alice)             // Person(Alice, 30)
```

### val vs var Constructor Parameters

```scala
class Point(val x: Double, val y: Double)
// val → immutable public field
// x, y สามารถอ่านได้จากข้างนอก

class MutablePoint(var x: Double, var y: Double)
// var → mutable public field

class PrivatePoint(private val x: Double, private val y: Double)
// private → ไม่สามารถเข้าถึงจากข้างนอก

class NoAccessor(x: Double, y: Double)
// ไม่มี val/var → private val (constructor-only)

val p = new Point(3.0, 4.0)
println(p.x)  // 3.0

val mp = new MutablePoint(1.0, 2.0)
mp.x = 10.0  // OK — var
println(mp.x) // 10.0

// val pp = new PrivatePoint(1.0, 2.0)
// pp.x  // Error: x is private
```

---

## Step 72 — Constructors

```scala
// Primary constructor (ใน class definition)
class Rectangle(val width: Double, val height: Double) {
  
  // Secondary constructor
  def this(side: Double) = this(side, side)  // square
  def this() = this(1.0, 1.0)               // unit square
  
  def area: Double = width * height
  def perimeter: Double = 2 * (width + height)
  
  override def toString: String = f"Rectangle($width%.1f × $height%.1f)"
}

val rect1 = new Rectangle(4.0, 3.0)
val square = new Rectangle(5.0)
val unit = new Rectangle()

println(rect1)         // Rectangle(4.0 × 3.0)
println(rect1.area)    // 12.0
println(square)        // Rectangle(5.0 × 5.0)
println(square.area)   // 25.0
```

### Constructor Body

```scala
class Temperature(val celsius: Double) {
  
  // Validation in constructor body
  require(celsius >= -273.15, s"Temperature $celsius°C is below absolute zero!")
  
  // Computed values
  val fahrenheit: Double = celsius * 9.0 / 5.0 + 32.0
  val kelvin: Double = celsius + 273.15
  
  // Secondary constructor from Fahrenheit
  def this(f: Double, isFahrenheit: Boolean) =
    this((f - 32) * 5.0 / 9.0)
  
  override def toString: String =
    f"${celsius}%.1f°C = ${fahrenheit}%.1f°F = ${kelvin}%.1f K"
}

val boiling = new Temperature(100.0)
val freezing = new Temperature(0.0)
val body = new Temperature(98.6, true)  // from Fahrenheit

println(boiling)   // 100.0°C = 212.0°F = 373.2 K
println(freezing)  // 0.0°C = 32.0°F = 273.2 K
println(body)      // 37.0°C = 98.6°F = 310.2 K

// try {
//   val invalid = new Temperature(-300.0)
// } catch {
//   case e: IllegalArgumentException => println(s"Error: ${e.getMessage}")
// }
```

---

## Step 73 — Fields และ Methods

```scala
class BankAccount(val owner: String, private var balance: Double = 0.0) {
  
  private var transactionCount = 0
  
  // Getter (computed property)
  def currentBalance: Double = balance
  def transactions: Int = transactionCount
  
  // Methods
  def deposit(amount: Double): Unit = {
    require(amount > 0, "Deposit amount must be positive")
    balance += amount
    transactionCount += 1
    println(f"  Deposited ฿$amount%.2f. Balance: ฿$balance%.2f")
  }
  
  def withdraw(amount: Double): Either[String, Double] = {
    if (amount <= 0) Left("Withdrawal amount must be positive")
    else if (amount > balance) Left(s"Insufficient funds (balance: $balance)")
    else {
      balance -= amount
      transactionCount += 1
      println(f"  Withdrew ฿$amount%.2f. Balance: ฿$balance%.2f")
      Right(amount)
    }
  }
  
  def transfer(amount: Double, to: BankAccount): Either[String, Unit] =
    for {
      _ <- withdraw(amount)
    } yield {
      to.deposit(amount)
    }
  
  override def toString: String =
    f"BankAccount($owner, ฿$balance%.2f, $transactionCount txns)"
}

val alice = new BankAccount("Alice", 10000.0)
val bob   = new BankAccount("Bob")

alice.deposit(5000.0)
alice.withdraw(2000.0)
alice.transfer(3000.0, bob)

println(s"\nAlice: $alice")
println(s"Bob: $bob")
```

---

## Step 74 — Getters และ Setters

```scala
class Circle private (private var _radius: Double) {
  
  // Getter
  def radius: Double = _radius
  
  // Setter with validation
  def radius_=(value: Double): Unit = {
    require(value > 0, s"Radius must be positive, got $value")
    _radius = value
  }
  
  def area: Double = math.Pi * _radius * _radius
  def circumference: Double = 2 * math.Pi * _radius
  
  override def toString: String = f"Circle(r=${_radius}%.2f)"
}

object Circle {
  def apply(radius: Double): Circle = {
    require(radius > 0)
    new Circle(radius)
  }
}

val c = Circle(5.0)
println(c)           // Circle(r=5.00)
println(c.area)      // 78.54...

c.radius = 10.0      // calls radius_=(10.0)
println(c)           // Circle(r=10.00)

// try { c.radius = -1.0 }  // IllegalArgumentException
```

---

## Step 75 — Object (Singleton)

```scala
// Object = singleton — มี instance เดียวเสมอ
object MathUtils {
  val PI = 3.141592653589793
  val E  = 2.718281828459045
  
  def square(x: Double): Double = x * x
  def cube(x: Double): Double = x * x * x
  
  def clamp(value: Double, min: Double, max: Double): Double =
    math.max(min, math.min(max, value))
  
  def lerp(a: Double, b: Double, t: Double): Double =
    a + (b - a) * clamp(t, 0.0, 1.0)
  
  def isPrime(n: Int): Boolean =
    n > 1 && (2 to math.sqrt(n).toInt).forall(n % _ != 0)
  
  def primes(max: Int): List[Int] =
    (2 to max).filter(isPrime).toList
}

println(MathUtils.PI)          // 3.14159...
println(MathUtils.square(5))   // 25.0
println(MathUtils.clamp(15.0, 0.0, 10.0))  // 10.0
println(MathUtils.lerp(0.0, 100.0, 0.75))  // 75.0
println(MathUtils.primes(30))  // List(2, 3, 5, 7, 11, 13, 17, 19, 23, 29)
```

### Object กับ main method

```scala
// Object ที่มี main method = executable program
object HelloApp {
  def main(args: Array[String]): Unit = {
    println("Hello from Object!")
    args.foreach(arg => println(s"Arg: $arg"))
  }
}

// Scala 3: @main annotation (ดีกว่า)
@main def helloApp(args: String*): Unit = {
  println("Hello!")
  args.foreach(a => println(s"  Arg: $a"))
}
```

---

## Step 76 — Companion Objects

Companion Object มีชื่อเดียวกับ class — ใช้สำหรับ factory methods, constants

```scala
// Class และ Companion Object
class User private (
  val id: Int,
  val name: String,
  val email: String,
  val role: User.Role
) {
  def isAdmin: Boolean = role == User.Role.Admin
  
  override def toString: String = s"User($id, $name, $role)"
}

object User {
  // Nested type (enum-like)
  enum Role {
    case Admin, Editor, Viewer
  }
  
  // Counter for auto-increment ID
  private var nextId = 1
  
  // Factory methods
  def apply(name: String, email: String, role: Role = Role.Viewer): User = {
    val id = nextId
    nextId += 1
    new User(id, name, email, role)
  }
  
  def admin(name: String, email: String): User =
    apply(name, email, Role.Admin)
  
  def fromMap(data: Map[String, String]): Option[User] =
    for {
      name  <- data.get("name")
      email <- data.get("email")
      roleStr <- data.get("role")
      role  <- scala.util.Try(Role.valueOf(roleStr)).toOption
    } yield apply(name, email, role)
}

// Usage
val alice = User("Alice", "alice@example.com", User.Role.Admin)
val bob   = User("Bob",   "bob@example.com")     // default: Viewer
val admin = User.admin("Charlie", "charlie@example.com")

println(alice)         // User(1, Alice, Admin)
println(bob)           // User(2, Bob, Viewer)
println(admin)         // User(3, Charlie, Admin)
println(alice.isAdmin) // true

val fromData = User.fromMap(Map("name" -> "Diana", "email" -> "d@d.com", "role" -> "Editor"))
println(fromData)  // Some(User(4, Diana, Editor))
```

---

## Step 77 — apply และ unapply

```scala
// apply — เรียก object เหมือน function
class Multiplier(factor: Int) {
  def apply(x: Int): Int = x * factor  // apply method
}

val triple = new Multiplier(3)
println(triple(5))    // 15 — calls triple.apply(5)
println(triple(10))   // 30

// Object apply — factory pattern
object Fraction {
  def apply(num: Int, den: Int): Fraction = {
    val g = gcd(math.abs(num), math.abs(den))
    new Fraction(num / g, den / g)
  }
  
  private def gcd(a: Int, b: Int): Int =
    if (b == 0) a else gcd(b, a % b)
}

class Fraction private (val num: Int, val den: Int) {
  def +(other: Fraction): Fraction =
    Fraction(num * other.den + other.num * den, den * other.den)
  
  def *(other: Fraction): Fraction =
    Fraction(num * other.num, den * other.den)
  
  override def toString: String = s"$num/$den"
}

val half  = Fraction(1, 2)
val third = Fraction(1, 3)
println(half + third)  // 5/6
println(half * third)  // 1/6

// unapply — Pattern matching extractor
object Email {
  def unapply(s: String): Option[(String, String)] = {
    val parts = s.split("@")
    if (parts.length == 2) Some((parts(0), parts(1)))
    else None
  }
}

"alice@example.com" match {
  case Email(user, domain) => println(s"User: $user, Domain: $domain")
  case _                   => println("Not an email")
}
// User: alice, Domain: example.com
```

---

## Step 78 — Inheritance Preview

```scala
// Base class
class Animal(val name: String) {
  def sound(): String = "..."
  def describe(): String = s"$name says ${sound()}"
}

// Subclasses
class Dog(name: String) extends Animal(name) {
  override def sound(): String = "Woof"
  def fetch(): String = s"$name fetches the ball!"
}

class Cat(name: String) extends Animal(name) {
  override def sound(): String = "Meow"
  def purr(): String = s"$name is purring..."
}

class Duck(name: String) extends Animal(name) {
  override def sound(): String = "Quack"
}

val animals: List[Animal] = List(
  new Dog("Rex"),
  new Cat("Whiskers"),
  new Duck("Donald")
)

// Polymorphism
animals.foreach(a => println(a.describe()))
// Rex says Woof
// Whiskers says Meow
// Donald says Quack

// Type checking
animals.foreach {
  case d: Dog  => println(d.fetch())
  case c: Cat  => println(c.purr())
  case _       => ()
}
```

---

## Step 79 — Abstract Classes

```scala
// Abstract class — ไม่ instantiate ตรงๆ
abstract class Shape {
  // Abstract methods (ไม่มี implementation)
  def area: Double
  def perimeter: Double
  
  // Concrete method (มี implementation)
  def describe(): String =
    f"${getClass.getSimpleName}: area=${area}%.2f, perimeter=${perimeter}%.2f"
  
  // Template method pattern
  def printInfo(): Unit = {
    println(s"--- ${getClass.getSimpleName} ---")
    println(describe())
  }
}

class Circle(val radius: Double) extends Shape {
  override def area: Double = math.Pi * radius * radius
  override def perimeter: Double = 2 * math.Pi * radius
}

class Rectangle(val width: Double, val height: Double) extends Shape {
  override def area: Double = width * height
  override def perimeter: Double = 2 * (width + height)
}

class Triangle(val a: Double, val b: Double, val c: Double) extends Shape {
  override def perimeter: Double = a + b + c
  override def area: Double = {
    val s = perimeter / 2
    math.sqrt(s * (s - a) * (s - b) * (s - c))
  }
}

val shapes = List(
  new Circle(5.0),
  new Rectangle(4.0, 6.0),
  new Triangle(3.0, 4.0, 5.0)
)

shapes.foreach(_.printInfo())

// Find largest by area
val largest = shapes.maxBy(_.area)
println(s"\nLargest: ${largest.describe()}")
```

---

## Step 80 — โปรแกรม Classes Complete

```scala
// LibrarySystem.scala — Complete OOP example

import scala.collection.mutable

object LibrarySystem {

  // ==============================
  // Domain Classes
  // ==============================
  
  enum Genre {
    case Fiction, NonFiction, Science, History, Technology, Art
  }
  
  case class ISBN(value: String) {
    require(value.matches("\\d{13}"), s"Invalid ISBN-13: $value")
    override def toString = value.grouped(4).mkString("-")
  }
  
  class Book(
    val isbn: ISBN,
    val title: String,
    val author: String,
    val genre: Genre,
    val year: Int,
    private var copies: Int
  ) {
    private val borrowHistory = mutable.ListBuffer[String]()
    
    def availableCopies: Int = copies
    
    def borrow(member: String): Either[String, Unit] =
      if (copies > 0) {
        copies -= 1
        borrowHistory += s"$member borrowed on ${java.time.LocalDate.now()}"
        Right(())
      } else Left(s"No copies available for '$title'")
    
    def returnBook(member: String): Unit = {
      copies += 1
      borrowHistory += s"$member returned on ${java.time.LocalDate.now()}"
    }
    
    def history: List[String] = borrowHistory.toList
    
    override def toString: String =
      s"Book(${isbn}, '$title' by $author, $genre, $year, copies=$copies)"
  }
  
  class Member(val id: Int, val name: String, val email: String) {
    private val borrowedBooks = mutable.Set[ISBN]()
    private var borrowCount = 0
    
    val maxBorrow: Int = 5
    
    def canBorrow: Boolean = borrowedBooks.size < maxBorrow
    def hasBorrowed(isbn: ISBN): Boolean = borrowedBooks.contains(isbn)
    def totalBorrowed: Int = borrowCount
    
    private[LibrarySystem] def addBorrowed(isbn: ISBN): Unit = {
      borrowedBooks += isbn
      borrowCount += 1
    }
    
    private[LibrarySystem] def removeBorrowed(isbn: ISBN): Unit =
      borrowedBooks -= isbn
    
    def currentlyBorrowing: Set[ISBN] = borrowedBooks.toSet
    
    override def toString: String =
      s"Member($id, $name, borrowing=${borrowedBooks.size}/$maxBorrow)"
  }
  
  // ==============================
  // Library Manager
  // ==============================
  
  class Library(val name: String) {
    private val books   = mutable.Map[ISBN, Book]()
    private val members = mutable.Map[Int, Member]()
    private var nextMemberId = 1
    
    def addBook(book: Book): Unit = books(book.isbn) = book
    
    def registerMember(name: String, email: String): Member = {
      val m = new Member(nextMemberId, name, email)
      members(nextMemberId) = m
      nextMemberId += 1
      m
    }
    
    def borrow(memberId: Int, isbn: ISBN): Either[String, Unit] =
      for {
        member <- members.get(memberId).toRight(s"Member $memberId not found")
        book   <- books.get(isbn).toRight(s"Book $isbn not found")
        _      <- if (member.canBorrow) Right(()) else Left("Borrow limit reached")
        _      <- if (!member.hasBorrowed(isbn)) Right(()) else Left("Already borrowing this book")
        _      <- book.borrow(member.name)
      } yield {
        member.addBorrowed(isbn)
      }
    
    def returnBook(memberId: Int, isbn: ISBN): Either[String, Unit] =
      for {
        member <- members.get(memberId).toRight(s"Member $memberId not found")
        book   <- books.get(isbn).toRight(s"Book $isbn not found")
        _      <- if (member.hasBorrowed(isbn)) Right(()) else Left("Not borrowing this book")
      } yield {
        book.returnBook(member.name)
        member.removeBorrowed(isbn)
      }
    
    def search(query: String): List[Book] =
      books.values.filter { b =>
        b.title.toLowerCase.contains(query.toLowerCase) ||
        b.author.toLowerCase.contains(query.toLowerCase)
      }.toList.sortBy(_.title)
    
    def booksByGenre(genre: Genre): List[Book] =
      books.values.filter(_.genre == genre).toList.sortBy(_.title)
    
    def memberStatus(memberId: Int): Option[String] =
      members.get(memberId).map { m =>
        val borrowed = m.currentlyBorrowing.flatMap(books.get).map(_.title)
        s"$m\n  Borrowing: ${borrowed.mkString(", ")}"
      }
    
    def generateReport(): String = {
      val sb = new StringBuilder()
      sb.append(s"=== $name Library Report ===\n\n")
      sb.append(s"Books: ${books.size}, Members: ${members.size}\n\n")
      
      sb.append("Top Borrowed Books:\n")
      books.values.toList.sortBy(-_.totalBorrowed).take(5).foreach { b =>
        sb.append(f"  ${b.title}%-30s (${b.availableCopies} available)\n")
      }
      sb.toString()
    }
  }
  
  // Helper method
  extension (book: Book) {
    def totalBorrowed: Int = book.history.count(_.contains("borrowed"))
  }
}

// ==============================
// Main Program
// ==============================

@main def libraryDemo(): Unit = {
  import LibrarySystem._
  import Genre._
  
  val lib = new Library("Scala City Library")
  
  // Add books
  val books = List(
    new Book(ISBN("9780134685991"), "Effective Java", "Joshua Bloch", Technology, 2018, 3),
    new Book(ISBN("9780201633610"), "Design Patterns", "GoF", Technology, 1994, 2),
    new Book(ISBN("9781492080695"), "Programming Scala", "Dean Wampler", Technology, 2021, 4),
    new Book(ISBN("9780062316097"), "Sapiens", "Yuval Noah Harari", History, 2015, 5),
    new Book(ISBN("9780385472579"), "1984", "George Orwell", Fiction, 1949, 6)
  )
  books.foreach(lib.addBook)
  
  // Register members
  val alice   = lib.registerMember("Alice", "alice@example.com")
  val bob     = lib.registerMember("Bob", "bob@example.com")
  val charlie = lib.registerMember("Charlie", "charlie@example.com")
  
  println("=== Library System Demo ===\n")
  
  // Borrow books
  def tryBorrow(member: LibrarySystem.Member, isbn: String): Unit = {
    val i = ISBN(isbn)
    lib.borrow(member.id, i) match {
      case Right(_)    => println(s"✓ ${member.name} borrowed $isbn")
      case Left(error) => println(s"✗ ${member.name} failed: $error")
    }
  }
  
  tryBorrow(alice, "9780134685991")
  tryBorrow(alice, "9780201633610")
  tryBorrow(bob,   "9781492080695")
  tryBorrow(bob,   "9780134685991")  // 2nd copy
  tryBorrow(bob,   "9780134685991")  // already borrowing
  
  println()
  println(lib.memberStatus(alice.id).getOrElse("Not found"))
  println()
  println(lib.memberStatus(bob.id).getOrElse("Not found"))
  
  // Search
  println("\n--- Search 'Scala' ---")
  lib.search("Scala").foreach(b => println(s"  ${b.title} by ${b.author}"))
  
  // Return
  println("\n--- Returning ---")
  lib.returnBook(alice.id, ISBN("9780134685991")) match {
    case Right(_)  => println("✓ Returned")
    case Left(err) => println(s"✗ $err")
  }
  
  println("\n" + lib.generateReport())
}
```

---

## สรุป Part 08

| Step | สิ่งที่เรียน |
|------|-------------|
| 71 | Classes เบื้องต้น |
| 72 | Constructors (primary, secondary) |
| 73 | Fields และ Methods |
| 74 | Getters และ Setters |
| 75 | Object (Singleton) |
| 76 | Companion Objects |
| 77 | apply และ unapply |
| 78 | Inheritance preview |
| 79 | Abstract classes |
| 80 | Library system โปรแกรม complete |

## แบบฝึกหัด

1. สร้าง `Matrix` class ที่รองรับ +, -, * operators
2. สร้าง `Stack[A]` class ที่มี push, pop, peek operations
3. implement `PriorityQueue[A]` ด้วย min-heap
4. สร้าง `EventBus` singleton สำหรับ publish/subscribe pattern
5. เพิ่ม late fee calculation ในระบบ Library

## ต่อไป

**[Part 09 →](part-09-case-classes-and-traits.md)** — Case Classes และ Traits
