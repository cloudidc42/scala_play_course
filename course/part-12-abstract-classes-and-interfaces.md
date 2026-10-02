# Part 12: Abstract Classes และ Trait-Based Interfaces

## Steps 111-120: Abstract Classes, Trait-Based Interfaces, Abstract Members, Template Method Pattern

---

## Step 111: Abstract Classes — คลาสนามธรรม

Abstract class คือ class ที่ไม่สามารถสร้าง instance ได้โดยตรง ใช้เป็น blueprint สำหรับ subclasses

```scala
// Abstract class with abstract and concrete members
abstract class Repository[T, ID] {
  
  // Abstract methods - subclass ต้องสร้าง implementation
  def findById(id: ID): Option[T]
  def findAll(): List[T]
  def save(entity: T): T
  def delete(id: ID): Boolean
  
  // Concrete methods - มี implementation พร้อมใช้
  def exists(id: ID): Boolean = findById(id).isDefined
  
  def count(): Int = findAll().size
  
  def findOrElse(id: ID, default: => T): T = 
    findById(id).getOrElse(default)
  
  def saveAll(entities: List[T]): List[T] = 
    entities.map(save)
  
  def deleteAll(ids: List[ID]): Int = 
    ids.count(delete)
}

// Domain model
case class User(
  id: Long,
  email: String,
  name: String,
  active: Boolean = true
)

// Concrete implementation - In-memory repository
class InMemoryUserRepository extends Repository[User, Long] {
  
  // Internal storage
  private var storage: Map[Long, User] = Map.empty
  private var nextId: Long = 1L
  
  override def findById(id: Long): Option[User] = storage.get(id)
  
  override def findAll(): List[User] = storage.values.toList.sortBy(_.id)
  
  override def save(user: User): User = {
    val savedUser = if (user.id == 0L) user.copy(id = nextId) else user
    if (user.id == 0L) nextId += 1
    storage = storage.updated(savedUser.id, savedUser)
    savedUser
  }
  
  override def delete(id: Long): Boolean = {
    if (storage.contains(id)) {
      storage = storage.removed(id)
      true
    } else false
  }
  
  // Additional query methods
  def findByEmail(email: String): Option[User] = 
    storage.values.find(_.email == email)
  
  def findActive(): List[User] = 
    storage.values.filter(_.active).toList
}

object AbstractClassDemo extends App {
  val repo = new InMemoryUserRepository()
  
  println("=== Repository Pattern ===")
  
  // Save users
  val alice = repo.save(User(0, "alice@example.com", "Alice Chen"))
  val bob = repo.save(User(0, "bob@example.com", "Bob Smith"))
  val carol = repo.save(User(0, "carol@example.com", "Carol Davis", active = false))
  
  println(s"Saved: $alice")
  println(s"Count: ${repo.count()}")
  
  // Find operations
  println(s"\nFind by ID 1: ${repo.findById(1)}")
  println(s"Exists ID 2: ${repo.exists(2)}")
  println(s"Exists ID 99: ${repo.exists(99)}")
  
  // Find active users
  println(s"\nActive users: ${repo.findActive()}")
  
  // Delete
  val deleted = repo.delete(2)
  println(s"\nDeleted Bob: $deleted")
  println(s"Count after delete: ${repo.count()}")
}
```

---

## Step 112: Trait-Based Interfaces — Interface ใน Scala

ใน Scala ใช้ `trait` แทน interface (คล้าย Java interface แต่ทรงพลังกว่า)

