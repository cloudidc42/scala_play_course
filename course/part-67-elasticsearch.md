# Part 67: Elasticsearch with Scala

## Steps 661-670: Elasticsearch Client, Indexing, Searching, Aggregations, Full-Text Search

---

## Step 661: Elasticsearch คืออะไร

Elasticsearch คือ distributed search and analytics engine ที่ใช้ Apache Lucene

```
Elasticsearch ใช้เมื่อ:
├── Full-text search (complex queries)
├── Log analytics (ELK stack)
├── E-commerce product search
├── Real-time analytics
└── Geospatial search

ต่างจาก PostgreSQL full-text:
├── Better relevance scoring (BM25)
├── Distributed (scale horizontally)
├── Near real-time (NRT)
├── Rich aggregations
└── Autocomplete, fuzzy, multi-language
```

### build.sbt

```scala
libraryDependencies ++= Seq(
  // Official Java High Level REST Client
  "co.elastic.clients" % "elasticsearch-java" % "8.13.4",

  // JSON Mapper
  "com.fasterxml.jackson.core" % "jackson-databind" % "2.17.1",

  // หรือ Elastic4s (Scala DSL)
  "com.sksamuel.elastic4s" %% "elastic4s-client-esjava" % "8.12.2",
  "com.sksamuel.elastic4s" %% "elastic4s-core"          % "8.12.2"
)
```

---

## Step 662: Client Setup

```scala
// app/db/ElasticsearchClient.scala
package db

import co.elastic.clients.elasticsearch.ElasticsearchAsyncClient
import co.elastic.clients.json.jackson.JacksonJsonpMapper
import co.elastic.clients.transport.rest_client.RestClientTransport
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.module.scala.DefaultScalaModule
import javax.inject.*
import org.apache.http.HttpHost
import org.elasticsearch.client.RestClient
import play.api.Configuration
import play.api.inject.ApplicationLifecycle
import scala.concurrent.Future

@Singleton
class ElasticsearchClient @Inject()(
  config: Configuration,
  lifecycle: ApplicationLifecycle
) {
  private val host = config.getOrElse("elasticsearch.host", "localhost")
  private val port = config.getOrElse("elasticsearch.port", 9200)

  private val restClient = RestClient.builder(new HttpHost(host, port)).build()

  private val mapper = new ObjectMapper().registerModule(DefaultScalaModule)
  private val transport = new RestClientTransport(restClient, new JacksonJsonpMapper(mapper))

  val client: ElasticsearchAsyncClient = new ElasticsearchAsyncClient(transport)

  lifecycle.addStopHook(() => Future.successful {
    transport.close()
    restClient.close()
  })
}
```

---

## Step 663: Index Mapping (Schema)

```scala
// app/search/ArticleIndex.scala
package search

import co.elastic.clients.elasticsearch.*
import co.elastic.clients.elasticsearch.indices.*
import co.elastic.clients.elasticsearch.indices.PutMappingRequest
import db.ElasticsearchClient
import javax.inject.*
import scala.concurrent.*
import scala.jdk.FutureConverters.*

@Singleton
class ArticleIndex @Inject()(
  esClient: ElasticsearchClient
)(implicit ec: ExecutionContext) {

  val IndexName = "articles"

  // Create index with mapping
  def createIndex(): Future[Unit] = {
    val client = esClient.client

    // Check if index exists
    client.indices().exists(_.index(IndexName)).asScala.flatMap { exists =>
      if (exists.value()) Future.successful(())
      else {
        client.indices().create { req =>
          req.index(IndexName)
            .settings { s =>
              s.numberOfShards("1")
               .numberOfReplicas("0")
               .analysis { a =>
                 a.analyzer("thai_analyzer", _ =>
                    _.custom(c => c.tokenizer("standard")
                                   .filter("lowercase", "asciifolding"))
                 )
               }
            }
            .mappings { m =>
              m.properties("title", p =>
                p.text(t => t.analyzer("thai_analyzer").searchAnalyzer("thai_analyzer").boost(3.0))
              )
              .properties("content", p =>
                p.text(t => t.analyzer("thai_analyzer"))
              )
              .properties("summary", p =>
                p.text(t => t.analyzer("thai_analyzer").boost(1.5))
              )
              .properties("tags", p =>
                p.keyword(k => k.normalizer("lowercase"))
              )
              .properties("authorName", p => p.keyword(k => k))
              .properties("status", p => p.keyword(k => k))
              .properties("viewCount", p => p.integer(i => i))
              .properties("publishedAt", p => p.date(d => d))
            }
        }.asScala.map(_ => ())
      }
    }
  }
}
```

