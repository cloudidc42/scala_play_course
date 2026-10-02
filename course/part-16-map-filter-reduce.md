# Part 16: Map, FlatMap, Filter, Fold, Reduce, Scan, Collect

## Steps 151-160: Practical Collection Operations

---

## Step 151: map — แปลงข้อมูลทุก Element

```scala
object MapDemo extends App {
  
  // Basic map
  val numbers = List(1, 2, 3, 4, 5)
  val doubled = numbers.map(_ * 2)
  val squared = numbers.map(n => n * n)
  val strings = numbers.map(n => s"Item $n")
  
  println("=== map basics ===")
  println(s"doubled: $doubled")
  println(s"squared: $squared")
  println(s"strings: $strings")
  
  // map on different types
  val words = List("hello", "world", "scala", "programming")
  val lengths = words.map(_.length)
  val upper = words.map(_.toUpperCase)
  val firstChars = words.map(_.head)
  
  println(s"\nlengths: $lengths")
  println(s"upper: $upper")
  println(s"firstChars: $firstChars")
  
  // map on Option
  val some: Option[Int] = Some(42)
  val none: Option[Int] = None
  
  println(s"\nsome.map(_*2) = ${some.map(_ * 2)}")
  println(s"none.map(_*2) = ${none.map(_ * 2)}")
  
  // map on Either
  val right: Either[String, Int] = Right(10)
  val left: Either[String, Int] = Left("error")
  
  println(s"right.map(_+1) = ${right.map(_ + 1)}")
  println(s"left.map(_+1) = ${left.map(_ + 1)}")
  
  // Transforming complex objects
  case class Product(id: String, name: String, price: Double, inStock: Boolean)
  case class ProductDTO(id: String, name: String, priceFormatted: String)
  
  val products = List(
    Product("P001", "Laptop", 45000.0, true),
    Product("P002", "Mouse", 890.0, false),
    Product("P003", "Keyboard", 1500.0, true)
  )
  
  val dtos = products.map(p => ProductDTO(
    id = p.id,
    name = p.name,
    priceFormatted = f"${p.price}%.2f THB"
  ))
  
  println("\n=== DTO transformation ===")
  dtos.foreach(println)
  
  // map on Map (collection)
  val prices: Map[String, Double] = Map("A" -> 100.0, "B" -> 200.0, "C" -> 150.0)
  val discountedPrices = prices.map { case (k, v) => k -> v * 0.9 }
  val priceStrings = prices.map { case (k, v) => k -> f"$v%.2f THB" }
  
  println(s"\ndiscounted: $discountedPrices")
  println(s"formatted: $priceStrings")
}
```

---

## Step 152: flatMap — map แล้ว flatten

```scala
object FlatMapDemo extends App {
  
  // flatMap = map + flatten
  val numbers = List(1, 2, 3, 4, 5)
  
  // map creates nested lists
  val nested = numbers.map(n => List(n, n * 10))
  println(s"nested: $nested")  // List(List(1,10), List(2,20), ...)
  
  // flatMap flattens the result
  val flat = numbers.flatMap(n => List(n, n * 10))
  println(s"flat: $flat")  // List(1, 10, 2, 20, ...)
  
  // equivalent to
  val equivalent = numbers.map(n => List(n, n * 10)).flatten
  println(s"equivalent: $equivalent")
  
  // Real use: Optional chaining
  case class Address(city: String, country: String)
  case class User(name: String, address: Option[Address])
  case class Company(name: String, primaryContact: Option[User])
  
  val company = Company("TechCorp", Some(User("Alice", Some(Address("Bangkok", "Thailand")))))
  val noContact = Company("EmptyCorp", None)
  val noAddress = Company("AddresslessCorp", Some(User("Bob", None)))
  
  // flatMap chains Option operations
  def getCity(company: Company): Option[String] = {
    company.primaryContact.flatMap(user => user.address.map(_.city))
  }
  
  println(s"\nCompany city: ${getCity(company)}")
  println(s"No contact city: ${getCity(noContact)}")
  println(s"No address city: ${getCity(noAddress)}")
  
  // flatMap with Either - short-circuit on Left
  def parseInt(s: String): Either[String, Int] = 
    scala.util.Try(s.toInt).toEither.left.map(_.getMessage)
  
  def divide(a: Int, b: Int): Either[String, Double] = 
    if (b == 0) Left("Division by zero") else Right(a.toDouble / b)
  
  def compute(s1: String, s2: String): Either[String, Double] = {
    parseInt(s1).flatMap(a => parseInt(s2).flatMap(b => divide(a, b)))
  }
  
  println(s"\ncompute('10','2') = ${compute("10", "2")}")
  println(s"compute('10','0') = ${compute("10", "0")}")
  println(s"compute('abc','2') = ${compute("abc", "2")}")
  
  // flatMap for cross-product
  val colors = List("red", "blue", "green")
  val sizes = List("S", "M", "L", "XL")
  
  val combinations = colors.flatMap(c => sizes.map(s => s"$c-$s"))
  println(s"\nColor-Size combinations: ${combinations.size}")
  combinations.foreach(c => print(s"$c "))
  println()
  
  // flatMap in for-comprehension (equivalent)
  val combForComp = for {
    color <- colors
    size  <- sizes
  } yield s"$color-$size"
  
  println(s"Via for-comp: $combForComp == combinations: ${combForComp == combinations}")
}
```

