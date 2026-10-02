# Part 11: OOP Inheritance และ Polymorphism ใน Scala

## Steps 101-110: Inheritance, Polymorphism, Method Overriding, Super, Abstract Override, Final

---

## Step 101: พื้นฐาน Inheritance ใน Scala

Inheritance (การสืบทอด) เป็นหัวใจสำคัญของ OOP ใน Scala ใช้ keyword `extends` เพื่อสืบทอด class

```scala
// Base class - คลาสหลักที่จะถูกสืบทอด
class Animal(val name: String, val age: Int) {
  
  // Regular method - method ปกติที่สามารถ override ได้
  def speak(): String = s"$name makes a sound"
  
  // Method ที่แสดงข้อมูล
  def describe(): String = s"$name is $age years old"
  
  override def toString: String = s"Animal($name, age=$age)"
}

// Subclass - คลาสที่สืบทอดจาก Animal
class Dog(name: String, age: Int, val breed: String) 
    extends Animal(name, age) {
  
  // Override method จาก parent class
  override def speak(): String = s"$name says: Woof!"
  
  // Method เพิ่มเติมที่ Dog มีเองแต่ Animal ไม่มี
  def fetch(): String = s"$name fetches the ball!"
  
  override def toString: String = s"Dog($name, breed=$breed)"
}

class Cat(name: String, age: Int, val indoor: Boolean) 
    extends Animal(name, age) {
  
  override def speak(): String = s"$name says: Meow!"
  
  def purr(): String = if (indoor) s"$name purrs softly" else s"$name purrs loudly"
  
  override def toString: String = s"Cat($name, indoor=$indoor)"
}

// ทดสอบการใช้งาน
object InheritanceBasicsDemo extends App {
  val animal = new Animal("Generic", 5)
  val dog = new Dog("Rex", 3, "Labrador")
  val cat = new Cat("Whiskers", 2, true)
  
  println("=== Basic Inheritance ===")
  println(animal.speak())       // Generic makes a sound
  println(dog.speak())          // Rex says: Woof!
  println(cat.speak())          // Whiskers says: Meow!
  
  println("\n=== Inherited Methods ===")
  println(dog.describe())       // Rex is 3 years old (inherited from Animal)
  println(cat.describe())       // Whiskers is 2 years old
  
  println("\n=== Subclass-specific Methods ===")
  println(dog.fetch())          // Rex fetches the ball!
  println(cat.purr())           // Whiskers purrs softly
  
  println("\n=== toString ===")
  println(animal)               // Animal(Generic, age=5)
  println(dog)                  // Dog(Rex, breed=Labrador)
  println(cat)                  // Cat(Whiskers, indoor=true)
}
```

---

## Step 102: Polymorphism — ความหลากหลายรูปแบบ

Polymorphism ช่วยให้เราสามารถใช้งาน object ของ subclass ผ่าน reference ของ parent class ได้