```scala
// Trait as interface - defines a contract
trait Printable {
  def print(): Unit
  def printToFile(filename: String): Unit = {
    // Default implementation
    import java.io.PrintWriter
    val writer = new PrintWriter(filename)
    try {
      writer.println(toString)
    } finally {
      writer.close()
    }
  }
}

trait Serializable2 {
  def toJson(): String
  def toXml(): String
  
  // Default implementation using toJson
  def serialize(format: String = "json"): String = format match {
    case "json" => toJson()
    case "xml"  => toXml()
    case _      => throw new IllegalArgumentException(s"Unknown format: $format")
  }
}

trait Validatable {
  def validate(): List[String]  // Returns list of validation errors
  
  def isValid(): Boolean = validate().isEmpty
  
  def validateOrThrow(): Unit = {
    val errors = validate()
    if (errors.nonEmpty) {
      throw new IllegalStateException(
        s"Validation failed:\n${errors.mkString("  - ", "\n  - ", "")}"
      )
    }
  }
}

// Implementing multiple traits
case class Product(
  id: String,
  name: String,
  price: Double,
  stock: Int,
  category: String
) extends Printable with Serializable2 with Validatable {
  
  override def print(): Unit = println(toString)
  
  override def toString: String = 
    s"Product[$id]: $name @ $price THB (stock: $stock, cat: $category)"
  
  override def toJson(): String = 
    s"""{"id":"$id","name":"$name","price":$price,"stock":$stock,"category":"$category"}"""
  
  override def toXml(): String = 
    s"""<product id="$id"><name>$name</name><price>$price</price><stock>$stock</stock></product>"""
  
  override def validate(): List[String] = {
    List(
      if (id.isEmpty) Some("ID cannot be empty") else None,
      if (name.isEmpty) Some("Name cannot be empty") else None,
      if (price <= 0) Some("Price must be positive") else None,
      if (stock < 0) Some("Stock cannot be negative") else None,
      if (category.isEmpty) Some("Category cannot be empty") else None
    ).flatten
  }
}

object TraitInterfaceDemo extends App {
  val product = Product("P001", "Laptop Pro", 45000.0, 10, "Electronics")
  
  println("=== Trait-Based Interfaces ===")
  product.print()
  
  println("\n=== Serialization ===")
  println(product.toJson())
  println(product.toXml())
  println(product.serialize("json"))
  
  println("\n=== Validation ===")
  println(s"Is valid: ${product.isValid()}")
  
  val invalidProduct = Product("", "Bad Product", -100, -5, "")
  println(s"Invalid product errors: ${invalidProduct.validate()}")
  
  try {
    invalidProduct.validateOrThrow()
  } catch {
    case e: IllegalStateException => println(s"Caught: ${e.getMessage}")
  }
}
```

---

## Step 113: Abstract Members ใน Trait

```scala
// Abstract members in traits (both methods and vals)
trait DataSource[T] {
  // Abstract type member
  type Record = T
  
  // Abstract val
  val sourceName: String
  val maxRecords: Int
  
  // Abstract method
  def fetch(): List[T]
  def fetchPage(page: Int, size: Int): List[T]
  
  // Concrete methods using abstract members
  def fetchFirst(n: Int): List[T] = fetch().take(n)
  
  def isEmpty(): Boolean = fetch().isEmpty
  
  def stats(): String = {
    val records = fetch()
    s"Source: $sourceName, Records: ${records.size}, Max: $maxRecords"
  }
}

// Abstract vals are initialized in the concrete class
class CSVDataSource(filename: String) extends DataSource[Map[String, String]] {
  
  override val sourceName: String = s"CSV($filename)"
  override val maxRecords: Int = 10000
  
  // Simulating CSV reading
  private lazy val data: List[Map[String, String]] = {
    // In real code, would read the file
    List(
      Map("name" -> "Alice", "age" -> "30", "dept" -> "Engineering"),
      Map("name" -> "Bob", "age" -> "25", "dept" -> "Marketing"),
      Map("name" -> "Carol", "age" -> "35", "dept" -> "Sales")
    )
  }
  
  override def fetch(): List[Map[String, String]] = data
  
  override def fetchPage(page: Int, size: Int): List[Map[String, String]] = {
    val start = (page - 1) * size
    data.slice(start, start + size)
  }
}

// Abstract class with template + abstract members
abstract class DataTransformer[In, Out] {
  
  // Abstract
  def transform(input: In): Out
  def validate(input: In): Boolean
  
  // Template method pattern
  def process(input: In): Option[Out] = {
    if (validate(input)) Some(transform(input))
    else None
  }
  
  def processAll(inputs: List[In]): List[Out] = 
    inputs.flatMap(process)
}

class StringToIntTransformer extends DataTransformer[String, Int] {
  override def validate(input: String): Boolean = 
    input.matches("-?\\d+")
  
  override def transform(input: String): Int = 
    input.toInt
}

object AbstractMembersDemo extends App {
  val csvSource = new CSVDataSource("employees.csv")
  
  println("=== Data Source ===")
  println(csvSource.stats())
  println(s"First 2 records:")
  csvSource.fetchFirst(2).foreach(r => println(s"  $r"))
  
  println("\n=== Data Transformer ===")
  val transformer = new StringToIntTransformer()
  val inputs = List("42", "abc", "100", "xyz", "-5")
  val results = transformer.processAll(inputs)
  println(s"Input: $inputs")
  println(s"Valid results: $results")
}
```

---

## Step 114: Template Method Pattern

