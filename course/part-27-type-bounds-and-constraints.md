# Part 27: Type Bounds and Constraints

## Steps 261-270: Upper/Lower Bounds, View Bounds, F-Bounded Polymorphism, Self Types, Structural Types

---

## Step 261: Upper Bounds ลึก

```scala
// T <: UpperBound — T must be subtype of UpperBound
object UpperBoundsDeep extends App {
  
  // Numeric computation with upper bound
  def sumList[A <: AnyVal](list: List[A])(implicit num: Numeric[A]): A =
    list.foldLeft(num.zero)(num.plus)
  
  println("=== Numeric Upper Bound ===")
  println(s"Sum ints: ${sumList(List(1, 2, 3, 4, 5))}")
  println(s"Sum doubles: ${sumList(List(1.5, 2.5, 3.0))}")
  
  // Collection-returning methods with upper bounds
  sealed trait JsonValue
  case class JsonInt(n: Int) extends JsonValue
  case class JsonString(s: String) extends JsonValue
  case class JsonList(items: List[JsonValue]) extends JsonValue
  
  def filterByType[A <: JsonValue : reflect.ClassTag](values: List[JsonValue]): List[A] =
    values.collect { case v: A => v }
  
  val mixed: List[JsonValue] = List(
    JsonInt(1), JsonString("hello"), JsonInt(2), 
    JsonString("world"), JsonInt(3)
  )
  
  println("\n=== Filter by Type ===")
  println(s"Ints: ${filterByType[JsonInt](mixed)}")
  println(s"Strings: ${filterByType[JsonString](mixed)}")
  
  // Upper bound with abstract class
  abstract class Transformer[A <: AnyRef, B <: AnyRef] {
    def transform(input: A): B
    def transformAll(inputs: List[A]): List[B] = inputs.map(transform)
  }
  
  class StringToLength extends Transformer[String, Integer] {
    def transform(input: String): Integer = Integer.valueOf(input.length)
  }
  
  val t = new StringToLength
  println(s"\n=== Transformer ===")
  println(s"Lengths: ${t.transformAll(List("hello", "world", "scala"))}")
  
  // Builder pattern with upper bound
  trait Builder[+T] {
    def build: T
  }
  
  class PersonBuilder {
    private var name: String = ""
    private var age: Int = 0
    private var email: String = ""
    
    def withName(n: String): this.type = { name = n; this }
    def withAge(a: Int): this.type = { age = a; this }
    def withEmail(e: String): this.type = { email = e; this }
    
    def build: (String, Int, String) = (name, age, email)
  }
  
  val person = new PersonBuilder()
    .withName("Alice")
    .withAge(30)
    .withEmail("alice@example.com")
    .build
  
  println(s"\n=== Builder ===")
  println(s"Person: $person")
}
```

---

## Step 262: Lower Bounds ลึก

```scala
object LowerBoundsDeep extends App {
  
  // Lower bound: T >: LowerBound — T must be supertype of LowerBound
  // Classic example: prepend to covariant list
  
  sealed trait Animal2
  case class Dog2(name: String) extends Animal2
  case class Cat2(name: String) extends Animal2
  case class GoldenRetriever(name: String) extends Dog2(name)
  
  // Without lower bound:
  // def prepend[A](item: A, list: List[A]): List[A] = item :: list
  
  // With lower bound — allows widening the type
  def addFirst[A, B >: A](item: B, list: List[A]): List[B] = item :: list
  
  val dogs: List[Dog2] = List(Dog2("Rex"), Dog2("Buddy"))
  val withCat: List[Animal2] = addFirst(Cat2("Whiskers"), dogs)  // Widened to Animal2
  
  println("=== Lower Bound Widening ===")
  println(s"dogs: $dogs")
  println(s"withCat: $withCat")
  
  // Lower bound in immutable Queue implementation
  class MyQueue[+A] private (
    private val in: List[A],
    private val out: List[A]
  ) {
    def enqueue[B >: A](item: B): MyQueue[B] = new MyQueue(item :: in, out)
    
    def dequeue: (A, MyQueue[A]) = out match {
      case head :: tail => (head, new MyQueue(in, tail))
      case Nil =>
        val reversed = in.reverse
        (reversed.head, new MyQueue(List.empty[A], reversed.tail))
    }
    
    def isEmpty: Boolean = in.isEmpty && out.isEmpty
    def size: Int = in.size + out.size
    
    override def toString: String = s"Queue(${(out ++ in.reverse).mkString(", ")})"
  }
  
  object MyQueue {
    def empty[A]: MyQueue[A] = new MyQueue(List.empty, List.empty)
    def apply[A](items: A*): MyQueue[A] = items.foldLeft(empty[A])(_.enqueue(_))
  }
  
  println("\n=== Functional Queue ===")
  val q1 = MyQueue(1, 2, 3)
  println(s"q1: $q1")
  
  val q2 = q1.enqueue(4).enqueue(5)
  println(s"q2: $q2")
  
  val (head, q3) = q2.dequeue
  println(s"dequeued: $head, remaining: $q3")
  
  // Lower bound for heterogeneous container widening
  val dogQ: MyQueue[Dog2] = MyQueue(Dog2("Rex"), Dog2("Buddy"))
  val animalQ: MyQueue[Animal2] = dogQ.enqueue(Cat2("Whiskers"))
  println(s"\nanimalQ: $animalQ")
}
```

