# Part 23: Advanced Pattern Matching

## Steps 221-230: Deep Patterns, Guards, @binding, Extractors

---

## Step 221: Pattern Matching พื้นฐาน (Review)

```scala
object PatternMatchingReview extends App {
  
  // Basic patterns
  def describe(x: Any): String = x match {
    case null        => "null"
    case 0           => "zero"
    case n: Int      => s"integer: $n"
    case s: String   => s"string: '$s'"
    case true        => "true"
    case false       => "false"
    case d: Double   => f"double: $d%.2f"
    case list: List[_] => s"list with ${list.size} elements"
    case _           => s"unknown: $x"
  }
  
  List(null, 0, 42, "hello", true, 3.14, List(1,2,3), Array(1)).foreach { x =>
    println(s"  ${describe(x)}")
  }
}
```

---

## Step 222: Deep Nested Patterns

```scala
object DeepPatterns extends App {
  
  // Nested case class patterns
  case class Address(city: String, country: String)
  case class Contact(email: String, phone: Option[String], address: Address)
  case class Person(name: String, age: Int, contact: Contact)
  
  def describeLocation(person: Person): String = person match {
    // Deep pattern - match nested structure
    case Person(name, _, Contact(_, _, Address("Bangkok", "Thailand"))) =>
      s"$name lives in Bangkok, Thailand"
    
    case Person(name, _, Contact(_, _, Address(city, "Thailand"))) =>
      s"$name lives in $city, Thailand"
    
    case Person(name, _, Contact(_, _, Address(city, country))) =>
      s"$name lives in $city, $country"
  }
  
  val p1 = Person("Alice", 30, Contact("alice@test.com", None, Address("Bangkok", "Thailand")))
  val p2 = Person("Bob", 25, Contact("bob@test.com", Some("+66812345678"), Address("Chiang Mai", "Thailand")))
  val p3 = Person("Carol", 35, Contact("carol@test.com", None, Address("London", "UK")))
  
  println("=== Deep Patterns ===")
  println(describeLocation(p1))
  println(describeLocation(p2))
  println(describeLocation(p3))
  
  // Matching inside collections
  def analyzeList(xs: List[Int]): String = xs match {
    case Nil                         => "empty"
    case x :: Nil                    => s"single: $x"
    case x :: y :: Nil               => s"pair: ($x, $y)"
    case 1 :: 2 :: rest              => s"starts with 1,2: then $rest"
    case x :: _ if x > 100          => s"starts big: $x"
    case x :: y :: _ if x > y       => s"decreasing start: $x > $y"
    case _ :: _ :: _ :: Nil          => s"exactly 3 elements"
    case _                           => s"other: $xs"
  }
  
  println("\n=== List Patterns ===")
  List(
    List.empty[Int],
    List(42),
    List(1, 2),
    List(1, 2, 3, 4),
    List(200, 50),
    List(10, 5, 20),
    List(5, 10, 15)
  ).foreach(l => println(s"  $l => ${analyzeList(l)}"))
  
  // Nested Option patterns
  def processOptional(opt: Option[Option[Int]]): String = opt match {
    case None               => "outer None"
    case Some(None)         => "inner None"
    case Some(Some(n)) if n > 0 => s"positive: $n"
    case Some(Some(n))      => s"non-positive: $n"
  }
  
  println("\n=== Nested Options ===")
  List(None, Some(None), Some(Some(5)), Some(Some(-3))).foreach { x =>
    println(s"  $x => ${processOptional(x)}")
  }
}
```

---

## Step 223: Guards ใน Pattern Matching