```scala
// Template Method Pattern - กำหนดขั้นตอนหลักใน abstract class
abstract class ReportGenerator {
  
  // Template method - defines the algorithm structure
  final def generate(): String = {
    val sb = new StringBuilder
    sb.append(generateHeader())
    sb.append("\n")
    sb.append(generateBody())
    sb.append("\n")
    sb.append(generateFooter())
    sb.toString()
  }
  
  // Steps to implement
  protected def generateHeader(): String
  protected def generateBody(): String
  protected def generateFooter(): String
  
  // Hook methods - optional override
  protected def getTitle(): String = "Report"
  protected def getDate(): String = java.time.LocalDate.now().toString
}

case class SalesData(product: String, amount: Double, quantity: Int)

class HTMLReportGenerator(data: List[SalesData]) extends ReportGenerator {
  
  override protected def getTitle(): String = "Sales Report"
  
  override protected def generateHeader(): String = s"""
    |<!DOCTYPE html>
    |<html><head><title>${getTitle()}</title></head>
    |<body><h1>${getTitle()}</h1>
    |<p>Date: ${getDate()}</p>""".stripMargin
  
  override protected def generateBody(): String = {
    val rows = data.map { d =>
      s"<tr><td>${d.product}</td><td>${d.amount}</td><td>${d.quantity}</td></tr>"
    }.mkString("\n")
    
    s"""
    |<table border="1">
    |  <tr><th>Product</th><th>Amount</th><th>Quantity</th></tr>
    |  $rows
    |</table>
    |<p>Total: ${data.map(_.amount * d.quantity).sum}</p>""".stripMargin
  }
  
  override protected def generateFooter(): String = 
    "</body></html>"
}

class CSVReportGenerator(data: List[SalesData]) extends ReportGenerator {
  
  override protected def generateHeader(): String = 
    "Product,Amount,Quantity,Total"
  
  override protected def generateBody(): String = {
    data.map { d =>
      s"${d.product},${d.amount},${d.quantity},${d.amount * d.quantity}"
    }.mkString("\n")
  }
  
  override protected def generateFooter(): String = {
    val totalAmount = data.map(d => d.amount * d.quantity).sum
    s"TOTAL,,,$totalAmount"
  }
}

class MarkdownReportGenerator(data: List[SalesData]) extends ReportGenerator {
  
  override protected def getTitle(): String = "Sales Summary"
  
  override protected def generateHeader(): String = 
    s"# ${getTitle()}\n\nGenerated: ${getDate()}\n"
  
  override protected def generateBody(): String = {
    val header = "| Product | Amount | Qty | Total |\n|---------|--------|-----|-------|"
    val rows = data.map { d =>
      f"| ${d.product} | ${d.amount}%.2f | ${d.quantity} | ${d.amount * d.quantity}%.2f |"
    }.mkString("\n")
    s"$header\n$rows"
  }
  
  override protected def generateFooter(): String = {
    val total = data.map(d => d.amount * d.quantity).sum
    f"\n**Grand Total: $total%.2f THB**"
  }
}

object TemplateMethodDemo extends App {
  val salesData = List(
    SalesData("Laptop Pro", 45000.0, 3),
    SalesData("Mouse", 890.0, 10),
    SalesData("Keyboard", 1500.0, 5),
    SalesData("Monitor", 12000.0, 2)
  )
  
  val generators: List[ReportGenerator] = List(
    new CSVReportGenerator(salesData),
    new MarkdownReportGenerator(salesData)
  )
  
  generators.foreach { gen =>
    println(s"\n=== ${gen.getClass.getSimpleName} ===")
    println(gen.generate())
    println("---")
  }
}
```

---

## Step 115: Sealed Abstract Classes

