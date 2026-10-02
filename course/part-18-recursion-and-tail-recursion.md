# Part 18: Recursion and Tail Recursion

## Steps 171-180: Recursion Patterns, @tailrec, Trampoline, Continuation-Passing Style

---

## Step 171: Recursion พื้นฐาน

```scala
import scala.annotation.tailrec

object RecursionBasics extends App {
  
  // Simple recursion - ระวัง stack overflow สำหรับ input ใหญ่
  def factorial(n: Int): Long = {
    if (n <= 0) 1L
    else n.toLong * factorial(n - 1)
  }
  
  def fibonacci(n: Int): Long = {
    if (n <= 1) n.toLong
    else fibonacci(n - 1) + fibonacci(n - 2)
  }
  
  // Sum of list
  def sum(list: List[Int]): Int = list match {
    case Nil     => 0
    case h :: t  => h + sum(t)
  }
  
  println("=== Basic Recursion ===")
  println(s"factorial(10) = ${factorial(10)}")
  println(s"fibonacci(15) = ${fibonacci(15)}")
  println(s"sum(1..10) = ${sum((1 to 10).toList)}")
  
  // Recursive tree structure
  sealed trait Tree[+A]
  case class Leaf[A](value: A) extends Tree[A]
  case class Branch[A](left: Tree[A], right: Tree[A]) extends Tree[A]
  
  def treeSum(tree: Tree[Int]): Int = tree match {
    case Leaf(v)        => v
    case Branch(l, r)   => treeSum(l) + treeSum(r)
  }
  
  def treeDepth[A](tree: Tree[A]): Int = tree match {
    case Leaf(_)       => 1
    case Branch(l, r)  => 1 + math.max(treeDepth(l), treeDepth(r))
  }
  
  def treeMap[A, B](tree: Tree[A])(f: A => B): Tree[B] = tree match {
    case Leaf(v)       => Leaf(f(v))
    case Branch(l, r)  => Branch(treeMap(l)(f), treeMap(r)(f))
  }
  
  val tree = Branch(
    Branch(Leaf(1), Leaf(2)),
    Branch(Leaf(3), Branch(Leaf(4), Leaf(5)))
  )
  
  println(s"\n=== Tree Recursion ===")
  println(s"tree sum: ${treeSum(tree)}")
  println(s"tree depth: ${treeDepth(tree)}")
  println(s"doubled: ${treeMap(tree)(_ * 2)}")
}
```

---

## Step 172: @tailrec — Tail-Recursive Optimization

```scala
import scala.annotation.tailrec

object TailRecursion extends App {
  
  // Regular recursion (NOT tail-recursive - builds stack frames)
  def factNormal(n: Int): Long = {
    if (n <= 0) 1L
    else n.toLong * factNormal(n - 1)  // Multiplication happens AFTER recursive call
  }
  
  // Tail-recursive with accumulator
  @tailrec  // Annotation ensures compiler optimizes this
  def factTail(n: Int, acc: Long = 1L): Long = {
    if (n <= 0) acc
    else factTail(n - 1, n.toLong * acc)  // Recursive call is the LAST operation
  }
  
  println("=== Tail Recursion ===")
  println(s"factNormal(10) = ${factNormal(10)}")
  println(s"factTail(10) = ${factTail(10)}")
  println(s"factTail(20) = ${factTail(20)}")
  
  // Tail-recursive sum
  @tailrec
  def sumTail(list: List[Int], acc: Int = 0): Int = list match {
    case Nil     => acc
    case h :: t  => sumTail(t, acc + h)
  }
  
  // Tail-recursive reverse
  @tailrec
  def reverseTail[A](list: List[A], acc: List[A] = List.empty): List[A] = list match {
    case Nil     => acc
    case h :: t  => reverseTail(t, h :: acc)
  }
  
  // Tail-recursive filter
  @tailrec
  def filterTail[A](list: List[A], pred: A => Boolean, acc: List[A] = List.empty): List[A] = 
    list match {
      case Nil     => acc.reverse
      case h :: t  => filterTail(t, pred, if (pred(h)) h :: acc else acc)
    }
  
  // Tail-recursive map
  @tailrec
  def mapTail[A, B](list: List[A], f: A => B, acc: List[B] = List.empty[B]): List[B] = 
    list match {
      case Nil     => acc.reverse
      case h :: t  => mapTail(t, f, f(h) :: acc)
    }
  
  val bigList = (1 to 100000).toList
  
  println(s"\n=== Tail-Recursive on Large Lists ===")
  println(s"sum(1..100000) = ${sumTail(bigList)}")
  println(s"First 5 reversed: ${reverseTail(bigList).take(5)}")
  println(s"Filter even count: ${filterTail(bigList, (n: Int) => n % 2 == 0).size}")
  
  // Fibonacci with tail recursion
  @tailrec
  def fibTail(n: Int, a: Long = 0L, b: Long = 1L): Long = {
    if (n == 0) a
    else fibTail(n - 1, b, a + b)
  }
  
  println(s"\nfib(0..15): ${(0 to 15).map(fibTail(_)).toList}")
  println(s"fib(90) = ${fibTail(90)}")
  
  // Greatest Common Divisor - naturally tail recursive
  @tailrec
  def gcd(a: Int, b: Int): Int = {
    if (b == 0) a
    else gcd(b, a % b)
  }
  
  println(s"\ngcd(48, 18) = ${gcd(48, 18)}")  // 6
  println(s"gcd(100, 75) = ${gcd(100, 75)}")  // 25
}
```

