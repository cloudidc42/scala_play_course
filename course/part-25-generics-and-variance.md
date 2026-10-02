# Part 25: Generics and Variance

## Steps 241-250: Type Parameters, Variance, Type Bounds, Higher-Kinded Types

---

## Step 241: Generic Classes และ Methods พื้นฐาน

```scala
// Generic class — type parameter T
class Box[A](val value: A) {
  def map[B](f: A => B): Box[B] = new Box(f(value))
  def flatMap[B](f: A => Box[B]): Box[B] = f(value)
  override def toString: String = s"Box($value)"
}

// Generic method
def identity[A](x: A): A = x
def swap[A, B](pair: (A, B)): (B, A) = (pair._2, pair._1)
def first[A](list: List[A]): Option[A] = list.headOption

// Multiple type parameters
class Pair[A, B](val first: A, val second: B) {
  def swap: Pair[B, A] = new Pair(second, first)
  def mapFirst[C](f: A => C): Pair[C, B] = new Pair(f(first), second)
  def mapSecond[C](f: B => C): Pair[A, C] = new Pair(first, f(second))
  override def toString: String = s"Pair($first, $second)"
}

// Generic data structures
sealed trait Tree[+A]
case object Leaf extends Tree[Nothing]
case class Branch[A](value: A, left: Tree[A], right: Tree[A]) extends Tree[A]

object Tree {
  def depth[A](tree: Tree[A]): Int = tree match {
    case Leaf => 0
    case Branch(_, l, r) => 1 + Math.max(depth(l), depth(r))
  }
  
  def size[A](tree: Tree[A]): Int = tree match {
    case Leaf => 0
    case Branch(_, l, r) => 1 + size(l) + size(r)
  }
  
  def map[A, B](tree: Tree[A])(f: A => B): Tree[B] = tree match {
    case Leaf => Leaf
    case Branch(v, l, r) => Branch(f(v), map(l)(f), map(r)(f))
  }
  
  def fold[A, B](tree: Tree[A])(leaf: B)(branch: (A, B, B) => B): B = tree match {
    case Leaf => leaf
    case Branch(v, l, r) => branch(v, fold(l)(leaf)(branch), fold(r)(leaf)(branch))
  }
  
  def toList[A](tree: Tree[A]): List[A] = fold(tree)(List.empty[A]) { 
    (v, l, r) => l ++ List(v) ++ r 
  }
}

object GenericsBasics extends App {
  
  println("=== Generic Box ===")
  val intBox = new Box(42)
  val strBox = intBox.map(_.toString)
  val doubled = intBox.map(_ * 2)
  println(s"intBox: $intBox")
  println(s"strBox: $strBox")
  println(s"doubled: $doubled")
  
  println("\n=== Generic Pair ===")
  val pair = new Pair("Alice", 30)
  println(s"pair: $pair")
  println(s"swapped: ${pair.swap}")
  println(s"mapFirst: ${pair.mapFirst(_.toUpperCase)}")
  
  println("\n=== Generic Tree ===")
  val tree: Tree[Int] = Branch(1,
    Branch(2, Branch(4, Leaf, Leaf), Branch(5, Leaf, Leaf)),
    Branch(3, Branch(6, Leaf, Leaf), Leaf)
  )
  
  println(s"depth: ${Tree.depth(tree)}")
  println(s"size: ${Tree.size(tree)}")
  println(s"inorder: ${Tree.toList(tree)}")
  
  val doubled2 = Tree.map(tree)(_ * 2)
  println(s"doubled: ${Tree.toList(doubled2)}")
  
  val sum = Tree.fold(tree)(0)((v, l, r) => v + l + r)
  println(s"sum: $sum")
}
```

---

## Step 242: Covariance (+T) และ Contravariance (-T)

