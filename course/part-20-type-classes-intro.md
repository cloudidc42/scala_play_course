# Part 20: Type Classes Introduction

## Steps 191-200: Type Classes, Ordering, Numeric, Show, Equal, Functor Basics

---

## Step 191: Type Class คืออะไร

```scala
// Type class = interface/trait + implicit instances
// Pattern: define behavior for types you don't own

// Define the type class
trait Show[A] {
  def show(value: A): String
}

// Instances for specific types
object Show {
  // Instance for Int
  implicit val intShow: Show[Int] = new Show[Int] {
    override def show(value: Int): String = value.toString
  }
  
  // Instance for String
  implicit val stringShow: Show[String] = new Show[String] {
    override def show(value: String): String = s""""$value""""
  }
  
  // Instance for Boolean
  implicit val boolShow: Show[Boolean] = new Show[Boolean] {
    override def show(value: Boolean): String = if (value) "yes" else "no"
  }
  
  // Instance for List - depends on Show[A]
  implicit def listShow[A](implicit showA: Show[A]): Show[List[A]] = new Show[List[A]] {
    override def show(list: List[A]): String = 
      list.map(showA.show).mkString("[", ", ", "]")
  }
  
  // Instance for Option
  implicit def optionShow[A](implicit showA: Show[A]): Show[Option[A]] = new Show[Option[A]] {
    override def show(opt: Option[A]): String = opt match {
      case Some(v) => s"Some(${showA.show(v)})"
      case None    => "None"
    }
  }
  
  // Summon syntax
  def apply[A](implicit show: Show[A]): Show[A] = show
  
  // Extension method syntax
  implicit class ShowOps[A](value: A)(implicit show: Show[A]) {
    def show: String = Show[A].show(value)
  }
}

object TypeClassIntro extends App {
  import Show._
  
  println("=== Show Type Class ===")
  println(42.show)
  println("hello".show)
  println(true.show)
  println(List(1, 2, 3).show)
  println(Some("hello").show)
  println((None: Option[Int]).show)
  
  // Generic function using type class
  def printAll[A: Show](items: List[A]): Unit = {
    items.foreach(item => println(s"  ${implicitly[Show[A]].show(item)}"))
  }
  
  println("\nInts:")
  printAll(List(1, 2, 3))
  
  println("Strings:")
  printAll(List("scala", "haskell", "java"))
  
  // Custom type with Show instance
  case class Color(r: Int, g: Int, b: Int)
  
  implicit val colorShow: Show[Color] = new Show[Color] {
    override def show(c: Color): String = f"Color(#${c.r}%02X${c.g}%02X${c.b}%02X)"
  }
  
  println(s"\nColor: ${Color(255, 128, 0).show}")
  println(s"Colors: ${List(Color(255, 0, 0), Color(0, 255, 0)).show}")
}
```

---

## Step 192: Ordering Type Class

```scala
// Ordering type class - for comparison
object OrderingTypeClass extends App {
  
  // Scala's built-in Ordering
  case class Person(name: String, age: Int)
  
  // Custom Ordering instances
  val byAge: Ordering[Person] = Ordering.by(_.age)
  val byName: Ordering[Person] = Ordering.by(_.name)
  val byAgeDesc: Ordering[Person] = byAge.reverse
  
  // Combined ordering
  val byAgehenName: Ordering[Person] = 
    Ordering.by((p: Person) => (p.age, p.name))
  
  val people = List(
    Person("Charlie", 30),
    Person("Alice", 25),
    Person("Bob", 30),
    Person("Diana", 25)
  )
  
  println("=== Ordering Type Class ===")
  println(s"By age: ${people.sorted(byAge)}")
  println(s"By name: ${people.sorted(byName)}")
  println(s"By age desc: ${people.sorted(byAgeDesc)}")
  println(s"By age then name: ${people.sorted(byAgehenName)}")
  
  // Using Ordering in generic functions
  def findTop[A](items: List[A], n: Int)(implicit ord: Ordering[A]): List[A] = {
    items.sorted(ord.reverse).take(n)
  }
  
  println(s"\nTop 3 by age desc: ${findTop(people, 3)(byAge)}")
  
  // Custom type class for domain-specific comparison
  trait Priority[A] {
    def priority(a: A): Int
    def compare(a: A, b: A): Int = priority(a) - priority(b)
    def isHigherPriority(a: A, b: A): Boolean = compare(a, b) > 0
  }
  
  sealed trait TaskStatus
  case object Urgent extends TaskStatus
  case object High extends TaskStatus
  case object Normal extends TaskStatus
  case object Low extends TaskStatus
  
  case class Task(id: String, title: String, status: TaskStatus, dueHours: Int)
  
  implicit val taskPriority: Priority[Task] = new Priority[Task] {
    override def priority(task: Task): Int = {
      val statusPriority = task.status match {
        case Urgent => 4
        case High   => 3
        case Normal => 2
        case Low    => 1
      }
      val urgencyBonus = if (task.dueHours <= 24) 2 else 0
      statusPriority + urgencyBonus
    }
  }
  
  val tasks = List(
    Task("T1", "Fix critical bug", Urgent, 2),
    Task("T2", "Update docs", Low, 72),
    Task("T3", "Code review", Normal, 12),
    Task("T4", "Deploy feature", High, 48),
    Task("T5", "Meeting prep", High, 4)
  )
  
  val prioritized = tasks.sortBy(t => -implicitly[Priority[Task]].priority(t))
  
  println("\n=== Prioritized Tasks ===")
  prioritized.foreach { t =>
    val p = implicitly[Priority[Task]].priority(t)
    println(s"  [P$p] ${t.title} (${t.status}, ${t.dueHours}h)")
  }
}
```

