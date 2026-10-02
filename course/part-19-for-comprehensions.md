# Part 19: For Comprehensions

## Steps 181-190: For Comprehensions Deep Dive, Monadic Operations, Desugaring

---

## Step 181: For Comprehension พื้นฐาน

```scala
object ForComprehensionBasics extends App {
  
  // Basic for comprehension
  val numbers = List(1, 2, 3, 4, 5)
  
  // Simple loop
  for (n <- numbers) println(s"  $n")
  
  // With yield - creates a new collection
  val doubled = for (n <- numbers) yield n * 2
  println(s"doubled: $doubled")
  
  // Equivalent to map
  val doubled2 = numbers.map(n => n * 2)
  println(s"doubled2: $doubled2")
  
  // With guard/filter
  val evens = for {
    n <- numbers
    if n % 2 == 0
  } yield n
  println(s"evens: $evens")
  
  // Multiple generators - cross product
  val colors = List("red", "blue")
  val sizes = List("S", "M", "L")
  
  val combinations = for {
    color <- colors
    size  <- sizes
  } yield s"$color-$size"
  
  println(s"\ncombinations: $combinations")
  
  // Nested computation
  val matrix = List(List(1, 2, 3), List(4, 5, 6), List(7, 8, 9))
  val flatElements = for {
    row  <- matrix
    elem <- row
    if elem % 2 != 0
  } yield elem
  
  println(s"odd elements: $flatElements")
  
  // Local variable in for comprehension
  val enriched = for {
    n <- numbers
    doubled = n * 2
    tripled = n * 3
  } yield s"$n -> $doubled/$tripled"
  
  println(s"\nenriched: $enriched")
}
```

---

## Step 182: Desugaring — For Comprehension คือ flatMap/map/filter

```scala
object Desugaring extends App {
  
  val xs = List(1, 2, 3)
  val ys = List(10, 20, 30)
  
  // For comprehension
  val forResult = for {
    x <- xs
    y <- ys
    if x + y > 20
  } yield x * y
  
  // Desugared equivalent
  val desugared = xs.flatMap { x =>
    ys.withFilter(y => x + y > 20).map { y =>
      x * y
    }
  }
  
  println(s"forResult: $forResult")
  println(s"desugared: $desugared")
  println(s"equal: ${forResult == desugared}")
  
  // Single generator with yield = map
  val single = for (x <- xs) yield x * 2
  val singleDesugared = xs.map(x => x * 2)
  println(s"\nsingle == singleDesugared: ${single == singleDesugared}")
  
  // Two generators = flatMap + map
  val two = for { x <- xs; y <- ys } yield (x, y)
  val twoDesugared = xs.flatMap(x => ys.map(y => (x, y)))
  println(s"two == twoDesugared: ${two == twoDesugared}")
  
  // Three generators = nested flatMaps
  val zs = List('a', 'b')
  val three = for { x <- xs; y <- ys; z <- zs } yield (x, y, z)
  val threeDesugared = xs.flatMap(x => ys.flatMap(y => zs.map(z => (x, y, z))))
  println(s"three count: ${three.size}, threeDesugared count: ${threeDesugared.size}")
  println(s"equal: ${three == threeDesugared}")
  
  // Without yield = foreach
  for (x <- xs) println(s"  x = $x")
  xs.foreach(x => println(s"  x = $x"))
  
  // withFilter vs filter
  // withFilter is lazy and doesn't create intermediate collection
  val filtered = for {
    x <- (1 to 1000000).toList
    if x % 2 == 0  // Uses withFilter - no intermediate collection
    if x < 10
  } yield x
  
  println(s"\nfiltered: $filtered")
}
```

---

## Step 183: For Comprehensions กับ Option