---

## Step 664: Indexing Documents

```scala
// app/search/ArticleSearchService.scala
package search

import co.elastic.clients.elasticsearch.*
import co.elastic.clients.elasticsearch.core.*
import db.ElasticsearchClient
import javax.inject.*
import models.*
import play.api.libs.json.*
import scala.concurrent.*
import scala.jdk.FutureConverters.*

// Document ที่จะ index
case class ArticleDocument(
  id: String,
  title: String,
  content: String,
  summary: Option[String],
  authorId: String,
  authorName: String,
  tags: List[String],
  status: String,
  viewCount: Int,
  publishedAt: Option[String]
)

object ArticleDocument {
  implicit val format: OFormat[ArticleDocument] = Json.format[ArticleDocument]

  def from(article: Article, authorName: String): ArticleDocument =
    ArticleDocument(
      id          = article.id,
      title       = article.title,
      content     = article.content.take(5000),  // limit content size
      summary     = article.summary,
      authorId    = article.authorId,
      authorName  = authorName,
      tags        = article.tags,
      status      = article.status,
      viewCount   = article.viewCount,
      publishedAt = article.publishedAt.map(_.toString)
    )
}

@Singleton
class ArticleSearchService @Inject()(
  esClient: ElasticsearchClient,
  articleIndex: ArticleIndex
)(implicit ec: ExecutionContext) {

  private val client = esClient.client
  private val index  = articleIndex.IndexName

  // Index single document
  def index(doc: ArticleDocument): Future[String] = {
    client.index { req =>
      req.index(index)
         .id(doc.id)
         .document(doc)
    }.asScala.map(_.id())
  }

  // Bulk index
  def bulkIndex(docs: Seq[ArticleDocument]): Future[Unit] = {
    val operations = docs.map { doc =>
      co.elastic.clients.elasticsearch.core.bulk.BulkOperation.Builder()
        .index(i => i.index(index).id(doc.id).document(doc))
        .build()
    }

    client.bulk(b => b.operations(operations.asJava)).asScala.map { response =>
      if (response.errors()) {
        // Log errors
        println(s"Bulk index had errors")
      }
    }
  }

  // Delete document
  def delete(id: String): Future[Unit] =
    client.delete(r => r.index(index).id(id)).asScala.map(_ => ())

  // Update document
  def update(doc: ArticleDocument): Future[Unit] =
    client.update[ArticleDocument, ArticleDocument](
      r => r.index(index).id(doc.id).doc(doc),
      classOf[ArticleDocument]
    ).asScala.map(_ => ())
}
```

---

## Step 665: Search Queries

