# Part 17: Immutability and Pure Functions

## Steps 161-170: Immutability Benefits, Pure Functions, Referential Transparency, Side Effects Management

---

## Step 161: Immutability — ทำไมถึงสำคัญ

```scala
// Immutability prevents bugs and makes code predictable
object ImmutabilityBenefits extends App {
  
  // Mutable - ปัญหาที่พบบ่อย
  class MutableShoppingCart {
    var items: scala.collection.mutable.ListBuffer[String] = 
      scala.collection.mutable.ListBuffer.empty
    var total: Double = 0.0
    
    def add(item: String, price: Double): Unit = {
      items += item
      total += price
    }
    
    // Bug potential: shared mutable state
  }
  
  // Immutable - ปลอดภัยกว่า
  case class ImmutableCart(items: List[String] = List.empty, total: Double = 0.0) {
    def add(item: String, price: Double): ImmutableCart = 
      copy(items = items :+ item, total = total + price)
    
    def remove(item: String): ImmutableCart = {
      val idx = items.indexOf(item)
      if (idx < 0) this
      else copy(items = items.patch(idx, Nil, 1))
    }
    
    def applyDiscount(pct: Double): ImmutableCart = 
      copy(total = total * (1 - pct))
    
    override def toString: String = 
      s"Cart(${items.mkString(", ")}, total=${total})"
  }
  
  println("=== Mutable vs Immutable Cart ===")
  
  // Immutable - can't accidentally share state
  val emptyCart = ImmutableCart()
  val cart1 = emptyCart.add("Laptop", 35000.0)
  val cart2 = cart1.add("Mouse", 850.0)
  val discounted = cart2.applyDiscount(0.10)
  
  // Original carts unchanged
  println(s"empty: $emptyCart")
  println(s"cart1: $cart1")
  println(s"cart2: $cart2")
  println(s"discounted: $discounted")
  
  // Thread safety: immutable objects can be shared safely
  val sharedCart = cart2
  // Multiple threads can read sharedCart without synchronization
  
  // Immutable collections
  val list1 = List(1, 2, 3)
  val list2 = 0 :: list1  // New list, list1 unchanged
  val list3 = list1 :+ 4   // New list, list1 unchanged
  
  println(s"\nlist1: $list1")
  println(s"list2: $list2")
  println(s"list3: $list3")
  
  // Structural sharing - efficient memory usage
  // list2 shares list1's structure: [0] -> list1
  // Not a full copy!
  
  // Map immutability
  val map1 = Map("a" -> 1, "b" -> 2)
  val map2 = map1 + ("c" -> 3)  // New map
  val map3 = map1 - "a"          // New map, without "a"
  
  println(s"\nmap1: $map1")
  println(s"map2: $map2")
  println(s"map3: $map3")
}
```

---

## Step 162: Pure Functions

```scala
// Pure functions: same input always gives same output, no side effects
object PureFunctions extends App {
  
  // IMPURE functions - avoid these
  var globalState = 0
  
  def impureAdd(x: Int): Int = {
    globalState += 1  // Side effect: modifies global state
    x + globalState   // Result depends on external state
  }
  
  // PURE functions - prefer these
  def pureAdd(x: Int, y: Int): Int = x + y  // Always same output for same input
  
  def pureFormatName(first: String, last: String): String = s"$last, $first"
  
  def pureCalculateTax(amount: Double, rate: Double): Double = amount * rate
  
  println("=== Pure vs Impure ===")
  println(s"impureAdd(5) = ${impureAdd(5)}")  // 6 (globalState = 1)
  println(s"impureAdd(5) = ${impureAdd(5)}")  // 7 (globalState = 2, different result!)
  
  println(s"\npureAdd(3, 4) = ${pureAdd(3, 4)}")  // Always 7
  println(s"pureAdd(3, 4) = ${pureAdd(3, 4)}")  // Always 7
  
  // Pure functions with complex data
  case class BankAccount(id: String, balance: Double, transactions: List[Double])
  
  // PURE: returns new state
  def deposit(account: BankAccount, amount: Double): BankAccount = {
    require(amount > 0, "Deposit amount must be positive")
    account.copy(
      balance = account.balance + amount,
      transactions = account.transactions :+ amount
    )
  }
  
  def withdraw(account: BankAccount, amount: Double): Either[String, BankAccount] = {
    if (amount <= 0) Left("Withdrawal amount must be positive")
    else if (amount > account.balance) Left(s"Insufficient funds: balance=${account.balance}, requested=$amount")
    else Right(account.copy(
      balance = account.balance - amount,
      transactions = account.transactions :+ -amount
    ))
  }
  
  def transfer(from: BankAccount, to: BankAccount, amount: Double): Either[String, (BankAccount, BankAccount)] = {
    withdraw(from, amount).map(newFrom => (newFrom, deposit(to, amount)))
  }
  
  val alice = BankAccount("ACC001", 10000.0, List.empty)
  val bob = BankAccount("ACC002", 5000.0, List.empty)
  
  println("\n=== Pure Banking Operations ===")
  val aliceAfterDeposit = deposit(alice, 5000.0)
  println(s"Alice after deposit: balance=${aliceAfterDeposit.balance}")
  
  withdraw(alice, 3000.0) match {
    case Right(acc) => println(s"Withdraw OK: balance=${acc.balance}")
    case Left(err)  => println(s"Withdraw failed: $err")
  }
  
  withdraw(alice, 50000.0) match {
    case Right(acc) => println(s"Withdraw OK: balance=${acc.balance}")
    case Left(err)  => println(s"Withdraw failed: $err")
  }
  
  transfer(alice, bob, 2000.0) match {
    case Right((newAlice, newBob)) => 
      println(s"Transfer OK: alice=${newAlice.balance}, bob=${newBob.balance}")
    case Left(err) => 
      println(s"Transfer failed: $err")
  }
}
```

