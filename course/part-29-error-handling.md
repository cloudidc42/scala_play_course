# Part 29: Error Handling

## Steps 281-290: Option, Either, Try, Validated, Error ADTs, Railway-Oriented Programming

---

## Step 281: Option — Handling Absence

```scala
object OptionHandling extends App {
  
  // Option = Some(value) | None
  def findUser(id: Int): Option[String] = id match {
    case 1 => Some("Alice")
    case 2 => Some("Bob")
    case _ => None
  }
  
  def findEmail(username: String): Option[String] = username match {
    case "Alice" => Some("alice@example.com")
    case "Bob"   => Some("bob@example.com")
    case _       => None
  }
  
  println("=== Option Operations ===")
  
  // Chain with flatMap (monadic chaining)
  val email1 = findUser(1).flatMap(findEmail)
  val email2 = findUser(99).flatMap(findEmail)
  
  println(s"User 1 email: $email1")
  println(s"User 99 email: $email2")
  
  // For comprehension — cleaner chaining
  def getUserEmail(userId: Int): Option[String] = for {
    username <- findUser(userId)
    email    <- findEmail(username)
  } yield email.toUpperCase
  
  println(s"\nUser 1 email (for): ${getUserEmail(1)}")
  println(s"User 99 email (for): ${getUserEmail(99)}")
  
  // Option operations
  val opt = Some(42)
  val none: Option[Int] = None
  
  println(s"\n--- Operations ---")
  println(s"map: ${opt.map(_ * 2)}")
  println(s"filter: ${opt.filter(_ > 50)}")
  println(s"getOrElse: ${none.getOrElse(0)}")
  println(s"orElse: ${none.orElse(Some(-1))}")
  println(s"fold: ${opt.fold("empty")(n => s"got $n")}")
  println(s"toList: ${opt.toList}")
  println(s"toRight: ${opt.toRight("not found")}")
  println(s"toLeft: ${none.toLeft("no value")}")
  
  // Collecting from multiple Options
  case class Config(host: String, port: Int, dbName: String)
  
  def buildConfig(
    host: Option[String],
    port: Option[Int],
    db: Option[String]
  ): Option[Config] = for {
    h <- host
    p <- port
    d <- db
  } yield Config(h, p, d)
  
  println(s"\n--- Config Building ---")
  println(buildConfig(Some("localhost"), Some(5432), Some("mydb")))
  println(buildConfig(Some("localhost"), None, Some("mydb")))
  
  // Real-world: null-safe navigation
  case class Company(name: String, address: Option[Address2])
  case class Address2(city: String, country: Option[Country])
  case class Country(name: String, phoneCode: String)
  
  def getPhoneCode(company: Company): Option[String] = for {
    address <- company.address
    country <- address.country
  } yield country.phoneCode
  
  val co1 = Company("TechCorp", Some(Address2("Bangkok", Some(Country("Thailand", "+66")))))
  val co2 = Company("StartupX", Some(Address2("Unknown", None)))
  val co3 = Company("Ghost", None)
  
  println(s"\n--- Null-safe Navigation ---")
  println(s"co1 phone: ${getPhoneCode(co1)}")
  println(s"co2 phone: ${getPhoneCode(co2)}")
  println(s"co3 phone: ${getPhoneCode(co3)}")
}
```

---

## Step 282: Either — Typed Errors