```scala
// app/search/ArticleSearchQueries.scala
package search

import co.elastic.clients.elasticsearch.*
import co.elastic.clients.elasticsearch.core.*
import co.elastic.clients.elasticsearch.core.search.*
import db.ElasticsearchClient
import javax.inject.*
import scala.concurrent.*
import scala.jdk.FutureConverters.*

@Singleton
class ArticleSearchQueries @Inject()(
  esClient: ElasticsearchClient,
  articleIndex: ArticleIndex
)(implicit ec: ExecutionContext) {

  private val client = esClient.client
  private val index  = articleIndex.IndexName

  // Full-text search
  def search(
    query: String,
    tags: Option[List[String]] = None,
    from: Int = 0,
    size: Int = 20
  ): Future[SearchResult] = {

    client.search[ArticleDocument]({ req =>
      req.index(index)
         .from(from)
         .size(size)
         .query { q =>
           q.bool { b =>
             // Must: filter by status
             b.filter(f => f.term(t => t.field("status").value("published")))

             // Should: full-text search on multiple fields
             .must(m => m.multiMatch { mm =>
               mm.query(query)
                 .fields("title^3", "summary^1.5", "content")
                 .`type`(TextQueryType.BestFields)
                 .fuzziness("AUTO")
             })

             // Tags filter (optional)
             tags.foreach { tagList =>
               b.filter(f => f.terms(t =>
                 t.field("tags").terms(tv =>
                   tv.value(tagList.map(co.elastic.clients.elasticsearch._types.FieldValue.of): _*)
                 )
               ))
             }

             b
           }
         }
         .highlight { h =>
           h.fields("title", _ => _.numberOfFragments(0))
            .fields("content", _ => _.numberOfFragments(3).fragmentSize(150))
         }
         .sort { s =>
           s.field(f => f.field("_score").order(co.elastic.clients.elasticsearch._types.SortOrder.Desc))
         }
    }, classOf[ArticleDocument]).asScala.map { response =>
      parseSearchResult(response, query)
    }
  }

  // Autocomplete (prefix search)
  def autocomplete(prefix: String): Future[Seq[String]] = {
    client.search[ArticleDocument]({ req =>
      req.index(index)
         .size(10)
         .query(q =>
           q.matchPhrasePrefix(m =>
             m.field("title").query(prefix)
           )
         )
         .source(s => s.filter(f => f.includes("title")))
    }, classOf[ArticleDocument]).asScala.map { response =>
      response.hits().hits().asScala.flatMap(h => Option(h.source()).map(_.title))
    }
  }

  // Similar articles (More Like This)
  def moreLikeThis(articleId: String, size: Int = 5): Future[Seq[ArticleDocument]] = {
    client.search[ArticleDocument]({ req =>
      req.index(index)
         .size(size)
         .query(q =>
           q.moreLikeThis { mlt =>
             mlt.like(l =>
               l.document(d => d.index(index).id(articleId))
             )
             .fields("title", "content", "tags")
             .minTermFreq(1)
             .maxQueryTerms(12)
           }
         )
    }, classOf[ArticleDocument]).asScala.map { response =>
      response.hits().hits().asScala.flatMap(h => Option(h.source()))
    }
  }

  private def parseSearchResult(response: SearchResponse[ArticleDocument], query: String): SearchResult = {
    val hits = response.hits().hits().asScala
    val total = response.hits().total().value()

    val results = hits.map { hit =>
      val doc = hit.source()
      val highlights = hit.highlight().asScala.map { case (field, frags) =>
        field -> frags.asScala.toSeq
      }.toMap

      SearchHit(
        id         = hit.id(),
        title      = highlights.get("title").flatMap(_.headOption).getOrElse(doc.title),
        summary    = highlights.get("content").flatMap(_.headOption).orElse(doc.summary),
        authorName = doc.authorName,
        tags       = doc.tags,
        score      = hit.score().toDouble,
        publishedAt = doc.publishedAt
      )
    }

    SearchResult(results.toSeq, total, query)
  }

  implicit class JavaListOps[T](list: java.util.List[T]) {
    def asScala: Seq[T] = scala.jdk.CollectionConverters.ListHasAsScala(list).asScala.toSeq
  }

  implicit class JavaMapOps[K, V](map: java.util.Map[K, V]) {
    def asScala: Map[K, V] = scala.jdk.CollectionConverters.MapHasAsScala(map).asScala.toMap
  }
}

case class SearchHit(
  id: String,
  title: String,
  summary: Option[String],
  authorName: String,
  tags: List[String],
  score: Double,
  publishedAt: Option[String]
)

case class SearchResult(
  hits: Seq[SearchHit],
  total: Long,
  query: String
)
```

---

## Step 666: Aggregations