---

## Step 163: Referential Transparency

```scala
// Referential transparency: expression can be replaced by its value
object ReferentialTransparency extends App {
  
  // Referentially transparent
  val x = 2 + 3  // Can replace any occurrence of (2+3) with 5
  
  def square(n: Int): Int = n * n
  // square(5) can always be replaced by 25
  
  // NOT referentially transparent
  import java.time.LocalDateTime
  def now(): LocalDateTime = LocalDateTime.now()
  // now() != now() - each call gives different result
  
  // Example: making time-dependent code testable
  // BAD: hardcoded time
  def isExpiredBad(expiryDate: java.time.LocalDate): Boolean = 
    expiryDate.isBefore(java.time.LocalDate.now())  // Depends on external state
  
  // GOOD: inject time as parameter
  def isExpiredGood(expiryDate: java.time.LocalDate, today: java.time.LocalDate): Boolean = 
    expiryDate.isBefore(today)  // Pure! Testable!
  
  val expiryDate = java.time.LocalDate.of(2024, 6, 1)
  val today2025 = java.time.LocalDate.of(2025, 1, 1)
  val today2023 = java.time.LocalDate.of(2023, 1, 1)
  
  println("=== Referential Transparency ===")
  println(s"isExpired (from 2025 view): ${isExpiredGood(expiryDate, today2025)}")  // true
  println(s"isExpired (from 2023 view): ${isExpiredGood(expiryDate, today2023)}")  // false
  
  // Pattern: Clock as dependency
  trait Clock {
    def now(): java.time.Instant
  }
  
  object SystemClock extends Clock {
    override def now(): java.time.Instant = java.time.Instant.now()
  }
  
  class FixedClock(fixedTime: java.time.Instant) extends Clock {
    override def now(): java.time.Instant = fixedTime
  }
  
  class SessionManager(clock: Clock) {
    private val sessions = scala.collection.mutable.Map[String, java.time.Instant]()
    private val SESSION_TTL = 3600L  // 1 hour in seconds
    
    def createSession(userId: String): String = {
      val sessionId = s"sess-${userId}-${System.nanoTime()}"
      sessions(sessionId) = clock.now()
      sessionId
    }
    
    def isValid(sessionId: String): Boolean = {
      sessions.get(sessionId).exists { createdAt =>
        val elapsed = clock.now().getEpochSecond - createdAt.getEpochSecond
        elapsed < SESSION_TTL
      }
    }
  }
  
  println("\n=== Clock Abstraction ===")
  
  // Test with fixed time
  val fixedTime = java.time.Instant.parse("2024-01-01T10:00:00Z")
  val testClock = new FixedClock(fixedTime)
  val sessionManager = new SessionManager(testClock)
  
  val sessionId = sessionManager.createSession("user001")
  println(s"Session valid: ${sessionManager.isValid(sessionId)}")
  
  // Simulate time passing
  val laterClock = new FixedClock(fixedTime.plusSeconds(7200))  // 2 hours later
  val laterManager = new SessionManager(laterClock)
  // Note: in real test, we'd inject the same session store
  println(s"With fixed clock, session is testable!")
}
```

