# Part 09 — Case Classes และ Traits
## Steps 81–90: Immutable Data และ Type-safe Interfaces

> **เป้าหมาย**: master case classes, traits, mixins, และ sealed hierarchies

---

## Step 81 — Case Classes

Case classes คือ immutable data classes ที่ Scala สร้าง methods ให้อัตโนมัติ

```scala
// Regular class vs Case class
class PersonClass(val name: String, val age: Int)
case class Person(name: String, age: Int)

// Case class ได้ automatic:
// 1. toString
// 2. equals / hashCode
// 3. copy
// 4. apply (factory)
// 5. unapply (pattern matching)

val alice = Person("Alice", 30)
val alice2 = Person("Alice", 30)

// toString
println(alice)        // Person(Alice,30)

// equals
println(alice == alice2)   // true (value equality!)
println(alice eq alice2)   // false (different objects)

// Regular class
val p1 = new PersonClass("Alice", 30)
val p2 = new PersonClass("Alice", 30)
println(p1 == p2)    // false (reference equality — ไม่ดี!)
println(p1.toString) // PersonClass@...hash (ไม่อ่านง่าย)

// hashCode
val set = Set(alice, alice2)
println(set.size)    // 1 (same hash, same equals)

// copy
val bob = alice.copy(name = "Bob")      // Person(Bob, 30)
val older = alice.copy(age = 31)        // Person(Alice, 31)
println(bob)    // Person(Bob,30)
println(older)  // Person(Alice,31)
```

### Case Class Fields

```scala
case class Product(
  id: Int,
  name: String,
  price: Double,
  category: String,
  inStock: Boolean = true  // default value
)

// Create
val laptop = Product(1, "MacBook Pro", 89000.0, "Electronics")
val chair  = Product(2, "Herman Miller", 45000.0, "Furniture", inStock = false)

// Access
println(laptop.name)     // MacBook Pro
println(laptop.price)    // 89000.0
println(laptop.inStock)  // true

// Update with copy
val discounted = laptop.copy(price = laptop.price * 0.9)
println(discounted)  // Product(1,MacBook Pro,80100.0,Electronics,true)

// Pattern matching
laptop match {
  case Product(_, name, price, "Electronics", true) =>
    println(f"$name: ฿$price%.2f (available)")
  case Product(_, name, _, _, false) =>
    println(s"$name is out of stock")
  case _ =>
    println("Other product")
}
```

---

## Step 82 — Case Class กับ Nested Structures

```scala
case class Address(
  street: String,
  city: String,
  country: String,
  zipCode: String
)

case class Contact(
  phone: Option[String],
  email: String,
  address: Address
)

case class Employee(
  id: Int,
  name: String,
  department: String,
  salary: Double,
  contact: Contact
)

// Create nested structure
val emp = Employee(
  id = 1001,
  name = "Alice Wonderland",
  department = "Engineering",
  salary = 150000.0,
  contact = Contact(
    phone = Some("02-555-1234"),
    email = "alice@company.com",
    address = Address(
      street = "123 Tech Street",
      city = "Bangkok",
      country = "Thailand",
      zipCode = "10110"
    )
  )
)

// Access nested
println(emp.contact.address.city)      // Bangkok
println(emp.contact.phone.getOrElse("N/A"))  // 02-555-1234

// Deep copy (nested update)
val promoted = emp.copy(
  salary = emp.salary * 1.2,
  contact = emp.contact.copy(
    address = emp.contact.address.copy(
      city = "Chiang Mai"
    )
  )
)

println(s"Old salary: ${emp.salary}")
println(s"New salary: ${promoted.salary}")
println(s"Old city: ${emp.contact.address.city}")
println(s"New city: ${promoted.contact.address.city}")
```

---

## Step 83 — Traits เบื้องต้น

Traits คือ interface + optional implementation (เหมือน interface ใน Java 8+)

```scala
// Basic trait
trait Greetable {
  def greet(): String
}

trait Farewell {
  def goodbye(): String = "Goodbye!"  // default implementation
}

// Class implements traits
class FriendlyPerson(val name: String) extends Greetable with Farewell {
  override def greet(): String = s"Hello, I'm $name!"
}

val fp = new FriendlyPerson("Alice")
println(fp.greet())    // Hello, I'm Alice!
println(fp.goodbye())  // Goodbye!

// Trait กับ abstract methods
trait Animal {
  def name: String
  def sound: String
  def breathe(): String = s"$name breathes"  // concrete
  def describe(): String = s"$name says $sound"  // concrete
}

class Dog(val name: String) extends Animal {
  def sound: String = "Woof"
  def fetch(): String = s"$name fetches!"
}

class Cat(val name: String) extends Animal {
  def sound: String = "Meow"
}

val rex = new Dog("Rex")
println(rex.describe())   // Rex says Woof
println(rex.breathe())    // Rex breathes
println(rex.fetch())      // Rex fetches!
```