```scala
// Covariance: Producer (+A) — if B extends A, then F[B] is subtype of F[A]
// Use: "output only", read-only, producers
sealed trait Animal { def name: String }
class Mammal(val name: String) extends Animal
class Dog(n: String) extends Mammal(n)
class Cat(n: String) extends Mammal(n)

// Covariant container — can only produce A, not consume it
class Cage[+A](val animal: A) {
  def describe: String = s"Cage containing ${animal.name}"
  // def put(a: A): Unit = ???  // NOT ALLOWED — would be unsound
}

// Contravariance: Consumer (-A) — if B extends A, then F[A] is subtype of F[B]
// Use: "input only", write-only, consumers
trait Shelter[-A] {
  def accept(animal: A): String
}

// Invariance: can read AND write
class MutableBox[A](var value: A) {
  def get: A = value
  def set(a: A): Unit = value = a
}

object VarianceDemo extends App {
  
  println("=== Covariance ===")
  val dogCage: Cage[Dog] = new Cage(new Dog("Rex"))
  
  // Covariance: Cage[Dog] is subtype of Cage[Mammal] is subtype of Cage[Animal]
  val mammalCage: Cage[Mammal] = dogCage   // OK — covariance
  val animalCage: Cage[Animal] = dogCage   // OK — covariance
  
  println(s"Dog: ${dogCage.describe}")
  println(s"As Mammal: ${mammalCage.describe}")
  println(s"As Animal: ${animalCage.describe}")
  
  println("\n=== Contravariance ===")
  
  val animalShelter: Shelter[Animal] = new Shelter[Animal] {
    def accept(animal: Animal): String = s"Accepted any animal: ${animal.name}"
  }
  
  // Contravariance: Shelter[Animal] is subtype of Shelter[Dog]
  // A shelter that accepts any Animal can also accept a Dog
  val dogShelter: Shelter[Dog] = animalShelter   // OK — contravariance
  
  println(dogShelter.accept(new Dog("Buddy")))
  
  println("\n=== List is Covariant ===")
  val dogs: List[Dog] = List(new Dog("Rex"), new Dog("Buddy"))
  val animals: List[Animal] = dogs   // OK — List is covariant
  println(s"dogs as animals: ${animals.map(_.name)}")
  
  println("\n=== Function Variance ===")
  // Function is contravariant in input, covariant in output
  // Function[-A, +B]
  
  val processAnimal: Animal => String = a => s"Processing ${a.name}"
  
  // Function[Animal, String] can be used as Function[Dog, String]
  // because Dog is a subtype of Animal (contravariant input)
  val processDog: Dog => String = processAnimal
  
  println(processDog(new Dog("Max")))
  
  // Invariant — MutableBox[Dog] is NOT MutableBox[Animal]
  val mutableDogBox = new MutableBox[Dog](new Dog("Spike"))
  // val mutableAnimalBox: MutableBox[Animal] = mutableDogBox  // COMPILE ERROR — invariant
}
```

---

## Step 243: Type Bounds — Upper and Lower

```scala
// Upper bound: T <: Animal — T must be Animal or subtype
def describe[A <: Animal](animal: A): String = 
  s"${animal.getClass.getSimpleName}: ${animal.name}"

// Lower bound: T >: Dog — T must be Dog or supertype  
def addToList[A >: Dog](dog: Dog, list: List[A]): List[A] = dog :: list

// View bound (deprecated) → use implicit parameter instead
// Context bound: T: Ordering — T must have Ordering[T] in scope
def max[A: Ordering](x: A, y: A): A = if (implicitly[Ordering[A]].gt(x, y)) x else y
def sort[A: Ordering](list: List[A]): List[A] = list.sorted

// Multiple bounds
trait Printable { def print(): Unit }
trait Serializable2 { def serialize(): String }

def processItem[A <: Printable with Serializable2](item: A): String = {
  item.print()
  item.serialize()
}

case class Document(title: String) extends Printable with Serializable2 {
  def print(): Unit = println(s"Printing: $title")
  def serialize(): String = s"""{"title":"$title"}"""
}

// Type bound in class definition
class SortedList[A <: Comparable[A]] private (private val data: List[A]) {
  def add(item: A): SortedList[A] = new SortedList(insert(item, data))
  
  private def insert(item: A, list: List[A]): List[A] = list match {
    case Nil => List(item)
    case h :: t =>
      if (item.compareTo(h) <= 0) item :: list
      else h :: insert(item, t)
  }
  
  def toList: List[A] = data
  override def toString: String = s"SortedList(${data.mkString(", ")})"
}

object SortedList {
  def empty[A <: Comparable[A]]: SortedList[A] = new SortedList(List.empty)
}

object TypeBoundsDemo extends App {
  
  println("=== Upper Bounds ===")
  println(describe(new Dog("Rex")))
  println(describe(new Cat("Whiskers")))
  
  println("\n=== Context Bounds (Ordering) ===")
  println(s"max(3, 5): ${max(3, 5)}")
  println(s"max('z', 'a'): ${max('z', 'a')}")
  println(s"sort: ${sort(List(3, 1, 4, 1, 5, 9, 2, 6))}")
  
  println("\n=== Document Processing ===")
  val doc = Document("Annual Report 2024")
  val json = processItem(doc)
  println(s"Serialized: $json")
  
  println("\n=== SortedList ===")
  val sl = SortedList.empty[Integer]
    .add(5).add(2).add(8).add(1).add(9).add(3)
  println(s"SortedList: $sl")
  
  val strList = SortedList.empty[String]
    .add("banana").add("apple").add("cherry").add("date")
  println(s"String SortedList: $strList")
}
```