---

## Step 153: filter, filterNot, partition, span

```scala
object FilterDemo extends App {
  
  val numbers = List(-5, -3, -1, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
  
  // Basic filter
  val positives = numbers.filter(_ > 0)
  val evens = numbers.filter(_ % 2 == 0)
  val negatives = numbers.filterNot(_ >= 0)
  
  println("=== filter ===")
  println(s"positives: $positives")
  println(s"evens: $evens")
  println(s"negatives: $negatives")
  
  // partition - split into two lists
  val (pos, nonPos) = numbers.partition(_ > 0)
  val (evenNums, oddNums) = numbers.filter(_ > 0).partition(_ % 2 == 0)
  
  println(s"\npartition > 0: pos=$pos, non-pos=$nonPos")
  println(s"even/odd: even=$evenNums, odd=$oddNums")
  
  // span - take while predicate, then rest
  val (initial, rest) = numbers.span(_ < 5)
  println(s"\nspan < 5: initial=$initial, rest=$rest")
  
  // takeWhile, dropWhile
  val ascending = List(1, 2, 3, 4, 5, 4, 3, 2, 1)
  val prefix = ascending.takeWhile(_ < 4)
  val suffix = ascending.dropWhile(_ < 4)
  println(s"\ntakeWhile < 4: $prefix")
  println(s"dropWhile < 4: $suffix")
  
  // Complex filtering
  case class Order(
    id: String,
    customerId: String,
    amount: Double,
    status: String,
    items: List[String]
  )
  
  val orders = List(
    Order("O001", "C001", 1500.0, "pending", List("Laptop", "Mouse")),
    Order("O002", "C002", 500.0, "completed", List("Book")),
    Order("O003", "C001", 3000.0, "cancelled", List("Phone")),
    Order("O004", "C003", 2200.0, "pending", List("Monitor", "Keyboard")),
    Order("O005", "C002", 750.0, "completed", List("Headphones")),
    Order("O006", "C001", 150.0, "pending", List("USB Hub"))
  )
  
  // Multi-condition filtering
  val highValuePending = orders.filter(o => o.status == "pending" && o.amount > 1000)
  val customerC001 = orders.filter(_.customerId == "C001")
  val multiItemOrders = orders.filter(_.items.size > 1)
  
  println("\n=== Complex Filtering ===")
  println(s"High-value pending: ${highValuePending.map(_.id)}")
  println(s"C001 orders: ${customerC001.map(o => s"${o.id}(${o.status})")}")
  println(s"Multi-item: ${multiItemOrders.map(_.id)}")
  
  // groupBy + filter
  val ordersByStatus = orders.groupBy(_.status)
  println("\n=== Orders by Status ===")
  ordersByStatus.foreach { case (status, ords) =>
    val total = ords.map(_.amount).sum
    println(f"  $status: ${ords.size} orders, total=$total%.2f THB")
  }
  
  // Filter with complex predicates
  def ordersForCustomerWithMinAmount(
    customerId: String,
    minAmount: Double
  ): List[Order] => List[Order] = 
    _.filter(o => o.customerId == customerId && o.amount >= minAmount)
  
  val c001HighOrders = ordersForCustomerWithMinAmount("C001", 1000.0)(orders)
  println(s"\nC001 orders >= 1000: ${c001HighOrders.map(o => f"${o.id}(${o.amount}%.0f)")}")
}
```