```scala
object PatternGuards extends App {
  
  // Guards with if conditions
  case class Order(id: String, amount: Double, status: String, priority: Int)
  
  def processOrder(order: Order): String = order match {
    case Order(id, amount, "pending", priority) if priority > 8 && amount > 10000 =>
      s"[$id] URGENT high-value: ${amount}THB"
    
    case Order(id, amount, "pending", _) if amount > 10000 =>
      s"[$id] High-value order: ${amount}THB"
    
    case Order(id, _, "pending", priority) if priority > 8 =>
      s"[$id] Urgent priority order"
    
    case Order(id, _, "cancelled", _) =>
      s"[$id] Already cancelled"
    
    case Order(id, amount, "pending", _) if amount <= 0 =>
      s"[$id] Invalid amount: $amount"
    
    case Order(id, _, status, _) =>
      s"[$id] Regular order (status: $status)"
  }
  
  val orders = List(
    Order("O001", 15000.0, "pending", 9),
    Order("O002", 5000.0, "pending", 5),
    Order("O003", 12000.0, "pending", 3),
    Order("O004", 3000.0, "cancelled", 7),
    Order("O005", -100.0, "pending", 1),
    Order("O006", 500.0, "shipped", 2)
  )
  
  println("=== Pattern Guards ===")
  orders.foreach(o => println(s"  ${processOrder(o)}"))
  
  // Guards with extractors
  object Even {
    def unapply(n: Int): Option[Int] = if (n % 2 == 0) Some(n) else None
  }
  
  object Odd {
    def unapply(n: Int): Option[Int] = if (n % 2 != 0) Some(n) else None
  }
  
  def classifyNumber(n: Int): String = n match {
    case Even(n) if n > 0  => s"positive even: $n"
    case Even(n)           => s"non-positive even: $n"
    case Odd(n) if n > 0   => s"positive odd: $n"
    case Odd(n)            => s"non-positive odd: $n"
  }
  
  println("\n=== Custom Extractors with Guards ===")
  List(-3, -2, 0, 1, 2, 5, 6).foreach { n =>
    println(s"  $n: ${classifyNumber(n)}")
  }
}
```

---

## Step 224: @ Binding Patterns

```scala
object AtBinding extends App {
  
  // @binding: bind the whole pattern while also deconstructing
  
  case class User(id: Long, name: String, role: String)
  case class Permission(userId: Long, resource: String, actions: List[String])
  
  def checkPermission(user: User, permission: Permission): String = {
    (user, permission) match {
      // Bind 'u' to the whole user, while also checking role
      case (u @ User(_, _, "admin"), _) =>
        s"${u.name} is admin - all permissions granted"
      
      // Bind 'p' and also check userId matches
      case (User(id, name, _), p @ Permission(uid, _, _)) if id == uid =>
        s"$name has permission for ${p.resource}: ${p.actions}"
      
      case (User(_, name, _), Permission(uid, resource, _)) =>
        s"$name (mismatched user $uid) cannot access $resource"
    }
  }
  
  val admin = User(1L, "Alice", "admin")
  val user = User(2L, "Bob", "user")
  val perm1 = Permission(2L, "orders", List("read", "write"))
  val perm2 = Permission(99L, "products", List("read"))
  
  println("=== @ Binding ===")
  println(checkPermission(admin, perm1))
  println(checkPermission(user, perm1))
  println(checkPermission(user, perm2))
  
  // @ binding in nested patterns
  def processNested(data: Option[List[Int]]): String = data match {
    case None                          => "nothing"
    case Some(Nil)                     => "empty list"
    case Some(all @ (head :: tail))    => 
      s"list starting with $head, full: $all"
    case Some(list @ _)               =>
      s"other list: $list"
  }
  
  println("\n=== Nested @ Binding ===")
  List(None, Some(List.empty[Int]), Some(List(1,2,3))).foreach { d =>
    println(s"  $d => ${processNested(d)}")
  }
  
  // @ in case class matching  
  case class Transaction(id: String, amount: Double, tags: List[String])
  
  def analyzeTransaction(tx: Transaction): String = tx match {
    case t @ Transaction(_, amount, _) if amount > 10000 =>
      s"High-value: ${t.id} (${t.tags.mkString(", ")})"
    
    case t @ Transaction(_, _, tags) if tags.contains("suspicious") =>
      s"Suspicious: ${t.id} - amount: ${t.amount}"
    
    case Transaction(id, _, _) =>
      s"Normal: $id"
  }
  
  val txns = List(
    Transaction("T1", 15000.0, List("purchase", "premium")),
    Transaction("T2", 500.0, List("suspicious", "foreign")),
    Transaction("T3", 200.0, List("regular"))
  )
  
  println("\n=== Transaction Analysis ===")
  txns.foreach(t => println(s"  ${analyzeTransaction(t)}"))
}
```

---

## Step 225: Pattern Matching กับ Sealed Hierarchies