```scala
// Polymorphism demonstration
class Shape {
  def area(): Double = 0.0
  def perimeter(): Double = 0.0
  def describe(): String = s"Shape: area=${area():.2f}, perimeter=${perimeter():.2f}"
}

class Circle(val radius: Double) extends Shape {
  override def area(): Double = Math.PI * radius * radius
  override def perimeter(): Double = 2 * Math.PI * radius
  override def describe(): String = s"Circle(r=$radius): area=${area():.2f}"
}

class Rectangle(val width: Double, val height: Double) extends Shape {
  override def area(): Double = width * height
  override def perimeter(): Double = 2 * (width + height)
  override def describe(): String = s"Rectangle(${width}x${height}): area=${area():.2f}"
}

class Triangle(val a: Double, val b: Double, val c: Double) extends Shape {
  // Heron's formula
  override def area(): Double = {
    val s = (a + b + c) / 2
    Math.sqrt(s * (s - a) * (s - b) * (s - c))
  }
  override def perimeter(): Double = a + b + c
  override def describe(): String = s"Triangle($a,$b,$c): area=${area():.2f}"
}

object PolymorphismDemo extends App {
  // Polymorphic collection - เก็บ Shape ได้หลายชนิด
  val shapes: List[Shape] = List(
    new Circle(5.0),
    new Rectangle(4.0, 6.0),
    new Triangle(3.0, 4.0, 5.0),
    new Circle(3.0),
    new Rectangle(10.0, 2.0)
  )
  
  println("=== Polymorphism - Shapes ===")
  shapes.foreach(s => println(s.describe()))
  
  // Calculate total area polymorphically
  val totalArea = shapes.map(_.area()).sum
  println(f"\nTotal area: $totalArea%.2f")
  
  // Find largest shape
  val largest = shapes.maxBy(_.area())
  println(s"Largest shape: ${largest.describe()}")
  
  // Filter circles only using pattern matching
  val circles = shapes.collect { case c: Circle => c }
  println(s"\nCircles: ${circles.map(c => s"r=${c.radius}").mkString(", ")}")
  
  // Sort by area
  val sorted = shapes.sortBy(_.area())
  println("\nShapes sorted by area:")
  sorted.foreach(s => println(s"  ${s.describe()}"))
}
```

---

## Step 103: Method Overriding กฎและ Best Practices

```scala
// Method overriding rules and best practices
class Vehicle(val make: String, val model: String, val year: Int) {
  
  // Method ที่ตั้งใจให้ override - ใช้ open design
  def fuelEfficiency(): String = "Unknown fuel efficiency"
  
  // Method ที่มีค่า default แต่ subclass อาจต้องการเปลี่ยน
  def startEngine(): String = s"$make $model engine starting..."
  
  def info(): String = s"$year $make $model"
  
  // toString override
  override def toString: String = info()
}

class ElectricCar(make: String, model: String, year: Int, val batteryKWh: Double) 
    extends Vehicle(make, model, year) {
  
  override def fuelEfficiency(): String = 
    f"${batteryKWh / 400 * 100}%.1f kWh per 100km"
  
  override def startEngine(): String = 
    s"$make $model electric motor activating silently..."
  
  // Additional method
  def chargeStatus(): String = s"Battery: ${batteryKWh}kWh capacity"
  
  override def info(): String = 
    s"${super.info()} [Electric, ${batteryKWh}kWh]"
}

class GasCar(make: String, model: String, year: Int, val engineCC: Int) 
    extends Vehicle(make, model, year) {
  
  override def fuelEfficiency(): String = 
    s"Approximately ${15000 / engineCC * 10} km/L"
  
  override def startEngine(): String = {
    val sound = if (engineCC > 2000) "VROOM!" else "vroom..."
    s"$make $model (${engineCC}cc) - $sound"
  }
  
  override def info(): String = 
    s"${super.info()} [Gas, ${engineCC}cc]"
}

class HybridCar(make: String, model: String, year: Int, 
               val engineCC: Int, val batteryKWh: Double) 
    extends Vehicle(make, model, year) {
  
  override def fuelEfficiency(): String = 
    s"~${engineCC / 100}km/L (hybrid mode)"
  
  override def startEngine(): String = 
    s"$make $model hybrid system starting (electric first)..."
  
  override def info(): String = 
    s"${super.info()} [Hybrid, ${engineCC}cc + ${batteryKWh}kWh]"
}

object MethodOverridingDemo extends App {
  val vehicles: List[Vehicle] = List(
    new ElectricCar("Tesla", "Model 3", 2024, 75.0),
    new GasCar("Toyota", "Camry", 2023, 2500),
    new HybridCar("Toyota", "Prius", 2024, 1800, 8.8),
    new ElectricCar("BYD", "Seal", 2024, 82.5),
    new GasCar("Honda", "Civic", 2023, 1500)
  )
  
  println("=== Vehicle Information ===")
  vehicles.foreach { v =>
    println(s"\n${v.info()}")
    println(s"  Efficiency: ${v.fuelEfficiency()}")
    println(s"  Start: ${v.startEngine()}")
  }
  
  // Find electric cars
  val electrics = vehicles.collect { case e: ElectricCar => e }
  println(s"\n=== Electric Cars (${electrics.size}) ===")
  electrics.foreach(e => println(s"  ${e.info()}, Battery: ${e.batteryKWh}kWh"))
}
```