---

## Step 244: Higher-Kinded Types (Type Constructors)

```scala
// Higher-kinded type: F[_] — type that takes a type to produce a type
// List is F[_]: takes Int to give List[Int]
// Option is F[_]: takes String to give Option[String]

// Define Functor using higher-kinded types
trait Functor[F[_]] {
  def map[A, B](fa: F[A])(f: A => B): F[B]
}

// Foldable
trait Foldable[F[_]] {
  def foldLeft[A, B](fa: F[A], b: B)(f: (B, A) => B): B
  def toList[A](fa: F[A]): List[A] = 
    foldLeft(fa, List.empty[A])((acc, a) => acc :+ a)
  def size[A](fa: F[A]): Int = foldLeft(fa, 0)((acc, _) => acc + 1)
  def isEmpty[A](fa: F[A]): Boolean = size(fa) == 0
}

// Instances for standard types
implicit val listFunctor: Functor[List] = new Functor[List] {
  def map[A, B](fa: List[A])(f: A => B): List[B] = fa.map(f)
}

implicit val optionFunctor: Functor[Option] = new Functor[Option] {
  def map[A, B](fa: Option[A])(f: A => B): Option[B] = fa.map(f)
}

// Tree Functor instance
implicit val treeFunctor: Functor[Tree] = new Functor[Tree] {
  def map[A, B](fa: Tree[A])(f: A => B): Tree[B] = Tree.map(fa)(f)
}

implicit val listFoldable: Foldable[List] = new Foldable[List] {
  def foldLeft[A, B](fa: List[A], b: B)(f: (B, A) => B): B = fa.foldLeft(b)(f)
}

// Generic functions that work with any F[_] having a Functor
def increment[F[_]: Functor](fa: F[Int]): F[Int] = 
  implicitly[Functor[F]].map(fa)(_ + 1)

def stringify[F[_]: Functor](fa: F[Int]): F[String] =
  implicitly[Functor[F]].map(fa)(_.toString)

object HigherKindedDemo extends App {
  
  println("=== Higher-Kinded Types ===")
  
  val ints = List(1, 2, 3, 4, 5)
  val opt = Option(42)
  val tree: Tree[Int] = Branch(3, Branch(1, Leaf, Leaf), Branch(5, Leaf, Leaf))
  
  println(s"increment list: ${increment(ints)}")
  println(s"increment option: ${increment(opt)}")
  println(s"increment tree: ${Tree.toList(increment(tree))}")
  
  println(s"\nstringify list: ${stringify(ints)}")
  println(s"stringify option: ${stringify(opt)}")
  
  // Foldable
  val listFold = implicitly[Foldable[List]]
  println(s"\nfoldable toList: ${listFold.toList(ints)}")
  println(s"foldable size: ${listFold.size(ints)}")
  println(s"foldable sum: ${listFold.foldLeft(ints, 0)(_ + _)}")
}
```

---

## Step 245-250: Phantom Types, Type Tags, Generic Patterns