---

## Step 154: fold, foldLeft, foldRight

```scala
object FoldDemo extends App {
  
  // foldLeft: accumulates from left to right
  // foldLeft(initial)((acc, elem) => newAcc)
  val numbers = List(1, 2, 3, 4, 5)
  
  val sum = numbers.foldLeft(0)(_ + _)
  val product = numbers.foldLeft(1)(_ * _)
  val sumSquares = numbers.foldLeft(0)((acc, n) => acc + n * n)
  
  println("=== foldLeft ===")
  println(s"sum: $sum")
  println(s"product: $product")
  println(s"sumSquares: $sumSquares")
  
  // Building complex structures with fold
  val words = List("Hello", "World", "From", "Scala")
  
  // Build a string
  val sentence = words.foldLeft("")((acc, word) => 
    if (acc.isEmpty) word else acc + " " + word
  )
  println(s"sentence: $sentence")
  
  // Build a map from list
  case class Product(id: String, name: String, price: Double)
  val products = List(
    Product("P001", "Laptop", 45000.0),
    Product("P002", "Mouse", 890.0),
    Product("P003", "Keyboard", 1500.0)
  )
  
  val productMap = products.foldLeft(Map.empty[String, Product]) { (acc, p) =>
    acc + (p.id -> p)
  }
  
  println(s"\nproductMap:")
  productMap.foreach { case (id, p) => println(s"  $id -> ${p.name}: ${p.price}") }
  
  // Counting with fold
  val items = List("apple", "banana", "apple", "cherry", "banana", "apple")
  val counts = items.foldLeft(Map.empty[String, Int]) { (acc, item) =>
    acc + (item -> (acc.getOrElse(item, 0) + 1))
  }
  println(s"\nItem counts: $counts")
  
  // foldRight - accumulates from right to left
  val foldRightResult = numbers.foldRight(List.empty[Int])((n, acc) => n :: acc)
  println(s"\nfoldRight (rebuild list): $foldRightResult")
  
  // Practical: Running statistics
  case class Stats(count: Int, sum: Double, min: Double, max: Double) {
    def avg: Double = if (count > 0) sum / count else 0
  }
  
  val scores = List(85.0, 92.0, 78.0, 95.0, 88.0, 72.0, 91.0)
  
  val stats = scores.foldLeft(
    Stats(0, 0.0, Double.MaxValue, Double.MinValue)
  ) { (acc, score) =>
    Stats(
      count = acc.count + 1,
      sum = acc.sum + score,
      min = Math.min(acc.min, score),
      max = Math.max(acc.max, score)
    )
  }
  
  println(s"\n=== Statistics ===")
  println(s"Count: ${stats.count}")
  println(f"Sum: ${stats.sum}%.1f")
  println(f"Avg: ${stats.avg}%.2f")
  println(f"Min: ${stats.min}%.1f, Max: ${stats.max}%.1f")
}
```

---

## Step 155: reduce, reduceLeft, reduceOption