---

## Step 104: Super Keyword — เรียกใช้ Method จาก Parent

```scala
// Using super keyword
class Logger {
  def log(message: String): Unit = {
    println(s"[LOG] $message")
  }
  
  def format(message: String): String = message
}

class TimestampLogger extends Logger {
  import java.time.LocalDateTime
  import java.time.format.DateTimeFormatter
  
  private val formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")
  
  override def format(message: String): String = {
    val timestamp = LocalDateTime.now().format(formatter)
    s"[$timestamp] $message"
  }
  
  // เรียก super เพื่อใช้งาน method ของ parent
  override def log(message: String): Unit = {
    super.log(format(message))  // เพิ่ม timestamp แล้วส่งต่อไป parent
  }
}

class LevelLogger(val level: String) extends TimestampLogger {
  
  override def format(message: String): String = {
    // Chain: เรียก super (TimestampLogger.format) แล้วเพิ่ม level
    s"[$level] ${super.format(message)}"
  }
  
  // Convenience methods
  def info(msg: String): Unit = {
    val savedLevel = level  // capture for closure
    log(s"INFO: $msg")
  }
  
  def error(msg: String): Unit = log(s"ERROR: $msg")
  def warn(msg: String): Unit = log(s"WARN: $msg")
}

// Super in constructors
class Person(val name: String, val age: Int) {
  def greet(): String = s"Hi, I'm $name"
}

class Employee(name: String, age: Int, val company: String, val role: String) 
    extends Person(name, age) {  // Calling parent constructor
  
  override def greet(): String = 
    s"${super.greet()}, I work at $company as $role"
  
  def businessCard(): String = 
    s"$name | $role @ $company | Age: $age"
}

class Manager(name: String, age: Int, company: String, 
             val teamSize: Int) 
    extends Employee(name, age, company, "Manager") {
  
  override def greet(): String = 
    s"${super.greet()}, managing a team of $teamSize"
}

object SuperKeywordDemo extends App {
  val logger = new LevelLogger("DEBUG")
  logger.log("Application started")
  logger.log("Processing data...")
  
  println()
  
  val person = new Person("Alice", 30)
  val employee = new Employee("Bob", 35, "TechCorp", "Engineer")
  val manager = new Manager("Carol", 40, "BigCo", 8)
  
  println("=== Super in Action ===")
  println(person.greet())
  println(employee.greet())
  println(manager.greet())
  
  println("\n=== Business Cards ===")
  println(employee.businessCard())
  println(manager.businessCard())  // inherited from Employee
}
```

---

## Step 105: Abstract Classes และ Abstract Methods