---

## Step 164: Managing Side Effects

```scala
// Side effect management patterns
object SideEffectManagement extends App {
  
  // Side effects are unavoidable (I/O, DB, network)
  // But we can isolate and manage them
  
  // Pattern 1: IO monad-like wrapper (simplified)
  class IO[+A](private val effect: () => A) {
    def run(): A = effect()
    
    def map[B](f: A => B): IO[B] = 
      new IO(() => f(this.run()))
    
    def flatMap[B](f: A => IO[B]): IO[B] = 
      new IO(() => f(this.run()).run())
    
    def andThen[B](next: IO[B]): IO[B] = 
      flatMap(_ => next)
  }
  
  object IO {
    def pure[A](value: A): IO[A] = new IO(() => value)
    def effect[A](f: => A): IO[A] = new IO(() => f)
    def println(msg: String): IO[Unit] = effect(Predef.println(msg))
    def readLine(): IO[String] = effect(scala.io.StdIn.readLine())
  }
  
  // Pure computation
  def greetUser(name: String): String = s"Hello, $name! Welcome to Scala."
  def validateName(name: String): Either[String, String] = {
    if (name.isEmpty) Left("Name cannot be empty")
    else if (name.length > 50) Left("Name too long")
    else Right(name.trim.capitalize)
  }
  
  // Side effects isolated in IO
  val program: IO[Unit] = for {
    _ <- IO.println("=== IO Effect Management ===")
    _ <- IO.println("Computing greeting...")
    result = validateName("  alice  ").map(greetUser)
    _ <- result match {
      case Right(msg) => IO.println(s"Result: $msg")
      case Left(err)  => IO.println(s"Error: $err")
    }
  } yield ()
  
  // All side effects happen here, at the "end of the world"
  program.run()
  
  // Pattern 2: Effect description (deferred execution)
  sealed trait AppEffect[+A]
  case class Log(message: String) extends AppEffect[Unit]
  case class ReadConfig(key: String) extends AppEffect[String]
  case class WriteData(key: String, value: String) extends AppEffect[Unit]
  case class Pure[A](value: A) extends AppEffect[A]
  
  // Build effect descriptions (pure!)
  def buildUserProcess(userId: String): List[AppEffect[_]] = List(
    Log(s"Processing user: $userId"),
    ReadConfig("user.validation.enabled"),
    Log(s"Validating user $userId"),
    WriteData(s"user:$userId:status", "active"),
    Log(s"User $userId processed successfully")
  )
  
  // Execute effects (side effects happen here)
  def runEffects(effects: List[AppEffect[_]]): Unit = {
    val configStore = Map("user.validation.enabled" -> "true")
    
    effects.foreach {
      case Log(msg) => println(s"  [LOG] $msg")
      case ReadConfig(key) => 
        println(s"  [CONFIG] Read $key = ${configStore.getOrElse(key, "")}")
      case WriteData(key, value) =>
        println(s"  [WRITE] $key = $value")
      case Pure(value) =>
        println(s"  [PURE] value = $value")
    }
  }
  
  println("\n=== Effect Description Pattern ===")
  val effects = buildUserProcess("U001")
  runEffects(effects)
  
  // Pattern 3: Pushing effects to the boundary
  // PURE CORE with IMPURE SHELL
  
  // Pure domain logic
  case class OrderState(items: List[String], status: String)
  
  def addItemToOrder(order: OrderState, item: String): OrderState = 
    order.copy(items = order.items :+ item)
  
  def submitOrder(order: OrderState): Either[String, OrderState] = 
    if (order.items.isEmpty) Left("Cannot submit empty order")
    else Right(order.copy(status = "submitted"))
  
  def cancelOrder(order: OrderState): Either[String, OrderState] = 
    if (order.status == "submitted") Right(order.copy(status = "cancelled"))
    else Left(s"Cannot cancel order in ${order.status} status")
  
  // Impure shell (all I/O here)
  def runOrderWorkflow(): Unit = {
    println("\n=== Pure Core / Impure Shell ===")
    
    // All pure logic
    val initialOrder = OrderState(List.empty, "new")
    val withLaptop = addItemToOrder(initialOrder, "Laptop")
    val withMouse = addItemToOrder(withLaptop, "Mouse")
    
    val result = for {
      submitted <- submitOrder(withMouse)
      _ = println(s"[I/O] Saving order to DB: ${submitted.status}")
      _ = println(s"[I/O] Sending email notification")
    } yield submitted
    
    result match {
      case Right(order) => println(s"Success: $order")
      case Left(err)    => println(s"Failed: $err")
    }
  }
  
  runOrderWorkflow()
}
```