```scala
object SealedPatterns extends App {
  
  // Sealed hierarchy = exhaustive pattern matching
  sealed trait Result[+A, +E]
  case class Ok[A](value: A) extends Result[A, Nothing]
  case class Err[E](error: E) extends Result[Nothing, E]
  case object Loading extends Result[Nothing, Nothing]
  case object NotFound extends Result[Nothing, Nothing]
  
  sealed trait AppError
  case class NetworkError(message: String, code: Int) extends AppError
  case class ValidationError(field: String, message: String) extends AppError
  case class DatabaseError(query: String, cause: String) extends AppError
  case object AuthorizationError extends AppError
  
  def handleResult[A](result: Result[A, AppError]): String = result match {
    case Ok(value) => s"Success: $value"
    case Loading   => "Loading..."
    case NotFound  => "Not found"
    case Err(NetworkError(msg, 404)) => s"Resource not found: $msg"
    case Err(NetworkError(msg, 500)) => s"Server error: $msg"
    case Err(NetworkError(msg, code)) => s"Network error $code: $msg"
    case Err(ValidationError(field, msg)) => s"Validation failed for $field: $msg"
    case Err(DatabaseError(query, cause)) => s"DB error in '$query': $cause"
    case Err(AuthorizationError) => s"Not authorized!"
    // Compiler warns if any case is missing - compile-time safety!
  }
  
  val results: List[Result[String, AppError]] = List(
    Ok("Data successfully loaded"),
    Loading,
    NotFound,
    Err(NetworkError("Connection timeout", 500)),
    Err(ValidationError("email", "Must contain @")),
    Err(AuthorizationError)
  )
  
  println("=== Sealed Pattern Matching ===")
  results.foreach(r => println(s"  ${handleResult(r)}"))
  
  // Recursive sealed type
  sealed trait Expr
  case class Literal(value: Double) extends Expr
  case class Variable(name: String) extends Expr
  case class BinaryOp(op: String, left: Expr, right: Expr) extends Expr
  case class UnaryOp(op: String, expr: Expr) extends Expr
  case class FunctionCall(name: String, args: List[Expr]) extends Expr
  
  type Env = Map[String, Double]
  
  def evaluate(expr: Expr, env: Env = Map.empty): Double = expr match {
    case Literal(v) => v
    case Variable(name) => env.getOrElse(name, throw new RuntimeException(s"Unknown variable: $name"))
    case BinaryOp("+", l, r) => evaluate(l, env) + evaluate(r, env)
    case BinaryOp("-", l, r) => evaluate(l, env) - evaluate(r, env)
    case BinaryOp("*", l, r) => evaluate(l, env) * evaluate(r, env)
    case BinaryOp("/", l, r) => evaluate(l, env) / evaluate(r, env)
    case UnaryOp("-", e)     => -evaluate(e, env)
    case FunctionCall("sin", List(e)) => Math.sin(evaluate(e, env))
    case FunctionCall("cos", List(e)) => Math.cos(evaluate(e, env))
    case FunctionCall("sqrt", List(e)) => Math.sqrt(evaluate(e, env))
    case e => throw new RuntimeException(s"Cannot evaluate: $e")
  }
  
  def prettyPrint(expr: Expr): String = expr match {
    case Literal(v) => v.toString
    case Variable(name) => name
    case BinaryOp(op, l, r) => s"(${prettyPrint(l)} $op ${prettyPrint(r)})"
    case UnaryOp(op, e) => s"$op${prettyPrint(e)}"
    case FunctionCall(name, args) => s"$name(${args.map(prettyPrint).mkString(", ")})"
  }
  
  // (3 + x) * sqrt(y)
  val expr = BinaryOp("*",
    BinaryOp("+", Literal(3), Variable("x")),
    FunctionCall("sqrt", List(Variable("y")))
  )
  
  val env = Map("x" -> 4.0, "y" -> 16.0)
  
  println(s"\n=== Expression Evaluator ===")
  println(s"Expression: ${prettyPrint(expr)}")
  println(s"With x=4, y=16: ${evaluate(expr, env)}")
}
```

---

## Step 226-230: Advanced Cases