---

## Step 84 — Trait Mixins

```scala
// Traits สำหรับ behavior composition
trait Serializable {
  def serialize(): String
}

trait Printable {
  def print(): Unit = println(serialize())
  def serialize(): String  // abstract — relies on other trait or class
}

trait Loggable {
  def log(msg: String): Unit = println(s"[LOG] $msg")
  def logInfo(msg: String): Unit = log(s"INFO: $msg")
  def logError(msg: String): Unit = log(s"ERROR: $msg")
}

trait Timestamped {
  def createdAt: java.time.Instant = java.time.Instant.now()
}

// Mixin multiple traits
class Event(val name: String, val data: Map[String, Any])
  extends Serializable
  with Printable
  with Loggable
  with Timestamped {
  
  override def serialize(): String =
    s"""{"event":"$name","data":${data.map { case (k,v) => s""""$k":"$v"""" }.mkString("{",",","}")}}"""
  
  def process(): Unit = {
    logInfo(s"Processing event: $name")
    print()
    logInfo("Done")
  }
}

val event = new Event("user.login", Map("userId" -> "123", "ip" -> "192.168.1.1"))
event.process()
// [LOG] INFO: Processing event: user.login
// {"event":"user.login","data":{"userId":"123","ip":"192.168.1.1"}}
// [LOG] INFO: Done
```

### Trait Stackable Modifications

```scala
abstract class Queue[A] {
  def enqueue(item: A): Unit
  def dequeue(): A
  def size: Int
}

import scala.collection.mutable

class BasicQueue[A] extends Queue[A] {
  private val items = mutable.Queue[A]()
  def enqueue(item: A): Unit = items.enqueue(item)
  def dequeue(): A = items.dequeue()
  def size: Int = items.size
}

// Stackable trait
trait LoggingQueue[A] extends Queue[A] {
  abstract override def enqueue(item: A): Unit = {
    println(s"  Enqueuing: $item")
    super.enqueue(item)
  }
  
  abstract override def dequeue(): A = {
    val item = super.dequeue()
    println(s"  Dequeued: $item")
    item
  }
}

trait BoundedQueue[A] extends Queue[A] {
  val maxSize: Int
  
  abstract override def enqueue(item: A): Unit = {
    if (size >= maxSize) throw new IllegalStateException(s"Queue full (max: $maxSize)")
    super.enqueue(item)
  }
}

// Mix them!
val q = new BasicQueue[Int] with LoggingQueue[Int] with BoundedQueue[Int] {
  val maxSize = 3
}

q.enqueue(1)  // Enqueuing: 1
q.enqueue(2)  // Enqueuing: 2
q.enqueue(3)  // Enqueuing: 3
// q.enqueue(4)  // IllegalStateException: Queue full

println(q.dequeue())  // Dequeued: 1  → 1
println(q.dequeue())  // Dequeued: 2  → 2
```

---

## Step 85 — Sealed Traits (ADTs)

Sealed trait = Algebraic Data Type — compiler รู้ทุก subtype