---

## Step 165: Functional Data Structures

```scala
// Immutable data structures for functional programming
object FunctionalDataStructures extends App {
  
  // Persistent (immutable) linked list
  sealed trait FList[+A] {
    def isEmpty: Boolean
    def head: A
    def tail: FList[A]
    
    def prepend[B >: A](elem: B): FList[B] = FCons(elem, this)
    
    def length: Int = this match {
      case FNil        => 0
      case FCons(_, t) => 1 + t.length
    }
    
    def map[B](f: A => B): FList[B] = this match {
      case FNil        => FNil
      case FCons(h, t) => FCons(f(h), t.map(f))
    }
    
    def filter(pred: A => Boolean): FList[A] = this match {
      case FNil        => FNil
      case FCons(h, t) => 
        if (pred(h)) FCons(h, t.filter(pred))
        else t.filter(pred)
    }
    
    def foldLeft[B](init: B)(f: (B, A) => B): B = this match {
      case FNil        => init
      case FCons(h, t) => t.foldLeft(f(init, h))(f)
    }
    
    def toList: List[A] = foldLeft(List.empty[A])((acc, a) => a :: acc).reverse
    
    override def toString: String = s"FList(${toList.mkString(", ")})"
  }
  
  case object FNil extends FList[Nothing] {
    override def isEmpty = true
    override def head = throw new NoSuchElementException("head of empty FList")
    override def tail = throw new NoSuchElementException("tail of empty FList")
  }
  
  case class FCons[+A](override val head: A, override val tail: FList[A]) extends FList[A] {
    override def isEmpty = false
  }
  
  object FList {
    def apply[A](elems: A*): FList[A] = 
      elems.foldRight(FNil: FList[A])(FCons(_, _))
    
    def empty[A]: FList[A] = FNil
  }
  
  println("=== Functional List ===")
  val list1 = FList(1, 2, 3, 4, 5)
  val list2 = list1.prepend(0)
  
  println(s"list1: $list1")
  println(s"list2 (prepend 0): $list2")
  println(s"list1 after prepend: $list1")  // Unchanged!
  
  val doubled = list1.map(_ * 2)
  val evens = list1.filter(_ % 2 == 0)
  val sum = list1.foldLeft(0)(_ + _)
  
  println(s"doubled: $doubled")
  println(s"evens: $evens")
  println(s"sum: $sum")
  
  // Immutable Stack
  case class ImmutableStack[+A](elements: List[A] = List.empty) {
    def push[B >: A](elem: B): ImmutableStack[B] = 
      ImmutableStack(elem :: elements)
    
    def pop: (Option[A], ImmutableStack[A]) = elements match {
      case Nil     => (None, this)
      case h :: t  => (Some(h), ImmutableStack(t))
    }
    
    def peek: Option[A] = elements.headOption
    def isEmpty: Boolean = elements.isEmpty
    def size: Int = elements.size
    
    override def toString: String = s"Stack(${elements.mkString(", ")})"
  }
  
  println("\n=== Immutable Stack ===")
  val stack0 = ImmutableStack[Int]()
  val stack1 = stack0.push(1)
  val stack2 = stack1.push(2)
  val stack3 = stack2.push(3)
  
  println(s"stack3: $stack3")
  val (top, stack2Again) = stack3.pop
  println(s"popped: $top, remaining: $stack2Again")
  println(s"stack3 unchanged: $stack3")
  
  // Immutable Queue using two stacks
  case class ImmutableQueue[+A] private(inbox: List[A], outbox: List[A]) {
    def enqueue[B >: A](elem: B): ImmutableQueue[B] = 
      ImmutableQueue(elem :: inbox, outbox)
    
    def dequeue: (Option[A], ImmutableQueue[A]) = outbox match {
      case h :: t => (Some(h), ImmutableQueue(inbox, t))
      case Nil    => inbox.reverse match {
        case h :: t => (Some(h), ImmutableQueue(List.empty, t))
        case Nil    => (None, this)
      }
    }
    
    def size: Int = inbox.size + outbox.size
    def isEmpty: Boolean = inbox.isEmpty && outbox.isEmpty
    
    override def toString: String = 
      s"Queue(${(outbox ++ inbox.reverse).mkString(", ")})"
  }
  
  object ImmutableQueue {
    def empty[A]: ImmutableQueue[A] = ImmutableQueue(List.empty, List.empty)
    def apply[A](elems: A*): ImmutableQueue[A] = 
      elems.foldLeft(empty[A])(_ enqueue _)
  }
  
  println("\n=== Immutable Queue ===")
  val q = ImmutableQueue(1, 2, 3)
  val q2 = q.enqueue(4).enqueue(5)
  println(s"q: $q")
  println(s"q2: $q2")
  
  val (first, rest) = q2.dequeue
  println(s"dequeued: $first, rest: $rest")
}
```