```scala
object ForWithOption extends App {
  
  // Option in for comprehension - short-circuits on None
  def divide(a: Int, b: Int): Option[Double] = 
    if (b == 0) None else Some(a.toDouble / b)
  
  def sqrt(n: Double): Option[Double] = 
    if (n < 0) None else Some(Math.sqrt(n))
  
  def parseDouble(s: String): Option[Double] = 
    scala.util.Try(s.toDouble).toOption
  
  // Chain operations - short-circuit if any step fails
  def compute(s: String, divisor: Int): Option[Double] = for {
    parsed  <- parseDouble(s)
    divided <- divide(parsed.toInt, divisor)
    result  <- sqrt(divided)
  } yield result
  
  println("=== Option Comprehension ===")
  println(compute("36", 4))   // Some(3.0): sqrt(36/4) = sqrt(9) = 3
  println(compute("abc", 4))  // None: parse fails
  println(compute("36", 0))   // None: divide fails
  println(compute("-16", 4))  // None: sqrt of negative
  
  // User lookup chain
  case class User(id: Long, name: String, adminId: Option[Long])
  case class Admin(id: Long, email: String, permissions: List[String])
  
  val users = Map(
    1L -> User(1L, "Alice", Some(10L)),
    2L -> User(2L, "Bob", None),
    3L -> User(3L, "Carol", Some(10L))
  )
  
  val admins = Map(
    10L -> Admin(10L, "admin@company.com", List("read", "write", "delete"))
  )
  
  def getUserAdmin(userId: Long): Option[Admin] = for {
    user  <- users.get(userId)
    adminId <- user.adminId
    admin <- admins.get(adminId)
  } yield admin
  
  println("\n=== User-Admin Chain ===")
  List(1L, 2L, 3L, 99L).foreach { id =>
    getUserAdmin(id) match {
      case Some(admin) => println(s"User $id -> Admin: ${admin.email}")
      case None        => println(s"User $id -> No admin found")
    }
  }
  
  // Chaining with transformation
  def lookupPermissions(userId: Long, permission: String): Option[Boolean] = for {
    admin <- getUserAdmin(userId)
  } yield admin.permissions.contains(permission)
  
  println("\n=== Permissions ===")
  println(s"User 1 can write: ${lookupPermissions(1L, "write")}")
  println(s"User 2 can write: ${lookupPermissions(2L, "write")}")
}
```

---

## Step 184: For Comprehensions กับ Either

```scala
object ForWithEither extends App {
  
  // Either in for - short-circuits on Left
  def parseAge(s: String): Either[String, Int] = {
    scala.util.Try(s.toInt) match {
      case scala.util.Success(n) if n >= 0 && n <= 150 => Right(n)
      case scala.util.Success(n) => Left(s"Age out of range: $n")
      case scala.util.Failure(_) => Left(s"Invalid age: $s")
    }
  }
  
  def parseEmail(s: String): Either[String, String] = {
    if (s.matches("[^@]+@[^@]+\\.[^@]+")) Right(s.toLowerCase.trim)
    else Left(s"Invalid email: $s")
  }
  
  def parseName(s: String): Either[String, String] = {
    val trimmed = s.trim
    if (trimmed.length >= 2 && trimmed.length <= 100) Right(trimmed.capitalize)
    else Left(s"Name length must be 2-100 chars: '$s'")
  }
  
  case class RegisterRequest(name: String, age: String, email: String)
  case class User(name: String, age: Int, email: String)
  
  def validateRegister(req: RegisterRequest): Either[String, User] = for {
    name  <- parseName(req.name)
    age   <- parseAge(req.age)
    email <- parseEmail(req.email)
  } yield User(name, age, email)
  
  println("=== Either Comprehension ===")
  
  val requests = List(
    RegisterRequest("Alice Chen", "30", "alice@example.com"),
    RegisterRequest("", "25", "bob@example.com"),      // Bad name
    RegisterRequest("Carol", "abc", "carol@example.com"), // Bad age
    RegisterRequest("David", "200", "david@example.com"), // Age out of range
    RegisterRequest("Eve", "28", "not-an-email"),          // Bad email
  )
  
  requests.foreach { req =>
    validateRegister(req) match {
      case Right(user) => println(s"OK: $user")
      case Left(err)   => println(s"ERROR: $err")
    }
  }
  
  // Chaining database operations
  type DBResult[A] = Either[String, A]
  
  def findUser(id: Long): DBResult[User] = id match {
    case 1 => Right(User("Alice", 30, "alice@example.com"))
    case 2 => Right(User("Bob", 25, "bob@example.com"))
    case _ => Left(s"User $id not found")
  }
  
  def findUserOrders(userId: Long): DBResult[List[String]] = userId match {
    case 1 => Right(List("O001", "O002", "O003"))
    case _ => Right(List.empty)
  }
  
  def getOrderDetails(orderId: String): DBResult[Map[String, Any]] = orderId match {
    case "O001" => Right(Map("id" -> "O001", "amount" -> 1500.0, "status" -> "delivered"))
    case "O002" => Right(Map("id" -> "O002", "amount" -> 3000.0, "status" -> "processing"))
    case _ => Left(s"Order $orderId not found")
  }
  
  println("\n=== DB Chain ===")
  val result = for {
    user   <- findUser(1L)
    orders <- findUserOrders(1L)
    detail <- getOrderDetails(orders.head)
  } yield (user, orders, detail)
  
  result match {
    case Right((user, orders, detail)) =>
      println(s"User: ${user.name}")
      println(s"Orders: $orders")
      println(s"First order: $detail")
    case Left(err) => println(s"Error: $err")
  }
}
```