```scala
// Search with aggregations (faceted search)
def facetedSearch(query: String, from: Int = 0, size: Int = 20): Future[FacetedResult] = {

  client.search[ArticleDocument]({ req =>
    req.index(index)
       .from(from)
       .size(size)
       .query(q => q.multiMatch { mm =>
         mm.query(query)
           .fields("title^3", "content")
           .fuzziness("AUTO")
       })
       .aggregations("tags", a => a
         .terms(t => t.field("tags").size(20))
       )
       .aggregations("authors", a => a
         .terms(t => t.field("authorName").size(10))
       )
       .aggregations("publishedByMonth", a => a
         .dateHistogram { dh =>
           dh.field("publishedAt")
             .calendarInterval(co.elastic.clients.elasticsearch._types.aggregations.CalendarInterval.Month)
             .minDocCount(1)
         }
       )
  }, classOf[ArticleDocument]).asScala.map { response =>

    // Parse tag aggregation
    val tagAgg = response.aggregations().get("tags")
      .sterms().buckets().array().asScala
      .map(b => (b.key().stringValue(), b.docCount()))

    // Parse author aggregation
    val authorAgg = response.aggregations().get("authors")
      .sterms().buckets().array().asScala
      .map(b => (b.key().stringValue(), b.docCount()))

    FacetedResult(
      total   = response.hits().total().value(),
      hits    = Seq.empty,  // parse hits similarly
      tagFacets    = tagAgg,
      authorFacets = authorAgg
    )
  }
}

case class FacetedResult(
  total: Long,
  hits: Seq[SearchHit],
  tagFacets: Seq[(String, Long)],
  authorFacets: Seq[(String, Long)]
)
```

---

## Step 667: Index Synchronization

```scala
// app/search/ArticleIndexer.scala — sync DB → ES

@Singleton
class ArticleIndexer @Inject()(
  articleRepo: ArticleRepository,
  userRepo: UserRepository,
  searchService: ArticleSearchService,
  system: ActorSystem
)(implicit ec: ExecutionContext) {

  // Full re-index (initial setup หรือ repair)
  def fullReindex(): Future[Int] = {
    for {
      articles <- articleRepo.findAll(limit = Int.MaxValue)
      authorIds = articles.map(_.authorId).distinct
      authors  <- userRepo.findByIds(authorIds)
      authorMap = authors.map(u => u.id -> u.name).toMap
      docs      = articles.map(a => ArticleDocument.from(a, authorMap.getOrElse(a.authorId, "Unknown")))
      _        <- searchService.bulkIndex(docs)
    } yield docs.size
  }

  // Real-time indexing (call after article save/update)
  def indexArticle(articleId: String): Future[Unit] = {
    for {
      articleOpt <- articleRepo.findById(articleId)
      result     <- articleOpt match {
        case None =>
          searchService.delete(articleId)
        case Some(article) =>
          for {
            authorOpt <- userRepo.findById(article.authorId)
            authorName = authorOpt.map(_.name).getOrElse("Unknown")
            doc        = ArticleDocument.from(article, authorName)
            _         <- searchService.index(doc)
          } yield ()
      }
    } yield result
  }
}
```

---

## Step 668: Search Controller

```scala
// app/controllers/SearchController.scala
package controllers

import javax.inject.*
import play.api.libs.json.*
import play.api.mvc.*
import search.*
import scala.concurrent.*

@Singleton
class SearchController @Inject()(
  val controllerComponents: ControllerComponents,
  searchQueries: ArticleSearchQueries,
  implicit val ec: ExecutionContext
) extends BaseController {

  def search(
    q: String,
    tags: Option[String] = None,
    page: Int = 1,
    pageSize: Int = 20
  ): Action[AnyContent] = Action.async {
    val from     = (page - 1) * pageSize
    val tagsList = tags.map(_.split(",").toList.map(_.trim).filter(_.nonEmpty))

    searchQueries.search(q, tagsList, from, pageSize).map { result =>
      Ok(Json.obj(
        "query"     -> q,
        "total"     -> result.total,
        "page"      -> page,
        "pageSize"  -> pageSize,
        "hits"      -> result.hits.map(h => Json.obj(
          "id"        -> h.id,
          "title"     -> h.title,
          "summary"   -> h.summary,
          "author"    -> h.authorName,
          "tags"      -> h.tags,
          "score"     -> h.score
        ))
      ))
    }
  }

  def autocomplete(q: String): Action[AnyContent] = Action.async {
    searchQueries.autocomplete(q).map { suggestions =>
      Ok(Json.toJson(suggestions))
    }
  }

  def similar(id: String): Action[AnyContent] = Action.async {
    searchQueries.moreLikeThis(id).map { docs =>
      Ok(Json.toJson(docs.map(d => Json.obj(
        "id"     -> d.id,
        "title"  -> d.title,
        "author" -> d.authorName
      ))))
    }
  }
}
```