---

## Step 173: Mutual Recursion และ Continuation-Passing Style

```scala
import scala.annotation.tailrec

object AdvancedRecursion extends App {
  
  // Mutual recursion - can't use @tailrec directly
  def isEven(n: Int): Boolean = {
    if (n == 0) true
    else isOdd(n - 1)
  }
  
  def isOdd(n: Int): Boolean = {
    if (n == 0) false
    else isEven(n - 1)
  }
  
  println("=== Mutual Recursion ===")
  println((0 to 6).map(n => s"$n: even=${isEven(n)}, odd=${isOdd(n)}").mkString(", "))
  
  // Continuation-Passing Style (CPS) - enables tail recursion for any recursive function
  // Transform: result = f(x)  =>  f(x, k) where k is continuation
  
  // Normal factorial
  def factNormal(n: Int): Long = 
    if (n <= 0) 1L else n.toLong * factNormal(n - 1)
  
  // CPS factorial - tail recursive!
  @tailrec
  def factCPS(n: Int, k: Long => Long): Long = {
    if (n <= 0) k(1L)
    else factCPS(n - 1, acc => k(n.toLong * acc))
  }
  
  println(s"\n=== CPS Factorial ===")
  println(s"factNormal(10) = ${factNormal(10)}")
  println(s"factCPS(10) = ${factCPS(10, identity)}")
  
  // CPS Fibonacci
  def fibCPS(n: Int, k: Long => Long): Long = {
    if (n <= 1) k(n.toLong)
    else fibCPS(n - 1, a => fibCPS(n - 2, b => k(a + b)))
  }
  
  println(s"fibCPS(10) = ${fibCPS(10, identity)}")
  
  // Tree traversal with continuation
  sealed trait Tree[+A]
  case class Leaf[A](value: A) extends Tree[A]
  case class Branch[A](left: Tree[A], right: Tree[A]) extends Tree[A]
  
  // Accumulator-based (tail recursive for left spine)
  def toListAcc[A](tree: Tree[A]): List[A] = {
    @tailrec
    def go(remaining: List[Tree[A]], acc: List[A]): List[A] = remaining match {
      case Nil => acc.reverse
      case h :: t => h match {
        case Leaf(v)       => go(t, v :: acc)
        case Branch(l, r)  => go(l :: r :: t, acc)
      }
    }
    go(List(tree), List.empty)
  }
  
  val bigTree = (1 to 100).foldLeft[Tree[Int]](Leaf(0)) { (acc, n) =>
    Branch(acc, Leaf(n))
  }
  
  println(s"\n=== Stack-Safe Tree Traversal ===")
  val elements = toListAcc(bigTree)
  println(s"Tree size: ${elements.size}, sum: ${elements.sum}")
  
  // Recursive descent parser
  sealed trait Expr
  case class Num(n: Int) extends Expr
  case class Add(left: Expr, right: Expr) extends Expr
  case class Mul(left: Expr, right: Expr) extends Expr
  
  def eval(expr: Expr): Int = expr match {
    case Num(n)      => n
    case Add(l, r)   => eval(l) + eval(r)
    case Mul(l, r)   => eval(l) * eval(r)
  }
  
  // (3 + 4) * (2 + 5)
  val expr = Mul(Add(Num(3), Num(4)), Add(Num(2), Num(5)))
  println(s"\n=== Expression Evaluation ===")
  println(s"(3+4)*(2+5) = ${eval(expr)}")  // 49
  
  // Stack-safe eval using work stack
  def evalSafe(expr: Expr): Int = {
    sealed trait Op
    case class EvalRight(right: Expr, combine: (Int, Int) => Int) extends Op
    case class CombineWith(left: Int, combine: (Int, Int) => Int) extends Op
    
    @tailrec
    def go(current: Expr, stack: List[Op]): Int = current match {
      case Num(n) => stack match {
        case Nil => n
        case EvalRight(right, f) :: rest => go(right, CombineWith(n, f) :: rest)
        case CombineWith(left, f) :: rest => go(Num(f(left, n)), rest)
      }
      case Add(l, r) => go(l, EvalRight(r, _ + _) :: stack)
      case Mul(l, r) => go(l, EvalRight(r, _ * _) :: stack)
    }
    go(expr, List.empty)
  }
  
  println(s"evalSafe (3+4)*(2+5) = ${evalSafe(expr)}")
}
```