```scala
object ReduceDemo extends App {
  
  // reduce: like fold but no initial value, throws on empty list
  val numbers = List(1, 2, 3, 4, 5)
  
  val sum = numbers.reduce(_ + _)
  val max = numbers.reduce((a, b) => if (a > b) a else b)
  val min = numbers.reduce(math.min)
  
  println("=== reduce ===")
  println(s"sum: $sum")
  println(s"max: $max")
  println(s"min: $min")
  
  // reduceOption - safe version (returns Option)
  val empty: List[Int] = List.empty
  println(s"\nreduce on empty: ${empty.reduceOption(_ + _)}")  // None - safe!
  
  // Aggregating with reduce
  case class Transaction(id: String, amount: Double, category: String)
  
  val transactions = List(
    Transaction("T001", 1500.0, "Food"),
    Transaction("T002", 500.0, "Transport"),
    Transaction("T003", 2000.0, "Shopping"),
    Transaction("T004", 300.0, "Food"),
    Transaction("T005", 800.0, "Entertainment"),
    Transaction("T006", 1200.0, "Shopping")
  )
  
  // Find max transaction
  val maxTx = transactions.maxBy(_.amount)
  val minTx = transactions.minBy(_.amount)
  
  println("\n=== Transaction Analysis ===")
  println(s"Max: ${maxTx.id} - ${maxTx.amount} (${maxTx.category})")
  println(s"Min: ${minTx.id} - ${minTx.amount} (${minTx.category})")
  
  // Total by category using fold (better than reduce for this)
  val totalByCategory = transactions.foldLeft(Map.empty[String, Double]) { (acc, tx) =>
    acc + (tx.category -> (acc.getOrElse(tx.category, 0.0) + tx.amount))
  }
  
  println("\nTotal by category:")
  totalByCategory.toList.sortBy(-_._2).foreach { case (cat, total) =>
    println(f"  $cat: $total%.2f THB")
  }
  
  // reduceLeft for string operations
  val words = List("Scala", "is", "a", "functional", "language")
  val sentence = words.reduceLeft((a, b) => s"$a $b")
  println(s"\nSentence: $sentence")
  
  // Custom reduce for business rules
  case class Bid(bidder: String, amount: Double, timestamp: Long)
  
  val bids = List(
    Bid("Alice", 1000.0, 1000),
    Bid("Bob", 1200.0, 2000),
    Bid("Carol", 1100.0, 3000),
    Bid("David", 1200.0, 4000),  // Same as Bob but later
    Bid("Eve", 1500.0, 5000)
  )
  
  // Find winner: highest bid, earliest if tie
  val winner = bids.reduce { (best, current) =>
    if (current.amount > best.amount) current
    else if (current.amount == best.amount && current.timestamp < best.timestamp) current
    else best
  }
  
  println(s"\nAuction winner: ${winner.bidder} with ${winner.amount} THB")
}
```

---

## Step 156: scan — Running Totals

```scala
object ScanDemo extends App {
  
  // scanLeft: like foldLeft but keeps all intermediate results
  val numbers = List(1, 2, 3, 4, 5)
  
  val runningSum = numbers.scanLeft(0)(_ + _)
  val runningProduct = numbers.scanLeft(1)(_ * _)
  
  println("=== scanLeft ===")
  println(s"numbers: $numbers")
  println(s"runningSum: $runningSum")    // [0, 1, 3, 6, 10, 15]
  println(s"runningProduct: $runningProduct")  // [1, 1, 2, 6, 24, 120]
  
  // Running max/min
  val unsorted = List(3, 1, 4, 1, 5, 9, 2, 6, 5, 3)
  val runningMax = unsorted.scanLeft(Int.MinValue)(math.max).tail
  val runningMin = unsorted.scanLeft(Int.MaxValue)(math.min).tail
  
  println(s"\nunsorted: $unsorted")
  println(s"runningMax: $runningMax")
  println(s"runningMin: $runningMin")
  
  // Financial: Running balance
  case class BankTransaction(description: String, amount: Double)
  
  val transactions = List(
    BankTransaction("Initial deposit", 10000.0),
    BankTransaction("Rent payment", -3500.0),
    BankTransaction("Salary", 35000.0),
    BankTransaction("Groceries", -2500.0),
    BankTransaction("Electric bill", -1200.0),
    BankTransaction("Bonus", 5000.0),
    BankTransaction("Credit card payment", -8000.0)
  )
  
  val runningBalance = transactions.scanLeft(0.0)((balance, tx) => balance + tx.amount).tail
  
  println("\n=== Running Balance ===")
  println(f"{'Date/Description':<25} ${"Amount":>10} ${"Balance":>10}")
  println("-" * 50)
  
  transactions.zip(runningBalance).foreach { case (tx, balance) =>
    val amountStr = if (tx.amount >= 0) f"+${tx.amount}%.2f" else f"${tx.amount}%.2f"
    println(f"${tx.description:<25} ${amountStr:>10} ${balance:>10.2f}")
  }
  
  println(f"\nFinal balance: ${runningBalance.last}%.2f THB")
  println(s"Low point: ${runningBalance.min}")
  
  // Stock price moving average using scan
  val stockPrices = List(100.0, 105.0, 103.0, 108.0, 112.0, 110.0, 115.0, 113.0, 118.0, 120.0)
  
  // 3-day moving average
  val movingAvg3 = stockPrices.sliding(3).map(window => window.sum / window.size).toList
  
  println("\n=== 3-Day Moving Average ===")
  println(s"Prices: $stockPrices")
  println(s"3-day MA: $movingAvg3")
}
```

