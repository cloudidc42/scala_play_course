# Part 61: Slick Introduction

## Steps 601-610: Slick Setup, Table Definition, Basic CRUD, PostgreSQL

---

## Step 601: Slick คืออะไร

Slick (Scala Language Integrated Connection Kit) คือ FRM (Functional Relational Mapper) สำหรับ Scala

```
Slick vs Hibernate (ORM):
Slick:
  - Functional style
  - Composable queries
  - Type-safe SQL
  - Non-blocking async

Hibernate:
  - Object-oriented
  - Session-based
  - Reflection heavy
  - Blocking by default
```

### build.sbt

```scala
val slickVersion = "3.5.1"

libraryDependencies ++= Seq(
  "com.typesafe.play"  %% "play-slick"            % "6.1.0",
  "com.typesafe.play"  %% "play-slick-evolutions"  % "6.1.0",
  "org.postgresql"      % "postgresql"              % "42.7.3",
  "com.github.tminglei" %% "slick-pg"              % "0.22.2",
  "com.github.tminglei" %% "slick-pg_play-json"    % "0.22.2"
)
```

---

## Step 602: Database Configuration

```hocon
# conf/application.conf
slick.dbs.default {
  profile = "slick.jdbc.PostgresProfile$"
  db {
    driver   = "org.postgresql.Driver"
    url      = "jdbc:postgresql://localhost:5432/myapp"
    user     = "myapp"
    password = "secret"
    numThreads = 10
    maxConnections = 20
    minConnections = 5
    connectionTimeout = 30000
    idleTimeout = 600000
    maxLifetime = 1800000
    keepAliveConnection = true
  }
}
```

---

## Step 603: Table Definition

```scala
// app/models/tables/UsersTable.scala
package models.tables

import slick.jdbc.PostgresProfile.api.*
import java.time.Instant

// Domain model
case class User(
  id: String,
  name: String,
  email: String,
  passwordHash: String,
  role: String = "user",
  createdAt: Instant = Instant.now(),
  updatedAt: Instant = Instant.now()
)

// Slick Table definition
class UsersTable(tag: Tag) extends Table[User](tag, "users") {
  def id           = column[String]("id", O.PrimaryKey)
  def name         = column[String]("name")
  def email        = column[String]("email")
  def passwordHash = column[String]("password_hash")
  def role         = column[String]("role", O.Default("user"))
  def createdAt    = column[Instant]("created_at")
  def updatedAt    = column[Instant]("updated_at")

  // Index สำหรับ email (unique)
  def emailIdx = index("users_email_idx", email, unique = true)

  // Map columns to case class
  def * = (id, name, email, passwordHash, role, createdAt, updatedAt).mapTo[User]
}

object UsersTable {
  val query = TableQuery[UsersTable]
}
```

```scala
// app/models/tables/ArticlesTable.scala
package models.tables

import slick.jdbc.PostgresProfile.api.*
import java.time.Instant

case class Article(
  id: String,
  title: String,
  content: String,
  summary: Option[String],
  authorId: String,
  status: String = "draft",
  tags: List[String] = Nil,
  viewCount: Int = 0,
  publishedAt: Option[Instant] = None,
  createdAt: Instant = Instant.now(),
  updatedAt: Instant = Instant.now()
)

class ArticlesTable(tag: Tag) extends Table[Article](tag, "articles") {
  def id          = column[String]("id", O.PrimaryKey)
  def title       = column[String]("title")
  def content     = column[String]("content")
  def summary     = column[Option[String]]("summary")
  def authorId    = column[String]("author_id")
  def status      = column[String]("status", O.Default("draft"))
  def tags        = column[List[String]]("tags")  // PostgreSQL array
  def viewCount   = column[Int]("view_count", O.Default(0))
  def publishedAt = column[Option[Instant]]("published_at")
  def createdAt   = column[Instant]("created_at")
  def updatedAt   = column[Instant]("updated_at")

  // Foreign key
  def authorFk = foreignKey("articles_author_fk", authorId, UsersTable.query)(
    _.id,
    onUpdate = ForeignKeyAction.Cascade,
    onDelete = ForeignKeyAction.Restrict
  )

  // Indexes
  def authorIdx = index("articles_author_idx", authorId)
  def statusIdx = index("articles_status_idx", status)

  def * = (id, title, content, summary, authorId, status, tags,
           viewCount, publishedAt, createdAt, updatedAt).mapTo[Article]
}

object ArticlesTable {
  val query = TableQuery[ArticlesTable]
}
```

