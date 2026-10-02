# Part 64: Doobie

## Steps 631-640: Doobie Setup, HikariCP, SQL Fragments, Error Handling, Transactions

---

## Step 631: Doobie คืออะไร

Doobie คือ functional JDBC layer สำหรับ Scala ที่ใช้ Cats Effect

```
Doobie vs Slick:
Doobie:
  - Pure SQL (เขียน SQL เอง)
  - Functional (Cats Effect IO)
  - Type-safe result mapping
  - Composable SQL fragments

Slick:
  - DSL query builder
  - Type-safe queries
  - Future-based
  - Auto-generate SQL
```

### build.sbt

```scala
val doobieVersion = "1.0.0-RC4"

libraryDependencies ++= Seq(
  "org.tpolecat" %% "doobie-core"       % doobieVersion,
  "org.tpolecat" %% "doobie-hikari"     % doobieVersion,
  "org.tpolecat" %% "doobie-postgres"   % doobieVersion,
  "org.tpolecat" %% "doobie-refined"    % doobieVersion,
  "org.typelevel" %% "cats-effect"      % "3.5.4",

  // สำหรับ Play integration (Cats Effect + Play)
  "org.typelevel" %% "cats-effect"      % "3.5.4",
  "com.github.valskalla" %% "odin-core" % "0.13.0"
)
```

---

## Step 632: HikariCP Connection Pool

```scala
// app/db/DatabaseModule.scala
package db

import cats.effect.*
import doobie.*
import doobie.hikari.*
import com.zaxxer.hikari.HikariConfig
import javax.inject.*
import play.api.Configuration
import play.api.inject.ApplicationLifecycle
import scala.concurrent.Future

@Singleton
class DatabaseModule @Inject()(
  config: Configuration,
  lifecycle: ApplicationLifecycle
) {
  // HikariCP config
  private val hikariConfig = new HikariConfig()
  hikariConfig.setJdbcUrl(config.get[String]("db.url"))
  hikariConfig.setUsername(config.get[String]("db.user"))
  hikariConfig.setPassword(config.get[String]("db.password"))
  hikariConfig.setMaximumPoolSize(config.getOrElse("db.maxPoolSize", 20))
  hikariConfig.setMinimumIdle(config.getOrElse("db.minIdle", 5))
  hikariConfig.setConnectionTimeout(30000)
  hikariConfig.setIdleTimeout(600000)
  hikariConfig.setMaxLifetime(1800000)

  // สร้าง Transactor ด้วย Cats Effect runtime
  private val runtime = cats.effect.unsafe.IORuntime.global

  val transactor: Transactor[IO] =
    HikariTransactor.fromHikariConfig[IO](hikariConfig).allocated
      .unsafeRunSync()(runtime)._1

  // Shutdown pool เมื่อ app stop
  lifecycle.addStopHook(() => Future.successful {
    // HikariCP cleanup
  })
}
```

---

## Step 633: Basic Queries

```scala
// app/repositories/DoobieUserRepository.scala
package repositories

import cats.effect.*
import doobie.*
import doobie.implicits.*
import doobie.postgres.implicits.*
import models.*
import javax.inject.*

@Singleton
class DoobieUserRepository @Inject()(dbModule: DatabaseModule) {

  private val xa = dbModule.transactor

  // SELECT
  def findById(id: String): IO[Option[User]] =
    sql"SELECT id, name, email, role, created_at FROM users WHERE id = $id"
      .query[User]
      .option
      .transact(xa)

  def findAll(limit: Int = 20, offset: Int = 0): IO[List[User]] =
    sql"""
      SELECT id, name, email, role, created_at
      FROM users
      ORDER BY created_at DESC
      LIMIT $limit OFFSET $offset
    """
      .query[User]
      .to[List]
      .transact(xa)

  def findByEmail(email: String): IO[Option[User]] =
    sql"SELECT id, name, email, role, created_at FROM users WHERE email = $email"
      .query[User]
      .option
      .transact(xa)

  // INSERT
  def create(user: User): IO[User] =
    sql"""
      INSERT INTO users (id, name, email, password_hash, role, created_at, updated_at)
      VALUES (${user.id}, ${user.name}, ${user.email}, ${user.passwordHash},
              ${user.role}, ${user.createdAt}, ${user.updatedAt})
    """
      .update
      .run
      .transact(xa)
      .map(_ => user)

  // INSERT RETURNING (PostgreSQL specific)
  def createAndReturn(name: String, email: String, passwordHash: String): IO[User] =
    sql"""
      INSERT INTO users (id, name, email, password_hash, role, created_at, updated_at)
      VALUES (gen_random_uuid()::text, $name, $email, $passwordHash, 'user', NOW(), NOW())
      RETURNING id, name, email, role, created_at
    """
      .query[User]
      .unique
      .transact(xa)

  // UPDATE
  def update(id: String, name: String, email: String): IO[Int] =
    sql"""
      UPDATE users SET name = $name, email = $email, updated_at = NOW()
      WHERE id = $id
    """
      .update
      .run
      .transact(xa)

  // DELETE
  def delete(id: String): IO[Int] =
    sql"DELETE FROM users WHERE id = $id"
      .update
      .run
      .transact(xa)

  def count(): IO[Long] =
    sql"SELECT COUNT(*) FROM users"
      .query[Long]
      .unique
      .transact(xa)
}
```

