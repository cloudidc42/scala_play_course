# Part 65: MongoDB with Scala

## Steps 641-650: ReactiveMongo Setup, CRUD, Queries, Aggregation, Indexes

---

## Step 641: ReactiveMongo คืออะไร

ReactiveMongo คือ async/non-blocking MongoDB driver สำหรับ Scala

```
MongoDB ใช้เมื่อ:
├── Schema-less data (ข้อมูลที่ structure ไม่แน่นอน)
├── Document-oriented (nested objects)
├── Rapid prototyping
├── Content management, catalogs
├── Real-time analytics
└── Flexible queries

MongoDB ไม่เหมาะเมื่อ:
├── Complex transactions ข้าม documents
├── Strict relationships (FK)
└── ACID-critical operations
```

### build.sbt

```scala
libraryDependencies ++= Seq(
  "org.reactivemongo" %% "reactivemongo"              % "1.1.0-RC6",
  "org.reactivemongo" %% "play2-reactivemongo"         % "1.1.0-RC12",
  "org.reactivemongo" %% "reactivemongo-play-json-compat" % "1.1.0-RC6"
)
```

---

## Step 642: Configuration

```hocon
# conf/application.conf
mongodb {
  uri     = "mongodb://localhost:27017/myapp"
  # หรือ MongoDB Atlas
  # uri = "mongodb+srv://user:pass@cluster.mongodb.net/myapp"
  
  db      = "myapp"
  
  failoverStrategy {
    initialDelay = 500ms
    retries      = 5
    factor       = 1.5
  }
}

play.modules.enabled += "play.modules.reactivemongo.ReactiveMongoModule"
```

---

## Step 643: Document Models

```scala
// app/models/mongo/Article.scala
package models.mongo

import play.api.libs.json.*
import reactivemongo.api.bson.*
import java.time.Instant

case class Article(
  _id: Option[BSONObjectID] = None,
  title: String,
  slug: String,
  content: String,
  summary: Option[String],
  author: AuthorRef,
  tags: List[String] = Nil,
  status: String = "draft",
  viewCount: Int = 0,
  metadata: ArticleMetadata = ArticleMetadata(),
  publishedAt: Option[Instant] = None,
  createdAt: Instant = Instant.now(),
  updatedAt: Instant = Instant.now()
)

case class AuthorRef(
  id: String,
  name: String,
  avatarUrl: Option[String] = None
)

case class ArticleMetadata(
  readTimeMinutes: Int = 0,
  wordCount: Int = 0,
  language: String = "th",
  seoTitle: Option[String] = None,
  seoDescription: Option[String] = None
)

// BSON (MongoDB) serializers
object Article {
  implicit val authorRefHandler: BSONDocumentHandler[AuthorRef] =
    Macros.handler[AuthorRef]

  implicit val metadataHandler: BSONDocumentHandler[ArticleMetadata] =
    Macros.handler[ArticleMetadata]

  implicit val articleHandler: BSONDocumentHandler[Article] =
    Macros.handler[Article]

  // JSON serializers สำหรับ API
  implicit val authorRefFormat: OFormat[AuthorRef] = Json.format[AuthorRef]
  implicit val metadataFormat: OFormat[ArticleMetadata] = Json.format[ArticleMetadata]
  implicit val articleFormat: OFormat[Article] = Json.format[Article]
}
```

---

## Step 644: Repository Layer