---

## Step 604: Custom Column Types

```scala
// app/models/tables/SlickProfile.scala
package models.tables

import com.github.tminglei.slickpg.*
import com.github.tminglei.slickpg.agg.PgAggFuncSupport
import play.api.libs.json.JsValue

// Custom PostgreSQL profile ที่รองรับ JSON, Arrays, etc.
trait MyPostgresProfile extends ExPostgresProfile
  with PgArraySupport
  with PgDateSupport
  with PgPlayJsonSupport
  with PgAggFuncSupport {

  def pgjson = "jsonb"

  override val api = MyAPI

  object MyAPI extends ExtPostgresAPI
    with ArrayImplicits
    with DateTimeImplicits
    with JsonImplicits {

    // Implicit สำหรับ List[String] ↔ PostgreSQL text[]
    implicit val strListTypeMapper = new SimpleArrayJdbcType[String]("text")
      .to(_.toList)
  }
}

object MyPostgresProfile extends MyPostgresProfile

// ใช้ custom profile แทน PostgresProfile
// import MyPostgresProfile.api.*  แทน
// import slick.jdbc.PostgresProfile.api.*
```

---

## Step 605: Repository Pattern

```scala
// app/repositories/UserRepository.scala
package repositories

import javax.inject.*
import models.tables.*
import MyPostgresProfile.api.*
import play.api.db.slick.DatabaseConfigProvider
import slick.jdbc.JdbcProfile
import scala.concurrent.*
import java.time.Instant
import java.util.UUID

@Singleton
class UserRepository @Inject()(
  dbConfigProvider: DatabaseConfigProvider
)(implicit ec: ExecutionContext) {

  private val db = dbConfigProvider.get[JdbcProfile].db
  private val users = UsersTable.query

  // CREATE
  def create(user: User): Future[User] =
    db.run(users += user).map(_ => user)

  // READ
  def findById(id: String): Future[Option[User]] =
    db.run(users.filter(_.id === id).result.headOption)

  def findByEmail(email: String): Future[Option[User]] =
    db.run(users.filter(_.email === email).result.headOption)

  def findAll(limit: Int = 20, offset: Int = 0): Future[Seq[User]] =
    db.run(
      users
        .sortBy(_.createdAt.desc)
        .drop(offset)
        .take(limit)
        .result
    )

  def count(): Future[Int] =
    db.run(users.length.result)

  // UPDATE
  def update(id: String, name: String, email: String): Future[Int] =
    db.run(
      users
        .filter(_.id === id)
        .map(u => (u.name, u.email, u.updatedAt))
        .update((name, email, Instant.now()))
    )

  def updatePassword(id: String, passwordHash: String): Future[Int] =
    db.run(
      users
        .filter(_.id === id)
        .map(u => (u.passwordHash, u.updatedAt))
        .update((passwordHash, Instant.now()))
    )

  // DELETE
  def delete(id: String): Future[Int] =
    db.run(users.filter(_.id === id).delete)

  // Upsert
  def upsert(user: User): Future[Int] =
    db.run(users.insertOrUpdate(user))
}
```

---

## Step 606: Article Repository