```scala
object AdvancedPatternMatching extends App {
  
  // Pattern matching on tuples
  def classifyPoint(point: (Double, Double)): String = point match {
    case (0.0, 0.0) => "origin"
    case (x, 0.0)   => s"on x-axis at $x"
    case (0.0, y)   => s"on y-axis at $y"
    case (x, y) if x > 0 && y > 0 => s"quadrant I ($x, $y)"
    case (x, y) if x < 0 && y > 0 => s"quadrant II ($x, $y)"
    case (x, y) if x < 0 && y < 0 => s"quadrant III ($x, $y)"
    case (x, y)     => s"quadrant IV ($x, $y)"
  }
  
  println("=== Point Classification ===")
  List((0.0,0.0),(3.0,0.0),(0.0,4.0),(2.0,3.0),(-1.0,2.0),(-2.0,-3.0),(1.0,-4.0))
    .foreach(p => println(s"  $p: ${classifyPoint(p)}"))
  
  // Pattern matching for parsing
  sealed trait Token
  case class NumberToken(value: Double) extends Token
  case class OperatorToken(op: Char) extends Token
  case class IdentifierToken(name: String) extends Token
  case object LeftParen extends Token
  case object RightParen extends Token
  
  def parseToken(s: String): Token = s.trim match {
    case n if n.matches("-?\\d+\\.?\\d*") => NumberToken(n.toDouble)
    case "+" | "-" | "*" | "/" if s.length == 1 => OperatorToken(s.head)
    case "(" => LeftParen
    case ")" => RightParen
    case id if id.matches("[a-zA-Z_][a-zA-Z0-9_]*") => IdentifierToken(id)
    case unknown => throw new IllegalArgumentException(s"Unknown token: $unknown")
  }
  
  val tokens = List("3.14", "+", "x", "*", "(", "y", ")", "-", "2")
  println("\n=== Token Parser ===")
  tokens.map(parseToken).foreach(t => println(s"  '$t' -> ${t.getClass.getSimpleName}"))
  
  // Pattern matching for state machine
  sealed trait State
  case object Idle extends State
  case object Processing extends State
  case class Error(msg: String) extends State
  case object Done extends State
  
  sealed trait Event2
  case class Start(data: String) extends Event2
  case class Complete(result: String) extends Event2
  case class Fail(reason: String) extends Event2
  case object Reset extends Event2
  
  def transition(state: State, event: Event2): (State, List[String]) = (state, event) match {
    case (Idle, Start(data))          => (Processing, List(s"Started with: $data"))
    case (Processing, Complete(res))  => (Done, List(s"Completed: $res"))
    case (Processing, Fail(reason))   => (Error(reason), List(s"Failed: $reason"))
    case (Error(_), Reset)            => (Idle, List("Reset to idle"))
    case (Done, Reset)                => (Idle, List("Reset after done"))
    case (s, e)                       => (s, List(s"Invalid: $e in $s"))
  }
  
  println("\n=== State Machine ===")
  val events: List[Event2] = List(
    Start("process data"),
    Complete("result-123"),
    Reset,
    Start("another task"),
    Fail("network error"),
    Reset
  )
  
  events.foldLeft((Idle: State, List.empty[String])) { case ((state, _), event) =>
    val (nextState, actions) = transition(state, event)
    println(s"  $state + $event => $nextState: ${actions.mkString(", ")}")
    (nextState, actions)
  }
}
```

---

## สรุป Part 23

| Pattern Type | Syntax | ตัวอย่าง |
|-------------|--------|---------|
| Literal | `case 42` | Match exact value |
| Type | `case n: Int` | Type check + bind |
| Constructor | `case Some(x)` | Deconstruct case class |
| Guard | `case x if x > 0` | Add condition |
| @ binding | `case p @ Person(n, _)` | Bind + deconstruct |
| Deep nested | `case Some(List(h, _))` | Multi-level match |
| Tuple | `case (x, y)` | Deconstruct tuple |
| Wildcard | `case _` | Match anything |
| Alt patterns | `case 1 \| 2 \| 3` | OR patterns |

---

## แบบฝึกหัด Part 23

**ข้อ 1:** สร้าง JSON AST ด้วย sealed trait แล้ว implement: `parse`, `render`, `get(path)`, `set(path, value)`

**ข้อ 2:** Implement rule engine ด้วย pattern matching: `case Rule(condition, action)` ที่ apply rules ไปยัง data

**ข้อ 3:** สร้าง HTTP router ด้วย pattern matching: `route(method, path)` → handler

**ข้อ 4:** Implement simple type checker สำหรับ expression language ด้วย sealed trait recursion

**ข้อ 5:** สร้าง workflow engine ที่ transition between states โดยใช้ exhaustive pattern matching

---

➡️ ต่อไป: [Part 24 — Extractors and Unapply](part-24-extractors-and-unapply.md)