```scala
// Simple enum-like ADT
sealed trait Color
case object Red   extends Color
case object Green extends Color
case object Blue  extends Color
case class Custom(r: Int, g: Int, b: Int) extends Color

def toHex(color: Color): String = color match {
  case Red                => "#FF0000"
  case Green              => "#00FF00"
  case Blue               => "#0000FF"
  case Custom(r, g, b)    => f"#$r%02X$g%02X$b%02X"
}

// Compiler warns if case is missing!
println(toHex(Red))              // #FF0000
println(toHex(Custom(128, 0, 255)))  // #8000FF

// Rich ADT
sealed trait Json {
  def isNull: Boolean = this == JsonNull
}

case object JsonNull extends Json
case class JsonBool(value: Boolean) extends Json
case class JsonNumber(value: Double) extends Json
case class JsonString(value: String) extends Json
case class JsonArray(items: List[Json]) extends Json
case class JsonObject(fields: Map[String, Json]) extends Json

def jsonToString(json: Json, indent: Int = 0): String = {
  val spaces = "  " * indent
  json match {
    case JsonNull           => "null"
    case JsonBool(b)        => b.toString
    case JsonNumber(n)      =>
      if (n == n.toLong) n.toLong.toString else n.toString
    case JsonString(s)      => s""""${s.replace("\"", "\\\"")}""""
    case JsonArray(items)   =>
      if (items.isEmpty) "[]"
      else {
        val inner = items.map(i => s"${spaces}  ${jsonToString(i, indent+1)}")
        s"[\n${inner.mkString(",\n")}\n$spaces]"
      }
    case JsonObject(fields) =>
      if (fields.isEmpty) "{}"
      else {
        val inner = fields.map { case (k, v) =>
          s"""${spaces}  "$k": ${jsonToString(v, indent+1)}"""
        }
        s"{\n${inner.mkString(",\n")}\n$spaces}"
      }
  }
}

val json = JsonObject(Map(
  "name"    -> JsonString("Alice"),
  "age"     -> JsonNumber(30),
  "active"  -> JsonBool(true),
  "scores"  -> JsonArray(List(JsonNumber(95), JsonNumber(87), JsonNumber(92))),
  "address" -> JsonObject(Map(
    "city"    -> JsonString("Bangkok"),
    "country" -> JsonString("Thailand")
  ))
))

println(jsonToString(json))
```

---

## Step 86 — Enum (Scala 3)

```scala
// Scala 3 Enums — clean ADTs
enum Direction {
  case North, South, East, West
  
  def opposite: Direction = this match {
    case North => South
    case South => North
    case East  => West
    case West  => East
  }
  
  def toVector: (Int, Int) = this match {
    case North => (0, 1)
    case South => (0, -1)
    case East  => (1, 0)
    case West  => (-1, 0)
  }
}

println(Direction.North.opposite)    // South
println(Direction.values.mkString(", "))  // North, South, East, West
println(Direction.valueOf("East"))   // East

// Parameterized enum
enum Planet(val mass: Double, val radius: Double) {
  case Mercury extends Planet(3.303e+23, 2.4397e6)
  case Venus   extends Planet(4.869e+24, 6.0518e6)
  case Earth   extends Planet(5.976e+24, 6.37814e6)
  case Mars    extends Planet(6.421e+23, 3.3972e6)
  
  val G = 6.67300e-11
  
  def surfaceGravity: Double = G * mass / (radius * radius)
  
  def surfaceWeight(otherMass: Double): Double =
    otherMass * surfaceGravity
}

val earthWeight = 75.0
val mass = earthWeight / Planet.Earth.surfaceGravity

println("Weight on planets:")
Planet.values.foreach { p =>
  println(f"  ${p}%-10s ${p.surfaceWeight(mass)}%.2f N")
}
```

---

## Step 87 — Trait กับ Generic Types

```scala
// Generic trait
trait Container[A] {
  def get: A
  def map[B](f: A => B): Container[B]
  def flatMap[B](f: A => Container[B]): Container[B]
}

// Implement
class Box[A](private val value: A) extends Container[A] {
  def get: A = value
  def map[B](f: A => B): Box[B] = new Box(f(value))
  def flatMap[B](f: A => Container[B]): Container[B] = f(value)
  override def toString: String = s"Box($value)"
}

object Box {
  def apply[A](value: A): Box[A] = new Box(value)
  def empty[A]: Box[Option[A]] = new Box(None)
}

val b1 = Box(42)
val b2 = b1.map(_ * 2)
val b3 = b1.map(_.toString)

println(b1)  // Box(42)
println(b2)  // Box(84)
println(b3)  // Box(42)

// Functor-like trait
trait Functor[F[_]] {
  def map[A, B](fa: F[A])(f: A => B): F[B]
}

// Foldable trait
trait Foldable[F[_]] {
  def foldLeft[A, B](fa: F[A], zero: B)(f: (B, A) => B): B
  def toList[A](fa: F[A]): List[A] = foldLeft(fa, List.empty[A])(_ :+ _)
  def size[A](fa: F[A]): Int = foldLeft(fa, 0)((acc, _) => acc + 1)
  def sum[A](fa: F[A])(implicit num: Numeric[A]): A =
    foldLeft(fa, num.zero)(num.plus)
}
```

---

## Step 88 — Trait self-types