---

## Step 185: For Comprehensions กับ Future

```scala
import scala.concurrent._
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._

object ForWithFuture extends App {
  
  // Async operations with Future
  def fetchUser(id: Long): Future[Map[String, Any]] = Future {
    Thread.sleep(50)  // Simulate latency
    id match {
      case 1 => Map("id" -> 1, "name" -> "Alice", "points" -> 500)
      case 2 => Map("id" -> 2, "name" -> "Bob", "points" -> 1200)
      case _ => throw new Exception(s"User $id not found")
    }
  }
  
  def fetchProductPrice(productId: String): Future[Double] = Future {
    Thread.sleep(30)
    productId match {
      case "laptop" => 45000.0
      case "mouse"  => 850.0
      case _        => throw new Exception(s"Product $productId not found")
    }
  }
  
  def calculateDiscount(userPoints: Int, price: Double): Future[Double] = Future {
    val discountPct = userPoints match {
      case p if p > 1000 => 0.15
      case p if p > 500  => 0.10
      case _             => 0.05
    }
    price * discountPct
  }
  
  // Sequential: each step waits for previous (but still non-blocking)
  val sequentialResult = for {
    user     <- fetchUser(1L)
    points   = user("points").asInstanceOf[Int]
    price    <- fetchProductPrice("laptop")
    discount <- calculateDiscount(points, price)
  } yield {
    Map(
      "user" -> user("name"),
      "price" -> price,
      "discount" -> discount,
      "final" -> (price - discount)
    )
  }
  
  println("=== Future Comprehension ===")
  val result = Await.result(sequentialResult, 5.seconds)
  result.foreach { case (k, v) => println(s"  $k: $v") }
  
  // Parallel: fetch independently then combine
  val user1Future = fetchUser(1L)
  val user2Future = fetchUser(2L)
  
  val combined = for {
    user1 <- user1Future
    user2 <- user2Future  // These run in parallel!
  } yield List(user1, user2)
  
  val users = Await.result(combined, 5.seconds)
  println(s"\n=== Parallel Fetch ===")
  users.foreach(u => println(s"  $u"))
  
  // Error handling with Future
  val failedFetch = for {
    user <- fetchUser(99L)  // Will fail
  } yield user
  
  val recovered = failedFetch.recover {
    case e: Exception => Map("error" -> e.getMessage)
  }
  
  println(s"\n=== Error Recovery ===")
  val errResult = Await.result(recovered, 5.seconds)
  println(s"Result: $errResult")
}
```

---

## Step 186-190: Advanced Patterns