---

## Step 669: Index Management

```scala
// app/search/IndexManagement.scala

def createAlias(indexName: String, alias: String): Future[Unit] =
  client.indices().updateAliases { req =>
    req.actions(a => a.add(add => add.index(indexName).alias(alias)))
  }.asScala.map(_ => ())

// Zero-downtime reindex ด้วย aliases
def reindexWithZeroDowntime(): Future[Unit] = {
  val newIndex = s"articles_${System.currentTimeMillis()}"
  val alias    = "articles_current"

  for {
    // 1. Create new index
    _ <- articleIndex.createIndex()  // creates newIndex

    // 2. Index all documents to new index
    _ <- fullReindex()

    // 3. Switch alias
    _ <- client.indices().updateAliases { req =>
      req.actions(
        a => a.remove(r => r.alias(alias)),  // remove from old
        a => a.add(add => add.index(newIndex).alias(alias))  // add to new
      )
    }.asScala
  } yield ()
}
```

---

## Step 670: Testing Elasticsearch

```scala
// test/search/ArticleSearchSpec.scala
class ArticleSearchSpec extends PlaySpec with GuiceOneAppPerTest {

  "ArticleSearchService" should {
    "index and search articles" in {
      val service = app.injector.instanceOf[ArticleSearchService]
      val queries = app.injector.instanceOf[ArticleSearchQueries]

      val doc = ArticleDocument(
        id         = java.util.UUID.randomUUID().toString,
        title      = "Introduction to Scala",
        content    = "Scala is a functional programming language",
        summary    = Some("Learn Scala basics"),
        authorId   = "user-1",
        authorName = "John Doe",
        tags       = List("scala", "programming"),
        status     = "published",
        viewCount  = 100,
        publishedAt = Some(java.time.Instant.now().toString)
      )

      val result = for {
        _       <- service.index(doc)
        _       <- Future { Thread.sleep(1000) }  // ES is near real-time
        results <- queries.search("Scala")
        _       <- service.delete(doc.id)
      } yield results

      val results = Await.result(result, 10.seconds)
      results.hits must not be empty
    }
  }
}
```

---

## สรุป Part 67

| Operation | Elasticsearch API | ตัวอย่าง |
|----------|------------------|---------|
| Create index | `client.indices().create()` | With mapping |
| Index doc | `client.index()` | With document |
| Bulk index | `client.bulk()` | Multiple docs |
| Delete doc | `client.delete()` | By id |
| Full-text search | `multi_match` query | title, content |
| Fuzzy search | `.fuzziness("AUTO")` | Typo tolerance |
| Autocomplete | `matchPhrasePrefix` | Prefix search |
| Filters | `bool.filter` | Exact match |
| Aggregations | `.aggregations()` | Facets, stats |
| Highlight | `.highlight()` | Search snippets |
| Similar docs | `moreLikeThis` | Recommendation |
| Aliases | `updateAliases` | Zero-downtime |

---

## แบบฝึกหัด Part 67

1. **Product Search**: สร้าง e-commerce product search ด้วย faceted filtering (price range, category, brand), sorting, และ pagination

2. **Log Analytics**: Implement log aggregation system ที่ index application logs ใน ES และ query ด้วย time range, log level, และ error patterns

3. **Multilingual Search**: Configure analyzers สำหรับ Thai, English, Chinese ใน same index ด้วย language-specific tokenizers

4. **Search Suggestions**: Implement "Did you mean?" suggestion ด้วย Elasticsearch phrase suggester และ term suggester

5. **Real-time Sync**: สร้าง CDC (Change Data Capture) pipeline ที่ listen PostgreSQL changes และ sync ไปยัง Elasticsearch แบบ real-time

---

[→ ไปยัง Part 68: Database Migrations](part-68-database-migrations.md)