---

## Step 157: collect — Pattern Match + Filter + Map

```scala
object CollectDemo extends App {
  
  // collect: filter + transform using PartialFunction
  val mixed: List[Any] = List(1, "hello", 2.5, 3, true, "world", 4, null, 5.0)
  
  val ints = mixed.collect { case n: Int => n }
  val strings = mixed.collect { case s: String => s }
  val doubles = mixed.collect { case d: Double => d }
  
  println("=== collect ===")
  println(s"ints: $ints")
  println(s"strings: $strings")
  println(s"doubles: $doubles")
  
  // collect with guard
  val numbers = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
  
  val largeEvenStrings = numbers.collect {
    case n if n > 5 && n % 2 == 0 => s"large-even-$n"
  }
  println(s"\nlarge even as strings: $largeEvenStrings")
  
  // Domain example
  sealed trait Event
  case class UserSignup(userId: String, email: String) extends Event
  case class UserLogin(userId: String, timestamp: Long) extends Event
  case class Purchase(userId: String, orderId: String, amount: Double) extends Event
  case class SystemError(code: Int, message: String) extends Event
  
  val events: List[Event] = List(
    UserSignup("U001", "alice@example.com"),
    UserLogin("U001", 1000L),
    Purchase("U001", "O001", 1500.0),
    SystemError(500, "Database connection failed"),
    UserSignup("U002", "bob@example.com"),
    Purchase("U001", "O002", 3000.0),
    UserLogin("U002", 2000L),
    SystemError(404, "Resource not found"),
    Purchase("U002", "O003", 500.0)
  )
  
  // Extract specific event types
  val signups = events.collect { case s: UserSignup => s }
  val purchases = events.collect { case p: Purchase => p }
  val errors = events.collect { case e: SystemError => e }
  
  println("\n=== Event Processing ===")
  println(s"Signups: ${signups.map(_.email)}")
  println(s"Purchase amounts: ${purchases.map(_.amount)}")
  println(s"Errors: ${errors.map(e => s"[${e.code}] ${e.message}")}")
  
  // Collect with transformation
  val largeTransactions = events.collect {
    case Purchase(userId, orderId, amount) if amount >= 1000 => 
      Map("user" -> userId, "order" -> orderId, "amount" -> f"$amount%.2f")
  }
  
  println("\nLarge transactions:")
  largeTransactions.foreach(println)
  
  // Using collectFirst - get first match
  val firstError = events.collectFirst { case e: SystemError => e }
  println(s"\nFirst error: $firstError")
  
  val firstLargePurchase = events.collectFirst {
    case Purchase(_, id, amount) if amount >= 2000 => s"$id: $amount"
  }
  println(s"First large purchase: $firstLargePurchase")
  
  // collect vs filter+map
  val result1 = numbers.filter(_ % 2 == 0).map(n => s"even-$n")
  val result2 = numbers.collect { case n if n % 2 == 0 => s"even-$n" }
  println(s"\nBoth approaches equal: ${result1 == result2}")
}
```

---

## Step 158: Combining Operations — Chain Processing