```scala
object AdvancedForComprehensions extends App {
  
  // Custom type that supports for comprehension
  // Must implement: map, flatMap, and optionally withFilter
  class Validated[+E, +A](val value: Either[List[E], A]) {
    
    def map[B](f: A => B): Validated[E, B] = 
      new Validated(value.map(f))
    
    // flatMap - accumulate errors
    def flatMap[EE >: E, B](f: A => Validated[EE, B]): Validated[EE, B] = 
      value match {
        case Left(errs) => f match {
          case _ => new Validated(Left(errs))
        }
        case Right(a) => f(a)
      }
    
    def withFilter(pred: A => Boolean): Validated[String, A] = 
      new Validated(value match {
        case Right(a) if pred(a) => Right(a)
        case Right(a) => Left(List("Filter condition failed"))
        case Left(errs) => Left(errs.asInstanceOf[List[String]])
      })
    
    def isValid: Boolean = value.isRight
    def errors: List[E] = value.left.getOrElse(List.empty)
    def result: Option[A] = value.toOption
  }
  
  object Validated {
    def valid[A](a: A): Validated[Nothing, A] = new Validated(Right(a))
    def invalid[E](errors: E*): Validated[E, Nothing] = new Validated(Left(errors.toList))
    
    def fromPredicate[A](a: A, pred: A => Boolean, err: String): Validated[String, A] =
      if (pred(a)) valid(a) else invalid(err)
  }
  
  // Form validation using custom Validated
  case class RegistrationForm(username: String, password: String, age: Int, email: String)
  
  def validateUsername(s: String): Validated[String, String] = {
    val trimmed = s.trim
    if (trimmed.length < 3) Validated.invalid("Username must be at least 3 chars")
    else if (trimmed.length > 20) Validated.invalid("Username must be at most 20 chars")
    else if (!trimmed.matches("[a-zA-Z0-9_]+")) Validated.invalid("Username can only contain letters, numbers, underscore")
    else Validated.valid(trimmed.toLowerCase)
  }
  
  def validatePassword(s: String): Validated[String, String] = {
    if (s.length < 8) Validated.invalid("Password must be at least 8 chars")
    else if (!s.exists(_.isUpper)) Validated.invalid("Password must contain uppercase letter")
    else if (!s.exists(_.isDigit)) Validated.invalid("Password must contain digit")
    else Validated.valid(s)
  }
  
  println("=== Custom Validated Type ===")
  
  val goodUser = for {
    username <- validateUsername("alice_chen")
    password <- validatePassword("SecurePass123")
  } yield (username, password)
  
  println(s"Good: ${goodUser.result}")
  
  val badUser = for {
    username <- validateUsername("a")  // Too short
    password <- validatePassword("weak")  // Multiple issues
  } yield (username, password)
  
  println(s"Bad valid: ${badUser.isValid}")
  
  // Multiple collections in comprehension
  println("\n=== Multi-Collection Comprehension ===")
  
  // Pythagorean triples
  val triples = for {
    c <- 1 to 20
    b <- 1 to c
    a <- 1 to b
    if a * a + b * b == c * c
  } yield (a, b, c)
  
  println(s"Pythagorean triples up to 20: $triples")
  
  // Sieve of Eratosthenes using for comprehension
  def sieve(nums: LazyList[Int]): LazyList[Int] = {
    nums.head #:: sieve(nums.tail.filter(_ % nums.head != 0))
  }
  
  val primes = sieve(LazyList.from(2))
  println(s"\nFirst 20 primes: ${primes.take(20).toList}")
  
  // Combination generation
  def combinations[A](list: List[A], n: Int): List[List[A]] = {
    if (n == 0) List(List.empty)
    else if (list.isEmpty) List.empty
    else {
      val withHead = combinations(list.tail, n - 1).map(list.head :: _)
      val withoutHead = combinations(list.tail, n)
      withHead ++ withoutHead
    }
  }
  
  println(s"\nCombinations of (1,2,3,4) choose 2: ${combinations(List(1,2,3,4), 2)}")
  
  // Nested for comprehension for matrix operations
  val matA = List(List(1, 2), List(3, 4))
  val matB = List(List(5, 6), List(7, 8))
  
  def matMul(a: List[List[Int]], b: List[List[Int]]): List[List[Int]] = {
    for (row <- a) yield
      for (col <- b.transpose) yield
        (row zip col).map { case (x, y) => x * y }.sum
  }
  
  println(s"\n=== Matrix Multiplication ===")
  println(s"A = $matA")
  println(s"B = $matB")
  println(s"A*B = ${matMul(matA, matB)}")
}
```

---

## สรุป Part 19

| Syntax | Desugars To | หมายเหตุ |
|--------|------------|---------|
| `for (x <- xs) yield f(x)` | `xs.map(x => f(x))` | Single generator |
| `for (x <- xs; y <- ys) yield (x,y)` | `xs.flatMap(x => ys.map(y => (x,y)))` | Two generators |
| `for (x <- xs; if pred(x)) yield x` | `xs.withFilter(pred).map(identity)` | With filter |
| `for (x <- xs) effect(x)` | `xs.foreach(x => effect(x))` | No yield |
| `for (x <- opt) yield f(x)` | `opt.map(f)` | With Option |
| `for { x <- opt1; y <- opt2 }` | `opt1.flatMap(x => opt2.map(y => ...))` | Chain Option |
| `for { x <- either }` | `either.map(...)` | Works same as Option |
| `for { x <- future }` | `future.map(...)` | Async chaining |

---

## แบบฝึกหัด Part 19

**ข้อ 1:** สร้าง `Result[E, A]` type ที่ support for-comprehension และ accumulate errors แทนที่จะ short-circuit

**ข้อ 2:** เขียน Sudoku validator ที่ใช้ for comprehensions ตรวจสอบ rows, columns, และ 3x3 boxes

**ข้อ 3:** Implement type-safe query builder ที่ใช้ for comprehensions: `for { user <- from("users"); if user.age > 18 } yield user.name`

**ข้อ 4:** สร้าง lazy data pipeline ด้วย `LazyList` และ for comprehensions ที่ process infinite streams

**ข้อ 5:** Implement graph path finding ด้วย for comprehensions ที่ explore paths

---

➡️ ต่อไป: [Part 20 — Type Classes Intro](part-20-type-classes-intro.md)