```scala
// Abstract classes - ไม่สามารถสร้าง instance ได้โดยตรง
abstract class DatabaseConnector(val host: String, val port: Int) {
  
  // Abstract methods - ต้อง implement ใน subclass
  def connect(): Boolean
  def disconnect(): Unit
  def query(sql: String): List[Map[String, Any]]
  def execute(sql: String): Int
  
  // Concrete methods - มี implementation แล้ว
  def connectionString(): String = s"$host:$port"
  
  def withConnection[T](block: => T): T = {
    val connected = connect()
    if (!connected) throw new RuntimeException(s"Cannot connect to $connectionString()")
    try {
      block
    } finally {
      disconnect()
    }
  }
  
  def queryFirst(sql: String): Option[Map[String, Any]] = {
    query(sql).headOption
  }
  
  override def toString: String = s"${getClass.getSimpleName}($connectionString())"
}

// Concrete implementation for PostgreSQL
class PostgreSQLConnector(host: String, port: Int, val database: String) 
    extends DatabaseConnector(host, port) {
  
  private var connected = false
  
  override def connect(): Boolean = {
    // Simulating connection
    println(s"Connecting to PostgreSQL: $host:$port/$database")
    connected = true
    connected
  }
  
  override def disconnect(): Unit = {
    println(s"Disconnecting from PostgreSQL: $host:$port/$database")
    connected = false
  }
  
  override def query(sql: String): List[Map[String, Any]] = {
    if (!connected) throw new RuntimeException("Not connected!")
    println(s"PostgreSQL Query: $sql")
    // Simulated results
    List(
      Map("id" -> 1, "name" -> "Alice", "db" -> database),
      Map("id" -> 2, "name" -> "Bob", "db" -> database)
    )
  }
  
  override def execute(sql: String): Int = {
    if (!connected) throw new RuntimeException("Not connected!")
    println(s"PostgreSQL Execute: $sql")
    1 // rows affected
  }
  
  override def connectionString(): String = 
    s"postgresql://$host:$port/$database"
}

// Mock implementation for testing
class MockDatabaseConnector extends DatabaseConnector("localhost", 0) {
  private val mockData = Map(
    "SELECT * FROM users" -> List(
      Map("id" -> 1, "name" -> "Test User"),
      Map("id" -> 2, "name" -> "Another User")
    )
  )
  
  override def connect(): Boolean = true
  override def disconnect(): Unit = ()
  
  override def query(sql: String): List[Map[String, Any]] = {
    mockData.getOrElse(sql, List.empty)
  }
  
  override def execute(sql: String): Int = 1
}

object AbstractClassDemo extends App {
  val pgConn = new PostgreSQLConnector("db.example.com", 5432, "myapp")
  
  println("=== PostgreSQL Connection ===")
  pgConn.withConnection {
    val results = pgConn.query("SELECT * FROM users")
    results.foreach(row => println(s"  Row: $row"))
    
    val first = pgConn.queryFirst("SELECT * FROM users WHERE id = 1")
    println(s"First: $first")
  }
  
  println("\n=== Mock Connection (for testing) ===")
  val mockConn = new MockDatabaseConnector()
  mockConn.withConnection {
    val results = mockConn.query("SELECT * FROM users")
    results.foreach(row => println(s"  Row: $row"))
  }
  
  // Polymorphism with abstract class
  val connectors: List[DatabaseConnector] = List(pgConn, mockConn)
  println("\n=== All Connectors ===")
  connectors.foreach(c => println(s"  $c"))
}
```

---

## Step 106: Abstract Override — Trait Stacking

```scala
// Abstract override สำหรับ Trait stacking (Stackable Traits)
trait Notification {
  def send(message: String): Unit
}

trait EmailNotification extends Notification {
  abstract override def send(message: String): Unit = {
    println(s"[Email] $message")
    super.send(message)
  }
}

trait SMSNotification extends Notification {
  abstract override def send(message: String): Unit = {
    println(s"[SMS] ${message.take(160)}")  // SMS limit
    super.send(message)
  }
}

trait SlackNotification extends Notification {
  abstract override def send(message: String): Unit = {
    println(s"[Slack] :bell: $message")
    super.send(message)
  }
}

trait LoggingNotification extends Notification {
  abstract override def send(message: String): Unit = {
    println(s"[LOG] Sending notification: $message")
    super.send(message)
  }
}

// Base implementation
class BaseNotification extends Notification {
  override def send(message: String): Unit = {
    println(s"[Base] Notification sent: $message")
  }
}

// Stack traits - ลำดับสำคัญมาก! ขวาไปซ้าย
object AbstractOverrideDemo extends App {
  println("=== Email + SMS Notification ===")
  val emailSms = new BaseNotification with EmailNotification with SMSNotification
  emailSms.send("Your order has been shipped!")
  
  println("\n=== All Channels ===")
  val allChannels = new BaseNotification 
    with LoggingNotification 
    with SlackNotification 
    with SMSNotification 
    with EmailNotification
  allChannels.send("Critical system alert!")
  
  println("\n=== Slack Only ===")
  val slackOnly = new BaseNotification with SlackNotification
  slackOnly.send("Meeting in 5 minutes")
}
```