---

## Step 263: F-Bounded Polymorphism

```scala
// F-Bounded: trait T[Self <: T[Self]] — self-referential type bound
// Used to return the actual subtype from methods in supertype

// Without F-bound (problem: returns wrong type)
trait Shape {
  def scale(factor: Double): Shape  // Returns Shape, not concrete type
}

// With F-bound (solution: returns concrete type)
trait FShape[Self <: FShape[Self]] {
  def scale(factor: Double): Self  // Returns Self type
  def translate(dx: Double, dy: Double): Self
}

case class FCircle(radius: Double, x: Double = 0, y: Double = 0) 
    extends FShape[FCircle] {
  def scale(factor: Double): FCircle = copy(radius = radius * factor)
  def translate(dx: Double, dy: Double): FCircle = copy(x = x + dx, y = y + dy)
  def area: Double = Math.PI * radius * radius
}

case class FRect(width: Double, height: Double, x: Double = 0, y: Double = 0)
    extends FShape[FRect] {
  def scale(factor: Double): FRect = copy(width = width * factor, height = height * factor)
  def translate(dx: Double, dy: Double): FRect = copy(x = x + dx, y = y + dy)
  def area: Double = width * height
}

// F-bound in type class
trait Comparable2[Self <: Comparable2[Self]] {
  def compareTo(other: Self): Int
  def lessThan(other: Self): Boolean = compareTo(other) < 0
  def greaterThan(other: Self): Boolean = compareTo(other) > 0
  def lessThanOrEqual(other: Self): Boolean = compareTo(other) <= 0
  def between(min: Self, max: Self): Boolean = !lessThan(min) && !greaterThan(max)
}

case class Version(major: Int, minor: Int, patch: Int) extends Comparable2[Version] {
  def compareTo(other: Version): Int = {
    val c1 = major.compare(other.major)
    if (c1 != 0) c1
    else {
      val c2 = minor.compare(other.minor)
      if (c2 != 0) c2
      else patch.compare(other.patch)
    }
  }
  override def toString: String = s"$major.$minor.$patch"
}

object FBoundedDemo extends App {
  
  println("=== F-Bounded Shape ===")
  val circle = FCircle(5.0)
  val scaled = circle.scale(2.0)  // Returns FCircle, not FShape
  val moved = scaled.translate(10, 20)
  println(s"original: $circle")
  println(s"scaled: $scaled, area=${scaled.area}")
  println(s"moved: $moved")
  
  val rect = FRect(4.0, 3.0)
  val bigRect = rect.scale(1.5).translate(5, 5)
  println(s"bigRect: $bigRect, area=${bigRect.area}")
  
  println("\n=== F-Bounded Comparable ===")
  val v1 = Version(1, 2, 3)
  val v2 = Version(1, 10, 0)
  val v3 = Version(2, 0, 0)
  
  println(s"$v1 < $v2: ${v1.lessThan(v2)}")
  println(s"$v3 > $v2: ${v3.greaterThan(v2)}")
  println(s"$v2 between $v1 and $v3: ${v2.between(v1, v3)}")
  
  // Sort versions
  val versions = List(v3, v1, v2, Version(1, 0, 0))
  implicit val versionOrdering: Ordering[Version] = Ordering.fromLessThan(_.lessThan(_))
  println(s"sorted: ${versions.sorted}")
}
```

