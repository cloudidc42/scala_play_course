# Part 99: Open Source Contributions — Steps 981-990

## บทนำ: Open Source in Scala

การมีส่วนร่วมกับ Scala open source community — ตั้งแต่ reporting bugs ไปจนถึง submitting SIPs, publishing libraries บน Sonatype/Maven Central

---

## Step 981: OSS Contribution Workflow

```bash
#!/bin/bash
# oss-contribution-workflow.sh

# ===== Standard workflow for Scala OSS projects =====

# 1. Fork the repository on GitHub
# Go to: https://github.com/scala/scala → Fork

# 2. Clone YOUR fork
git clone https://github.com/YOUR_USERNAME/scala.git
cd scala

# 3. Add upstream remote
git remote add upstream https://github.com/scala/scala.git

# 4. Create feature branch
git checkout -b fix/issue-12345-list-performance

# 5. Make changes following project code style
# Check CONTRIBUTING.md for specific guidelines

# 6. Run tests
sbt test
sbt "testOnly *ListSpec*"

# 7. Check formatting (most projects use scalafmt)
sbt scalafmt
sbt scalafmtCheck

# 8. Commit with descriptive message
git commit -m "Fix: improve List.collect performance for large lists

Fixes #12345

Previous implementation created intermediate List for each collect call.
New implementation uses a single pass with direct accumulation.

Benchmarks show 3x improvement for List of 100k elements:
- Before: 523 us/op
- After:  178 us/op"

# 9. Sync with upstream before pushing
git fetch upstream
git rebase upstream/main

# 10. Push and create PR
git push origin fix/issue-12345-list-performance
# Then go to GitHub and create Pull Request
```

---

## Step 982: Finding Good First Issues

```scala
// FindingIssues.scala

/*
===== Where to find Scala OSS contribution opportunities =====

1. Scala Standard Library:
   https://github.com/scala/scala/issues?q=label%3A"good+first+issue"

2. Cats / Cats Effect:
   https://github.com/typelevel/cats/issues?q=label%3A"good+first+issue"

3. ZIO:
   https://github.com/zio/zio/issues?q=label%3A"good+first+issue"

4. Akka:
   https://github.com/akka/akka/issues?q=label%3A"help+wanted"

5. Spark:
   https://issues.apache.org/jira/browse/SPARK?q=labels+%3D+%22starter%22

6. Play Framework:
   https://github.com/playframework/playframework/issues

===== Types of Contributions =====

Small (good for beginners):
  - Fix typos in documentation
  - Improve error messages
  - Add missing ScalaDoc
  - Write missing tests
  - Fix formatting issues

Medium:
  - Bug fixes with tests
  - Performance improvements
  - New utility methods
  - Improve binary compatibility

Large:
  - New features (requires design discussion)
  - Breaking changes (requires SIP)
  - Major refactors
*/

// ===== Reading source code strategy =====
object ReadingOSSCode {
  
  // 1. Start from tests (they show intended behavior)
  // 2. Read the trait/class definition
  // 3. Trace the implementation
  // 4. Check git blame for context
  // 5. Look at related issues/PRs
  
  def cloneAndExplore(): Unit = {
    // Commands to understand a project:
    // git log --oneline --graph --all | head -20   → recent history
    // git log --author="Martin Odersky" --oneline  → specific contributor
    // git log --all --full-history -- "src/library/scala/collection/immutable/List.scala"  → file history
    // git bisect start + git bisect good/bad        → find regression
    println("Use git tools to understand project history")
  }
}
```

---

## Step 983: Writing Quality Code for PRs

```scala
// QualityCodeForPR.scala

/*
===== What reviewers look for =====

1. Correctness: Does it solve the reported issue?
2. Tests: Is there adequate test coverage?
3. Performance: No regressions?
4. Backwards compatibility: Binary compatible?
5. Documentation: ScalaDoc updated?
6. Code style: Follows project conventions?
*/

// ===== Good ScalaDoc =====
object ScalaDocExample {
  
  /**
   * Finds the first element satisfying the predicate, or `None` if none exists.
   *
   * @note This is equivalent to `filter(p).headOption` but more efficient
   *       as it short-circuits on the first match.
   * @param p The predicate to test each element against
   * @tparam A The element type
   * @return `Some(element)` if found, `None` otherwise
   * @example
   * {{{
   * List(1, 2, 3, 4, 5).findFirst(_ > 3) // Returns Some(4)
   * List(1, 2, 3).findFirst(_ > 10)       // Returns None
   * }}}
   *
   * @since 2.14.0
   */
  def findFirst[A](xs: List[A])(p: A => Boolean): Option[A] = xs match {
    case Nil     => None
    case h :: t  => if (p(h)) Some(h) else findFirst(t)(p)
  }
}

// ===== Complete test coverage =====
class FindFirstSpec extends org.scalatest.flatspec.AnyFlatSpec
    with org.scalatest.matchers.should.Matchers {
  
  "findFirst" should "return first matching element" in {
    ScalaDocExample.findFirst(List(1, 2, 3, 4))(_ > 2) shouldBe Some(3)
  }
  
  it should "return None when no match" in {
    ScalaDocExample.findFirst(List(1, 2, 3))(_ > 10) shouldBe None
  }
  
  it should "work on empty list" in {
    ScalaDocExample.findFirst(List.empty[Int])(_ > 0) shouldBe None
  }
  
  it should "return first not second match" in {
    ScalaDocExample.findFirst(List(3, 4, 5))(_ > 2) shouldBe Some(3)
  }
  
  it should "be equivalent to filter.headOption" in {
    val list = List(1, 2, 3, 4, 5)
    val p = (_: Int) > 3
    ScalaDocExample.findFirst(list)(p) shouldBe list.filter(p).headOption
  }
}
```