```scala
object CombinedOpsDemo extends App {
  
  // Real-world e-commerce analytics
  case class SaleItem(
    transactionId: String,
    customerId: String,
    productId: String,
    productName: String,
    category: String,
    quantity: Int,
    unitPrice: Double,
    timestamp: Long
  ) {
    def revenue: Double = quantity * unitPrice
  }
  
  val sales = List(
    SaleItem("T001", "C001", "P001", "Laptop Pro", "Electronics", 1, 45000.0, 1000L),
    SaleItem("T002", "C002", "P002", "Python Book", "Books", 2, 650.0, 1100L),
    SaleItem("T003", "C001", "P003", "Wireless Mouse", "Electronics", 3, 850.0, 1200L),
    SaleItem("T004", "C003", "P004", "Mechanical Keyboard", "Electronics", 1, 2500.0, 1300L),
    SaleItem("T005", "C002", "P001", "Laptop Pro", "Electronics", 2, 45000.0, 1400L),
    SaleItem("T006", "C004", "P005", "Data Science Book", "Books", 1, 780.0, 1500L),
    SaleItem("T007", "C001", "P006", "USB-C Hub", "Accessories", 2, 1200.0, 1600L),
    SaleItem("T008", "C003", "P007", "Monitor 4K", "Electronics", 1, 15000.0, 1700L),
    SaleItem("T009", "C005", "P008", "Ergonomic Chair", "Furniture", 1, 8500.0, 1800L),
    SaleItem("T010", "C002", "P003", "Wireless Mouse", "Electronics", 1, 850.0, 1900L)
  )
  
  println("=== E-Commerce Analytics ===")
  
  // 1. Total revenue
  val totalRevenue = sales.map(_.revenue).sum
  println(f"\n1. Total Revenue: $totalRevenue%.2f THB")
  
  // 2. Revenue by category
  val revenueByCategory = sales
    .groupBy(_.category)
    .map { case (cat, items) => cat -> items.map(_.revenue).sum }
    .toList
    .sortBy(-_._2)
  
  println("\n2. Revenue by Category:")
  revenueByCategory.foreach { case (cat, rev) =>
    val pct = rev / totalRevenue * 100
    println(f"   $cat%-15s: $rev%10.2f THB ($pct%.1f%%)")
  }
  
  // 3. Top products by revenue
  val topProducts = sales
    .groupBy(_.productName)
    .map { case (name, items) => 
      name -> items.map(_.revenue).sum 
    }
    .toList
    .sortBy(-_._2)
    .take(3)
  
  println("\n3. Top 3 Products:")
  topProducts.zipWithIndex.foreach { case ((name, rev), idx) =>
    println(f"   ${idx + 1}. $name: $rev%.2f THB")
  }
  
  // 4. Customer analysis
  val customerStats = sales
    .groupBy(_.customerId)
    .map { case (cId, items) =>
      val totalSpent = items.map(_.revenue).sum
      val avgOrder = totalSpent / items.size
      (cId, items.size, totalSpent, avgOrder)
    }
    .toList
    .sortBy(-_._3)
  
  println("\n4. Customer Analysis:")
  println(f"   ${"Customer":<10} ${"Orders":>6} ${"Total":>12} ${"Avg Order":>12}")
  println("   " + "-" * 45)
  customerStats.foreach { case (cId, orders, total, avg) =>
    println(f"   $cId%-10s $orders%6d $total%12.2f $avg%12.2f")
  }
  
  // 5. Items that are selling well (revenue > 5000)
  val topItems = sales
    .filter(_.revenue > 5000)
    .sortBy(-_.revenue)
    .map(s => f"${s.productName} (${s.revenue}%.0f THB)")
  
  println(s"\n5. High-value transactions: $topItems")
  
  // 6. Category diversity per customer
  val customerDiversity = sales
    .groupBy(_.customerId)
    .map { case (cId, items) =>
      cId -> items.map(_.category).distinct.size
    }
    .toList
    .sortBy(-_._2)
  
  println("\n6. Category diversity per customer:")
  customerDiversity.foreach { case (cId, cats) =>
    println(s"   $cId: $cats categories")
  }
}
```

---

## Step 159: zip, unzip, zipWithIndex