```scala
// app/repositories/ArticleMongoRepository.scala
package repositories

import javax.inject.*
import models.mongo.*
import play.modules.reactivemongo.*
import reactivemongo.api.bson.*
import reactivemongo.api.bson.collection.*
import reactivemongo.api.commands.*
import scala.concurrent.*

@Singleton
class ArticleMongoRepository @Inject()(
  val reactiveMongoApi: ReactiveMongoApi
)(implicit ec: ExecutionContext) {

  // Collection reference
  private def collection: Future[BSONCollection] =
    reactiveMongoApi.database.map(_.collection[BSONCollection]("articles"))

  // CREATE
  def create(article: Article): Future[Article] = {
    val withId = article.copy(_id = Some(BSONObjectID.generate()))
    collection.flatMap(_.insert.one(withId)).map(_ => withId)
  }

  // READ by ID
  def findById(id: String): Future[Option[Article]] =
    for {
      col    <- collection
      bsonId <- Future.fromTry(BSONObjectID.parse(id))
      result <- col.find(BSONDocument("_id" -> bsonId)).one[Article]
    } yield result

  // READ by slug
  def findBySlug(slug: String): Future[Option[Article]] =
    collection.flatMap(
      _.find(BSONDocument("slug" -> slug)).one[Article]
    )

  // READ many
  def findPublished(limit: Int = 20, skip: Int = 0): Future[Seq[Article]] =
    collection.flatMap(
      _.find(BSONDocument("status" -> "published"))
        .sort(BSONDocument("publishedAt" -> -1))
        .skip(skip)
        .cursor[Article]()
        .collect[Seq](limit)
    )

  // UPDATE
  def update(id: String, article: Article): Future[UpdateWriteResult] =
    for {
      col    <- collection
      bsonId <- Future.fromTry(BSONObjectID.parse(id))
      result <- col.update.one(
        q = BSONDocument("_id" -> bsonId),
        u = BSONDocument("$set" -> BSON.writeDocument(article.copy(
          updatedAt = java.time.Instant.now()
        )))
      )
    } yield result

  // PARTIAL UPDATE
  def publish(id: String): Future[UpdateWriteResult] =
    for {
      col    <- collection
      bsonId <- Future.fromTry(BSONObjectID.parse(id))
      now    = java.time.Instant.now()
      result <- col.update.one(
        q = BSONDocument("_id" -> bsonId),
        u = BSONDocument("$set" -> BSONDocument(
          "status"      -> "published",
          "publishedAt" -> now,
          "updatedAt"   -> now
        ))
      )
    } yield result

  def incrementViewCount(id: String): Future[UpdateWriteResult] =
    for {
      col    <- collection
      bsonId <- Future.fromTry(BSONObjectID.parse(id))
      result <- col.update.one(
        q = BSONDocument("_id" -> bsonId),
        u = BSONDocument(
          "$inc" -> BSONDocument("viewCount" -> 1),
          "$set" -> BSONDocument("updatedAt"  -> java.time.Instant.now())
        )
      )
    } yield result

  // DELETE
  def delete(id: String): Future[DeleteWriteResult] =
    for {
      col    <- collection
      bsonId <- Future.fromTry(BSONObjectID.parse(id))
      result <- col.delete.one(BSONDocument("_id" -> bsonId))
    } yield result
}
```

---

## Step 645: Complex Queries

```scala
// app/repositories/ArticleMongoQueries.scala
package repositories

import reactivemongo.api.bson.*
import reactivemongo.api.bson.collection.*
import scala.concurrent.*

class ArticleMongoQueries(collection: BSONCollection)(implicit ec: ExecutionContext) {

  // Find by tags (any)
  def findByAnyTag(tags: List[String]): Future[Seq[Article]] =
    collection
      .find(BSONDocument("tags" -> BSONDocument("$in" -> tags)))
      .sort(BSONDocument("publishedAt" -> -1))
      .cursor[Article]()
      .collect[Seq](20)

  // Find by all tags
  def findByAllTags(tags: List[String]): Future[Seq[Article]] =
    collection
      .find(BSONDocument("tags" -> BSONDocument("$all" -> tags)))
      .cursor[Article]()
      .collect[Seq](20)

  // Text search (requires text index)
  def search(query: String, limit: Int = 20): Future[Seq[Article]] =
    collection
      .find(BSONDocument(
        "$text"   -> BSONDocument("$search" -> query),
        "status"  -> "published"
      ))
      .sort(BSONDocument("score" -> BSONDocument("$meta" -> "textScore")))
      .cursor[Article]()
      .collect[Seq](limit)

  // Date range query
  def findInDateRange(from: java.time.Instant, to: java.time.Instant): Future[Seq[Article]] =
    collection
      .find(BSONDocument(
        "publishedAt" -> BSONDocument(
          "$gte" -> from,
          "$lte" -> to
        ),
        "status" -> "published"
      ))
      .cursor[Article]()
      .collect[Seq]()

  // Nested document query
  def findByAuthor(authorId: String): Future[Seq[Article]] =
    collection
      .find(BSONDocument("author.id" -> authorId))
      .sort(BSONDocument("createdAt" -> -1))
      .cursor[Article]()
      .collect[Seq](50)

  // Array contains
  def findByTag(tag: String): Future[Seq[Article]] =
    collection
      .find(BSONDocument("tags" -> tag))
      .cursor[Article]()
      .collect[Seq](20)
}
```