---

## Step 984: Scala Improvement Process (SIP)

```markdown
# SIP Template: SIP-XX — [Feature Name]

## Author
Jane Doe <jane@example.com>

## Summary
[One paragraph summary of the proposal]

## Motivation
[Why is this needed? What problem does it solve?]

Example of current pain point:
```scala
// Current: verbose
val result = xs.map(f).filter(g).headOption.getOrElse(default)

// Proposed: concise
val result = xs.findMap(x => Option.when(g(x))(f(x))).getOrElse(default)
```

## Proposed Solution
[Detailed description with examples]

```scala
// New method on collection
extension [A, B](xs: IterableOnce[A]) {
  def findMap[B](f: A => Option[B]): Option[B]
}
```

## Compatibility
- [x] Backwards compatible (additive only)
- [x] Source compatible
- [x] Binary compatible
- [ ] Requires language change

## Alternatives Considered
1. Adding to standard library without syntax change (rejected: too verbose)
2. Implementing via macro (rejected: complexity)

## Open Questions
1. Should this be in `IterableOnce` or `Iterable`?
2. Name: `findMap` vs `collectFirst` vs `mapFind`?

## References
- [scala-contributors discussion](...)
- [Related PR](...)
```

---

## Step 985: Publishing to Maven Central

```scala
// build.sbt — Library publication setup

name := "scala-json-validator"
organization := "io.github.myusername"
version := "1.0.0"
scalaVersion := "3.3.3"

crossScalaVersions := Seq("2.13.14", "3.3.3")

// Publication info
description := "Type-safe JSON validation for Scala"
homepage := Some(url("https://github.com/myusername/scala-json-validator"))
licenses := Seq("Apache-2.0" -> url("https://www.apache.org/licenses/LICENSE-2.0"))

// Developer info (required by Maven Central)
developers := List(
  Developer(
    id    = "myusername",
    name  = "My Name",
    email = "me@example.com",
    url   = url("https://github.com/myusername")
  )
)

// SCM (required by Maven Central)
scmInfo := Some(ScmInfo(
  url("https://github.com/myusername/scala-json-validator"),
  "scm:git@github.com:myusername/scala-json-validator.git"
))

// Sonatype publishing
publishTo := sonatypePublishToBundle.value
sonatypeCredentialHost := "s01.oss.sonatype.org"

// Sign artifacts (GPG required)
// gpgWkdir := file("/home/user/.gnupg")

// Exclude test artifacts
publishArtifact in Test := false
```

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
          gpg-passphrase: GPG_PASSPHRASE
      
      - name: Import GPG key
        env:
          GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
        run: |
          echo "${{ secrets.GPG_PRIVATE_KEY }}" | gpg --batch --import
      
      - name: Publish to Maven Central
        env:
          SONATYPE_USERNAME: ${{ secrets.SONATYPE_USERNAME }}
          SONATYPE_PASSWORD: ${{ secrets.SONATYPE_PASSWORD }}
          GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
        run: |
          sbt +publishSigned
          sbt sonatypeBundleRelease
```

---

## Step 986: Versioning & Binary Compatibility

```scala
// BinaryCompatibility.scala

/*
===== Semantic Versioning for Scala Libraries =====

MAJOR.MINOR.PATCH (1.2.3)

MAJOR: Breaking changes (binary incompatible)
  - Remove public method
  - Change method signature
  - Change trait/class hierarchy

MINOR: New features, backwards compatible
  - Add new public method
  - Add new parameter with default value
  - New class/trait

PATCH: Bug fixes
  - Fix incorrect behavior
  - Performance improvement
  - Documentation

===== Binary Compatibility (MiMa) =====
MiMa = Migration Manager: detects binary incompatibilities
*/