---

## Step 634: SQL Fragments (Fragment)

```scala
// SQL Fragments ช่วย compose dynamic queries อย่าง type-safe

import doobie.*
import doobie.implicits.*

def findArticles(
  status: Option[String] = None,
  authorId: Option[String] = None,
  tag: Option[String] = None,
  search: Option[String] = None,
  limit: Int = 20,
  offset: Int = 0
): IO[List[Article]] = {

  // Base query
  val base = fr"""
    SELECT id, title, content, summary, author_id, status, tags, view_count, published_at, created_at
    FROM articles
  """

  // Build WHERE fragments
  val filters = List(
    status.map(s    => fr"status = $s"),
    authorId.map(id => fr"author_id = $id"),
    tag.map(t       => fr"$t = ANY(tags)"),
    search.map(q    => fr"(title ILIKE ${"%" + q + "%"} OR content ILIKE ${"%" + q + "%"})")
  ).flatten

  val where = filters match {
    case Nil  => Fragment.empty
    case list => fr"WHERE" ++ list.reduceLeft(_ ++ fr"AND" ++ _)
  }

  val orderBy = fr"ORDER BY published_at DESC NULLS LAST"
  val pagination = fr"LIMIT $limit OFFSET $offset"

  // Compose complete query
  val query = base ++ where ++ orderBy ++ pagination

  query.query[Article].to[List].transact(xa)
}

// Fragment สำหรับ IN clause
def findByIds(ids: List[String]): IO[List[Article]] = {
  val inClause = Fragments.in(fr"id", ids)
  val query = fr"SELECT * FROM articles WHERE" ++ inClause
  query.query[Article].to[List].transact(xa)
}
```

---

## Step 635: Read Instances (Custom Type Mapping)

```scala
import doobie.*
import doobie.implicits.*
import java.time.Instant

// Custom Read instance สำหรับ case class
case class Article(
  id: String,
  title: String,
  content: String,
  summary: Option[String],
  authorId: String,
  status: String,
  tags: List[String],
  viewCount: Int,
  publishedAt: Option[Instant],
  createdAt: Instant
)

// Doobie ใช้ HList-based derivation สำหรับ case classes
// ถ้า fields ตรง order กับ SQL columns → auto-derived
// query[Article] จะ map โดยอัตโนมัติ

// Custom mapping สำหรับ enum
sealed trait ArticleStatus
case object Draft     extends ArticleStatus
case object Published extends ArticleStatus
case object Archived  extends ArticleStatus

implicit val articleStatusMeta: Meta[ArticleStatus] =
  Meta[String].timap {
    case "draft"     => Draft
    case "published" => Published
    case "archived"  => Archived
    case other       => throw new Exception(s"Unknown status: $other")
  } {
    case Draft     => "draft"
    case Published => "published"
    case Archived  => "archived"
  }

// PostgreSQL Array → List[String]
implicit val listStringMeta: Meta[List[String]] =
  doobie.postgres.implicits.pgArrayGet[String].timap(_.toList)(_.toArray)
```

---

## Step 636: Transactions