---

## Step 646: Aggregation Pipeline

```scala
// app/repositories/ArticleAggregation.scala
package repositories

import reactivemongo.api.bson.*
import reactivemongo.api.bson.collection.*
import scala.concurrent.*

class ArticleAggregation(collection: BSONCollection)(implicit ec: ExecutionContext) {

  import collection.AggregationFramework.*

  // Tag statistics
  def tagStats(): Future[Seq[(String, Int)]] = {
    collection.aggregateWith[BSONDocument]() { framework =>
      import framework.*

      val pipeline = List(
        Match(BSONDocument("status" -> "published")),
        Unwind(field = "tags"),
        Group(BSONString("$tags"))(
          "count" -> SumAll
        ),
        Sort(Descending("count")),
        Limit(20)
      )

      pipeline.head -> pipeline.tail
    }
    .collect[Seq]()
    .map(_.map { doc =>
      val tag   = doc.getAsOpt[String]("_id").getOrElse("")
      val count = doc.getAsOpt[Int]("count").getOrElse(0)
      (tag, count)
    })
  }

  // Monthly article count
  def monthlyStats(): Future[Seq[(String, Int)]] = {
    collection.aggregateWith[BSONDocument]() { framework =>
      import framework.*

      val pipeline = List(
        Match(BSONDocument("status" -> "published")),
        Group(BSONDocument(
          "year"  -> BSONDocument("$year"  -> BSONString("$publishedAt")),
          "month" -> BSONDocument("$month" -> BSONString("$publishedAt"))
        ))(
          "count"      -> SumAll,
          "totalViews" -> SumField("viewCount")
        ),
        Sort(Descending("_id.year"), Descending("_id.month")),
        Limit(12)
      )

      pipeline.head -> pipeline.tail
    }
    .collect[Seq]()
    .map(_.map { doc =>
      val id    = doc.getAsOpt[BSONDocument]("_id").get
      val year  = id.getAsOpt[Int]("year").getOrElse(0)
      val month = id.getAsOpt[Int]("month").getOrElse(0)
      val count = doc.getAsOpt[Int]("count").getOrElse(0)
      (s"$year-${month.toString.padLeft(2, '0')}", count)
    })
  }

  // Top authors by article count
  def topAuthors(limit: Int = 10): Future[Seq[(String, String, Int)]] = {
    collection.aggregateWith[BSONDocument]() { framework =>
      import framework.*

      val pipeline = List(
        Match(BSONDocument("status" -> "published")),
        Group(BSONString("$author.id"))(
          "authorName"   -> FirstField("author.name"),
          "articleCount" -> SumAll,
          "totalViews"   -> SumField("viewCount")
        ),
        Sort(Descending("articleCount")),
        Limit(limit)
      )

      pipeline.head -> pipeline.tail
    }
    .collect[Seq]()
    .map(_.flatMap { doc =>
      for {
        authorId   <- doc.getAsOpt[String]("_id")
        authorName <- doc.getAsOpt[String]("authorName")
        count      <- doc.getAsOpt[Int]("articleCount")
      } yield (authorId, authorName, count)
    })
  }

  // Faceted search (multiple aggregations in parallel)
  def facetedSearch(query: String): Future[SearchFacets] = {
    collection.aggregateWith[BSONDocument]() { framework =>
      import framework.*

      val pipeline = List(
        Match(BSONDocument(
          "$text"  -> BSONDocument("$search" -> query),
          "status" -> "published"
        )),
        Facet(
          "tags" -> List(
            Unwind(field = "tags"),
            Group(BSONString("$tags"))("count" -> SumAll),
            Sort(Descending("count")),
            Limit(10)
          ),
          "results" -> List(
            Sort(Descending("score" -> BSONDocument("$meta" -> "textScore"))),
            Limit(20),
            Project(BSONDocument("content" -> 0))  // exclude content
          )
        )
      )

      pipeline.head -> pipeline.tail
    }
    .collect[Seq]()
    .map(_ => SearchFacets(Nil, Nil))  // parse result
  }
}

case class SearchFacets(tags: List[(String, Int)], articles: List[Article])
```