// build.sbt
/*
import com.typesafe.tools.mima.plugin.MimaPlugin.autoImport._

mimaPreviousArtifacts := Set("io.github.myusername" %% "scala-json-validator" % "1.0.0")

// Report binary incompatibilities (fail build on violation)
mimaFailOnProblem := true

// Ignore specific methods if intentionally changed (with MAJOR bump)
mimaBinaryIssueFilters ++= Seq(
  ProblemFilters.exclude[MissingMethodProblem]("io.github.myusername.Validator.old_method"),
)
*/

// ===== What breaks binary compatibility? =====
// BEFORE (1.0.0)
class Validator {
  def validate(input: String): Boolean = input.nonEmpty
}

// AFTER — BINARY INCOMPATIBLE (adding parameter breaks existing callers)
// class Validator {
//   def validate(input: String, strict: Boolean = false): Boolean = ...
// }

// AFTER — BINARY COMPATIBLE (new overload)
class ValidatorV2 {
  def validate(input: String): Boolean = input.nonEmpty
  def validate(input: String, strict: Boolean): Boolean = validate(input)
}
```

---

## Step 987: Writing a Scala Library

```scala
// Example library: scala-result

// Core types
sealed trait Result[+E, +A] {
  def map[B](f: A => B): Result[E, B]
  def flatMap[EE >: E, B](f: A => Result[EE, B]): Result[EE, B]
  def fold[B](onError: E => B, onSuccess: A => B): B
  def isSuccess: Boolean
  def isError: Boolean
}

final case class Success[+A](value: A) extends Result[Nothing, A] {
  def map[B](f: A => B): Result[Nothing, B] = Success(f(value))
  def flatMap[EE, B](f: A => Result[EE, B]): Result[EE, B] = f(value)
  def fold[B](onError: Nothing => B, onSuccess: A => B): B = onSuccess(value)
  val isSuccess = true
  val isError   = false
}

final case class Failure[+E](error: E) extends Result[E, Nothing] {
  def map[B](f: Nothing => B): Result[E, Nothing] = this
  def flatMap[EE >: E, B](f: Nothing => Result[EE, B]): Result[EE, B] = this
  def fold[B](onError: E => B, onSuccess: Nothing => B): B = onError(error)
  val isSuccess = false
  val isError   = true
}

object Result {
  def success[A](value: A): Result[Nothing, A] = Success(value)
  def failure[E](error: E): Result[E, Nothing] = Failure(error)
  
  def fromTry[A](t: scala.util.Try[A]): Result[Throwable, A] = t match {
    case scala.util.Success(v) => Success(v)
    case scala.util.Failure(e) => Failure(e)
  }
  
  def fromOption[E, A](opt: Option[A], ifNone: => E): Result[E, A] = opt match {
    case Some(v) => Success(v)
    case None    => Failure(ifNone)
  }
}

// Usage example
object ResultDemo {
  def parseAge(s: String): Result[String, Int] =
    scala.util.Try(s.toInt)
      .toOption
      .flatMap(n => if (n >= 0 && n <= 150) Some(n) else None)
      .fold[Result[String, Int]](Failure(s"Invalid age: $s"))(Success(_))
  
  def demo(): Unit = {
    parseAge("25").fold(
      err => println(s"Error: $err"),
      age => println(s"Age: $age")
    )
  }
}
```

---

## Step 988: Contributing to Spark

```scala
// SparkContribution.scala

/*
===== Spark Contribution Process =====

1. Browse JIRA: https://issues.apache.org/jira/browse/SPARK
2. Find "starter" or "good first contribution" issues
3. Comment: "I'd like to work on this"
4. Fork: https://github.com/apache/spark

===== Build & Test =====
./build/sbt "sql/test"           # SQL module tests
./build/sbt "core/test"          # Core tests
./build/sbt scalastyle           # Style check

===== PR Requirements =====
- Tests for new functionality
- Documentation (for user-facing changes)
- No new Checkstyle violations
- PR title: [SPARK-XXXXX][MODULE] Description
*/

// Example: Add new DataFrame method
// (Simplified for illustration)
object DataFrameExtensions {
  import org.apache.spark.sql.{DataFrame, Column}
  import org.apache.spark.sql.functions._
  
  implicit class DataFrameOps(df: DataFrame) {
    
    /**
     * Adds percentage column for a numeric column.
     *
     * @param valueCol The column to compute percentages for
     * @param newColName Name for the new percentage column
     */
    def withPercent(valueCol: String, newColName: String = s"${valueCol}_pct"): DataFrame = {
      val total = df.agg(sum(valueCol)).collect()(0).getDouble(0)
      df.withColumn(newColName, col(valueCol) / total * 100)
    }
  }
}
```

---

## Step 989: Scala Community

```markdown
# Scala Community Channels