```scala
// Self-type annotation: requires another trait to be mixed in
trait UserService {
  def findUser(id: Int): Option[String]
}

trait EmailService {
  def sendEmail(to: String, subject: String): Unit
}

// Notification requires both UserService and EmailService
trait NotificationService { self: UserService with EmailService =>
  def notifyUser(userId: Int, message: String): Unit = {
    findUser(userId) match {
      case Some(email) =>
        sendEmail(email, message)
        println(s"Notified user $userId: $message")
      case None =>
        println(s"User $userId not found")
    }
  }
}

// Concrete implementation mixes everything
class AppService extends UserService
  with EmailService
  with NotificationService {
  
  private val users = Map(1 -> "alice@example.com", 2 -> "bob@example.com")
  
  def findUser(id: Int): Option[String] = users.get(id)
  
  def sendEmail(to: String, subject: String): Unit =
    println(s"[EMAIL] To: $to | Subject: $subject")
}

val app = new AppService()
app.notifyUser(1, "Welcome!")   // notifies alice
app.notifyUser(99, "Hello")     // user not found
```

---

## Step 89 — Case Classes เป็น Domain Model

```scala
// Domain-Driven Design with case classes

// Value Objects
case class Money(amount: BigDecimal, currency: String) {
  require(amount >= 0, "Amount cannot be negative")
  
  def +(other: Money): Money = {
    require(currency == other.currency, "Cannot add different currencies")
    Money(amount + other.amount, currency)
  }
  
  def *(factor: BigDecimal): Money = Money(amount * factor, currency)
  
  override def toString: String = f"${currency} ${amount}%.2f"
}

case class CustomerId(value: String) {
  require(value.nonEmpty, "Customer ID cannot be empty")
}

case class OrderId(value: Int) {
  require(value > 0, "Order ID must be positive")
}

// Entities
case class OrderItem(
  productId: Int,
  name: String,
  quantity: Int,
  unitPrice: Money
) {
  require(quantity > 0)
  def subtotal: Money = unitPrice * quantity
}

case class Order(
  id: OrderId,
  customerId: CustomerId,
  items: List[OrderItem],
  status: Order.Status
) {
  def total: Money = items.map(_.subtotal).reduce(_ + _)
  
  def addItem(item: OrderItem): Order =
    copy(items = items :+ item)
  
  def cancel: Either[String, Order] =
    if (status == Order.Status.Delivered) Left("Cannot cancel delivered order")
    else Right(copy(status = Order.Status.Cancelled))
}

object Order {
  enum Status {
    case Pending, Confirmed, Shipped, Delivered, Cancelled
  }
  
  def create(customerId: CustomerId, items: List[OrderItem]): Either[String, Order] =
    if (items.isEmpty) Left("Order must have at least one item")
    else Right(Order(
      id         = OrderId(scala.util.Random.nextInt(99999) + 1),
      customerId = customerId,
      items      = items,
      status     = Status.Pending
    ))
}

// Usage
val thb = (amount: BigDecimal) => Money(amount, "THB")

val order = Order.create(
  CustomerId("CUST-001"),
  List(
    OrderItem(1, "Laptop", 1, thb(45000)),
    OrderItem(2, "Mouse",  2, thb(500)),
    OrderItem(3, "Bag",    1, thb(1500))
  )
)

order match {
  case Right(o) =>
    println(s"Order ${o.id.value} created")
    o.items.foreach(i => println(f"  ${i.name}%-10s x${i.quantity} = ${i.subtotal}"))
    println(s"  Total: ${o.total}")
  case Left(err) => println(s"Error: $err")
}
```

---

## Step 90 — โปรแกรม Traits Complete