```scala
// Sealed abstract class - กำหนดว่า subclass ต้องอยู่ในไฟล์เดียวกัน
sealed abstract class ApiResponse[+T] {
  def isSuccess: Boolean
  def map[B](f: T => B): ApiResponse[B]
}

case class Success[T](data: T, statusCode: Int = 200) extends ApiResponse[T] {
  override def isSuccess: Boolean = true
  override def map[B](f: T => B): ApiResponse[B] = Success(f(data), statusCode)
}

case class Failure(error: String, statusCode: Int) extends ApiResponse[Nothing] {
  override def isSuccess: Boolean = false
  override def map[B](f: Nothing => B): ApiResponse[B] = this
}

case class Loading[T]() extends ApiResponse[T] {
  override def isSuccess: Boolean = false
  override def map[B](f: T => B): ApiResponse[B] = Loading()
}

// Using sealed abstract class
object SealedAbstractDemo extends App {
  // Simulate API calls
  def fetchUser(id: Int): ApiResponse[Map[String, Any]] = id match {
    case 1 => Success(Map("id" -> 1, "name" -> "Alice", "email" -> "alice@test.com"))
    case 2 => Success(Map("id" -> 2, "name" -> "Bob", "email" -> "bob@test.com"))
    case _ => Failure("User not found", 404)
  }
  
  def processResponse(response: ApiResponse[Map[String, Any]]): String = {
    // Exhaustive matching - compiler warns if case is missing
    response match {
      case Success(data, code) => 
        s"[${code}] User: ${data("name")} <${data("email")}>"
      case Failure(err, code) => 
        s"[${code}] Error: $err"
      case Loading() => 
        s"Loading..."
    }
  }
  
  println("=== Sealed Abstract Class ===")
  println(processResponse(fetchUser(1)))
  println(processResponse(fetchUser(2)))
  println(processResponse(fetchUser(99)))
  println(processResponse(Loading()))
  
  // Chaining with map
  val result = fetchUser(1).map(user => user("name").toString.toUpperCase)
  println(s"\nMapped result: $result")
}
```

---

## Step 116: Interface Segregation — แยก Traits

```scala
// Interface Segregation Principle - แยก interface ให้เล็กและเฉพาะเจาะจง
// แทนที่จะมี interface ใหญ่ interface เดียว

// แยกความสามารถออกเป็น traits เล็กๆ
trait Readable[T] {
  def read(id: Long): Option[T]
  def readAll(): List[T]
}

trait Writable[T] {
  def write(item: T): T
  def writeAll(items: List[T]): List[T] = items.map(write)
}

trait Deletable {
  def delete(id: Long): Boolean
  def deleteAll(ids: List[Long]): Int = ids.count(delete)
}

trait Searchable[T] {
  def search(query: String): List[T]
  def searchByField(field: String, value: String): List[T]
}

trait Pageable[T] {
  def findPage(page: Int, size: Int): (List[T], Int) // (items, totalCount)
}

// Combine traits as needed
case class Article(id: Long, title: String, content: String, author: String, published: Boolean)

// Full CRUD repository
class ArticleRepository 
    extends Readable[Article] 
    with Writable[Article] 
    with Deletable 
    with Searchable[Article]
    with Pageable[Article] {
  
  private var articles: Map[Long, Article] = Map.empty
  private var idCounter: Long = 1L
  
  override def read(id: Long): Option[Article] = articles.get(id)
  
  override def readAll(): List[Article] = articles.values.toList.sortBy(_.id)
  
  override def write(article: Article): Article = {
    val saved = if (article.id == 0L) article.copy(id = idCounter) else article
    if (article.id == 0L) idCounter += 1
    articles = articles.updated(saved.id, saved)
    saved
  }
  
  override def delete(id: Long): Boolean = {
    if (articles.contains(id)) {
      articles = articles.removed(id)
      true
    } else false
  }
  
  override def search(query: String): List[Article] = {
    val q = query.toLowerCase
    articles.values.filter(a => 
      a.title.toLowerCase.contains(q) || a.content.toLowerCase.contains(q)
    ).toList
  }
  
  override def searchByField(field: String, value: String): List[Article] = {
    field match {
      case "author"    => articles.values.filter(_.author == value).toList
      case "published" => articles.values.filter(a => a.published.toString == value).toList
      case _           => List.empty
    }
  }
  
  override def findPage(page: Int, size: Int): (List[Article], Int) = {
    val all = readAll()
    val start = (page - 1) * size
    (all.slice(start, start + size), all.size)
  }
}

// Read-only repository (no write/delete)
class ReadOnlyArticleView(source: List[Article]) extends Readable[Article] with Searchable[Article] {
  
  override def read(id: Long): Option[Article] = source.find(_.id == id)
  override def readAll(): List[Article] = source
  
  override def search(query: String): List[Article] = {
    val q = query.toLowerCase
    source.filter(a => a.title.toLowerCase.contains(q))
  }
  
  override def searchByField(field: String, value: String): List[Article] = {
    field match {
      case "author" => source.filter(_.author == value)
      case _        => List.empty
    }
  }
}

object InterfaceSegregationDemo extends App {
  val repo = new ArticleRepository()
  
  // Write articles
  val a1 = repo.write(Article(0, "Scala Basics", "Learn Scala...", "Alice", true))
  val a2 = repo.write(Article(0, "Advanced Scala", "Deep dive...", "Bob", true))
  val a3 = repo.write(Article(0, "Scala Draft", "Work in progress...", "Alice", false))
  val a4 = repo.write(Article(0, "Functional Programming", "Pure functions...", "Alice", true))
  
  println("=== Interface Segregation ===")
  println(s"Total articles: ${repo.readAll().size}")
  
  println("\n--- Search ---")
  repo.search("scala").foreach(a => println(s"  Found: ${a.title}"))
  
  println("\n--- Search by author ---")
  repo.searchByField("author", "Alice").foreach(a => println(s"  ${a.title} [${if(a.published) "published" else "draft"}]"))
  
  println("\n--- Pagination ---")
  val (page1, total) = repo.findPage(1, 2)
  println(s"Page 1 (total: $total):")
  page1.foreach(a => println(s"  ${a.title}"))
  
  println("\n--- Read-only view ---")
  val publishedArticles = repo.readAll().filter(_.published)
  val readOnlyView = new ReadOnlyArticleView(publishedArticles)
  println(s"Published articles: ${readOnlyView.readAll().size}")
  readOnlyView.search("scala").foreach(a => println(s"  Found: ${a.title}"))
}
```

