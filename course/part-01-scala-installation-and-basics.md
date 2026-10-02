# Part 01 — ติดตั้ง Scala และเริ่มต้นเขียนโปรแกรม
## Steps 1–10: จาก Zero สู่ Hello World

> **เป้าหมาย**: ติดตั้ง Scala, รู้จัก REPL, และเขียนโปรแกรมแรก

---

## Step 1 — ทำความรู้จักกับ Scala

Scala ย่อมาจาก **Scalable Language** — ภาษาที่ออกแบบมาให้เติบโตตามความต้องการของโปรแกรม ตั้งแต่ script เล็กๆ ไปจนถึงระบบ distributed ขนาดใหญ่

### ทำไมต้องเรียน Scala?

```
ภาษา Java        → Verbose, boilerplate มาก
ภาษา Python      → ช้า, dynamic typing
ภาษา Scala       → กระชับ, type-safe, fast, functional + OOP
```

### จุดเด่นของ Scala

1. **Type Safety** — จับ bug ตั้งแต่ compile time
2. **Functional Programming** — immutable, pure functions
3. **JVM Ecosystem** — ใช้ library Java ทั้งหมดได้
4. **Concurrency** — Akka, Futures, coroutines
5. **Big Data** — Apache Spark เขียนด้วย Scala
6. **Play Framework** — Web framework ที่ทรงพลัง

### บริษัทที่ใช้ Scala

- **Twitter** — backend infrastructure
- **LinkedIn** — data pipeline
- **Netflix** — data processing
- **Airbnb** — data platform
- **Databricks** — Apache Spark (creators)
- **Goldman Sachs** — trading systems

---

## Step 2 — ติดตั้ง JDK

Scala ทำงานบน JVM จึงต้องติดตั้ง JDK ก่อน

### macOS

```bash
# ใช้ Homebrew
brew install openjdk@17

# เพิ่ม PATH
echo 'export PATH="/opt/homebrew/opt/openjdk@17/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# ตรวจสอบ
java -version
# openjdk version "17.x.x" ...
```

### Linux (Ubuntu/Debian)

```bash
# อัปเดต package list
sudo apt update

# ติดตั้ง JDK 17
sudo apt install -y openjdk-17-jdk

# ตรวจสอบ
java -version
javac -version
```

### Linux (Fedora/RHEL)

```bash
sudo dnf install java-17-openjdk-devel
java -version
```

### Windows