---

## Step 107: Final — ป้องกันการ Override

```scala
// final keyword ป้องกัน override
class SecurityService {
  
  // final method ไม่สามารถ override ได้
  final def authenticate(token: String): Boolean = {
    // Critical security logic - ห้าม override เพื่อความปลอดภัย
    token.length > 10 && token.startsWith("Bearer ")
  }
  
  // final def validate(...)  // ถ้า uncomment แล้วทำ override จะ compile error
  
  // Non-final method - สามารถ override ได้
  def authorize(userId: String, resource: String): Boolean = {
    println(s"Default authorization for $userId -> $resource")
    true
  }
}

// final class ไม่สามารถ extend ได้เลย
final class ImmutablePoint(val x: Double, val y: Double) {
  def distanceTo(other: ImmutablePoint): Double = {
    val dx = x - other.x
    val dy = y - other.y
    Math.sqrt(dx * dx + dy * dy)
  }
  
  def translate(dx: Double, dy: Double): ImmutablePoint = 
    new ImmutablePoint(x + dx, y + dy)
  
  override def toString: String = f"Point($x%.2f, $y%.2f)"
}

// class SubPoint extends ImmutablePoint(0, 0)  // Compile error!

// final val ใน trait
trait Constants {
  final val PI = 3.14159265358979
  final val E = 2.71828182845904
  final val GoldenRatio = 1.61803398874989
  
  // Regular val - สามารถ override ได้
  val defaultPrecision: Int = 6
}

class MathCalculator extends Constants {
  // override val PI = 3.14  // Compile error! PI is final
  override val defaultPrecision: Int = 10  // This is OK
  
  def circleArea(r: Double): Double = PI * r * r
  def sphereVolume(r: Double): Double = (4.0 / 3.0) * PI * r * r * r
}

object FinalDemo extends App {
  val security = new SecurityService()
  println("=== Final Methods ===")
  println(security.authenticate("Bearer validToken123"))  // true
  println(security.authenticate("short"))                  // false
  
  val p1 = new ImmutablePoint(0, 0)
  val p2 = new ImmutablePoint(3, 4)
  println(s"\n=== Final Class ===")
  println(s"Distance: ${p1.distanceTo(p2)}")  // 5.0
  println(s"Translated: ${p1.translate(1, 2)}")
  
  val calc = new MathCalculator()
  println(s"\n=== Final Val in Trait ===")
  println(f"Circle area (r=5): ${calc.circleArea(5):.4f}")
  println(f"Sphere volume (r=3): ${calc.sphereVolume(3):.4f}")
}
```

---

## Step 108: Inheritance กับ Constructor Chaining