---

## Step 117: Abstract Class vs Trait — เมื่อไหรใช้อะไร

```scala
// When to use abstract class vs trait
// Abstract Class: เมื่อต้องการ constructor parameters, ใช้ Java, สืบทอดได้แค่ 1
// Trait: เมื่อต้องการ mixin หลายตัว, ไม่ต้องการ constructor params

// USE CASE 1: Abstract class with constructor params
abstract class ConfigurableService(val serviceName: String, val version: String) {
  def start(): Unit
  def stop(): Unit
  def healthCheck(): Boolean
  
  def info(): String = s"$serviceName v$version"
}

// USE CASE 2: Trait for cross-cutting concerns  
trait Cacheable[T] {
  private val cache = scala.collection.mutable.Map[String, T]()
  
  def cached(key: String)(compute: => T): T = {
    cache.getOrElseUpdate(key, compute)
  }
  
  def invalidate(key: String): Unit = cache.remove(key)
  def clearCache(): Unit = cache.clear()
  def cacheSize(): Int = cache.size
}

trait Retryable {
  def retry[T](maxAttempts: Int, delay: Long = 100)(operation: => T): T = {
    def attempt(remaining: Int): T = {
      try {
        operation
      } catch {
        case e: Exception if remaining > 1 =>
          Thread.sleep(delay)
          attempt(remaining - 1)
        case e: Exception => throw e
      }
    }
    attempt(maxAttempts)
  }
}

trait MetricsRecorder {
  private var callCount = 0
  private var totalDuration = 0L
  
  def timed[T](name: String)(block: => T): T = {
    val start = System.currentTimeMillis()
    callCount += 1
    try {
      block
    } finally {
      totalDuration += System.currentTimeMillis() - start
    }
  }
  
  def metrics(): Map[String, Any] = Map(
    "calls" -> callCount,
    "totalMs" -> totalDuration,
    "avgMs" -> (if (callCount > 0) totalDuration / callCount else 0)
  )
}

// Combining abstract class with multiple traits
class UserService(serviceName: String, version: String)
    extends ConfigurableService(serviceName, version)
    with Cacheable[String]
    with Retryable
    with MetricsRecorder {
  
  private var running = false
  
  override def start(): Unit = {
    println(s"Starting ${info()}")
    running = true
  }
  
  override def stop(): Unit = {
    println(s"Stopping ${info()}")
    running = false
    clearCache()
  }
  
  override def healthCheck(): Boolean = running
  
  def getUserName(userId: Long): String = {
    timed("getUserName") {
      cached(s"user:$userId") {
        retry(3, 50) {
          // Simulate API call
          Thread.sleep(10)
          s"User_$userId"
        }
      }
    }
  }
}

object AbstractVsTraitDemo extends App {
  val service = new UserService("UserService", "1.0.0")
  service.start()
  
  println(s"\n=== Service: ${service.info()} ===")
  println(s"Health: ${service.healthCheck()}")
  
  // Multiple calls - second should be cached
  println("\n--- User lookups ---")
  for (i <- 1 to 3) {
    println(s"User 1: ${service.getUserName(1)}")
    println(s"User 2: ${service.getUserName(2)}")
  }
  
  println(s"\nCache size: ${service.cacheSize()}")
  println(s"Metrics: ${service.metrics()}")
  
  service.stop()
}
```