```scala
object ZipDemo extends App {
  
  // zip - combine two lists element by element
  val names = List("Alice", "Bob", "Carol", "David")
  val scores = List(95, 87, 92, 78)
  val grades = List("A", "B+", "A-", "B")
  
  val combined = names.zip(scores)
  println("=== zip ===")
  println(combined)
  
  // zip with transformation
  val report = names.zip(scores).map { case (name, score) =>
    s"$name: $score"
  }
  println(report)
  
  // zipWithIndex - add index
  println("\n=== zipWithIndex ===")
  names.zipWithIndex.foreach { case (name, idx) =>
    println(s"  ${idx + 1}. $name")
  }
  
  // zip3 - Scala doesn't have it natively, use lazip
  val fullReport = names.zip(scores).zip(grades).map { case ((name, score), grade) =>
    s"$name: $score ($grade)"
  }
  println(s"\nFull report: $fullReport")
  
  // unzip - split pairs
  val (unzippedNames, unzippedScores) = combined.unzip
  println(s"\nunzipped names: $unzippedNames")
  println(s"unzipped scores: $unzippedScores")
  
  // zipAll - zip with default values for shorter list
  val shortList = List(1, 2, 3)
  val longList = List("a", "b", "c", "d", "e")
  
  val zippedAll = shortList.zipAll(longList, -1, "?")
  println(s"\nzipAll: $zippedAll")
  
  // Practical: Parallel processing results
  val tasks = List("task1", "task2", "task3", "task4")
  val results = List(Right("success"), Left("error"), Right("success"), Right("done"))
  
  val taskResults = tasks.zip(results).map { case (task, result) =>
    result match {
      case Right(msg) => s"$task: OK ($msg)"
      case Left(err)  => s"$task: FAILED ($err)"
    }
  }
  
  println("\n=== Task Results ===")
  taskResults.foreach(println)
  
  // Building lookup tables with zip
  val keys = List("name", "age", "city")
  val values = List("Alice", "30", "Bangkok")
  
  val record = keys.zip(values).toMap
  println(s"\nRecord: $record")
  
  // Transpose and zip for matrix operations
  val matrix = List(
    List(1, 2, 3),
    List(4, 5, 6),
    List(7, 8, 9)
  )
  
  val transposed = matrix.transpose
  println(s"\nOriginal: $matrix")
  println(s"Transposed: $transposed")
}
```

---

## Step 160: Real-World Analytics Pipeline

```scala
object AnalyticsPipeline extends App {
  
  import java.time.{LocalDate, Month}
  
  case class SalesRecord(
    date: LocalDate,
    region: String,
    salesperson: String,
    product: String,
    amount: Double,
    units: Int
  )
  
  // Generate sample data
  val regions = List("North", "South", "East", "West", "Central")
  val salespeople = List("Alice", "Bob", "Carol", "David", "Eve", "Frank")
  val products = List("Product A", "Product B", "Product C", "Product D")
  
  val rng = new scala.util.Random(42)
  
  val salesData: List[SalesRecord] = {
    for {
      month <- 1 to 12
      day   <- List(5, 10, 15, 20, 25)
    } yield SalesRecord(
      date = LocalDate.of(2024, month, day),
      region = regions(rng.nextInt(regions.size)),
      salesperson = salespeople(rng.nextInt(salespeople.size)),
      product = products(rng.nextInt(products.size)),
      amount = 1000 + rng.nextDouble() * 9000,
      units = 1 + rng.nextInt(20)
    )
  }
  
  println(s"=== Sales Analytics (${salesData.size} records) ===")
  
  // Monthly totals
  val monthlyTotals = salesData
    .groupBy(_.date.getMonth)
    .map { case (month, records) => month -> records.map(_.amount).sum }
    .toList
    .sortBy(_._1)
  
  println("\n--- Monthly Revenue ---")
  monthlyTotals.foreach { case (month, total) =>
    val bar = "#" * (total / 2000).toInt
    println(f"  ${month.toString.take(3)}%-4s: $total%8.0f  $bar")
  }
  
  // Top performers
  val topSalespeople = salesData
    .groupBy(_.salesperson)
    .map { case (name, records) => 
      (name, records.map(_.amount).sum, records.size) 
    }
    .toList
    .sortBy(-_._2)
    .take(3)
  
  println("\n--- Top Salespeople ---")
  topSalespeople.foreach { case (name, total, count) =>
    println(f"  $name%-10s: $total%8.0f THB ($count orders, avg ${total/count}%.0f)")
  }
  
  // Product analysis
  val productAnalysis = salesData
    .groupBy(_.product)
    .map { case (prod, records) =>
      val revenue = records.map(_.amount).sum
      val units = records.map(_.units).sum
      val avgPrice = revenue / units
      (prod, revenue, units, avgPrice)
    }
    .toList
    .sortBy(-_._2)
  
  println("\n--- Product Performance ---")
  println(f"  ${"Product":<12} ${"Revenue":>10} ${"Units":>6} ${"Avg Price":>10}")
  println("  " + "-" * 45)
  productAnalysis.foreach { case (prod, rev, units, avg) =>
    println(f"  $prod%-12s $rev%10.0f $units%6d $avg%10.0f")
  }
  
  // Regional performance with running totals
  val regionTotals = salesData
    .groupBy(_.region)
    .map { case (region, records) => region -> records.map(_.amount).sum }
    .toList
    .sortBy(-_._2)
  
  val grandTotal = regionTotals.map(_._2).sum
  val regionCumulative = regionTotals.scanLeft(("", 0.0)) { (acc, curr) =>
    curr._1 -> (acc._2 + curr._2)
  }.tail
  
  println("\n--- Regional Performance (Cumulative) ---")
  regionTotals.zip(regionCumulative).foreach { case ((region, total), (_, cumulative)) =>
    val pct = cumulative / grandTotal * 100
    println(f"  $region%-10s: $total%8.0f (cumulative: $pct%.1f%%)")
  }
  
  // Quarterly summary
  val quarterlySummary = salesData
    .groupBy(r => (r.date.getYear, (r.date.getMonthValue - 1) / 3 + 1))
    .map { case ((year, quarter), records) =>
      val total = records.map(_.amount).sum
      val count = records.size
      (s"Q$quarter $year", total, count)
    }
    .toList
    .sortBy(_._1)
  
  println("\n--- Quarterly Summary ---")
  quarterlySummary.foreach { case (quarter, total, count) =>
    println(f"  $quarter: $total%9.0f THB ($count transactions)")
  }
}
```