```scala
// Constructor chaining in inheritance
class Vehicle2(val id: String, val make: String) {
  println(s"Vehicle2 constructor: $id")
  
  def describe(): String = s"Vehicle $id by $make"
}

class Car(id: String, make: String, val model: String, val year: Int) 
    extends Vehicle2(id, make) {  // Calling parent constructor
  println(s"Car constructor: $id $model")
  
  override def describe(): String = 
    s"${super.describe()} - $model ($year)"
}

class SportsCar(id: String, make: String, model: String, year: Int, 
               val horsepower: Int) 
    extends Car(id, make, model, year) {
  println(s"SportsCar constructor: $id ${horsepower}hp")
  
  override def describe(): String = 
    s"${super.describe()} [${horsepower}hp]"
  
  def topSpeed(): Int = 100 + (horsepower / 10)
}

// Multiple constructor params and auxiliary constructors
class DatabaseConfig(
  val host: String,
  val port: Int,
  val database: String,
  val username: String,
  val password: String,
  val maxConnections: Int
) {
  def connectionUrl: String = s"jdbc:postgresql://$host:$port/$database"
}

class ProductionDbConfig(host: String, database: String) 
    extends DatabaseConfig(
      host = host,
      port = 5432,
      database = database,
      username = "prod_user",
      password = sys.env.getOrElse("DB_PASSWORD", "default"),
      maxConnections = 20
    ) {
  
  def isSecure: Boolean = host.endsWith(".rds.amazonaws.com")
}

class TestDbConfig 
    extends DatabaseConfig("localhost", 5432, "test_db", "test", "test", 5) {
  
  override def connectionUrl: String = super.connectionUrl + "?sslmode=disable"
}

object ConstructorChainingDemo extends App {
  println("=== Constructor Chain ===")
  val sports = new SportsCar("SC001", "Ferrari", "F8", 2024, 720)
  println(sports.describe())
  println(s"Top speed: ~${sports.topSpeed()} km/h")
  
  println("\n=== Database Configs ===")
  val prod = new ProductionDbConfig("myapp.rds.amazonaws.com", "production")
  val test = new TestDbConfig()
  
  println(s"Prod URL: ${prod.connectionUrl}")
  println(s"Prod secure: ${prod.isSecure}")
  println(s"Test URL: ${test.connectionUrl}")
  println(s"Max connections - Prod: ${prod.maxConnections}, Test: ${test.maxConnections}")
}
```

---

## Step 109: Type Checking และ Type Casting

```scala
// Type checking with isInstanceOf and asInstanceOf (and pattern matching)
abstract class Payment {
  def amount: Double
  def currency: String = "THB"
  def process(): String
}

case class CreditCardPayment(
  amount: Double, 
  cardNumber: String, 
  expiryMonth: Int, 
  expiryYear: Int
) extends Payment {
  override def process(): String = 
    s"Credit card $maskedCard charged $amount $currency"
  
  private def maskedCard: String = 
    "**** **** **** " + cardNumber.takeRight(4)
  
  def isExpired: Boolean = {
    val now = java.time.YearMonth.now()
    java.time.YearMonth.of(expiryYear, expiryMonth).isBefore(now)
  }
}

case class BankTransferPayment(
  amount: Double,
  bankCode: String,
  accountNumber: String
) extends Payment {
  override def process(): String = 
    s"Bank transfer $amount $currency to $bankCode/$maskedAccount"
  
  private def maskedAccount: String = 
    accountNumber.dropRight(4).map(_ => '*') + accountNumber.takeRight(4)
}

case class CryptoPayment(
  amount: Double,
  override val currency: String,
  walletAddress: String
) extends Payment {
  override def process(): String = 
    s"Crypto payment $amount $currency to ${walletAddress.take(10)}..."
  
  def amountInTHB(rate: Double): Double = amount * rate
}

object TypeCheckingDemo extends App {
  val payments: List[Payment] = List(
    CreditCardPayment(1500.0, "4532015112830366", 12, 2025),
    BankTransferPayment(5000.0, "KBANK", "1234567890"),
    CryptoPayment(0.05, "BTC", "1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2"),
    CreditCardPayment(3000.0, "5425233430109903", 6, 2024),
    BankTransferPayment(10000.0, "SCB", "9876543210")
  )
  
  println("=== Processing Payments ===")
  payments.foreach(p => println(p.process()))
  
  // Type checking with isInstanceOf (old style - ไม่แนะนำ)
  println("\n=== Type Checking (old style) ===")
  payments.foreach { p =>
    if (p.isInstanceOf[CreditCardPayment]) {
      val cc = p.asInstanceOf[CreditCardPayment]
      if (cc.isExpired) println(s"WARNING: Card expired! ${cc.process()}")
    }
  }
  
  // Better: Pattern matching (recommended)
  println("\n=== Type Checking (pattern matching) ===")
  payments.foreach {
    case cc: CreditCardPayment if cc.isExpired =>
      println(s"EXPIRED CARD: ${cc.process()}")
    case cc: CreditCardPayment =>
      println(s"Valid CC: ${cc.process()}")
    case bt: BankTransferPayment =>
      println(s"Bank: ${bt.process()}")
    case cp: CryptoPayment =>
      val thbAmount = cp.amountInTHB(1500000.0) // BTC rate
      println(f"Crypto: ${cp.process()} (~$thbAmount%.0f THB)")
  }
  
  // Statistics by type
  val ccTotal = payments.collect { case p: CreditCardPayment => p.amount }.sum
  val btTotal = payments.collect { case p: BankTransferPayment => p.amount }.sum
  
  println(s"\n=== Totals ===")
  println(f"Credit Card: $ccTotal%.2f THB")
  println(f"Bank Transfer: $btTotal%.2f THB")
}
```