---

## Step 174: Trampoline Pattern

```scala
// Trampoline - bounce between calls without growing the stack
object TrampolinePattern extends App {
  
  sealed trait Bounce[+A] {
    final def run: A = {
      @tailrec
      def loop(current: Bounce[A]): A = current match {
        case Done(result) => result
        case Continue(thunk) => loop(thunk())
        case FlatMap(sub, f) => sub match {
          case Done(result) => loop(f(result))
          case Continue(thunk) => loop(FlatMap(thunk(), f))
          case FlatMap(sub2, f2) => loop(FlatMap(sub2, (x: Any) => FlatMap(f2(x), f)))
        }
      }
      loop(this)
    }
  }
  
  case class Done[A](result: A) extends Bounce[A]
  case class Continue[A](thunk: () => Bounce[A]) extends Bounce[A]
  case class FlatMap[A, B](sub: Bounce[A], f: A => Bounce[B]) extends Bounce[B]
  
  object Bounce {
    def pure[A](a: A): Bounce[A] = Done(a)
    def suspend[A](f: => Bounce[A]): Bounce[A] = Continue(() => f)
    
    implicit class BounceOps[A](bounce: Bounce[A]) {
      def flatMap[B](f: A => Bounce[B]): Bounce[B] = FlatMap(bounce, f)
      def map[B](f: A => B): Bounce[B] = FlatMap(bounce, a => Done(f(a)))
    }
  }
  
  import Bounce._
  
  // Stack-safe factorial using trampoline
  def factorial(n: BigInt, acc: BigInt = 1): Bounce[BigInt] = {
    if (n <= 0) Done(acc)
    else Continue(() => factorial(n - 1, n * acc))
  }
  
  // Stack-safe even/odd (mutual recursion)
  def isEven(n: Long): Bounce[Boolean] = {
    if (n == 0L) Done(true)
    else Continue(() => isOdd(n - 1))
  }
  
  def isOdd(n: Long): Bounce[Boolean] = {
    if (n == 0L) Done(false)
    else Continue(() => isEven(n - 1))
  }
  
  println("=== Trampoline ===")
  println(s"factorial(20) = ${factorial(20).run}")
  println(s"factorial(100) = ${factorial(100).run}")
  
  println(s"\nisEven(1000) = ${isEven(1000L).run}")
  println(s"isOdd(999) = ${isOdd(999L).run}")
  println(s"isEven(100000) = ${isEven(100000L).run}")  // Would overflow without trampoline
  
  // Fibonacci using trampoline
  def fib(n: Int): Bounce[BigInt] = {
    def go(n: Int, a: BigInt, b: BigInt): Bounce[BigInt] = {
      if (n == 0) Done(a)
      else Continue(() => go(n - 1, b, a + b))
    }
    go(n, 0, 1)
  }
  
  println(s"\nfib(100) = ${fib(100).run}")
  println(s"fib(1000) starts with: ${fib(1000).run.toString.take(20)}...")
}
```

---

## Step 175: Real-World Recursive Algorithms