---

## Step 647: Indexes

```scala
// app/db/MongoIndexes.scala
package db

import javax.inject.*
import play.modules.reactivemongo.*
import reactivemongo.api.bson.*
import reactivemongo.api.bson.collection.*
import reactivemongo.api.indexes.{Index, IndexType}
import play.api.inject.ApplicationLifecycle
import scala.concurrent.*

@Singleton
class MongoIndexes @Inject()(
  reactiveMongoApi: ReactiveMongoApi,
  lifecycle: ApplicationLifecycle
)(implicit ec: ExecutionContext) {

  // สร้าง indexes เมื่อ app start
  ensureIndexes()

  private def ensureIndexes(): Unit = {
    reactiveMongoApi.database.foreach { db =>
      val articles = db.collection[BSONCollection]("articles")

      // Text index สำหรับ full-text search
      articles.indexesManager.ensure(
        Index(
          key     = Seq("title" -> IndexType.Text, "content" -> IndexType.Text),
          name    = Some("articles_text_idx"),
          weights = Some(BSONDocument("title" -> 10, "content" -> 1))
        )
      )

      // Status + publishedAt (compound index)
      articles.indexesManager.ensure(
        Index(
          key  = Seq("status" -> IndexType.Ascending, "publishedAt" -> IndexType.Descending),
          name = Some("articles_status_published_idx")
        )
      )

      // Author lookup
      articles.indexesManager.ensure(
        Index(
          key  = Seq("author.id" -> IndexType.Ascending),
          name = Some("articles_author_idx")
        )
      )

      // Slug (unique)
      articles.indexesManager.ensure(
        Index(
          key    = Seq("slug" -> IndexType.Ascending),
          name   = Some("articles_slug_idx"),
          unique = true
        )
      )

      // Tags
      articles.indexesManager.ensure(
        Index(
          key  = Seq("tags" -> IndexType.Ascending),
          name = Some("articles_tags_idx")
        )
      )
    }
  }
}
```

---

## Step 648: Change Streams (Real-time)

```scala
// MongoDB Change Streams สำหรับ real-time updates

import org.apache.pekko.stream.scaladsl.Source
import reactivemongo.akkastream.*

def watchArticles(): Source[Article, ?] = {
  Source.futureSource(
    reactiveMongoApi.database.map { db =>
      db.collection[BSONCollection]("articles")
        .watch[Article]()
        .source
    }
  )
}

// ใช้ใน SSE endpoint
def articleEvents(): Action[AnyContent] = Action {
  val source = watchArticles()
    .map { article =>
      val json = Json.toJson(article).toString
      s"data: $json\n\n"
    }
    .map(s => org.apache.pekko.util.ByteString(s))

  Ok.chunked(source)
    .withHeaders(
      "Content-Type"  -> "text/event-stream",
      "Cache-Control" -> "no-cache"
    )
}
```

---

## Step 649: GridFS (File Storage)

