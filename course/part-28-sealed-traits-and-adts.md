# Part 28: Sealed Traits and Algebraic Data Types (ADTs)

## Steps 271-280: Sealed Traits, Sum Types, Product Types, GADTs, Domain Modeling

---

## Step 271: ADT คืออะไร

```scala
// Algebraic Data Type = Sum Type + Product Type
// Sum type (OR): A | B | C — เลือกได้ 1 อย่าง (sealed trait + case classes)
// Product type (AND): A & B — มีทั้งหมด (case class)

// Sum type: Payment เป็น Card | BankTransfer | Cash | Crypto
sealed trait Payment
case class Card(number: String, expiry: String, cvv: String) extends Payment
case class BankTransfer(accountNo: String, bankCode: String, amount: Double) extends Payment
case class Cash(amount: Double, currency: String) extends Payment
case class Crypto(walletAddress: String, coinType: String, amount: Double) extends Payment

// Product type: Address มี street AND city AND country AND postalCode
case class Address(
  street: String,
  city: String,
  country: String,
  postalCode: String
)

// Nested ADT
sealed trait OrderStatus
case object Pending extends OrderStatus
case object Confirmed extends OrderStatus
case class Processing(stage: String, estimatedCompletion: java.time.LocalDate) extends OrderStatus
case class Shipped(trackingNumber: String, carrier: String) extends OrderStatus
case class Delivered(deliveredAt: java.time.LocalDate) extends OrderStatus
case class Cancelled(reason: String, refundAmount: Double) extends OrderStatus
case class Returned(reason: String, returnedAt: java.time.LocalDate) extends OrderStatus

case class Order(
  id: String,
  customerId: String,
  items: List[String],
  payment: Payment,
  shippingAddress: Address,
  status: OrderStatus
)

object ADTBasics extends App {
  
  // Process payment — exhaustive match guaranteed by compiler
  def processPayment(payment: Payment): String = payment match {
    case Card(number, expiry, _) =>
      s"Charging card ending in ${number.takeRight(4)}, expires $expiry"
    case BankTransfer(account, bank, amount) =>
      s"Transferring $amount from $account at bank $bank"
    case Cash(amount, currency) =>
      s"Receiving $amount $currency in cash"
    case Crypto(wallet, coin, amount) =>
      s"Receiving $amount $coin to wallet ${wallet.take(8)}..."
  }
  
  // Order status description
  def describeStatus(status: OrderStatus): String = status match {
    case Pending => "Order received, awaiting confirmation"
    case Confirmed => "Order confirmed, preparing to ship"
    case Processing(stage, date) => s"In processing: $stage (ETA: $date)"
    case Shipped(tracking, carrier) => s"Shipped via $carrier, tracking: $tracking"
    case Delivered(date) => s"Delivered on $date"
    case Cancelled(reason, refund) => s"Cancelled: $reason (refund: $refund)"
    case Returned(reason, date) => s"Returned on $date: $reason"
  }
  
  println("=== Payment Processing ===")
  val payments = List(
    Card("4111111111111111", "12/25", "123"),
    BankTransfer("1234567890", "SCB", 5000.0),
    Cash(100.0, "THB"),
    Crypto("1A1zP1eP5QGefi2DMPTfTL5SLmv7Divf", "BTC", 0.001)
  )
  payments.foreach(p => println(s"  ${processPayment(p)}"))
  
  println("\n=== Order Status ===")
  val statuses: List[OrderStatus] = List(
    Pending,
    Confirmed,
    Processing("Quality Check", java.time.LocalDate.now().plusDays(2)),
    Shipped("TH123456789", "DHL"),
    Delivered(java.time.LocalDate.now()),
    Cancelled("Out of stock", 250.0)
  )
  statuses.foreach(s => println(s"  ${describeStatus(s)}"))
}
```

---

## Step 272: Expression Problem และ ADTs