---

## Step 193: Numeric Type Class

```scala
// Numeric type class - generic arithmetic
object NumericTypeClass extends App {
  
  // Generic sum using Numeric
  def sum[T: Numeric](items: List[T]): T = {
    val num = implicitly[Numeric[T]]
    items.foldLeft(num.zero)(num.plus)
  }
  
  def average[T: Numeric](items: List[T]): Double = {
    val num = implicitly[Numeric[T]]
    if (items.isEmpty) 0.0
    else num.toDouble(sum(items)) / items.size
  }
  
  def normalize[T: Numeric](items: List[T]): List[Double] = {
    val num = implicitly[Numeric[T]]
    val doubles = items.map(num.toDouble)
    val min = doubles.min
    val max = doubles.max
    val range = max - min
    if (range == 0) items.map(_ => 0.5)
    else doubles.map(v => (v - min) / range)
  }
  
  println("=== Numeric Type Class ===")
  
  val ints = List(1, 2, 3, 4, 5)
  val doubles = List(1.5, 2.5, 3.5, 4.5)
  val longs = List(100L, 200L, 300L)
  
  println(s"sum(ints) = ${sum(ints)}")
  println(s"sum(doubles) = ${sum(doubles)}")
  println(s"sum(longs) = ${sum(longs)}")
  println(s"avg(ints) = ${average(ints)}")
  
  println(s"\nnormalized ints: ${normalize(ints)}")
  println(s"normalized doubles: ${normalize(doubles)}")
  
  // Statistics using Numeric
  case class Stats[T](
    count: Int,
    sum: T,
    min: T,
    max: T
  )(implicit num: Numeric[T]) {
    def avg: Double = num.toDouble(sum) / count
    override def toString: String = 
      f"Stats(count=$count, sum=${num.toDouble(sum)}%.2f, min=${num.toDouble(min)}%.2f, max=${num.toDouble(max)}%.2f, avg=$avg%.2f)"
  }
  
  def computeStats[T: Numeric](items: List[T]): Option[Stats[T]] = {
    val num = implicitly[Numeric[T]]
    if (items.isEmpty) None
    else Some(Stats(
      count = items.size,
      sum = items.foldLeft(num.zero)(num.plus),
      min = items.min,
      max = items.max
    ))
  }
  
  println(s"\nStats: ${computeStats(ints)}")
  println(s"Stats: ${computeStats(doubles)}")
  println(s"Empty: ${computeStats(List.empty[Int])}")
}
```

---

## Step 194: Eq Type Class