```scala
// Phantom types — type parameter used only at compile time
// Prevent using wrong type at runtime

// State machine with phantom types
sealed trait Sealed
sealed trait Opened
sealed trait Locked

class Safe[State] private (val contents: String) {
  override def toString: String = s"Safe($contents)"
}

object Safe {
  def empty: Safe[Sealed] = new Safe("empty")
  
  def open(safe: Safe[Sealed], code: Int): Either[String, Safe[Opened]] = 
    if (code == 1234) Right(new Safe(safe.contents)) 
    else Left("Wrong code")
  
  def put(safe: Safe[Opened], item: String): Safe[Opened] = 
    new Safe(item)
  
  def seal(safe: Safe[Opened]): Safe[Sealed] = 
    new Safe(safe.contents)
  
  def lock(safe: Safe[Sealed]): Safe[Locked] = 
    new Safe(safe.contents)
  
  // Can only get content from open safe
  def getContents(safe: Safe[Opened]): String = safe.contents
}

// Phantom type for units (prevent adding apples to oranges)
class Quantity[Unit](val value: Double) {
  def +(other: Quantity[Unit]): Quantity[Unit] = new Quantity(value + other.value)
  def *(scalar: Double): Quantity[Unit] = new Quantity(value * scalar)
  override def toString: String = s"$value"
}

object Quantity {
  sealed trait Meter
  sealed trait Second  
  sealed trait Kilogram
  sealed trait MetersPerSecond
  
  type Length = Quantity[Meter]
  type Time   = Quantity[Second]
  type Mass   = Quantity[Kilogram]
  type Speed  = Quantity[MetersPerSecond]
  
  def meters(n: Double): Length = new Quantity[Meter](n)
  def seconds(n: Double): Time  = new Quantity[Second](n)
  def kg(n: Double): Mass       = new Quantity[Kilogram](n)
  
  def speed(distance: Length, time: Time): Speed = 
    new Quantity[MetersPerSecond](distance.value / time.value)
}

object PhantomTypesDemo extends App {
  
  import Quantity._
  
  println("=== Phantom Types: Safe State Machine ===")
  
  val empty = Safe.empty
  println(s"Empty safe: $empty")
  
  Safe.open(empty, 9999) match {
    case Left(err)  => println(s"Failed to open: $err")
    case Right(open) => println(s"Opened: $open")
  }
  
  Safe.open(empty, 1234) match {
    case Left(err)   => println(s"Failed: $err")
    case Right(open) =>
      val loaded = Safe.put(open, "Gold bars")
      println(s"Contents: ${Safe.getContents(loaded)}")
      val sealed2 = Safe.seal(loaded)
      val locked = Safe.lock(sealed2)
      println(s"Locked: $locked")
      // Safe.getContents(sealed2)  // COMPILE ERROR — can't read sealed safe
  }
  
  println("\n=== Phantom Types: Physical Units ===")
  
  val dist  = meters(100.0)
  val time  = seconds(9.58)
  val spd   = speed(dist, time)
  
  println(s"Distance: $dist meters")
  println(s"Time: $time seconds")
  println(s"Speed: $spd m/s")
  
  val d1 = meters(50.0)
  val d2 = meters(30.0)
  println(s"Total distance: ${d1 + d2} meters")
  
  // val wrong = dist + time  // COMPILE ERROR — can't add meters to seconds
  println("Type safety prevents mixing units at compile time!")
}
```

---

## สรุป Part 25

| Concept | Syntax | ความหมาย | ตัวอย่าง |
|---------|--------|---------|---------|
| Generic class | `class Box[A]` | Parameterized type | `Box[Int]`, `Box[String]` |
| Covariance | `+A` | F[Sub] <: F[Super] | `List[+A]`, `Option[+A]` |
| Contravariance | `-A` | F[Super] <: F[Sub] | `Function[-A,+B]` |
| Invariance | `A` | No subtyping | `MutableBox[A]` |
| Upper bound | `A <: T` | A must be T or subtype | `[A <: Animal]` |
| Lower bound | `A >: T` | A must be T or supertype | `[A >: Dog]` |
| Context bound | `A: TC` | Requires TC[A] implicit | `[A: Ordering]` |
| Higher-kinded | `F[_]` | Type constructor | `Functor[F[_]]` |
| Phantom type | compile-only param | State at type level | `Safe[Locked]` |

---

## แบบฝึกหัด Part 25

**ข้อ 1:** Implement `Either[+L, +R]` จาก scratch พร้อม `map`, `flatMap`, `fold`, `swap`

**ข้อ 2:** สร้าง generic `Cache[K, V]` ด้วย LRU eviction และ TTL expiry

**ข้อ 3:** Implement `Applicative[F[_]]` type class พร้อม instances สำหรับ `List` และ `Option`

**ข้อ 4:** สร้าง phantom type สำหรับ HTTP request states: `Raw`, `Validated`, `Authenticated`, `Authorized`

**ข้อ 5:** Implement `Traverse[F[_]]` ที่ traverse F[A] with function A => G[B] เพื่อได้ G[F[B]]

---

➡️ ต่อไป: [Part 26 — Implicits and Given](part-26-implicits-and-given.md)