```scala
// app/repositories/ArticleRepository.scala
package repositories

import javax.inject.*
import models.tables.*
import MyPostgresProfile.api.*
import play.api.db.slick.DatabaseConfigProvider
import slick.jdbc.JdbcProfile
import scala.concurrent.*
import java.time.Instant

@Singleton
class ArticleRepository @Inject()(
  dbConfigProvider: DatabaseConfigProvider
)(implicit ec: ExecutionContext) {

  private val db       = dbConfigProvider.get[JdbcProfile].db
  private val articles = ArticlesTable.query

  def create(article: Article): Future[Article] =
    db.run(articles += article).map(_ => article)

  def findById(id: String): Future[Option[Article]] =
    db.run(articles.filter(_.id === id).result.headOption)

  // Complex query: filter + sort + paginate
  def findPublished(
    limit: Int = 20,
    offset: Int = 0,
    search: Option[String] = None,
    tag: Option[String] = None
  ): Future[Seq[Article]] = {

    var query = articles.filter(_.status === "published")

    // Add search filter
    search.foreach { term =>
      query = query.filter(a =>
        (a.title.toLowerCase like s"%${term.toLowerCase}%") ||
        (a.content.toLowerCase like s"%${term.toLowerCase}%")
      )
    }

    // Add tag filter (PostgreSQL array contains)
    tag.foreach { t =>
      query = query.filter(_.tags @> List(t))
    }

    db.run(
      query
        .sortBy(_.publishedAt.desc.nullsLast)
        .drop(offset)
        .take(limit)
        .result
    )
  }

  def countPublished(search: Option[String] = None): Future[Int] =
    db.run(articles.filter(_.status === "published").length.result)

  def findByAuthor(authorId: String): Future[Seq[Article]] =
    db.run(
      articles
        .filter(_.authorId === authorId)
        .sortBy(_.createdAt.desc)
        .result
    )

  def update(id: String, title: String, content: String, summary: Option[String]): Future[Int] =
    db.run(
      articles
        .filter(_.id === id)
        .map(a => (a.title, a.content, a.summary, a.updatedAt))
        .update((title, content, summary, Instant.now()))
    )

  def publish(id: String): Future[Int] =
    db.run(
      articles
        .filter(_.id === id)
        .map(a => (a.status, a.publishedAt, a.updatedAt))
        .update(("published", Some(Instant.now()), Instant.now()))
    )

  def incrementViewCount(id: String): Future[Int] =
    db.run(
      sqlu"UPDATE articles SET view_count = view_count + 1 WHERE id = $id"
    )

  def delete(id: String): Future[Int] =
    db.run(articles.filter(_.id === id).delete)
}
```

---

## Step 607: Database Evolutions (Schema Migration)

```sql
-- conf/evolutions/default/1.sql

# --- !Ups

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE users (
  id            VARCHAR(36) PRIMARY KEY DEFAULT gen_random_uuid()::text,
  name          VARCHAR(255) NOT NULL,
  email         VARCHAR(255) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  role          VARCHAR(50)  NOT NULL DEFAULT 'user',
  created_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX users_email_idx ON users(email);

CREATE TABLE articles (
  id           VARCHAR(36) PRIMARY KEY DEFAULT gen_random_uuid()::text,
  title        VARCHAR(500) NOT NULL,
  content      TEXT         NOT NULL,
  summary      TEXT,
  author_id    VARCHAR(36)  NOT NULL REFERENCES users(id),
  status       VARCHAR(50)  NOT NULL DEFAULT 'draft',
  tags         TEXT[]       NOT NULL DEFAULT '{}',
  view_count   INTEGER      NOT NULL DEFAULT 0,
  published_at TIMESTAMPTZ,
  created_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX articles_author_idx ON articles(author_id);
CREATE INDEX articles_status_idx ON articles(status);
CREATE INDEX articles_published_at_idx ON articles(published_at DESC NULLS LAST);

# --- !Downs

DROP TABLE IF EXISTS articles;
DROP TABLE IF EXISTS users;
```

---

## Step 608: Transactions

```scala
// ใช้ DBIO.seq สำหรับ transaction

def createArticleWithTags(article: Article, tags: Seq[String]): Future[Article] = {
  val updatedArticle = article.copy(tags = tags.toList)

  val action = for {
    _           <- articles += updatedArticle
    _           <- DBIO.seq(
                     tags.map(tag =>
                       sqlu"INSERT INTO article_tags(article_id, tag) VALUES (${article.id}, $tag)"
                     ): _*
                   )
  } yield updatedArticle

  db.run(action.transactionally)
}

// Transfer money ระหว่าง accounts
def transfer(fromId: String, toId: String, amount: BigDecimal): Future[Unit] = {
  val action = for {
    from <- users.filter(_.id === fromId).result.headOption
    to   <- users.filter(_.id === toId).result.headOption

    _ <- (from, to) match {
      case (Some(f), Some(t)) =>
        // ทั้ง 2 operations ใน transaction เดียว
        DBIO.seq(
          sqlu"UPDATE accounts SET balance = balance - $amount WHERE user_id = $fromId",
          sqlu"UPDATE accounts SET balance = balance + $amount WHERE user_id = $toId"
        )
      case _ =>
        DBIO.failed(new Exception("User not found"))
    }
  } yield ()

  db.run(action.transactionally)
}
```

---