---

## สรุป Part 16

| Operation | Signature | ใช้งาน |
|-----------|-----------|--------|
| `map` | `List[A] => (A => B) => List[B]` | แปลงทุก element |
| `flatMap` | `List[A] => (A => List[B]) => List[B]` | map แล้ว flatten |
| `filter` | `List[A] => (A => Boolean) => List[A]` | กรองตาม predicate |
| `foldLeft` | `List[A] => B => ((B,A) => B) => B` | สะสมจากซ้ายไปขวา |
| `reduce` | `List[A] => ((A,A) => A) => A` | รวมโดยไม่มีค่าเริ่มต้น |
| `scan` | `List[A] => B => ((B,A) => B) => List[B]` | running total |
| `collect` | `List[A] => PartialFunction[A,B] => List[B]` | filter + transform |
| `zip` | `List[A] => List[B] => List[(A,B)]` | จับคู่ lists |
| `groupBy` | `List[A] => (A => K) => Map[K, List[A]]` | จัดกลุ่ม |
| `partition` | `List[A] => (A => Boolean) => (List[A], List[A])` | แบ่งเป็น 2 กลุ่ม |

---

## แบบฝึกหัด Part 16

**ข้อ 1:** สร้าง word frequency analyzer: รับ text แล้ว return Map[String, Int] ของจำนวนครั้งที่แต่ละคำปรากฏ (case-insensitive, remove punctuation) พร้อม top-N words

**ข้อ 2:** Implement histogram function ที่รับ `List[Double]` และ number of buckets แล้ว return `Map[String, Int]` ของ frequency ใน each bucket

**ข้อ 3:** สร้าง time-series analysis: รับ `List[(timestamp, value)]` แล้ย compute moving average, bollinger bands (mean ± 2σ)

**ข้อ 4:** Implement matrix multiplication โดยใช้ `map`, `zip`, `foldLeft` เท่านั้น (ไม่ใช้ for-loop)

**ข้อ 5:** สร้าง data cleaning pipeline: รับ `List[Map[String, String]]` แล้ว apply: trim whitespace, remove duplicates, validate required fields, transform types — คืน `(List[Map[String, Any]], List[String])` (cleaned, errors)

---

➡️ ต่อไป: [Part 17 — Immutability and Pure Functions](part-17-immutability-and-pure-functions.md)