---

## Step 110: Real-World Inheritance — Employee Hierarchy

```scala
// Real-world example: Company employee hierarchy
import scala.util.{Try, Success, Failure}

abstract class Employee(
  val id: String,
  val name: String,
  val department: String
) {
  def baseSalary: Double
  def calculateBonus(): Double
  
  def totalCompensation: Double = baseSalary + calculateBonus()
  
  def payslip(): String = {
    f"""
    |=== PAYSLIP ===
    |Employee: $name ($id)
    |Department: $department
    |Base Salary: $baseSalary%.2f THB
    |Bonus: ${calculateBonus()}%.2f THB
    |Total: $totalCompensation%.2f THB
    |""".stripMargin
  }
  
  override def toString: String = s"$name ($id) - $department"
}

class SalariedEmployee(
  id: String, name: String, department: String,
  override val baseSalary: Double
) extends Employee(id, name, department) {
  
  override def calculateBonus(): Double = baseSalary * 0.10 // 10% bonus
}

class HourlyEmployee(
  id: String, name: String, department: String,
  val hourlyRate: Double,
  val hoursWorked: Double
) extends Employee(id, name, department) {
  
  override val baseSalary: Double = hourlyRate * hoursWorked
  
  override def calculateBonus(): Double = {
    if (hoursWorked > 160) (hoursWorked - 160) * hourlyRate * 1.5
    else 0.0
  }
}

class CommissionEmployee(
  id: String, name: String, department: String,
  override val baseSalary: Double,
  val salesAmount: Double,
  val commissionRate: Double
) extends Employee(id, name, department) {
  
  override def calculateBonus(): Double = salesAmount * commissionRate
}

class ExecutiveEmployee(
  id: String, name: String, department: String,
  override val baseSalary: Double,
  val stockOptions: Int,
  val stockPrice: Double
) extends Employee(id, name, department) {
  
  override def calculateBonus(): Double = baseSalary * 0.30 // 30% bonus
  
  def stockValue: Double = stockOptions * stockPrice
  
  override def totalCompensation: Double = 
    super.totalCompensation + stockValue
  
  override def payslip(): String = 
    super.payslip() + f"\nStock Options Value: $stockValue%.2f THB\n"
}

// HR System
class HRSystem(val employees: List[Employee]) {
  
  def totalPayroll(): Double = employees.map(_.totalCompensation).sum
  
  def departmentSummary(): Map[String, Double] = {
    employees.groupBy(_.department).map { case (dept, emps) =>
      dept -> emps.map(_.totalCompensation).sum
    }
  }
  
  def topEarners(n: Int): List[Employee] = {
    employees.sortBy(-_.totalCompensation).take(n)
  }
  
  def findById(id: String): Option[Employee] = 
    employees.find(_.id == id)
  
  def report(): String = {
    val sb = new StringBuilder
    sb.append("=== HR PAYROLL REPORT ===\n")
    sb.append(f"Total Employees: ${employees.size}\n")
    sb.append(f"Total Payroll: ${totalPayroll()}%.2f THB\n\n")
    
    sb.append("--- Department Breakdown ---\n")
    departmentSummary().toList.sortBy(_._1).foreach { case (dept, total) =>
      sb.append(f"  $dept: $total%.2f THB\n")
    }
    
    sb.append("\n--- Top 3 Earners ---\n")
    topEarners(3).zipWithIndex.foreach { case (emp, idx) =>
      sb.append(f"  ${idx + 1}. $emp: ${emp.totalCompensation}%.2f THB\n")
    }
    
    sb.toString()
  }
}

object EmployeeHierarchyDemo extends App {
  val employees: List[Employee] = List(
    new SalariedEmployee("E001", "Alice Chen", "Engineering", 85000),
    new SalariedEmployee("E002", "Bob Smith", "Marketing", 65000),
    new HourlyEmployee("E003", "Carol Davis", "Operations", 350, 180),
    new HourlyEmployee("E004", "David Lee", "Operations", 280, 160),
    new CommissionEmployee("E005", "Eve Wilson", "Sales", 40000, 500000, 0.05),
    new CommissionEmployee("E006", "Frank Brown", "Sales", 40000, 850000, 0.05),
    new ExecutiveEmployee("E007", "Grace Kim", "Management", 150000, 1000, 250.0)
  )
  
  val hr = new HRSystem(employees)
  
  println(hr.report())
  
  println("=== Individual Payslips ===")
  hr.findById("E007").foreach(e => println(e.payslip()))
  hr.findById("E005").foreach(e => println(e.payslip()))
}
```