```scala
object EitherHandling extends App {
  
  // Either[Left=Error, Right=Value] — Right is the "happy path"
  
  sealed trait AppError
  case class NotFound(id: String) extends AppError
  case class ValidationError(field: String, message: String) extends AppError
  case class DatabaseError(cause: String) extends AppError
  case class UnauthorizedError(userId: String) extends AppError
  
  type Result[A] = Either[AppError, A]
  
  case class User(id: String, name: String, email: String, role: String)
  case class Order(id: String, userId: String, total: Double, items: Int)
  
  def findUser(id: String): Result[User] = id match {
    case "u1" => Right(User("u1", "Alice", "alice@example.com", "admin"))
    case "u2" => Right(User("u2", "Bob", "bob@example.com", "user"))
    case _    => Left(NotFound(id))
  }
  
  def findOrders(userId: String): Result[List[Order]] = userId match {
    case "u1" => Right(List(Order("o1", "u1", 1500.0, 3), Order("o2", "u1", 850.0, 2)))
    case "u2" => Right(List(Order("o3", "u2", 250.0, 1)))
    case _    => Left(DatabaseError(s"No orders table entry for $userId"))
  }
  
  def authorize(user: User, action: String): Result[User] = 
    if (user.role == "admin" || action == "read") Right(user)
    else Left(UnauthorizedError(user.id))
  
  def validateOrder(order: Order): Result[Order] = {
    if (order.total < 0) Left(ValidationError("total", "Total cannot be negative"))
    else if (order.items <= 0) Left(ValidationError("items", "Order must have at least 1 item"))
    else Right(order)
  }
  
  // Chain operations with for comprehension
  def getAdminOrders(userId: String): Result[List[Order]] = for {
    user   <- findUser(userId)
    authed <- authorize(user, "list_orders")
    orders <- findOrders(authed.id)
    valid  <- orders.map(validateOrder).foldLeft(Right(List.empty): Result[List[Order]]) {
                case (Right(acc), Right(order)) => Right(acc :+ order)
                case (Left(err), _)             => Left(err)
                case (_, Left(err))             => Left(err)
              }
  } yield valid
  
  println("=== Either Error Handling ===")
  List("u1", "u2", "u99") foreach { userId =>
    getAdminOrders(userId) match {
      case Right(orders) => 
        println(s"$userId orders:")
        orders.foreach(o => println(s"  ${o.id}: $${o.total} (${o.items} items)"))
      case Left(NotFound(id)) => 
        println(s"$userId: User not found: $id")
      case Left(UnauthorizedError(uid)) => 
        println(s"$userId: Unauthorized access by $uid")
      case Left(DatabaseError(cause)) => 
        println(s"$userId: DB error: $cause")
      case Left(err) => 
        println(s"$userId: Error: $err")
    }
  }
  
  // Either transformations
  val result: Result[Int] = Right(42)
  println(s"\n--- Transformations ---")
  println(s"map: ${result.map(_ * 2)}")
  println(s"flatMap: ${result.flatMap(n => Right(n.toString))}")
  println(s"swap: ${result.swap}")
  println(s"fold: ${result.fold(e => s"Error: $e", n => s"Value: $n")}")
  println(s"toOption: ${result.toOption}")
  
  val error: Result[Int] = Left(NotFound("x"))
  println(s"left.getOrElse: ${error.getOrElse(0)}")
  println(s"left.orElse: ${error.orElse(Right(-1))}")
}
```

---

## Step 283: Try — Exception Handling

```scala
import scala.util.{Try, Success, Failure}

object TryHandling extends App {
  
  // Try = Success(value) | Failure(exception)
  // Useful when calling Java libraries or exception-throwing code
  
  def parseInt(s: String): Try[Int] = Try(s.toInt)
  def divide(a: Int, b: Int): Try[Double] = Try(a.toDouble / b)
  
  println("=== Try Operations ===")
  println(parseInt("42"))        // Success(42)
  println(parseInt("hello"))     // Failure(NumberFormatException)
  println(divide(10, 3))         // Success(3.333...)
  println(divide(10, 0))         // Success(Infinity) — not an exception in Java!
  
  // Try chaining
  def parseAndDouble(s: String): Try[Int] = for {
    n      <- parseInt(s)
    doubled = n * 2
    result <- if (doubled > 100) Failure(new Exception("Too large")) else Success(doubled)
  } yield result
  
  List("10", "60", "abc").foreach { s =>
    parseAndDouble(s) match {
      case Success(v) => println(s"  '$s' -> $v")
      case Failure(e) => println(s"  '$s' -> Error: ${e.getMessage}")
    }
  }
  
  // Try → Either (for typed errors)
  def readFile(path: String): Either[String, String] =
    Try(scala.io.Source.fromFile(path).mkString)
      .toEither
      .left.map(e => s"Failed to read file: ${e.getMessage}")
  
  println(s"\n--- File Reading ---")
  readFile("/etc/hostname") match {
    case Right(content) => println(s"  Content: ${content.trim}")
    case Left(err)      => println(s"  Error: $err")
  }
  readFile("/nonexistent/path") match {
    case Right(content) => println(s"  Content: $content")
    case Left(err)      => println(s"  Error: $err")
  }
  
  // Try with resources (close after use)
  def withCloseable[A, B <: java.io.Closeable](resource: => B)(f: B => A): Try[A] = {
    val res = Try(resource)
    try {
      res.map(f)
    } finally {
      res.foreach(r => Try(r.close()))
    }
  }
  
  // Try operations
  val t: Try[Int] = Success(42)
  println(s"\n--- Operations ---")
  println(s"map: ${t.map(_ * 2)}")
  println(s"filter: ${t.filter(_ > 50)}")
  println(s"recover: ${Failure[Int](new Exception("oops")).recover { case _ => -1 }}")
  println(s"recoverWith: ${Failure[Int](new Exception("oops")).recoverWith { case e => Success(0) }}")
  println(s"toOption: ${t.toOption}")
  println(s"toEither: ${t.toEither}")
  println(s"getOrElse: ${Failure[Int](new Exception).getOrElse(999)}")
}
```