```scala
// GameEngine.scala — Trait-based game entity system

@main def gameEngineDemo(): Unit =
  
  println("=== Trait-based Game Engine ===\n")
  
  // ==============================
  // Component traits
  // ==============================
  
  trait Positioned {
    var x: Double
    var y: Double
    
    def moveTo(nx: Double, ny: Double): Unit = { x = nx; y = ny }
    def moveBy(dx: Double, dy: Double): Unit = { x += dx; y += dy }
    def distanceTo(other: Positioned): Double =
      math.sqrt(math.pow(other.x - x, 2) + math.pow(other.y - y, 2))
  }
  
  trait Renderable {
    def symbol: String
    def render(): String
  }
  
  trait Damageable {
    var health: Int
    val maxHealth: Int
    def isAlive: Boolean = health > 0
    def takeDamage(amount: Int): Unit = health = math.max(0, health - amount)
    def heal(amount: Int): Unit = health = math.min(maxHealth, health + amount)
    def healthBar: String = {
      val pct = health.toDouble / maxHealth
      val filled = (pct * 10).toInt
      "█" * filled + "░" * (10 - filled) + f" ${health}/$maxHealth"
    }
  }
  
  trait Attackable { self: Damageable =>
    val attackPower: Int
    def attack(target: Damageable): Unit = {
      val dmg = attackPower + scala.util.Random.nextInt(5)
      println(s"  → deals $dmg damage!")
      target.takeDamage(dmg)
    }
  }
  
  trait Inventoried {
    val inventory: scala.collection.mutable.Map[String, Int] = 
      scala.collection.mutable.Map.empty
    
    def addItem(item: String, qty: Int = 1): Unit =
      inventory(item) = inventory.getOrElse(item, 0) + qty
    
    def useItem(item: String): Boolean =
      inventory.get(item).exists { qty =>
        if (qty > 1) inventory(item) = qty - 1
        else inventory.remove(item)
        true
      }
  }
  
  // ==============================
  // Game entities
  // ==============================
  
  class Hero(name: String, initX: Double, initY: Double)
    extends Positioned
    with Damageable
    with Attackable
    with Inventoried
    with Renderable {
    
    var x = initX
    var y = initY
    var health = 100
    val maxHealth = 100
    val attackPower = 15
    val symbol = "⚔"
    
    def render(): String = s"$symbol $name [${healthBar}] at ($x,$y)"
    def levelUp(): Unit = println(s"$name leveled up!")
  }
  
  class Monster(kind: String, initX: Double, initY: Double, val difficulty: Int)
    extends Positioned
    with Damageable
    with Attackable
    with Renderable {
    
    var x = initX
    var y = initY
    var health = difficulty * 20
    val maxHealth = difficulty * 20
    val attackPower = difficulty * 5
    val symbol = if (difficulty >= 3) "👹" else "👾"
    val name = s"$kind (Lv.$difficulty)"
    
    def render(): String = s"$symbol $name [${healthBar}] at ($x,$y)"
  }
  
  // ==============================
  // Game loop
  // ==============================
  
  val hero = new Hero("Sir Scala", 0.0, 0.0)
  hero.addItem("Health Potion", 3)
  hero.addItem("Power Scroll")
  
  val monsters = List(
    new Monster("Goblin", 5.0, 3.0, 1),
    new Monster("Orc",    8.0, 1.0, 2),
    new Monster("Dragon", 3.0, 6.0, 4)
  )
  
  println("Initial state:")
  println(hero.render())
  monsters.foreach(m => println(m.render()))
  
  println("\n=== Battle ===")
  
  // Sort monsters by distance
  val sorted = monsters.sortBy(hero.distanceTo)
  
  sorted.foreach { monster =>
    if (hero.isAlive && monster.isAlive) {
      val dist = hero.distanceTo(monster)
      println(s"\nEncountering ${monster.name} (distance: $dist%.1f)")
      
      // Battle round
      var round = 1
      while (hero.isAlive && monster.isAlive && round <= 5) {
        print(s"Round $round: ${hero.symbol} attacks ${monster.symbol}")
        hero.attack(monster)
        
        if (monster.isAlive) {
          print(s"         ${monster.symbol} attacks back ${hero.symbol}")
          monster.attack(hero)
          
          // Use potion if low health
          if (hero.health < 30 && hero.useItem("Health Potion")) {
            hero.heal(40)
            println(s"  💊 Used Health Potion! HP: ${hero.health}")
          }
        }
        round += 1
      }
      
      if (!monster.isAlive) println(s"  ✓ ${monster.name} defeated!")
      else println(s"  ✗ ${monster.name} survived!")
    }
  }
  
  println("\n=== Final State ===")
  println(hero.render())
  println(s"Inventory: ${hero.inventory.toList.mkString(", ")}")
  
  println("\n=== Game Engine Demo Complete ===")
```

---

## สรุป Part 09

| Step | สิ่งที่เรียน |
|------|-------------|
| 81 | Case classes และ automatic methods |
| 82 | Nested case classes |
| 83 | Traits เบื้องต้น |
| 84 | Trait mixins และ stackable |
| 85 | Sealed traits (ADTs) |
| 86 | Enum (Scala 3) |
| 87 | Generic traits |
| 88 | Self-types |
| 89 | Domain modeling |
| 90 | Game engine (traits complete) |

## แบบฝึกหัด

1. สร้าง `Tree[A]` ADT พร้อม `map`, `fold`, `flatten`
2. สร้าง validation framework ด้วย traits
3. implement `Observable` pattern ด้วย traits
4. สร้าง `Parser[A]` monad สำหรับ text parsing
5. เพิ่ม magic items และ spell system ใน game engine

## ต่อไป

**[Part 10 →](part-10-build-tools-and-sbt.md)** — SBT Build Tool และ Project Structure