```scala
// AST (Abstract Syntax Tree) — classic ADT use case
sealed trait Expr
case class Num(value: Double) extends Expr
case class Var(name: String) extends Expr
case class Add(left: Expr, right: Expr) extends Expr
case class Mul(left: Expr, right: Expr) extends Expr
case class Div(left: Expr, right: Expr) extends Expr
case class Neg(expr: Expr) extends Expr
case class Pow(base: Expr, exp: Expr) extends Expr
case class Let(name: String, value: Expr, body: Expr) extends Expr

type Env = Map[String, Double]

object ExprEval {
  
  def eval(expr: Expr, env: Env = Map.empty): Either[String, Double] = expr match {
    case Num(v) => Right(v)
    case Var(name) => env.get(name).toRight(s"Undefined variable: $name")
    case Add(l, r) => for { lv <- eval(l, env); rv <- eval(r, env) } yield lv + rv
    case Mul(l, r) => for { lv <- eval(l, env); rv <- eval(r, env) } yield lv * rv
    case Div(l, r) => for {
      lv <- eval(l, env)
      rv <- eval(r, env)
      result <- if (rv == 0) Left("Division by zero") else Right(lv / rv)
    } yield result
    case Neg(e) => eval(e, env).map(-_)
    case Pow(base, exp) => for { bv <- eval(base, env); ev <- eval(exp, env) } yield Math.pow(bv, ev)
    case Let(name, value, body) => 
      for { v <- eval(value, env); result <- eval(body, env + (name -> v)) } yield result
  }
  
  def prettyPrint(expr: Expr): String = expr match {
    case Num(v)     => if (v == v.toLong) v.toLong.toString else v.toString
    case Var(name)  => name
    case Add(l, r)  => s"(${prettyPrint(l)} + ${prettyPrint(r)})"
    case Mul(l, r)  => s"(${prettyPrint(l)} * ${prettyPrint(r)})"
    case Div(l, r)  => s"(${prettyPrint(l)} / ${prettyPrint(r)})"
    case Neg(e)     => s"-(${prettyPrint(e)})"
    case Pow(b, e)  => s"(${prettyPrint(b)} ^ ${prettyPrint(e)})"
    case Let(n, v, b) => s"let $n = ${prettyPrint(v)} in ${prettyPrint(b)}"
  }
  
  def simplify(expr: Expr): Expr = expr match {
    case Add(Num(0), e) => simplify(e)
    case Add(e, Num(0)) => simplify(e)
    case Mul(Num(1), e) => simplify(e)
    case Mul(e, Num(1)) => simplify(e)
    case Mul(Num(0), _) => Num(0)
    case Mul(_, Num(0)) => Num(0)
    case Neg(Neg(e))    => simplify(e)
    case Add(l, r)      => Add(simplify(l), simplify(r))
    case Mul(l, r)      => Mul(simplify(l), simplify(r))
    case Div(l, r)      => Div(simplify(l), simplify(r))
    case Neg(e)         => Neg(simplify(e))
    case Pow(b, e)      => Pow(simplify(b), simplify(e))
    case other          => other
  }
}

object ExprDemo extends App {
  import ExprEval._
  
  // 2 * (x + 3) / (y - 1)
  val expr = Div(Mul(Num(2), Add(Var("x"), Num(3))), Add(Var("y"), Neg(Num(1))))
  val env = Map("x" -> 5.0, "y" -> 4.0)
  
  println("=== Expression Evaluator ===")
  println(s"Expr: ${prettyPrint(expr)}")
  println(s"Result: ${eval(expr, env)}")
  
  // Let binding: let x = 10 in (x * x + 2 * x + 1)
  val quad = Let("x", Num(10),
    Add(Add(Mul(Var("x"), Var("x")), Mul(Num(2), Var("x"))), Num(1)))
  println(s"\nLet expr: ${prettyPrint(quad)}")
  println(s"Result: ${eval(quad)}")
  
  // Simplification
  val messy = Mul(Num(1), Add(Num(0), Mul(Var("x"), Num(1))))
  println(s"\nBefore: ${prettyPrint(messy)}")
  println(s"After: ${prettyPrint(simplify(messy))}")
}
```

---

## Step 273: Effect ADTs — Modeling Effectful Operations