---

## Step 264: Self Types

```scala
// Self type annotation: this: OtherType => 
// Declares that this trait requires another trait to be mixed in

trait Logger {
  def log(msg: String): Unit = println(s"[LOG] $msg")
  def logError(msg: String): Unit = println(s"[ERROR] $msg")
}

trait MetricsCollector {
  def recordMetric(name: String, value: Double): Unit =
    println(s"[METRIC] $name = $value")
}

trait UserRepository {
  this: Logger with MetricsCollector =>  // Requires Logger + MetricsCollector
  
  private var users = Map.empty[String, (String, Int)]
  
  def createUser(id: String, name: String, age: Int): Unit = {
    log(s"Creating user: $id")
    users = users + (id -> (name, age))
    recordMetric("users.created", 1.0)
    log(s"User created: $id")
  }
  
  def findUser(id: String): Option[(String, Int)] = {
    log(s"Finding user: $id")
    val result = users.get(id)
    recordMetric("users.lookup", 1.0)
    result
  }
  
  def allUsers: Map[String, (String, Int)] = users
}

// Can only instantiate with required traits
class UserService extends UserRepository with Logger with MetricsCollector

// Cake Pattern — dependency injection via self types
trait DatabaseModule {
  trait DatabaseDriver {
    def query(sql: String): List[String]
    def execute(sql: String): Int
  }
  
  val db: DatabaseDriver
}

trait UserModule { self: DatabaseModule =>
  class UserService2 {
    def getUsers: List[String] = self.db.query("SELECT * FROM users")
    def createUser2(name: String): Int = self.db.execute(s"INSERT INTO users VALUES('$name')")
  }
  
  val userService: UserService2 = new UserService2
}

trait OrderModule { self: DatabaseModule with UserModule =>
  class OrderService {
    def getOrders(userId: String): List[String] = 
      self.db.query(s"SELECT * FROM orders WHERE user_id='$userId'")
  }
  
  val orderService: OrderService = new OrderService
}

// Concrete wiring
object ProductionApp extends DatabaseModule with UserModule with OrderModule {
  val db: DatabaseDriver = new DatabaseDriver {
    def query(sql: String): List[String] = {
      println(s"  QUERY: $sql")
      List(s"result-1", s"result-2")
    }
    def execute(sql: String): Int = {
      println(s"  EXECUTE: $sql")
      1
    }
  }
}

object SelfTypesDemo extends App {
  
  println("=== Self Type (Repository) ===")
  val service = new UserService
  service.createUser("u1", "Alice", 30)
  service.createUser("u2", "Bob", 25)
  println(s"  Found: ${service.findUser("u1")}")
  println(s"  All users: ${service.allUsers}")
  
  println("\n=== Cake Pattern ===")
  val users = ProductionApp.userService.getUsers
  println(s"Users: $users")
  
  ProductionApp.userService.createUser2("Carol")
  
  val orders = ProductionApp.orderService.getOrders("u1")
  println(s"Orders: $orders")
}
```

---

## Step 265-270: Structural Types and Type Constraints