```scala
// Eq type class - type-safe equality
trait Eq[A] {
  def eqv(a: A, b: A): Boolean
  def neqv(a: A, b: A): Boolean = !eqv(a, b)
}

object Eq {
  def apply[A](implicit eq: Eq[A]): Eq[A] = eq
  
  // Default: use ==
  def fromEquals[A]: Eq[A] = (a, b) => a == b
  
  // Instances
  implicit val intEq: Eq[Int] = fromEquals
  implicit val stringEq: Eq[String] = (a, b) => a == b
  implicit val boolEq: Eq[Boolean] = fromEquals
  
  implicit def listEq[A: Eq]: Eq[List[A]] = { (xs, ys) =>
    val elemEq = Eq[A]
    xs.length == ys.length && xs.zip(ys).forall { case (x, y) => elemEq.eqv(x, y) }
  }
  
  implicit def optionEq[A: Eq]: Eq[Option[A]] = { (a, b) =>
    val elemEq = Eq[A]
    (a, b) match {
      case (None, None)       => true
      case (Some(x), Some(y)) => elemEq.eqv(x, y)
      case _                  => false
    }
  }
  
  implicit class EqOps[A: Eq](lhs: A) {
    def ===(rhs: A): Boolean = Eq[A].eqv(lhs, rhs)
    def !==(rhs: A): Boolean = Eq[A].neqv(lhs, rhs)
  }
}

object EqTypeClass extends App {
  import Eq._
  
  println("=== Eq Type Class ===")
  println(s"3 === 3: ${3 === 3}")
  println(s"3 === 4: ${3 === 4}")
  println(s""""hello" === "hello": ${"hello" === "hello"}")
  
  println(s"List(1,2) === List(1,2): ${List(1,2) === List(1,2)}")
  println(s"List(1,2) === List(1,3): ${List(1,2) === List(1,3)}")
  
  println(s"Some(1) === Some(1): ${Some(1) === Some(1)}")
  println(s"None === None: ${(None: Option[Int]) === None}")
  
  // Custom Eq for domain types
  case class Money(amount: BigDecimal, currency: String)
  
  implicit val moneyEq: Eq[Money] = (a, b) => 
    a.currency == b.currency && a.amount.compare(b.amount) == 0
  
  val m1 = Money(BigDecimal("100.000"), "THB")
  val m2 = Money(BigDecimal("100.0"), "THB")
  
  println(s"\n100.000 THB === 100.0 THB: ${m1 === m2}")  // true (BigDecimal comparison)
  
  // Type-safe: prevents comparing wrong types at compile time
  // 3 === "3"  // Compile error!
  // 3 === 3.0  // Compile error!
}
```

---

## Step 195: Functor Type Class

```scala
// Functor - can map over
trait Functor[F[_]] {
  def map[A, B](fa: F[A])(f: A => B): F[B]
  
  // Derived operations
  def lift[A, B](f: A => B): F[A] => F[B] = fa => map(fa)(f)
  def as[A, B](fa: F[A], b: B): F[B] = map(fa)(_ => b)
  def void[A](fa: F[A]): F[Unit] = as(fa, ())
}

object Functor {
  def apply[F[_]](implicit F: Functor[F]): Functor[F] = F
  
  // Instances
  implicit val optionFunctor: Functor[Option] = new Functor[Option] {
    override def map[A, B](fa: Option[A])(f: A => B): Option[B] = fa.map(f)
  }
  
  implicit val listFunctor: Functor[List] = new Functor[List] {
    override def map[A, B](fa: List[A])(f: A => B): List[B] = fa.map(f)
  }
  
  type EitherString[A] = Either[String, A]
  implicit val eitherFunctor: Functor[EitherString] = new Functor[EitherString] {
    override def map[A, B](fa: EitherString[A])(f: A => B): EitherString[B] = fa.map(f)
  }
  
  implicit class FunctorOps[F[_]: Functor, A](fa: F[A]) {
    def fmap[B](f: A => B): F[B] = Functor[F].map(fa)(f)
  }
}

object FunctorTypeClass extends App {
  import Functor._
  
  println("=== Functor Type Class ===")
  
  // map over different structures uniformly
  def doubleAll[F[_]: Functor](container: F[Int]): F[Int] = 
    Functor[F].map(container)(_ * 2)
  
  println(s"Option: ${doubleAll(Some(5))}")
  println(s"Option None: ${doubleAll(None: Option[Int])}")
  println(s"List: ${doubleAll(List(1, 2, 3, 4, 5))}")
  
  // Using FunctorOps
  println(s"\nfmap on Option: ${Some(10).fmap(_ + 5)}")
  println(s"fmap on List: ${List(1,2,3).fmap(_ * 10)}")
  
  // Generic transformation
  def transform[F[_]: Functor, A, B](container: F[A], f: A => B): F[B] = 
    container.fmap(f)
  
  println(s"\nTransform: ${transform(Some("hello"), (s: String) => s.length)}")
  println(s"Transform: ${transform(List("a", "bb", "ccc"), (s: String) => s.length)}")
  
  // Functor laws
  // 1. Identity: map(identity) == identity
  // 2. Composition: map(g compose f) == map(g) compose map(f)
  
  val xs = List(1, 2, 3)
  val f = (n: Int) => n * 2
  val g = (n: Int) => n + 1
  
  val law1 = xs.fmap(identity) == xs
  val law2 = xs.fmap(g compose f) == xs.fmap(f).fmap(g)
  
  println(s"\nFunctor laws:")
  println(s"Identity: $law1")
  println(s"Composition: $law2")
}
```

---

## Step 196-200: Putting It All Together