```scala
// Free monad style — describe operations as data
sealed trait ConsoleOp[+A]
case class ReadLine[A](next: String => A) extends ConsoleOp[A]
case class WriteLine[A](line: String, next: A) extends ConsoleOp[A]
case class Exit[A](code: Int) extends ConsoleOp[A]

// Simple interpreter
def runConsole[A](program: ConsoleOp[A]): A = program match {
  case ReadLine(next)       => next(scala.io.StdIn.readLine())
  case WriteLine(line, next) => println(line); next
  case Exit(code)            => sys.exit(code)
}

// Task ADT — Describe async computations
sealed trait Task[+A] {
  def map[B](f: A => B): Task[B] = FlatMap(this, (a: A) => Pure(f(a)))
  def flatMap[B](f: A => Task[B]): Task[B] = FlatMap(this, f)
  def *>[B](other: Task[B]): Task[B] = flatMap(_ => other)
  def recover[B >: A](f: Throwable => Task[B]): Task[B] = Recover(this, f)
}

case class Pure[A](value: A) extends Task[A]
case class Delay[A](thunk: () => A) extends Task[A]
case class FlatMap[A, B](task: Task[A], f: A => Task[B]) extends Task[B]
case class Recover[A](task: Task[A], f: Throwable => Task[A]) extends Task[A]
case class Async[A](callback: (Either[Throwable, A] => Unit) => Unit) extends Task[A]

object Task {
  def pure[A](value: A): Task[A] = Pure(value)
  def delay[A](thunk: => A): Task[A] = Delay(() => thunk)
  def fail[A](err: Throwable): Task[A] = Delay(() => throw err)
  
  def run[A](task: Task[A]): Either[Throwable, A] = try {
    Right(runUnsafe(task))
  } catch {
    case e: Throwable => Left(e)
  }
  
  private def runUnsafe[A](task: Task[A]): A = task match {
    case Pure(v)          => v
    case Delay(thunk)     => thunk()
    case FlatMap(t, f)    => runUnsafe(f(runUnsafe(t)))
    case Recover(t, f)    =>
      try runUnsafe(t)
      catch { case e: Throwable => runUnsafe(f(e)) }
    case Async(_)         => throw new RuntimeException("Cannot run Async synchronously")
  }
}

object EffectADTDemo extends App {
  
  // Build programs as data
  val fetchUser: Task[String] = Task.delay {
    println("  Fetching user...")
    "Alice"  // Simulate DB call
  }
  
  val fetchOrders: Task[List[String]] = Task.delay {
    println("  Fetching orders...")
    List("O-001", "O-002", "O-003")
  }
  
  val processOrder: String => Task[String] = order => Task.delay {
    println(s"  Processing $order...")
    s"Processed: $order"
  }
  
  // Compose tasks
  val program: Task[List[String]] = for {
    user    <- fetchUser
    _       <- Task.delay(println(s"  Got user: $user"))
    orders  <- fetchOrders
    results <- Task.pure(orders.map(o => s"$user:$o"))
  } yield results
  
  println("=== Task ADT ===")
  Task.run(program) match {
    case Right(results) => results.foreach(println)
    case Left(err)      => println(s"Error: $err")
  }
  
  // Error recovery
  val risky: Task[Int] = Task.delay {
    println("  Attempting risky operation...")
    throw new RuntimeException("Something went wrong")
  }
  
  val safe: Task[Int] = risky.recover { case e =>
    Task.delay {
      println(s"  Recovering from: ${e.getMessage}")
      -1
    }
  }
  
  println("\n=== Error Recovery ===")
  Task.run(safe) match {
    case Right(v)  => println(s"Result: $v")
    case Left(err) => println(s"Unrecovered: $err")
  }
}
```

---

## Step 274-280: Real-World ADT Patterns