```scala
// Doobie transactions ใช้ ConnectionIO monad

def createArticleWithTags(
  article: Article,
  tagIds: List[String]
): IO[Article] = {

  // สร้าง program ใน ConnectionIO monad
  val program: ConnectionIO[Article] = for {
    // Insert article
    _ <- sql"""
      INSERT INTO articles (id, title, content, author_id, status, created_at)
      VALUES (${article.id}, ${article.title}, ${article.content},
              ${article.authorId}, 'draft', NOW())
    """.update.run

    // Insert tag relationships
    _ <- Update[String](
      "INSERT INTO article_tags (article_id, tag_id) VALUES (?, ?)"
    ).updateMany(tagIds.map(tid => article.id + "," + tid))

    // Return created article
    created <- sql"SELECT * FROM articles WHERE id = ${article.id}"
                 .query[Article].unique
  } yield created

  // Run as transaction
  program.transact(xa)
}

// Rollback on error
def transferCredits(fromId: String, toId: String, amount: Int): IO[Unit] = {
  val program: ConnectionIO[Unit] = for {
    fromBalance <- sql"SELECT credits FROM users WHERE id = $fromId FOR UPDATE"
                     .query[Int].unique
    _           <- if (fromBalance < amount)
                     FC.raiseError(new Exception("Insufficient credits"))
                   else
                     sql"UPDATE users SET credits = credits - $amount WHERE id = $fromId".update.run
    _           <- sql"UPDATE users SET credits = credits + $amount WHERE id = $toId".update.run
  } yield ()

  // Auto-rollback on exception
  program.transact(xa)
}
```

---

## Step 637: Error Handling

```scala
import cats.effect.*
import doobie.*

def safeFind(id: String): IO[Either[String, User]] =
  findById(id)
    .map {
      case Some(user) => Right(user)
      case None       => Left(s"User $id not found")
    }
    .handleErrorWith { ex =>
      IO.pure(Left(s"Database error: ${ex.getMessage}"))
    }

// หรือใช้ attempt
def findOrFail(id: String): IO[User] =
  findById(id).flatMap {
    case Some(user) => IO.pure(user)
    case None       => IO.raiseError(new NoSuchElementException(s"User $id not found"))
  }

// Error types
sealed trait DbError
case class NotFound(id: String)        extends DbError
case class DuplicateKey(field: String) extends DbError
case class DbException(ex: Throwable)  extends DbError

def createSafe(user: User): IO[Either[DbError, User]] =
  create(user)
    .map(Right(_))
    .handleError {
      case ex: org.postgresql.util.PSQLException if ex.getSQLState == "23505" =>
        Left(DuplicateKey("email"))
      case ex =>
        Left(DbException(ex))
    }
```

---

## Step 638: Play Integration

```scala
// app/controllers/UserController.scala — using Doobie with Play

import cats.effect.unsafe.implicits.global  // for unsafeRunSync/unsafeToFuture

@Singleton
class UserController @Inject()(
  val controllerComponents: ControllerComponents,
  userRepo: DoobieUserRepository,
  implicit val ec: ExecutionContext
) extends BaseController {

  // Convert IO[T] → Future[T] สำหรับ Play
  def getUser(id: String): Action[AnyContent] = Action.async {
    userRepo.findById(id)
      .unsafeToFuture()  // IO → Future
      .map {
        case Some(user) => Ok(Json.toJson(user))
        case None       => NotFound(Json.obj("error" -> s"User $id not found"))
      }
      .recover {
        case ex: Exception =>
          InternalServerError(Json.obj("error" -> ex.getMessage))
      }
  }

  def createUser(): Action[JsValue] = Action.async(parse.json) { request =>
    request.body.validate[CreateUserRequest].fold(
      errors => Future.successful(BadRequest(Json.obj("error" -> "Invalid input"))),
      req =>
        userRepo.createAndReturn(req.name, req.email, hashPassword(req.password))
          .unsafeToFuture()
          .map(user => Created(Json.toJson(user)))
          .recover {
            case ex: org.postgresql.util.PSQLException if ex.getSQLState == "23505" =>
              Conflict(Json.obj("error" -> "Email already exists"))
          }
    )
  }

  private def hashPassword(plain: String): String =
    org.mindrot.jbcrypt.BCrypt.hashpw(plain, org.mindrot.jbcrypt.BCrypt.gensalt())
}
```