## Discussion
- **Discord**: https://discord.com/invite/scala (most active)
- **Discourse**: https://users.scala-lang.org (long-form discussions)
- **Reddit**: r/scala
- **Twitter/X**: #Scala

## Conferences
- **Scala Days** (annual, EU/US alternating)
- **Functional Scala** (London, December)
- **ScalaCon** (online)
- **Typelevel Summit** (alongside ScalaDays)

## Learning Resources
- **Scala Book**: https://docs.scala-lang.org/scala3/book/introduction.html
- **Coursera**: "Functional Programming in Scala" (Martin Odersky)
- **Rock the JVM**: https://rockthejvm.com
- **Alvin Alexander**: https://alvinalexander.com/scala/

## Contributing to Docs
- Scala docs: https://github.com/scala/docs.scala-lang
- Very welcoming to first-time contributors
- Fix typos, add examples, translate content

## Important Projects to Know
| Project | GitHub | Focus |
|---------|--------|-------|
| Scala 3 | scala/scala3 | Language |
| Cats | typelevel/cats | FP library |
| ZIO | zio/zio | Effect system |
| Akka | akka/akka | Actor model |
| Spark | apache/spark | Big data |
| http4s | http4s/http4s | HTTP |
| Doobie | tpolecat/doobie | DB |
```

---

## Step 990: Building Your OSS Profile

```scala
// BuildingOSSProfile.scala

/*
===== Steps to build an OSS profile =====

1. Start with documentation fixes (low barrier, high value)
   - Fix typos, improve clarity
   - Add missing examples
   
2. Move to small bug fixes
   - Pick "good first issue" with failing test
   - Write fix + additional tests
   
3. Work on features
   - Discuss design in issue before coding
   - Get feedback early with draft PR
   
4. Maintain your own library
   - Solve a problem you have
   - Publish to Maven Central
   - Write blog post about it
   
5. Review others' PRs
   - Great way to learn codebase
   - Shows you understand the code
   
6. Speak at meetups/conferences
   - Start with local Scala meetup
   - Submit to Scala Days
   
===== GitHub Profile Optimization =====

- Pin your best Scala repositories
- Keep README up-to-date with badges:
  - Build status (GitHub Actions)
  - Coverage (Codecov/Coveralls)
  - Maven Central version
  - Scaladoc link
  
- Write good commit messages (explains WHY, not WHAT)
- Respond promptly to issues/PRs

===== OSS README Template =====
*/

// README.md template for Scala library
/*
# scala-result

[![Build Status](https://github.com/myusername/scala-result/workflows/CI/badge.svg)](...)
[![Maven Central](https://img.shields.io/maven-central/v/io.github.myusername/scala-result.svg)](...)
[![Scaladoc](https://javadoc.io/badge2/io.github.myusername/scala-result_2.13/scaladoc.svg)](...)
[![License](https://img.shields.io/badge/license-Apache2-blue.svg)](LICENSE)

Type-safe result type for Scala, inspired by Rust's `Result<T, E>`.

## Installation

```sbt
libraryDependencies += "io.github.myusername" %% "scala-result" % "1.0.0"
```

## Quick Start

```scala
import myusername.result._

def parseAge(s: String): Result[String, Int] = ...

parseAge("25") match {
  case Success(age) => println(s"Age: $age")
  case Failure(err) => println(s"Error: $err")
}
```

## Contributing
See [CONTRIBUTING.md](CONTRIBUTING.md)
*/
```

---

## สรุป Part 99: Open Source Contributions

| Activity | Effort | Impact |
|----------|--------|--------|
| Fix documentation | Low | Medium |
| Bug fix with test | Medium | High |
| New feature | High | High |
| Own library | Very High | Very High |
| Community involvement | Ongoing | Career |

---

## แบบฝึกหัด Part 99

1. **First Contribution**: หา 1 issue ใน Scala OSS project (cats, zio, akka) ที่ label "good first issue" และ submit PR สำหรับ documentation improvement หรือ small bug fix

2. **Publish Library**: สร้าง Scala library เล็กๆ (เช่น utility functions ที่คุณใช้บ่อย), setup CI/CD ด้วย GitHub Actions, publish ไปยัง Maven Central

3. **MiMa Check**: setup MiMa ใน build.sbt, simulate binary-breaking change, verify MiMa detects it

4. **ScalaDoc**: เขียน complete ScalaDoc สำหรับ library ที่สร้างใน exercise 2 พร้อม @param, @return, @throws, @example

5. **SIP Draft**: เขียน draft SIP สำหรับ feature ที่คุณอยาก add ใน Scala standard library (ไม่ต้อง submit จริง) ตาม SIP template

---

## ไปต่อ: Part 100 — World-Class Practices
[→ Part 100: World-Class Practices](./part-100-world-class-practices.md)