---

## Step 166-170: Value Objects, Lens Pattern, Builder-style Updates

```scala
// Comprehensive immutability in real systems
object ImmutableDesignPatterns extends App {
  
  // Value Object pattern - immutable with meaningful equality
  final case class Money(amount: BigDecimal, currency: String) {
    def +(other: Money): Either[String, Money] = 
      if (currency != other.currency) Left(s"Currency mismatch: $currency != ${other.currency}")
      else Right(Money(amount + other.amount, currency))
    
    def *(factor: BigDecimal): Money = 
      Money((amount * factor).setScale(2, BigDecimal.RoundingMode.HALF_UP), currency)
    
    def >(other: Money): Boolean = {
      require(currency == other.currency)
      amount > other.amount
    }
    
    override def toString: String = f"${amount.toDouble}%.2f $currency"
  }
  
  object Money {
    def thb(amount: Double): Money = 
      Money(BigDecimal(amount).setScale(2, BigDecimal.RoundingMode.HALF_UP), "THB")
    def usd(amount: Double): Money = 
      Money(BigDecimal(amount).setScale(2, BigDecimal.RoundingMode.HALF_UP), "USD")
  }
  
  // Lens pattern - focus on nested immutable structure
  // Without lens (verbose copy chains)
  case class Address(street: String, city: String, country: String, zipCode: String)
  case class ContactInfo(email: String, phone: String, address: Address)
  case class Person(name: String, age: Int, contact: ContactInfo)
  
  val person = Person(
    "Alice Chen",
    30,
    ContactInfo(
      "alice@example.com",
      "+66812345678",
      Address("123 Main St", "Bangkok", "Thailand", "10110")
    )
  )
  
  // Update nested field - verbose but correct
  def updateCity(p: Person, newCity: String): Person = 
    p.copy(contact = p.contact.copy(
      address = p.contact.address.copy(city = newCity)
    ))
  
  // Simple lens implementation
  case class Lens[S, A](get: S => A, set: (S, A) => S) {
    def modify(f: A => A): S => S = s => set(s, f(get(s)))
    def composeLens[B](other: Lens[A, B]): Lens[S, B] = Lens(
      get = s => other.get(get(s)),
      set = (s, b) => set(s, other.set(get(s), b))
    )
  }
  
  // Define lenses
  val personContact = Lens[Person, ContactInfo](_.contact, (p, c) => p.copy(contact = c))
  val contactAddress = Lens[ContactInfo, Address](_.address, (c, a) => c.copy(address = a))
  val addressCity = Lens[Address, String](_.city, (a, c) => a.copy(city = c))
  
  // Compose lenses
  val personCity = personContact composeLens contactAddress composeLens addressCity
  
  println("=== Lens Pattern ===")
  println(s"Original city: ${personCity.get(person)}")
  val movedPerson = personCity.set(person, "Chiang Mai")
  println(s"New city: ${personCity.get(movedPerson)}")
  println(s"Original unchanged: ${personCity.get(person)}")
  
  // Modify pattern
  val upperCityPerson = personCity.modify(_.toUpperCase)(person)
  println(s"Upper city: ${personCity.get(upperCityPerson)}")
  
  // Immutable configuration with defaults
  case class ServiceConfig private(
    host: String,
    port: Int,
    maxConnections: Int,
    timeout: Int,
    debug: Boolean
  )
  
  object ServiceConfig {
    def default: ServiceConfig = ServiceConfig(
      host = "localhost",
      port = 8080,
      maxConnections = 100,
      timeout = 30000,
      debug = false
    )
    
    // Builder-style updates preserving immutability
    implicit class ConfigOps(config: ServiceConfig) {
      def withHost(h: String): ServiceConfig = config.copy(host = h)
      def withPort(p: Int): ServiceConfig = config.copy(port = p)
      def withMaxConnections(n: Int): ServiceConfig = config.copy(maxConnections = n)
      def withTimeout(ms: Int): ServiceConfig = config.copy(timeout = ms)
      def withDebug(d: Boolean): ServiceConfig = config.copy(debug = d)
    }
  }
  
  import ServiceConfig._
  
  val prodConfig = ServiceConfig.default
    .withHost("api.production.com")
    .withPort(443)
    .withMaxConnections(500)
    .withDebug(false)
  
  val devConfig = ServiceConfig.default
    .withDebug(true)
    .withTimeout(60000)
  
  println(s"\n=== Immutable Config ===")
  println(s"Production: $prodConfig")
  println(s"Development: $devConfig")
  println(s"Default unchanged: ${ServiceConfig.default}")
  
  // Event Sourcing - immutable audit trail
  sealed trait AccountEvent
  case class AccountOpened(accountId: String, ownerId: String, initialBalance: Double) extends AccountEvent
  case class MoneyDeposited(amount: Double) extends AccountEvent
  case class MoneyWithdrawn(amount: Double) extends AccountEvent
  case class AccountClosed(reason: String) extends AccountEvent
  
  case class AccountSnapshot(
    accountId: String,
    ownerId: String,
    balance: Double,
    isOpen: Boolean,
    history: List[AccountEvent]
  )
  
  def applyEvent(snapshot: AccountSnapshot, event: AccountEvent): AccountSnapshot = event match {
    case AccountOpened(id, owner, balance) => snapshot.copy(
      accountId = id, ownerId = owner, balance = balance, isOpen = true,
      history = snapshot.history :+ event
    )
    case MoneyDeposited(amount) => snapshot.copy(
      balance = snapshot.balance + amount,
      history = snapshot.history :+ event
    )
    case MoneyWithdrawn(amount) => snapshot.copy(
      balance = snapshot.balance - amount,
      history = snapshot.history :+ event
    )
    case AccountClosed(_) => snapshot.copy(
      isOpen = false,
      history = snapshot.history :+ event
    )
  }
  
  def replayEvents(events: List[AccountEvent]): AccountSnapshot = 
    events.foldLeft(AccountSnapshot("", "", 0.0, false, List.empty))(applyEvent)
  
  val events = List(
    AccountOpened("ACC001", "Alice", 1000.0),
    MoneyDeposited(5000.0),
    MoneyWithdrawn(2000.0),
    MoneyDeposited(10000.0),
    MoneyWithdrawn(500.0)
  )
  
  println("\n=== Event Sourcing ===")
  val finalState = replayEvents(events)
  println(s"Account: ${finalState.accountId}")
  println(f"Balance: ${finalState.balance}%.2f THB")
  println(s"Open: ${finalState.isOpen}")
  println(s"Event count: ${finalState.history.size}")
  
  // Can replay to any point in time
  val afterFirstDeposit = replayEvents(events.take(2))
  println(f"Balance after 2 events: ${afterFirstDeposit.balance}%.2f THB")
}
```