```scala
// Complete example combining multiple type classes
object TypeClassSystem extends App {
  
  // Monoid type class
  trait Monoid[A] {
    def empty: A
    def combine(a: A, b: A): A
    def combineAll(list: List[A]): A = list.foldLeft(empty)(combine)
  }
  
  object Monoid {
    def apply[A](implicit M: Monoid[A]): Monoid[A] = M
    
    implicit val intAddMonoid: Monoid[Int] = new Monoid[Int] {
      override def empty: Int = 0
      override def combine(a: Int, b: Int): Int = a + b
    }
    
    implicit val stringMonoid: Monoid[String] = new Monoid[String] {
      override def empty: String = ""
      override def combine(a: String, b: String): String = a + b
    }
    
    implicit def listMonoid[A]: Monoid[List[A]] = new Monoid[List[A]] {
      override def empty: List[A] = List.empty
      override def combine(a: List[A], b: List[A]): List[A] = a ++ b
    }
    
    implicit def mapMonoid[K, V: Monoid]: Monoid[Map[K, V]] = new Monoid[Map[K, V]] {
      val valMonoid = implicitly[Monoid[V]]
      override def empty: Map[K, V] = Map.empty
      override def combine(a: Map[K, V], b: Map[K, V]): Map[K, V] = {
        val keys = a.keySet ++ b.keySet
        keys.map { k =>
          k -> valMonoid.combine(
            a.getOrElse(k, valMonoid.empty),
            b.getOrElse(k, valMonoid.empty)
          )
        }.toMap
      }
    }
  }
  
  import Monoid._
  
  println("=== Monoid Type Class ===")
  println(s"Int: ${Monoid[Int].combineAll(List(1,2,3,4,5))}")
  println(s"String: ${Monoid[String].combineAll(List("Hello", " ", "World"))}")
  println(s"List: ${Monoid[List[Int]].combineAll(List(List(1,2), List(3,4), List(5)))}")
  
  // Map monoid for aggregation
  val data = List(
    Map("a" -> 1, "b" -> 2),
    Map("b" -> 3, "c" -> 4),
    Map("a" -> 5, "c" -> 6)
  )
  
  val merged = Monoid[Map[String, Int]].combineAll(data)
  println(s"\nMerged maps: $merged")
  // Expected: Map(a->6, b->5, c->10)
  
  // Generic fold using Monoid
  def foldMap[A, B: Monoid](items: List[A])(f: A => B): B = {
    val M = Monoid[B]
    items.foldLeft(M.empty)((acc, a) => M.combine(acc, f(a)))
  }
  
  case class SaleRecord(product: String, amount: Double, region: String)
  
  val sales = List(
    SaleRecord("Laptop", 45000.0, "North"),
    SaleRecord("Mouse", 850.0, "South"),
    SaleRecord("Laptop", 43000.0, "North"),
    SaleRecord("Keyboard", 1500.0, "East"),
    SaleRecord("Mouse", 900.0, "North"),
    SaleRecord("Laptop", 46000.0, "South")
  )
  
  // Count by product using foldMap
  implicit val productCountMonoid: Monoid[Map[String, Int]] = Monoid.mapMonoid[String, Int]
  
  val countByProduct = foldMap(sales)(s => Map(s.product -> 1))
  println(s"\nCount by product: $countByProduct")
  
  // Total by region
  val totalByRegion = foldMap(sales)(s => Map(s.region -> s.amount.toInt))
  println(s"Total by region: $totalByRegion")
  
  // All products list
  val allProducts = foldMap(sales)(s => List(s.product))
  println(s"All products: $allProducts")
  println(s"Distinct: ${allProducts.distinct}")
}
```

---

## สรุป Part 20

| Type Class | What it Does | Key Methods |
|-----------|-------------|-------------|
| `Show[A]` | Convert to String | `show(a: A): String` |
| `Ordering[A]` | Compare elements | `compare(a, b): Int` |
| `Numeric[A]` | Arithmetic operations | `plus`, `minus`, `times` |
| `Eq[A]` | Type-safe equality | `eqv(a, b): Boolean` |
| `Functor[F[_]]` | map over structure | `map(fa)(f)` |
| `Monoid[A]` | Combine + identity | `empty`, `combine` |

---

## แบบฝึกหัด Part 20

**ข้อ 1:** Implement `Encoder[A]` type class ที่ convert type A ไปเป็น `Map[String, Any]` (JSON-like) พร้อม instances สำหรับ primitive types และ case classes

**ข้อ 2:** สร้าง `Comparable[A]` type class ที่ rich กว่า Ordering โดยมี `min`, `max`, `clamp`, `between`

**ข้อ 3:** Implement `Hash[A]` type class สำหรับ type-safe hashing พร้อม consistent hash codes

**ข้อ 4:** สร้าง `Foldable[F[_]]` type class ที่มี `foldLeft`, `foldRight`, `toList`, `size`, `exists`, `forall`

**ข้อ 5:** Combine Show + Eq + Ordering + Monoid เพื่อสร้าง generic data analysis function ที่ group, sort, aggregate data ได้

---

➡️ ต่อไป: [Part 21 — List and Vector](part-21-list-and-vector.md)