## Step 609: Compiled Queries (Performance)

```scala
// Compiled queries สำหรับ frequently used queries
// ป้องกัน query compilation overhead

val findByIdCompiled = Compiled { (id: Rep[String]) =>
  users.filter(_.id === id)
}

val findByEmailCompiled = Compiled { (email: Rep[String]) =>
  users.filter(_.email === email)
}

val findPublishedArticlesCompiled = Compiled {
  (limit: ConstColumn[Long], offset: ConstColumn[Long]) =>
    articles
      .filter(_.status === "published")
      .sortBy(_.publishedAt.desc)
      .drop(offset)
      .take(limit)
}

// ใช้งาน compiled query
def findById(id: String): Future[Option[User]] =
  db.run(findByIdCompiled(id).result.headOption)

def findPublished(limit: Int, offset: Int): Future[Seq[Article]] =
  db.run(findPublishedArticlesCompiled(limit.toLong, offset.toLong).result)
```

---

## Step 610: Testing Slick

```scala
// test/repositories/UserRepositorySpec.scala
package repositories

import org.scalatestplus.play.*
import org.scalatestplus.play.guice.GuiceOneAppPerTest
import play.api.test.*
import models.tables.User
import java.time.Instant
import scala.concurrent.*
import scala.concurrent.duration.*

class UserRepositorySpec extends PlaySpec with GuiceOneAppPerTest {

  "UserRepository" should {
    "create and retrieve a user" in {
      val repo = app.injector.instanceOf[UserRepository]

      val user = User(
        id           = java.util.UUID.randomUUID().toString,
        name         = "Test User",
        email        = s"test${System.currentTimeMillis()}@example.com",
        passwordHash = "hashed"
      )

      val result = for {
        _     <- repo.create(user)
        found <- repo.findById(user.id)
      } yield found

      val found = Await.result(result, 5.seconds)
      found must be(defined)
      found.get.name mustBe "Test User"
    }

    "update a user" in {
      val repo = app.injector.instanceOf[UserRepository]
      val userId = java.util.UUID.randomUUID().toString
      val user = User(userId, "Old Name", s"old${userId}@test.com", "hash")

      val result = for {
        _       <- repo.create(user)
        updated <- repo.update(userId, "New Name", s"new${userId}@test.com")
        found   <- repo.findById(userId)
      } yield (updated, found)

      val (rows, found) = Await.result(result, 5.seconds)
      rows mustBe 1
      found.get.name mustBe "New Name"
    }
  }
}
```

---

## สรุป Part 61

| Concept | Slick API | ตัวอย่าง |
|---------|----------|---------|
| Table definition | `Table[T](tag, "table_name")` | `UsersTable` |
| Column | `column[T]("col_name", options)` | `O.PrimaryKey`, `O.Default` |
| Query | `TableQuery[T]` | `UsersTable.query` |
| Filter | `.filter(_.col === value)` | `WHERE id = ?` |
| Sort | `.sortBy(_.col.desc)` | `ORDER BY col DESC` |
| Paginate | `.drop(n).take(m)` | `OFFSET n LIMIT m` |
| Run | `db.run(query.result)` | Returns `Future[T]` |
| Transaction | `action.transactionally` | ACID transaction |
| Compiled | `Compiled { query }` | Pre-compiled query |
| Insert | `table += row` | `INSERT INTO` |
| Update | `.map(...).update(values)` | `UPDATE SET` |
| Delete | `.filter(...).delete` | `DELETE WHERE` |

---

## แบบฝึกหัด Part 61

1. **Blog Database**: Design และ implement ทุก tables สำหรับ blog system (users, articles, comments, tags, likes) ด้วย Slick table definitions และ proper foreign keys

2. **Repository Layer**: สร้าง complete repository layer ด้วย CRUD operations สำหรับทุก entities และ test ด้วย test database

3. **Compiled Queries**: Identify 5 frequently used queries และ convert เป็น compiled queries เพื่อ optimize performance

4. **Transaction Batch**: Implement batch import ที่ insert หลาย records ใน single transaction ด้วย error rollback

5. **Custom Column Types**: Implement custom Slick column type สำหรับ PostgreSQL `JSONB` ที่ serialize/deserialize Scala case class อัตโนมัติ

---

[→ ไปยัง Part 62: Slick Queries](part-62-slick-queries.md)