---

## Step 118: Dependency Inversion ด้วย Abstract Types

```scala
// Dependency Inversion - depend on abstractions, not concretions
trait EmailSender {
  def send(to: String, subject: String, body: String): Boolean
}

trait SMSSender {
  def send(to: String, message: String): Boolean
}

trait NotificationLogger {
  def logNotification(channel: String, recipient: String, success: Boolean): Unit
}

// High-level module depends on abstractions
class NotificationService(
  private val emailSender: EmailSender,
  private val smsSender: SMSSender,
  private val logger: NotificationLogger
) {
  
  def sendOrderConfirmation(
    email: String, 
    phone: String, 
    orderId: String
  ): (Boolean, Boolean) = {
    val emailSubject = s"Order Confirmation #$orderId"
    val emailBody = s"Your order #$orderId has been confirmed. Thank you!"
    val smsBody = s"Order #$orderId confirmed. Check email for details."
    
    val emailResult = emailSender.send(email, emailSubject, emailBody)
    logger.logNotification("email", email, emailResult)
    
    val smsResult = smsSender.send(phone, smsBody)
    logger.logNotification("sms", phone, smsResult)
    
    (emailResult, smsResult)
  }
}

// Concrete implementations
class SmtpEmailSender(host: String, port: Int) extends EmailSender {
  override def send(to: String, subject: String, body: String): Boolean = {
    println(s"[SMTP $host:$port] Email to $to: $subject")
    true // Simulated success
  }
}

class TwilioSMSSender(apiKey: String) extends SMSSender {
  override def send(to: String, message: String): Boolean = {
    println(s"[Twilio] SMS to $to: $message")
    true
  }
}

class ConsoleLogger extends NotificationLogger {
  override def logNotification(channel: String, recipient: String, success: Boolean): Unit = {
    val status = if (success) "OK" else "FAILED"
    println(s"[LOG] $channel -> $recipient: $status")
  }
}

// Mock implementations for testing
class MockEmailSender extends EmailSender {
  val sentEmails: scala.collection.mutable.ListBuffer[(String, String, String)] = 
    scala.collection.mutable.ListBuffer.empty
  
  override def send(to: String, subject: String, body: String): Boolean = {
    sentEmails.append((to, subject, body))
    true
  }
}

object DependencyInversionDemo extends App {
  println("=== Production Setup ===")
  val prodService = new NotificationService(
    emailSender = new SmtpEmailSender("smtp.gmail.com", 587),
    smsSender = new TwilioSMSSender("twilio-api-key"),
    logger = new ConsoleLogger()
  )
  
  val (emailOk, smsOk) = prodService.sendOrderConfirmation(
    "customer@example.com",
    "+6681234567",
    "ORD-2024-001"
  )
  println(s"Results: email=$emailOk, sms=$smsOk")
  
  println("\n=== Test Setup ===")
  val mockEmail = new MockEmailSender()
  val testService = new NotificationService(
    emailSender = mockEmail,
    smsSender = new MockEmailSender() { // Inline class
      override def send(to: String, subject: String, body: String): Boolean = true
    }.asInstanceOf[SMSSender],
    logger = new NotificationLogger {
      override def logNotification(channel: String, recipient: String, success: Boolean): Unit = ()
    }
  )
  
  // In tests, we can verify what was sent
  testService.sendOrderConfirmation("test@test.com", "+66000000000", "TEST-001")
  println(s"Emails sent in test: ${mockEmail.sentEmails.size}")
  mockEmail.sentEmails.foreach { case (to, subj, _) => println(s"  To: $to, Subject: $subj") }
}
```

---

## Step 119: Mixin Composition