```scala
// GridFS สำหรับ store ไฟล์ขนาดใหญ่ใน MongoDB

import reactivemongo.api.gridfs.*

class MongoFileService @Inject()(
  reactiveMongoApi: ReactiveMongoApi
)(implicit ec: ExecutionContext, mat: Materializer) {

  private def gfs = reactiveMongoApi.database.map(
    GridFS[BSONSerializationPack.type](_, "files")
  )

  def save(name: String, contentType: String, data: Array[Byte]): Future[String] = {
    import org.apache.pekko.stream.scaladsl.Source
    import org.apache.pekko.util.ByteString

    val source = Source.single(ByteString(data))

    gfs.flatMap { fs =>
      fs.writeFromInputStream(
        fs.fileToSave(
          filename    = Some(name),
          contentType = Some(contentType)
        ),
        new java.io.ByteArrayInputStream(data)
      ).map(_._id.toString)
    }
  }

  def find(id: String): Future[Option[Array[Byte]]] = {
    gfs.flatMap { fs =>
      BSONObjectID.parse(id).toOption match {
        case None => Future.successful(None)
        case Some(bsonId) =>
          fs.find[BSONDocument, _](
            BSONDocument("_id" -> bsonId)
          ).headOption.flatMap {
            case None       => Future.successful(None)
            case Some(file) =>
              val baos = new java.io.ByteArrayOutputStream()
              fs.readToOutputStream(file, baos).map(_ => Some(baos.toByteArray))
          }
      }
    }
  }
}
```

---

## Step 650: Testing ReactiveMongo

```scala
// test/repositories/ArticleMongoSpec.scala
package repositories

import org.scalatestplus.play.*
import org.scalatestplus.play.guice.GuiceOneAppPerTest
import play.api.test.*
import models.mongo.*
import scala.concurrent.*
import scala.concurrent.duration.*

class ArticleMongoSpec extends PlaySpec with GuiceOneAppPerTest {

  "ArticleMongoRepository" should {
    "create and find article" in {
      val repo = app.injector.instanceOf[ArticleMongoRepository]

      val article = Article(
        title   = "Test Article",
        slug    = s"test-${System.currentTimeMillis()}",
        content = "Test content",
        author  = AuthorRef("user-1", "Test Author")
      )

      val result = for {
        created <- repo.create(article)
        found   <- repo.findBySlug(article.slug)
        _       <- repo.delete(created._id.get.stringify)
      } yield found

      val found = Await.result(result, 5.seconds)
      found must be(defined)
      found.get.title mustBe "Test Article"
    }

    "increment view count" in {
      val repo = app.injector.instanceOf[ArticleMongoRepository]
      val slug = s"views-${System.currentTimeMillis()}"

      val result = for {
        created <- repo.create(Article("View Test", slug, "Content", None, AuthorRef("u1", "User")))
        id       = created._id.get.stringify
        _       <- repo.incrementViewCount(id)
        _       <- repo.incrementViewCount(id)
        found   <- repo.findById(id)
        _       <- repo.delete(id)
      } yield found

      val found = Await.result(result, 5.seconds)
      found.get.viewCount mustBe 2
    }
  }
}
```

---

## สรุป Part 65

| Operation | ReactiveMongo API | MongoDB Operator |
|----------|------------------|-----------------|
| INSERT | `collection.insert.one(doc)` | insertOne |
| SELECT | `.find(query).one[T]` / `.cursor.collect` | find |
| UPDATE | `.update.one(q, u)` | updateOne |
| PARTIAL UPDATE | `$set`, `$inc`, `$push` | update operators |
| DELETE | `.delete.one(query)` | deleteOne |
| Text search | `$text: {$search: "..."}` | text index |
| Aggregation | `.aggregateWith[T]` | aggregate pipeline |
| Change stream | `.watch[T]().source` | changeStream |
| Indexes | `indexesManager.ensure` | createIndex |
| Array query | `$in`, `$all`, direct match | array operators |

---

## แบบฝึกหัด Part 65

1. **Content CMS**: สร้าง CMS backend ด้วย MongoDB ที่ support flexible content types (blog post, product, page) แต่ละ type มี schema ต่างกัน

2. **Aggregation Dashboard**: Build analytics dashboard ที่ use MongoDB aggregation pipeline เพื่อแสดง page views, top content, และ user engagement

3. **Full-Text Search**: Implement full-text search ด้วย MongoDB text indexes และ Atlas Search (ถ้า available) พร้อม relevance scoring

4. **Change Streams**: สร้าง real-time notification system ที่ listen MongoDB change streams และ push updates ไปยัง connected clients ผ่าน SSE

5. **Migration PostgreSQL→MongoDB**: Migrate existing relational blog schema ไปยัง MongoDB พร้อม document denormalization strategy

---

[→ ไปยัง Part 66: Redis with Scala](part-66-redis-with-scala.md)