---

## Step 284: Validated — Accumulating Errors

```scala
// Unlike Either which short-circuits, Validated accumulates ALL errors
// Useful for form validation, config parsing

sealed trait Validated[+E, +A] {
  def map[B](f: A => B): Validated[E, B]
  def isValid: Boolean
  
  def andThen[B](f: A => Validated[E, B]): Validated[E, B]
  
  def zip[E2 >: E, B](other: Validated[E2, B]): Validated[E2, (A, B)]
}

case class Valid[+A](value: A) extends Validated[Nothing, A] {
  def map[B](f: A => B): Validated[Nothing, B] = Valid(f(value))
  def isValid: Boolean = true
  def andThen[B](f: A => Validated[Nothing, B]): Validated[Nothing, B] = f(value)
  def zip[E2, B](other: Validated[E2, B]): Validated[E2, (A, B)] = other match {
    case Valid(b)      => Valid((value, b))
    case Invalid(errs) => Invalid(errs)
  }
}

case class Invalid[+E](errors: List[E]) extends Validated[E, Nothing] {
  def map[B](f: Nothing => B): Validated[E, B] = this
  def isValid: Boolean = false
  def andThen[B](f: Nothing => Validated[E, B]): Validated[E, B] = this
  def zip[E2 >: E, B](other: Validated[E2, B]): Validated[E2, (Nothing, B)] = other match {
    case Valid(_)           => Invalid(errors)
    case Invalid(otherErrs) => Invalid(errors ++ otherErrs)
  }
}

object Validated {
  def valid[A](value: A): Validated[Nothing, A] = Valid(value)
  def invalid[E](error: E): Validated[E, Nothing] = Invalid(List(error))
  def invalidNel[E](errors: List[E]): Validated[E, Nothing] = Invalid(errors)
  
  def fromEither[E, A](either: Either[E, A]): Validated[E, A] = either match {
    case Right(v) => Valid(v)
    case Left(e)  => Invalid(List(e))
  }
  
  def fromOption[E, A](option: Option[A], error: => E): Validated[E, A] = option match {
    case Some(v) => Valid(v)
    case None    => Invalid(List(error))
  }
}

// Form validation example
case class RegistrationForm(
  username: String,
  email: String,
  password: String,
  age: Int
)

object FormValidator {
  
  def validateUsername(username: String): Validated[String, String] = {
    if (username.isEmpty)                   Validated.invalid("Username cannot be empty")
    else if (username.length < 3)           Validated.invalid("Username must be at least 3 characters")
    else if (username.length > 20)          Validated.invalid("Username cannot exceed 20 characters")
    else if (!username.matches("[a-zA-Z0-9_]+")) Validated.invalid("Username can only contain letters, numbers, and underscores")
    else Validated.valid(username.toLowerCase)
  }
  
  def validateEmail(email: String): Validated[String, String] = {
    if (email.isEmpty)                          Validated.invalid("Email cannot be empty")
    else if (!email.contains("@"))              Validated.invalid("Email must contain @")
    else if (!email.matches("^[^@]+@[^@]+\\.[^@]{2,}$")) Validated.invalid("Email format is invalid")
    else Validated.valid(email.trim.toLowerCase)
  }
  
  def validatePassword(password: String): Validated[String, String] = {
    val errors = List(
      if (password.length < 8)               Some("Password must be at least 8 characters") else None,
      if (!password.exists(_.isUpper))        Some("Password must contain uppercase letter") else None,
      if (!password.exists(_.isLower))        Some("Password must contain lowercase letter") else None,
      if (!password.exists(_.isDigit))        Some("Password must contain a digit") else None,
      if (!password.exists(!_.isLetterOrDigit)) Some("Password must contain special character") else None
    ).flatten
    
    if (errors.isEmpty) Validated.valid(password)
    else Validated.invalidNel(errors)
  }
  
  def validateAge(age: Int): Validated[String, Int] = {
    if (age < 13)  Validated.invalid("Must be at least 13 years old")
    else if (age > 150) Validated.invalid("Please enter a valid age")
    else Validated.valid(age)
  }
  
  def validate(form: RegistrationForm): Validated[String, RegistrationForm] = {
    val usernameV = validateUsername(form.username)
    val emailV    = validateEmail(form.email)
    val passwordV = validatePassword(form.password)
    val ageV      = validateAge(form.age)
    
    // Collect all errors
    val allErrors = List(usernameV, emailV, passwordV, ageV).collect {
      case Invalid(errs) => errs
    }.flatten
    
    if (allErrors.isEmpty)
      Valid(form.copy(
        username = (usernameV: @unchecked) match { case Valid(v) => v; case _ => form.username },
        email = (emailV: @unchecked) match { case Valid(v) => v; case _ => form.email }
      ))
    else
      Invalid(allErrors)
  }
}

object ValidatedDemo extends App {
  println("=== Validated Error Accumulation ===")
  
  val forms = List(
    RegistrationForm("alice123", "alice@example.com", "SecurePass1!", 25),
    RegistrationForm("ab", "not-an-email", "short", 10),
    RegistrationForm("", "valid@email.com", "NoSpecial1", 30),
    RegistrationForm("valid_user", "valid@email.com", "Valid1@Pass", 200)
  )
  
  forms.foreach { form =>
    println(s"\nValidating: $form")
    FormValidator.validate(form) match {
      case Valid(f)        => println(s"  Valid! username=${f.username}, email=${f.email}")
      case Invalid(errors) => errors.foreach(e => println(s"  Error: $e"))
    }
  }
}
```