```scala
// Mixin Composition - combining traits creatively
trait JsonSerializable {
  def toJson(): String
  
  def toPrettyJson(): String = {
    // Simple pretty printing
    val json = toJson()
    json.replace("{", "{\n  ")
        .replace(",", ",\n  ")
        .replace("}", "\n}")
  }
}

trait Auditable {
  val createdAt: java.time.Instant = java.time.Instant.now()
  var updatedAt: java.time.Instant = java.time.Instant.now()
  var createdBy: String = "system"
  
  def touch(user: String): Unit = {
    updatedAt = java.time.Instant.now()
    createdBy = user
  }
  
  def auditInfo(): String = 
    s"Created: $createdAt by $createdBy, Updated: $updatedAt"
}

trait SoftDeletable {
  var isDeleted: Boolean = false
  var deletedAt: Option[java.time.Instant] = None
  
  def softDelete(): Unit = {
    isDeleted = true
    deletedAt = Some(java.time.Instant.now())
  }
  
  def restore(): Unit = {
    isDeleted = false
    deletedAt = None
  }
}

trait TaggableEntity {
  private var tags: Set[String] = Set.empty
  
  def addTag(tag: String): Unit = tags = tags + tag.toLowerCase.trim
  def removeTag(tag: String): Unit = tags = tags - tag.toLowerCase.trim
  def hasTag(tag: String): Boolean = tags.contains(tag.toLowerCase.trim)
  def getTags: Set[String] = tags
}

// Rich domain model using multiple traits
class BlogPost(
  val id: Long,
  var title: String,
  var content: String
) extends JsonSerializable 
    with Auditable 
    with SoftDeletable 
    with TaggableEntity {
  
  override def toJson(): String = {
    val tagsJson = getTags.map(t => s""""$t"""").mkString("[", ",", "]")
    s"""{"id":$id,"title":"$title","deleted":$isDeleted,"tags":$tagsJson}"""
  }
  
  override def toString: String = s"BlogPost($id: $title)"
}

object MixinCompositionDemo extends App {
  val post = new BlogPost(1L, "Scala Mixins", "Learn about mixins...")
  
  println("=== Mixin Composition ===")
  println(post)
  println(post.toJson())
  
  // Add tags
  post.addTag("Scala")
  post.addTag("Programming")
  post.addTag("FP")
  println(s"\nTags: ${post.getTags}")
  println(s"Has 'scala': ${post.hasTag("scala")}")
  
  // Audit
  post.touch("alice@example.com")
  println(s"\nAudit: ${post.auditInfo()}")
  
  // Soft delete
  post.softDelete()
  println(s"\nIs deleted: ${post.isDeleted}")
  println(s"Deleted at: ${post.deletedAt}")
  
  post.restore()
  println(s"After restore - Is deleted: ${post.isDeleted}")
  
  println("\n--- Pretty JSON ---")
  println(post.toPrettyJson())
}
```

---

## Step 120: Production Example — Plugin System

```scala
// Plugin/Extension System using abstract classes and traits
abstract class Plugin(val name: String, val version: String) {
  def initialize(): Boolean
  def execute(context: PluginContext): PluginResult
  def shutdown(): Unit
  
  def metadata: Map[String, String] = Map(
    "name" -> name,
    "version" -> version,
    "class" -> getClass.getName
  )
}

case class PluginContext(
  requestId: String,
  data: Map[String, Any],
  config: Map[String, String]
)

sealed trait PluginResult
case class PluginSuccess(output: Map[String, Any], message: String) extends PluginResult
case class PluginError(error: String, code: Int) extends PluginResult

trait DataTransformPlugin {
  def transformData(data: Map[String, Any]): Map[String, Any]
}

trait ValidationPlugin {
  def validateInput(data: Map[String, Any]): List[String]
}

// Concrete plugin implementations
class DataEnrichmentPlugin extends Plugin("DataEnrichment", "1.0.0") 
    with DataTransformPlugin {
  
  override def initialize(): Boolean = {
    println(s"[$name] Initializing...")
    true
  }
  
  override def execute(context: PluginContext): PluginResult = {
    val enriched = transformData(context.data)
    PluginSuccess(enriched, s"Enriched ${context.data.size} fields")
  }
  
  override def shutdown(): Unit = println(s"[$name] Shutting down")
  
  override def transformData(data: Map[String, Any]): Map[String, Any] = {
    data + ("enrichedAt" -> java.time.Instant.now().toString) +
           ("version" -> version)
  }
}

class InputValidationPlugin extends Plugin("InputValidation", "2.0.0") 
    with ValidationPlugin {
  
  override def initialize(): Boolean = true
  
  override def execute(context: PluginContext): PluginResult = {
    val errors = validateInput(context.data)
    if (errors.isEmpty) {
      PluginSuccess(context.data, "Validation passed")
    } else {
      PluginError(s"Validation failed: ${errors.mkString(", ")}", 400)
    }
  }
  
  override def shutdown(): Unit = ()
  
  override def validateInput(data: Map[String, Any]): List[String] = {
    val errors = scala.collection.mutable.ListBuffer[String]()
    if (!data.contains("userId")) errors += "userId is required"
    if (!data.contains("action")) errors += "action is required"
    if (data.get("userId").exists(v => v.toString.isEmpty)) errors += "userId cannot be empty"
    errors.toList
  }
}

// Plugin Manager
class PluginManager {
  private var plugins: List[Plugin] = List.empty
  
  def register(plugin: Plugin): Boolean = {
    if (plugin.initialize()) {
      plugins = plugins :+ plugin
      println(s"Registered plugin: ${plugin.name} v${plugin.version}")
      true
    } else {
      println(s"Failed to initialize: ${plugin.name}")
      false
    }
  }
  
  def executeAll(context: PluginContext): List[(String, PluginResult)] = {
    plugins.map(p => p.name -> p.execute(context))
  }
  
  def shutdown(): Unit = plugins.foreach(_.shutdown())
  
  def listPlugins(): Unit = {
    println("Registered Plugins:")
    plugins.foreach(p => println(s"  - ${p.name} v${p.version}"))
  }
}

object PluginSystemDemo extends App {
  val manager = new PluginManager()
  
  manager.register(new InputValidationPlugin())
  manager.register(new DataEnrichmentPlugin())
  
  println("\n=== Plugin Manager ===")
  manager.listPlugins()
  
  println("\n--- Valid Request ---")
  val validCtx = PluginContext(
    requestId = "REQ-001",
    data = Map("userId" -> "U123", "action" -> "purchase", "amount" -> 1500.0),
    config = Map("env" -> "production")
  )
  
  val results = manager.executeAll(validCtx)
  results.foreach { case (name, result) =>
    result match {
      case PluginSuccess(output, msg) => println(s"[$name] SUCCESS: $msg")
      case PluginError(err, code) => println(s"[$name] ERROR ($code): $err")
    }
  }
  
  println("\n--- Invalid Request ---")
  val invalidCtx = PluginContext(
    requestId = "REQ-002",
    data = Map("amount" -> 500.0), // Missing userId and action
    config = Map()
  )
  
  val failResults = manager.executeAll(invalidCtx)
  failResults.foreach { case (name, result) =>
    result match {
      case PluginSuccess(_, msg) => println(s"[$name] SUCCESS: $msg")
      case PluginError(err, code) => println(s"[$name] ERROR ($code): $err")
    }
  }
  
  manager.shutdown()
}
```