1. ดาวน์โหลด [Adoptium JDK 17](https://adoptium.net/)
2. รัน installer และกด Next ตลอด
3. เปิด Command Prompt และพิมพ์:

```cmd
java -version
```

### ใช้ SDKMAN (แนะนำสำหรับ macOS/Linux)

```bash
# ติดตั้ง SDKMAN
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

# ติดตั้ง JDK ผ่าน SDKMAN
sdk install java 17.0.9-tem

# ดู versions ที่มี
sdk list java

# สลับ version
sdk use java 17.0.9-tem
```

---

## Step 3 — ติดตั้ง Scala

### วิธีที่ 1: Coursier (แนะนำ)

Coursier เป็น Scala artifact fetcher ที่เร็วและทันสมัย

```bash
# macOS
brew install coursier/formulas/coursier && cs setup

# Linux
curl -fL https://github.com/coursier/launchers/raw/master/cs-x86_64-pc-linux.gz | gzip -d > cs
chmod +x cs
./cs setup
```

หลัง setup:

```bash
# ตรวจสอบ Scala
scala -version
# Scala code runner version 3.x.x -- Copyright 2002-2024, LAMP/EPFL

scalac -version
# Scala compiler version 3.x.x -- Copyright 2002-2024, LAMP/EPFL
```

### วิธีที่ 2: SDKMAN

```bash
sdk install scala 3.3.1
scala -version
```

### วิธีที่ 3: Homebrew (macOS)

```bash
brew install scala
```

### ตรวจสอบการติดตั้ง

```bash
which scala
which scalac
scala -version
```

---

## Step 4 — ติดตั้ง SBT (Simple Build Tool)

SBT คือ build tool หลักของ Scala (เหมือน Maven/Gradle ของ Java)

### macOS

```bash
brew install sbt
sbt -version
# sbt version in this project: ...
# sbt script version: 1.x.x
```

### Linux (Ubuntu/Debian)

```bash
echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" | sudo tee /etc/apt/sources.list.d/sbt.list
curl -sL "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x2EE0EA64E40A89B84B2DF73499E82A75642AC823" | sudo apt-key add
sudo apt update
sudo apt install sbt
```

### Windows

1. ดาวน์โหลด [sbt installer](https://www.scala-sbt.org/download.html)
2. รัน .msi installer
3. เปิด Command Prompt ใหม่

```cmd
sbt -version
```

---

## Step 5 — ติดตั้ง IDE

### IntelliJ IDEA (แนะนำ)

1. ดาวน์โหลด [IntelliJ IDEA Community](https://www.jetbrains.com/idea/download/) (ฟรี)
2. ติดตั้งและเปิดโปรแกรม
3. ไปที่ **Plugins** → ค้นหา **Scala** → Install
4. Restart IDE

### VS Code + Metals

```bash
# ติดตั้ง VS Code
# ติดตั้ง extension "Scala (Metals)" จาก marketplace

# หรือผ่าน command line
code --install-extension scalameta.metals
```

### Neovim + Metals (สำหรับ advanced users)

```lua
-- ใน init.lua
require('lspconfig').metals.setup{}
```

---

## Step 6 — Scala REPL เบื้องต้น

REPL ย่อจาก **Read-Eval-Print Loop** — เหมือน Python shell สำหรับ Scala

```bash
# เปิด REPL
scala
```

```
Welcome to Scala 3.3.1 (17.0.9, Java OpenJDK 64-Bit Server VM).
Type in expressions for evaluation. Or try :help.

scala>
```

### คำสั่งพื้นฐานใน REPL

```scala
// พิมพ์ Hello World
scala> println("Hello, World!")
Hello, World!

// คำนวณ
scala> 1 + 1
val res0: Int = 2

scala> 10 * 3.14
val res1: Double = 31.400000000000002

// กำหนดตัวแปร
scala> val name = "Scala"
val name: String = Scala

scala> val version = 3
val version: Int = 3

scala> s"Hello, $name $version!"
val res2: String = Hello, Scala 3!
```

### คำสั่ง REPL ที่ควรรู้

```
:help      — แสดงความช่วยเหลือ
:quit      — ออกจาก REPL
:type      — แสดง type ของ expression
:paste     — paste หลายบรรทัด (กด Ctrl+D เพื่อจบ)
:history   — ดูประวัติคำสั่ง
:reset     — รีเซ็ต REPL
```

```scala
// :type ตรวจสอบ type
scala> :type 42
Int

scala> :type "hello"
String

scala> :type List(1, 2, 3)
List[Int]
```

---

## Step 7 — Hello World โปรแกรมแรก

### สร้างไฟล์ Hello.scala

```bash
mkdir ~/scala-learning
cd ~/scala-learning
```

```scala
// Hello.scala
@main def hello(): Unit =
  println("Hello, World!")
  println("ยินดีต้อนรับสู่โลก Scala!")
```

### Compile และ Run

```bash
# Compile
scalac Hello.scala

# Run
scala hello

# หรือ run โดยตรง (Scala 3)
scala Hello.scala
```

**Output:**
```
Hello, World!
ยินดีต้อนรับสู่โลก Scala!
```

### Scala 2 Style (Object-based)

```scala
// HelloScala2.scala
object HelloScala2 {
  def main(args: Array[String]): Unit = {
    println("Hello from Scala 2 style!")
  }
}
```

```bash
scalac HelloScala2.scala
scala HelloScala2
```

### Scala 3 Style (Modern — แนะนำ)

```scala
// HelloScala3.scala
@main def main(): Unit = {
  println("Hello from Scala 3!")
}
```

```bash
scala HelloScala3.scala
```

---

## Step 8 — สร้างโปรเจกต์แรกด้วย SBT

### สร้างโปรเจกต์

```bash
# สร้าง directory
mkdir my-first-scala-project
cd my-first-scala-project

# สร้างโปรเจกต์ด้วย sbt template
sbt new scala/scala3.g8
```

ระบบจะถามชื่อโปรเจกต์:
```
name [Scala 3 Project Template]: my-first-project
```

### โครงสร้างโปรเจกต์

```
my-first-project/
├── build.sbt          ← Build definition
├── project/
│   ├── build.properties
│   └── plugins.sbt
└── src/
    ├── main/
    │   └── scala/
    │       └── Main.scala
    └── test/
        └── scala/
            └── MySuite.scala
```

### build.sbt

```scala
// build.sbt
val scala3Version = "3.3.1"

lazy val root = project
  .in(file("."))
  .settings(
    name         := "my-first-project",
    version      := "0.1.0-SNAPSHOT",
    scalaVersion := scala3Version,

    libraryDependencies += "org.scalameta" %% "munit" % "0.7.29" % Test
  )
```

### Main.scala

```scala
// src/main/scala/Main.scala
@main def main(): Unit =
  println("Hello from SBT project!")
  
  // ลองใช้ features ต่างๆ
  val numbers = List(1, 2, 3, 4, 5)
  val doubled = numbers.map(_ * 2)
  println(s"Doubled: $doubled")
  
  val sum = numbers.sum
  println(s"Sum: $sum")
```

### รัน SBT

```bash
# เข้า SBT shell
sbt

# ใน SBT shell
sbt:my-first-project> run
sbt:my-first-project> compile
sbt:my-first-project> test
sbt:my-first-project> clean

# หรือรันตรงๆ
sbt run
sbt compile
sbt test
```

**Output:**
```
[info] running main 
Hello from SBT project!
Doubled: List(2, 4, 6, 8, 10)
Sum: 15
```

---

## Step 9 — เข้าใจ Scala Syntax พื้นฐาน

### Comments

```scala
// Single-line comment

/*
  Multi-line comment
  สามารถเขียนหลายบรรทัดได้
*/

/**
 * Scaladoc comment — สำหรับ documentation
 * @param name ชื่อผู้ใช้
 * @return ข้อความทักทาย
 */
def greet(name: String): String = s"Hello, $name!"
```

### Expressions vs Statements

Scala เป็น **expression-oriented language** — เกือบทุกอย่างมีค่าส่งคืน

```scala
// Expression: มีค่า
val x = if (true) 1 else 2      // x = 1
val y = {
  val a = 10
  val b = 20
  a + b                          // ค่าสุดท้ายใน block
}                                // y = 30

// Expression แบบ match
val day = 1
val dayName = day match {
  case 1 => "Monday"
  case 2 => "Tuesday"
  case 3 => "Wednesday"
  case _ => "Other"
}
println(dayName)  // Monday
```

### Semicolons และ Braces

```scala
// Scala 3: ใช้ indentation แทน braces (optional)
def add(a: Int, b: Int): Int =
  a + b

// หรือใช้ braces ก็ได้
def multiply(a: Int, b: Int): Int = {
  a * b
}

// Semicolon ไม่จำเป็น (แต่ใส่ได้)
val a = 1; val b = 2; val c = a + b
```

### val vs var

```scala
val pi = 3.14159      // Immutable — เปลี่ยนค่าไม่ได้
var counter = 0       // Mutable — เปลี่ยนค่าได้

// pi = 3.14  // Error! val cannot be reassigned

counter = counter + 1  // OK
counter += 1           // OK
println(counter)       // 2
```

**หลักการ**: ใช้ `val` เสมอ เว้นแต่จำเป็นต้องใช้ `var`

### Type Annotations

```scala
// Explicit type
val name: String = "Scala"
val age: Int = 30
val price: Double = 9.99
val active: Boolean = true

// Type Inference (Scala เดาเองได้)
val name2 = "Scala"    // String
val age2 = 30          // Int
val price2 = 9.99      // Double
val active2 = true     // Boolean
```

---

## Step 10 — โปรแกรม Hello World แบบ Complete

มารวมทุกอย่างที่เรียนมาสร้างโปรแกรม complete แรก

### สร้าง project structure

```
hello-scala/
├── build.sbt
├── project/
│   └── build.properties
└── src/
    └── main/
        └── scala/
            └── HelloApp.scala
```

### project/build.properties

```
sbt.version=1.9.7
```

### build.sbt

```scala
val scala3Version = "3.3.1"

lazy val root = project
  .in(file("."))
  .settings(
    name         := "hello-scala",
    version      := "0.1.0",
    scalaVersion := scala3Version
  )
```

### src/main/scala/HelloApp.scala

```scala
// HelloApp.scala — โปรแกรมแรกแบบ Complete

// Import libraries
import scala.io.StdIn.readLine

@main def helloApp(): Unit =
  
  // ==============================
  // 1. Basic Output
  // ==============================
  println("=" * 50)
  println("ยินดีต้อนรับสู่หลักสูตร Scala!")
  println("=" * 50)
  
  // ==============================
  // 2. Variables
  // ==============================
  val courseName: String = "Scala Master Course"
  val totalParts: Int = 100
  val hoursPerPart: Double = 3.5
  val isFreeCourse: Boolean = false
  
  println(s"\nชื่อหลักสูตร: $courseName")
  println(s"จำนวน Parts: $totalParts")
  println(s"ชั่วโมงต่อ Part: $hoursPerPart")
  println(s"ฟรีหรือไม่: $isFreeCourse")
  
  // ==============================
  // 3. Calculations
  // ==============================
  val totalHours = totalParts * hoursPerPart
  println(s"\nเวลาเรียนทั้งหมด: $totalHours ชั่วโมง")
  
  val daysAt2HoursPerDay = totalHours / 2.0
  println(f"ถ้าเรียนวันละ 2 ชั่วโมง: $daysAt2HoursPerDay%.0f วัน")
  
  // ==============================
  // 4. Conditional Expression
  // ==============================
  val level = if (totalParts >= 100) "World-class" else "Beginner"
  println(s"\nระดับหลักสูตร: $level")
  
  // ==============================
  // 5. Lists and Collections
  // ==============================
  val phases = List(
    "Scala Fundamentals",
    "OOP & Functional",
    "Collections & Types",
    "Concurrency",
    "Play Framework",
    "Big Data",
    "Microservices",
    "Production"
  )
  
  println("\nPhases ในหลักสูตร:")
  phases.zipWithIndex.foreach { case (phase, index) =>
    println(s"  Phase ${index + 1}: $phase")
  }
  
  // ==============================
  // 6. String Operations
  // ==============================
  val greeting = "hello scala"
  println(s"\n'$greeting' in uppercase: ${greeting.toUpperCase}")
  println(s"'$greeting' reversed: ${greeting.reverse}")
  println(s"Length: ${greeting.length}")
  
  // ==============================
  // 7. Simple Calculation Function
  // ==============================
  def calculateProgress(currentPart: Int, total: Int): Double =
    (currentPart.toDouble / total) * 100
  
  println("\nความคืบหน้า:")
  for (part <- List(10, 25, 50, 75, 100))
    println(f"  Part $part: ${calculateProgress(part, totalParts)}%.1f%%")
  
  // ==============================
  // 8. Pattern Matching Preview
  // ==============================
  def getPhaseDescription(phase: Int): String = phase match
    case p if p <= 2  => "พื้นฐาน Scala"
    case p if p <= 4  => "Functional Programming"
    case p if p <= 6  => "Web Development"
    case p if p <= 8  => "Big Data & Cloud"
    case _            => "Advanced Topics"
  
  println("\nตัวอย่าง Pattern Matching:")
  for (phase <- 1 to 10 by 2)
    println(s"  Phase $phase: ${getPhaseDescription(phase)}")
  
  println("\n" + "=" * 50)
  println("เริ่มต้นการเดินทางสู่ Scala Master!")
  println("=" * 50)
```

### รัน

```bash
cd hello-scala
sbt run
```

**Expected Output:**
```
==================================================
ยินดีต้อนรับสู่หลักสูตร Scala!
==================================================

ชื่อหลักสูตร: Scala Master Course
จำนวน Parts: 100
ชั่วโมงต่อ Part: 3.5
ฟรีหรือไม่: false

เวลาเรียนทั้งหมด: 350.0 ชั่วโมง
ถ้าเรียนวันละ 2 ชั่วโมง: 175 วัน

ระดับหลักสูตร: World-class

Phases ในหลักสูตร:
  Phase 1: Scala Fundamentals
  Phase 2: OOP & Functional
  Phase 3: Collections & Types
  Phase 4: Concurrency
  Phase 5: Play Framework
  Phase 6: Big Data
  Phase 7: Microservices
  Phase 8: Production

'hello scala' in uppercase: HELLO SCALA
'hello scala' reversed: alacs olleh
Length: 11

ความคืบหน้า:
  Part 10: 10.0%
  Part 25: 25.0%
  Part 50: 50.0%
  Part 75: 75.0%
  Part 100: 100.0%

ตัวอย่าง Pattern Matching:
  Phase 1: พื้นฐาน Scala
  Phase 3: Functional Programming
  Phase 5: Web Development
  Phase 7: Big Data & Cloud
  Phase 9: Advanced Topics

==================================================
เริ่มต้นการเดินทางสู่ Scala Master!
==================================================
```

---

## สรุป Part 01

| Step | สิ่งที่เรียน |
|------|-------------|
| 1 | ทำความรู้จัก Scala และ use cases |
| 2 | ติดตั้ง JDK 17 |
| 3 | ติดตั้ง Scala ผ่าน Coursier |
| 4 | ติดตั้ง SBT build tool |
| 5 | ติดตั้ง IntelliJ IDEA / VS Code |
| 6 | Scala REPL เบื้องต้น |
| 7 | Hello World โปรแกรมแรก |
| 8 | สร้างโปรเจกต์ SBT |
| 9 | Scala Syntax พื้นฐาน |
| 10 | โปรแกรม Complete แรก |

## แบบฝึกหัด

1. ติดตั้ง Scala และ SBT บนเครื่องของคุณ
2. เปิด REPL และลองคำนวณ `(100 + 200) * 3 / 2`
3. สร้างโปรเจกต์ SBT ใหม่และแก้ไข Main.scala ให้แสดงชื่อของคุณ
4. ใน REPL ลองพิมพ์ `:type List(1,2,3)` และสังเกตผลลัพธ์
5. เขียนโปรแกรมที่รับ input ชื่อจาก user และแสดง greeting

## ต่อไป

**[Part 02 →](part-02-variables-and-types.md)** — Variables, Data Types และ Type Inference