```scala
import scala.annotation.tailrec

object RealWorldRecursion extends App {
  
  // Merge Sort
  def mergeSort[A: Ordering](list: List[A]): List[A] = {
    val ord = implicitly[Ordering[A]]
    
    def merge(left: List[A], right: List[A]): List[A] = (left, right) match {
      case (Nil, r) => r
      case (l, Nil) => l
      case (lh :: lt, rh :: rt) =>
        if (ord.lteq(lh, rh)) lh :: merge(lt, right)
        else rh :: merge(left, rt)
    }
    
    if (list.length <= 1) list
    else {
      val (left, right) = list.splitAt(list.length / 2)
      merge(mergeSort(left), mergeSort(right))
    }
  }
  
  println("=== Merge Sort ===")
  val unsorted = List(3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5)
  println(s"unsorted: $unsorted")
  println(s"sorted: ${mergeSort(unsorted)}")
  
  val strings = List("banana", "apple", "cherry", "date")
  println(s"strings sorted: ${mergeSort(strings)}")
  
  // Quick Sort (functional style)
  def quickSort[A: Ordering](list: List[A]): List[A] = {
    val ord = implicitly[Ordering[A]]
    list match {
      case Nil | _ :: Nil => list
      case pivot :: rest =>
        val smaller = rest.filter(x => ord.lt(x, pivot))
        val equal = rest.filter(x => ord.equiv(x, pivot)) :+ pivot
        val larger = rest.filter(x => ord.gt(x, pivot))
        quickSort(smaller) ++ equal ++ quickSort(larger)
    }
  }
  
  println(s"\n=== Quick Sort ===")
  println(s"quick sorted: ${quickSort(unsorted)}")
  
  // Binary Search (recursive)
  @tailrec
  def binarySearch[A: Ordering](
    arr: Vector[A], 
    target: A, 
    low: Int = 0, 
    high: Int = -1
  ): Option[Int] = {
    val ord = implicitly[Ordering[A]]
    val h = if (high < 0) arr.length - 1 else high
    
    if (low > h) None
    else {
      val mid = low + (h - low) / 2
      val midVal = arr(mid)
      
      if (ord.equiv(midVal, target)) Some(mid)
      else if (ord.lt(midVal, target)) binarySearch(arr, target, mid + 1, h)
      else binarySearch(arr, target, low, mid - 1)
    }
  }
  
  println("\n=== Binary Search ===")
  val sorted = Vector(1, 3, 5, 7, 9, 11, 13, 15, 17, 19)
  println(s"search 7 in $sorted = ${binarySearch(sorted, 7)}")  // Some(3)
  println(s"search 10 = ${binarySearch(sorted, 10)}")           // None
  println(s"search 1 = ${binarySearch(sorted, 1)}")             // Some(0)
  println(s"search 19 = ${binarySearch(sorted, 19)}")           // Some(9)
  
  // Flatten nested lists
  def flatten[A](nested: List[Any]): List[A] = nested match {
    case Nil => List.empty
    case (h: List[_]) :: t => flatten[A](h) ++ flatten[A](t)
    case (h: A @unchecked) :: t => h :: flatten[A](t)
  }
  
  println("\n=== Flatten ===")
  val nested = List(1, List(2, 3, List(4, 5)), 6, List(7, List(8)))
  println(s"flattened: ${flatten[Int](nested)}")
  
  // Power set
  def powerSet[A](set: Set[A]): Set[Set[A]] = {
    if (set.isEmpty) Set(Set.empty)
    else {
      val elem = set.head
      val rest = powerSet(set.tail)
      rest ++ rest.map(_ + elem)
    }
  }
  
  println("\n=== Power Set ===")
  val s = Set(1, 2, 3)
  println(s"powerSet($s) = ${powerSet(s)}")
  println(s"size = ${powerSet(s).size}")  // 2^3 = 8
  
  // Permutations
  def permutations[A](list: List[A]): List[List[A]] = list match {
    case Nil | _ :: Nil => List(list)
    case _ =>
      for {
        elem <- list
        rest <- permutations(list.filterNot(_ == elem))
      } yield elem :: rest
  }
  
  println("\n=== Permutations ===")
  println(s"perms(1,2,3): ${permutations(List(1, 2, 3))}")
}
```

---

## สรุป Part 18

| Pattern | เมื่อไหรใช้ | Complexity |
|---------|------------|-----------|
| Simple recursion | ปัญหาขนาดเล็ก | Risk: stack overflow |
| @tailrec | Tail position recursion | O(1) stack |
| Accumulator | Non-tail → tail | เพิ่ม acc parameter |
| Mutual recursion | 2+ functions เรียกกัน | ระวัง stack |
| CPS | ทำให้ tail-recursive ได้เสมอ | Harder to read |
| Trampoline | Stack-safe mutual recursion | Runtime overhead |
| Work stack | Tree traversal | Complex but efficient |

---

## แบบฝึกหัด Part 18

**ข้อ 1:** Implement `@tailrec` flatten สำหรับ `List[List[A]]` → `List[A]`

**ข้อ 2:** สร้าง tail-recursive JSON-like tree printer ที่ print structured output

**ข้อ 3:** Implement Towers of Hanoi recursively แล้วแปลงเป็น tail-recursive

**ข้อ 4:** สร้าง stack-safe `map` และ `flatMap` สำหรับ tree structure โดยใช้ work list

**ข้อ 5:** Implement Ackermann function ด้วย Trampoline (Ackermann grows so fast it needs stack protection)

---

➡️ ต่อไป: [Part 19 — For Comprehensions](part-19-for-comprehensions.md)