---

## สรุป Part 12

| แนวคิด | Scala Syntax | การใช้งาน |
|--------|-------------|----------|
| Abstract class | `abstract class Foo` | Blueprint ที่มี constructor params |
| Abstract method | `def foo(): T` ใน abstract class | ต้อง implement ใน subclass |
| Trait interface | `trait Foo { def bar(): T }` | Interface ที่ mix ได้หลายตัว |
| Default implementation | method body ใน trait | Shared behavior ไม่ต้อง override |
| Abstract val | `val x: T` ใน trait/abstract | ค่าที่ subclass ต้องกำหนด |
| Template method | `final def template()` | กำหนด algorithm structure |
| Mixin composition | `with Trait1 with Trait2` | เพิ่มความสามารถหลายอย่าง |
| Sealed abstract | `sealed abstract class` | จำกัด subclass ในไฟล์เดียว |

---

## แบบฝึกหัด Part 12

**ข้อ 1:** สร้าง `abstract class FileProcessor` ที่ implement Template Method Pattern สำหรับการประมวลผลไฟล์: open → validate → process → close โดยมี subclass สำหรับ CSV, JSON, XML

**ข้อ 2:** ออกแบบ trait-based permission system: `trait Readable`, `trait Writable`, `trait Executable`, `trait Administrable` แล้วสร้าง `User`, `Admin`, `ReadOnlyUser` class ที่ mix traits เหมาะสม

**ข้อ 3:** สร้าง `abstract class GameCharacter` ที่มี abstract method `attack()`, `defend()`, `specialMove()` พร้อม subclasses: `Warrior`, `Mage`, `Archer` แต่ละตัวมีสถิติและ abilities ต่างกัน

**ข้อ 4:** ใช้ Dependency Inversion สร้าง payment processing system ที่ depend บน trait `PaymentGateway` และสร้าง implementations สำหรับ Stripe, PayPal, และ Mock

**ข้อ 5:** สร้าง plugin system ที่ซับซ้อนขึ้นโดยมี plugin pipeline ที่ result ของ plugin แรกเป็น input ของ plugin ถัดไป (pipeline pattern)

---

➡️ ต่อไป: [Part 13 — Companion Objects](part-13-companion-objects.md)