```scala
// Domain-driven design with ADTs
object DomainModeling extends App {
  
  // Value objects as ADTs
  sealed trait EmailAddress {
    def value: String
  }
  
  object EmailAddress {
    private case class ValidEmail(value: String) extends EmailAddress
    
    def apply(s: String): Either[String, EmailAddress] = {
      val trimmed = s.trim.toLowerCase
      if (trimmed.matches("^[^@]+@[^@]+\\.[^@]{2,}$"))
        Right(ValidEmail(trimmed))
      else
        Left(s"Invalid email: $s")
    }
    
    def unsafe(s: String): EmailAddress = apply(s).fold(
      err => throw new IllegalArgumentException(err),
      identity
    )
  }
  
  sealed trait Money {
    def amount: BigDecimal
    def currency: String
    def +(other: Money): Either[String, Money] =
      if (currency == other.currency) Right(Money.of(amount + other.amount, currency))
      else Left(s"Cannot add $currency and ${other.currency}")
    def *(factor: BigDecimal): Money = Money.of(amount * factor, currency)
  }
  
  object Money {
    private case class MoneyValue(amount: BigDecimal, currency: String) extends Money
    
    def of(amount: BigDecimal, currency: String): Money = MoneyValue(amount, currency)
    def thb(amount: Double): Money = MoneyValue(BigDecimal(amount), "THB")
    def usd(amount: Double): Money = MoneyValue(BigDecimal(amount), "USD")
  }
  
  // Event Sourcing with ADTs
  sealed trait AccountEvent
  case class AccountOpened(id: String, owner: String, initialBalance: Money) extends AccountEvent
  case class MoneyDeposited(amount: Money, description: String) extends AccountEvent
  case class MoneyWithdrawn(amount: Money, description: String) extends AccountEvent
  case class AccountClosed(reason: String) extends AccountEvent
  case class InterestApplied(rate: BigDecimal, amount: Money) extends AccountEvent
  
  case class AccountState(
    id: String,
    owner: String,
    balance: Money,
    isOpen: Boolean,
    transactions: List[AccountEvent]
  )
  
  def applyEvent(state: Option[AccountState], event: AccountEvent): Either[String, AccountState] =
    (state, event) match {
      case (None, AccountOpened(id, owner, balance)) =>
        Right(AccountState(id, owner, balance, true, List(event)))
      
      case (Some(s), _) if !s.isOpen =>
        Left("Cannot apply event to closed account")
      
      case (Some(s), MoneyDeposited(amount, _)) =>
        s.balance + amount match {
          case Right(newBalance) => Right(s.copy(balance = newBalance, transactions = s.transactions :+ event))
          case Left(err) => Left(err)
        }
      
      case (Some(s), MoneyWithdrawn(amount, _)) =>
        if (s.balance.amount < amount.amount) Left("Insufficient funds")
        else s.balance + Money.of(-amount.amount, amount.currency) match {
          case Right(newBalance) => Right(s.copy(balance = newBalance, transactions = s.transactions :+ event))
          case Left(err) => Left(err)
        }
      
      case (Some(s), AccountClosed(reason)) =>
        Right(s.copy(isOpen = false, transactions = s.transactions :+ event))
      
      case (Some(s), InterestApplied(rate, amount)) =>
        s.balance + amount match {
          case Right(newBalance) => Right(s.copy(balance = newBalance, transactions = s.transactions :+ event))
          case Left(err) => Left(err)
        }
      
      case _ => Left("Invalid event sequence")
    }
  
  def replayEvents(events: List[AccountEvent]): Either[String, AccountState] = {
    events.foldLeft[Either[String, Option[AccountState]]](Right(None)) { (stateEither, event) =>
      stateEither.flatMap(state => applyEvent(state, event).map(Some(_)))
    }.flatMap(_.toRight("No events to replay"))
  }
  
  println("=== Domain Modeling with ADTs ===")
  
  // Validate email
  println("--- Email Validation ---")
  List("alice@example.com", "not-an-email", "bob@test.co.th").foreach { email =>
    EmailAddress(email) match {
      case Right(e) => println(s"  Valid: ${e.value}")
      case Left(e)  => println(s"  Invalid: $e")
    }
  }
  
  // Event sourcing
  println("\n--- Event Sourcing ---")
  val events = List(
    AccountOpened("ACC-001", "Alice", Money.thb(1000.0)),
    MoneyDeposited(Money.thb(5000.0), "Salary"),
    MoneyWithdrawn(Money.thb(1500.0), "Rent"),
    InterestApplied(BigDecimal("0.01"), Money.thb(45.0)),
    MoneyWithdrawn(Money.thb(200.0), "Food")
  )
  
  replayEvents(events) match {
    case Right(state) =>
      println(s"  Account: ${state.id}")
      println(s"  Owner: ${state.owner}")
      println(s"  Balance: ${state.balance.amount} ${state.balance.currency}")
      println(s"  Transactions: ${state.transactions.size}")
      println(s"  Open: ${state.isOpen}")
    case Left(err) =>
      println(s"  Error: $err")
  }
  
  // Test insufficient funds
  println("\n--- Insufficient Funds ---")
  val overdraftAttempt = events :+ MoneyWithdrawn(Money.thb(10000.0), "Attempt overdraft")
  replayEvents(overdraftAttempt) match {
    case Right(_)  => println("  Should not succeed")
    case Left(err) => println(s"  Correctly rejected: $err")
  }
}
```

---

## สรุป Part 28

| Concept | ความหมาย | ตัวอย่าง |
|---------|---------|---------|
| Sum type | A OR B OR C | `sealed trait` + case classes |
| Product type | A AND B AND C | `case class(a: A, b: B)` |
| Unit type | singleton | `case object` |
| Empty type | impossible | `Nothing` |
| ADT | Sum + Product | Domain models, ASTs |
| Sealed | exhaustive matching | `sealed trait` ใน file เดียว |
| GADTs | generalized ADTs | Type-indexed ADTs |
| Event sourcing | replay events | State = fold(events) |

---

## แบบฝึกหัด Part 28

**ข้อ 1:** สร้าง `Result[+A, +E]` ADT ที่แตกต่างจาก Either — มี `Pending`, `Success`, `Failure`, `Cancelled`

**ข้อ 2:** Implement JSON ADT พร้อม parser ที่แปลง String → JsonValue

**ข้อ 3:** สร้าง `Workflow[S, A]` ADT สำหรับ multi-step business process ที่ track state transitions

**ข้อ 4:** Implement immutable `FileSystem` ADT: `File`, `Directory`, `SymLink` พร้อม operations

**ข้อ 5:** สร้าง DSL ด้วย ADT สำหรับ query builder: `SELECT cols FROM table WHERE conditions ORDER BY`

---

➡️ ต่อไป: [Part 29 — Error Handling](part-29-error-handling.md)