---

## Step 285-290: Railway-Oriented Programming

```scala
// Railway-oriented: chain operations where any can fail
// Like a railway track that can switch to error track

object RailwayOriented extends App {
  
  sealed trait DomainError {
    def message: String
  }
  case class InputError(message: String) extends DomainError
  case class BusinessError(message: String) extends DomainError
  case class InfraError(message: String) extends DomainError
  
  type Rail[A] = Either[DomainError, A]
  
  case class OrderRequest(userId: String, itemId: String, quantity: Int, couponCode: Option[String])
  case class ValidatedRequest(userId: String, itemId: String, quantity: Int, discountPct: Double)
  case class EnrichedOrder(userId: String, item: String, price: Double, quantity: Int, discount: Double)
  case class PlacedOrder(id: String, total: Double, message: String)
  
  // Step 1: Validate input
  def validateInput(req: OrderRequest): Rail[ValidatedRequest] = {
    if (req.userId.isEmpty) Left(InputError("User ID required"))
    else if (req.itemId.isEmpty) Left(InputError("Item ID required"))
    else if (req.quantity <= 0) Left(InputError("Quantity must be positive"))
    else if (req.quantity > 100) Left(InputError("Max quantity is 100"))
    else {
      val discount = req.couponCode match {
        case Some("SAVE10") => 10.0
        case Some("SAVE20") => 20.0
        case Some(code)     => return Left(InputError(s"Invalid coupon: $code"))
        case None           => 0.0
      }
      Right(ValidatedRequest(req.userId, req.itemId, req.quantity, discount))
    }
  }
  
  // Step 2: Enrich with domain data
  val itemDb = Map(
    "ITEM-001" -> ("Laptop", 45000.0),
    "ITEM-002" -> ("Mouse", 850.0),
    "ITEM-003" -> ("Keyboard", 1200.0)
  )
  
  val userDb = Set("USER-001", "USER-002", "USER-003")
  
  def enrichOrder(req: ValidatedRequest): Rail[EnrichedOrder] = {
    if (!userDb.contains(req.userId))
      Left(BusinessError(s"User not found: ${req.userId}"))
    else itemDb.get(req.itemId) match {
      case None => Left(BusinessError(s"Item not found: ${req.itemId}"))
      case Some((name, price)) =>
        if (price * req.quantity > 500000)
          Left(BusinessError("Order exceeds maximum value"))
        else
          Right(EnrichedOrder(req.userId, name, price, req.quantity, req.discountPct))
    }
  }
  
  // Step 3: Apply business rules
  def applyRules(order: EnrichedOrder): Rail[EnrichedOrder] = {
    // Check inventory (simulated)
    val inStock = order.item != "Laptop" || order.quantity <= 5
    if (!inStock) Left(BusinessError(s"Insufficient stock for ${order.item}"))
    else Right(order)
  }
  
  // Step 4: Place order
  def placeOrder(order: EnrichedOrder): Rail[PlacedOrder] = {
    val subtotal = order.price * order.quantity
    val discount = subtotal * order.discount / 100
    val total = subtotal - discount
    val id = s"ORD-${System.currentTimeMillis() % 10000}"
    Right(PlacedOrder(id, total, 
      s"Order placed: ${order.quantity}x ${order.item} = ฿$total (saved ฿$discount)"))
  }
  
  // Railway: chain all steps
  def processOrder(req: OrderRequest): Rail[PlacedOrder] = for {
    validated <- validateInput(req)
    enriched  <- enrichOrder(validated)
    checked   <- applyRules(enriched)
    placed    <- placeOrder(checked)
  } yield placed
  
  println("=== Railway-Oriented Order Processing ===")
  
  val testCases = List(
    OrderRequest("USER-001", "ITEM-002", 2, Some("SAVE10")),    // Happy path
    OrderRequest("", "ITEM-001", 1, None),                      // Validation error
    OrderRequest("USER-999", "ITEM-001", 1, None),              // User not found
    OrderRequest("USER-001", "ITEM-001", 10, None),             // Stock error
    OrderRequest("USER-002", "ITEM-001", 1, Some("INVALID")),   // Bad coupon
    OrderRequest("USER-003", "ITEM-003", 1, Some("SAVE20"))     // With discount
  )
  
  testCases.foreach { req =>
    println(s"\nRequest: userId=${req.userId}, item=${req.itemId}, qty=${req.quantity}, coupon=${req.couponCode}")
    processOrder(req) match {
      case Right(order)            => println(s"  SUCCESS: ${order.message}")
      case Left(InputError(msg))   => println(s"  INPUT ERROR: $msg")
      case Left(BusinessError(msg)) => println(s"  BUSINESS ERROR: $msg")
      case Left(InfraError(msg))   => println(s"  INFRA ERROR: $msg")
    }
  }
  
  // Parallel error handling with cats-style Validated + Either
  def collectResults[E, A](results: List[Either[E, A]]): Either[List[E], List[A]] = {
    val (errors, values) = results.partitionMap(identity)
    if (errors.isEmpty) Right(values) else Left(errors)
  }
  
  val itemIds = List("ITEM-001", "ITEM-999", "ITEM-002", "ITEM-888")
  val lookupResults = itemIds.map(id => itemDb.get(id).toRight(s"Not found: $id"))
  
  println(s"\n=== Collecting Results ===")
  collectResults(lookupResults) match {
    case Right(items)  => println(s"All found: $items")
    case Left(errors)  => println(s"Some errors: $errors")
  }
}
```