---

## สรุป Part 17

| แนวคิด | Scala | Production Benefit |
|--------|-------|-------------------|
| Immutable values | `val`, `case class` | Thread-safe, predictable |
| Pure functions | no side effects | Testable, composable |
| Referential transparency | replace expr with value | Cacheable, memoizable |
| Structural sharing | List prepend | Efficient memory |
| Lens pattern | focus/set nested | Clean nested updates |
| Event sourcing | immutable event log | Full audit trail |
| IO isolation | IO wrapper | Controlled effects |
| Value objects | `final case class` | Domain integrity |

---

## แบบฝึกหัด Part 17

**ข้อ 1:** Implement `PersistentMap[K, V]` ที่ immutable พร้อม operations: `get`, `put`, `remove`, `merge` โดยใช้ structural sharing

**ข้อ 2:** สร้าง pure function pipeline สำหรับ user registration: validate → normalize → checkDuplicate → createUser — แต่ละ step ต้องเป็น pure function

**ข้อ 3:** Implement simple Lens library ที่ support `get`, `set`, `modify`, `compose`, `andThen` พร้อม macros-free syntax

**ข้อ 4:** สร้าง shopping cart ด้วย event sourcing: ทุก action (AddItem, RemoveItem, ApplyDiscount, Checkout) เป็น event ที่ append-only

**ข้อ 5:** Implement `IO` monad ง่ายๆ ที่มี `map`, `flatMap`, `for-comprehension` support และ `unsafeRun` ที่ execute side effects

---

➡️ ต่อไป: [Part 18 — Recursion and Tail Recursion](part-18-recursion-and-tail-recursion.md)