---

## สรุป Part 11

| แนวคิด | Keyword/Method | ตัวอย่าง |
|--------|---------------|---------|
| Inheritance | `extends` | `class Dog extends Animal` |
| Override method | `override def` | `override def speak()` |
| Call parent | `super.method()` | `super.speak()` |
| Abstract class | `abstract class` | `abstract class Shape` |
| Abstract method | ไม่มี body | `def area(): Double` |
| Final method | `final def` | ป้องกัน override |
| Final class | `final class` | ป้องกัน extend |
| Type check | `isInstanceOf[T]` | หรือ pattern matching |
| Type cast | `asInstanceOf[T]` | ใช้ด้วยความระมัดระวัง |
| Polymorphism | parent reference | `val a: Animal = new Dog(...)` |

---

## แบบฝึกหัด Part 11

**ข้อ 1:** สร้าง class hierarchy สำหรับระบบอาหาร: `Food` (abstract) → `MainDish`, `Dessert`, `Beverage` โดยมี method `calories(): Int`, `price(): Double`, `describe(): String`

**ข้อ 2:** สร้าง `BankAccount` (abstract) พร้อม subclasses: `SavingsAccount` (ดอกเบี้ยรายปี), `CheckingAccount` (ไม่มีดอกเบี้ยแต่มี overdraft limit), `InvestmentAccount` (ผลตอบแทนผันแปร) โดยแต่ละอันต้องมี `calculateInterest()` ที่คืนค่าต่างกัน

**ข้อ 3:** ใช้ abstract override และ trait stacking สร้างระบบ pipeline การประมวลผลข้อมูล: `DataProcessor` trait → `ValidationProcessor`, `TransformationProcessor`, `AuditProcessor`

**ข้อ 4:** สร้าง `Vehicle` hierarchy ที่ใช้ `final` อย่างเหมาะสม กำหนดว่า method ไหนควรเป็น final และทำไม

**ข้อ 5:** สร้าง employee payroll ที่ซับซ้อนขึ้น โดยเพิ่ม `PartTimeEmployee` และ `ContractEmployee` พร้อม override `totalCompensation()` ให้คำนวณแตกต่างจากพนักงานประจำ

---

➡️ ต่อไป: [Part 12 — Abstract Classes and Interfaces](part-12-abstract-classes-and-interfaces.md)