---

## สรุป Part 29

| Type | Short-circuits | Errors | ใช้เมื่อ |
|------|---------------|--------|---------|
| `Option[A]` | Yes | None | Simple absence |
| `Either[E, A]` | Yes | 1 error | Typed errors |
| `Try[A]` | Yes | Throwable | Java exceptions |
| `Validated[E, A]` | No | Multiple | Form validation |
| `IO[A]` | No | Lazy | Side effects |
| `Future[A]` | Short | Non-blocking | Async |

---

## แบบฝึกหัด Part 29

**ข้อ 1:** สร้าง `Result[A]` monad ที่ combine `Option` + `Either` + error messages

**ข้อ 2:** Implement config parser ที่ใช้ Validated เพื่อ collect ทุก missing/invalid fields พร้อมกัน

**ข้อ 3:** สร้าง retry mechanism: `retry[A](n: Int)(task: => Try[A]): Try[A]` with exponential backoff

**ข้อ 4:** Implement `handleErrors` middleware ที่แปลง domain errors → HTTP responses

**ข้อ 5:** สร้าง `SafeParser[A]` ที่ parse JSON string → Either[List[ParseError], A] โดย accumulate errors

---

➡️ ต่อไป: [Part 30 — Standard Library Deep Dive](part-30-standard-library-deep-dive.md)