```scala
import scala.language.reflectiveCalls

// Structural types — duck typing
type Closeable = { def close(): Unit }
type Readable = { def read(): String }

def withResource[A <: Closeable](resource: A)(f: A => Unit): Unit = {
  try f(resource)
  finally resource.close()
}

class FileResource(name: String) {
  def read(): String = s"Contents of $name"
  def write(data: String): Unit = println(s"Writing to $name: $data")
  def close(): Unit = println(s"Closing $name")
}

class NetworkConnection(host: String) {
  def send(data: String): Unit = println(s"Sending to $host: $data")
  def receive(): String = s"Response from $host"
  def close(): Unit = println(s"Closing connection to $host")
}

// Type constraint combinations
def processAll[A](items: List[A])(
  implicit ev1: A <:< Product,    // A must be a Product (case class, tuple)
  ev2: A <:< Serializable         // A must be Serializable
): List[String] = items.map(_.productIterator.mkString(", "))

case class Config(host: String, port: Int)
case class Point2D(x: Double, y: Double)

// Generalized type constraints
def toDouble[A](value: A)(implicit ev: A =:= Int): Double = ev(value).toDouble

// Implicit evidence pattern
trait IsNumeric[A] {
  def toDouble(value: A): Double
  def fromDouble(d: Double): A
  def add(a: A, b: A): A
  def multiply(a: A, b: A): A
}

object IsNumeric {
  implicit val intNumeric: IsNumeric[Int] = new IsNumeric[Int] {
    def toDouble(n: Int): Double = n.toDouble
    def fromDouble(d: Double): Int = d.toInt
    def add(a: Int, b: Int): Int = a + b
    def multiply(a: Int, b: Int): Int = a * b
  }
  
  implicit val doubleNumeric: IsNumeric[Double] = new IsNumeric[Double] {
    def toDouble(d: Double): Double = d
    def fromDouble(d: Double): Double = d
    def add(a: Double, b: Double): Double = a + b
    def multiply(a: Double, b: Double): Double = a * b
  }
}

def dotProduct[A: IsNumeric](xs: List[A], ys: List[A]): A = {
  val num = implicitly[IsNumeric[A]]
  xs.zip(ys).map { case (x, y) => num.multiply(x, y) }
    .foldLeft(num.fromDouble(0.0))(num.add)
}

object StructuralTypesDemo extends App {
  
  println("=== Structural Types ===")
  withResource(new FileResource("data.txt")) { f =>
    println(s"Read: ${f.read()}")
    f.write("hello world")
  }
  
  withResource(new NetworkConnection("api.example.com")) { conn =>
    conn.send("GET /health")
    println(s"Received: ${conn.receive()}")
  }
  
  println("\n=== Type Constraint (Product) ===")
  val configs = List(Config("localhost", 8080), Config("prod.example.com", 443))
  val points = List(Point2D(1.0, 2.0), Point2D(3.0, 4.0))
  
  println(s"Configs: ${processAll(configs)}")
  println(s"Points: ${processAll(points)}")
  
  println("\n=== Custom Numeric Type Class ===")
  val xs = List(1, 2, 3)
  val ys = List(4, 5, 6)
  println(s"dot product: ${dotProduct(xs, ys)}")  // 1*4 + 2*5 + 3*6 = 32
  
  val xd = List(1.0, 2.0, 3.0)
  val yd = List(4.0, 5.0, 6.0)
  println(s"dot product (double): ${dotProduct(xd, yd)}")
  
  // =:= constraint
  val i: Int = 42
  println(s"\n=:= constraint: ${toDouble(i)}")
}
```

---

## สรุป Part 27

| Feature | Syntax | ใช้เมื่อ |
|---------|--------|---------|
| Upper bound | `[A <: T]` | A must be subtype of T |
| Lower bound | `[A >: T]` | A must be supertype of T |
| F-bounded | `[Self <: F[Self]]` | Methods return subtype |
| Self type | `this: T =>` | Require mixin dependency |
| View bound (deprecated) | `[A <% T]` | Use implicit conversion |
| Context bound | `[A: TC]` | Require type class instance |
| Type equality | `A =:= B` | Compile-time type equality |
| Subtype evidence | `A <:< B` | Compile-time subtype proof |
| Structural type | `{ def m: T }` | Duck typing |

---

## แบบฝึกหัด Part 27

**ข้อ 1:** Implement F-bounded `Cloneable[Self <: Cloneable[Self]]` trait พร้อม deep copy

**ข้อ 2:** สร้าง Cake Pattern สำหรับ e-commerce app: `ProductModule`, `CartModule`, `CheckoutModule`

**ข้อ 3:** Implement `Matrix[A: IsNumeric]` class พร้อม matrix multiplication

**ข้อ 4:** สร้าง type-safe SQL DSL ด้วย phantom types และ type bounds

**ข้อ 5:** Implement `Graph[V <: Vertex, E <: Edge]` with `shortestPath` using Dijkstra

---

➡️ ต่อไป: [Part 28 — Sealed Traits and ADTs](part-28-sealed-traits-and-adts.md)