---

## Step 639: Testing Doobie

```scala
// test/repositories/DoobieUserRepositorySpec.scala
package repositories

import cats.effect.*
import doobie.*
import doobie.implicits.*
import org.scalatestplus.play.*
import org.scalatestplus.play.guice.GuiceOneAppPerTest
import cats.effect.unsafe.implicits.global

class DoobieUserRepositorySpec extends PlaySpec with GuiceOneAppPerTest {

  "DoobieUserRepository" should {
    "create and retrieve user" in {
      val repo = app.injector.instanceOf[DoobieUserRepository]

      val program = for {
        user  <- repo.createAndReturn("Test", s"test${System.currentTimeMillis()}@test.com", "hash")
        found <- repo.findById(user.id)
        _     <- repo.delete(user.id)
      } yield found

      val result = program.unsafeRunSync()
      result must be(defined)
      result.get.name mustBe "Test"
    }

    "handle duplicate email" in {
      val repo  = app.injector.instanceOf[DoobieUserRepository]
      val email = s"dup${System.currentTimeMillis()}@test.com"

      val program = for {
        user1 <- repo.createAndReturn("User1", email, "hash")
        result <- repo.createAndReturn("User2", email, "hash").attempt
        _      <- repo.delete(user1.id)
      } yield result

      val result = program.unsafeRunSync()
      result.isLeft mustBe true
    }
  }
}
```

---

## Step 640: Advanced Patterns

```scala
// Pagination with total count
case class Page[A](items: List[A], total: Long, limit: Int, offset: Int) {
  def hasNext: Boolean = offset + limit < total
  def hasPrev: Boolean = offset > 0
  def pageNumber: Int  = offset / limit + 1
}

def findWithPage(limit: Int, offset: Int): IO[Page[Article]] = {
  val itemsQuery = fr"""
    SELECT id, title, content, summary, author_id, status, tags,
           view_count, published_at, created_at
    FROM articles
    WHERE status = 'published'
    ORDER BY published_at DESC NULLS LAST
    LIMIT $limit OFFSET $offset
  """.query[Article].to[List]

  val countQuery =
    fr"SELECT COUNT(*) FROM articles WHERE status = 'published'".query[Long].unique

  // Run both in same transaction
  (for {
    items <- itemsQuery
    total <- countQuery
  } yield Page(items, total, limit, offset)).transact(xa)
}
```

---

## สรุป Part 64

| Concept | Doobie API | ตัวอย่าง |
|---------|-----------|---------|
| SELECT | `sql"...".query[T].to[List]` | `query[User].option` |
| INSERT | `sql"...".update.run` | `RETURNING` for full row |
| UPDATE | `sql"...".update.run` | Returns affected rows |
| DELETE | `sql"...".update.run` | Returns affected rows |
| Transaction | `.transact(xa)` | `ConnectionIO` monad |
| Fragments | `fr"..." ++ fr"..."` | Dynamic SQL |
| Error handling | `.handleError`, `.attempt` | `IO[Either[E, A]]` |
| Play integration | `.unsafeToFuture()` | `IO` → `Future` |
| Custom types | `Meta[T]` | Enum mapping |
| Batch operations | `Update[T].updateMany` | Bulk insert |

---

## แบบฝึกหัด Part 64

1. **Full CRUD Service**: สร้าง complete Doobie-based service layer สำหรับ blog API ที่มี users, articles, comments พร้อม transactions

2. **Dynamic Search**: Implement search API ที่ใช้ Fragment composition สำหรับ dynamic WHERE clauses พร้อม full-text search

3. **Bulk Operations**: สร้าง data import endpoint ที่ accept CSV, parse เป็น records, และ bulk insert ด้วย Doobie batch operations

4. **Play Integration**: Integrate Doobie กับ Play application ที่มี proper error handling, logging, และ transaction management

5. **Migration from Slick**: Convert existing Slick repository เป็น Doobie โดยให้ behavior เหมือนกัน รวมถึง tests ครบ

---

[→ ไปยัง Part 65: MongoDB with Scala](part-65-mongodb-with-scala.md)
