# Part 62: Slick Queries

## Steps 611-620: Complex Queries, Filtering, Sorting, GroupBy, Aggregates, Raw SQL

---

## Step 611: Query Composition

```scala
// Slick queries สามารถ compose ได้เหมือน collections

import MyPostgresProfile.api.*
import models.tables.*

val allArticles = ArticlesTable.query

// Filter
val published = allArticles.filter(_.status === "published")

// Sort
val byDate = published.sortBy(_.publishedAt.desc)

// Paginate
val page1 = byDate.take(10)

// Compose: query เป็น val ที่ reuse ได้
val baseQuery = allArticles
  .filter(_.status === "published")
  .sortBy(_.publishedAt.desc.nullsLast)

// เพิ่ม filter ได้ทีหลัง
def withTag(tag: String) = baseQuery.filter(_.tags @> List(tag))
def withSearch(q: String) = baseQuery.filter(a =>
  a.title.toLowerCase like s"%${q.toLowerCase}%"
)
```

---

## Step 612: Join Queries

```scala
// Inner Join
val articlesWithAuthors =
  ArticlesTable.query
    .join(UsersTable.query)
    .on(_.authorId === _.id)
    .filter { case (article, _) => article.status === "published" }
    .sortBy { case (article, _) => article.publishedAt.desc }
    .map { case (article, author) => (article, author) }

// Left Join (article + optional comment count)
val articlesWithCommentCounts =
  ArticlesTable.query
    .joinLeft(CommentsTable.query)
    .on(_.id === _.articleId)
    .groupBy { case (article, _) => article }
    .map { case (article, group) =>
      (article, group.map(_._2).length)
    }

// Multiple Joins
val articlesFullInfo =
  for {
    article  <- ArticlesTable.query if article.status === "published"
    author   <- UsersTable.query    if author.id === article.authorId
  } yield (article, author)

// Cross Join with filter (for-comprehension style)
val recentArticlesByUser = for {
  user    <- UsersTable.query
  article <- ArticlesTable.query
  if article.authorId === user.id
  if article.status === "published"
} yield (user.name, article.title, article.publishedAt)
```

---

## Step 613: Aggregate Functions

```scala
// Count
val articleCount = ArticlesTable.query.length.result
// SQL: SELECT COUNT(*) FROM articles

// Count with filter
val publishedCount = ArticlesTable.query.filter(_.status === "published").length.result

// Sum
val totalViews = ArticlesTable.query.map(_.viewCount).sum.result
// SQL: SELECT SUM(view_count) FROM articles

// Average
val avgViews = ArticlesTable.query.map(_.viewCount.asColumnOf[Double]).avg.result

// Max/Min
val maxViews = ArticlesTable.query.map(_.viewCount).max.result
val minViews = ArticlesTable.query.map(_.viewCount).min.result

// GroupBy + Aggregate
val viewsByStatus = ArticlesTable.query
  .groupBy(_.status)
  .map { case (status, group) =>
    (status, group.length, group.map(_.viewCount).sum)
  }
  .result
// SQL: SELECT status, COUNT(*), SUM(view_count) FROM articles GROUP BY status

// GroupBy with Having
val popularAuthors = ArticlesTable.query
  .filter(_.status === "published")
  .groupBy(_.authorId)
  .map { case (authorId, group) =>
    (authorId, group.length)
  }
  .filter { case (_, count) => count >= 5 }
  .sortBy { case (_, count) => count.desc }
  .result
// SQL: SELECT author_id, COUNT(*) FROM articles
//      WHERE status = 'published'
//      GROUP BY author_id HAVING COUNT(*) >= 5
//      ORDER BY COUNT(*) DESC
```

---

## Step 614: Sub-queries

```scala
// Sub-query: articles by prolific authors
val prolificAuthorIds = ArticlesTable.query
  .filter(_.status === "published")
  .groupBy(_.authorId)
  .map { case (authorId, group) => (authorId, group.length) }
  .filter { case (_, count) => count >= 3 }
  .map { case (authorId, _) => authorId }

val articlesByProlificAuthors = ArticlesTable.query
  .filter(_.authorId in prolificAuthorIds)
  .result

// Exists sub-query
val usersWithArticles = UsersTable.query
  .filter { user =>
    ArticlesTable.query
      .filter(a => a.authorId === user.id && a.status === "published")
      .exists
  }
  .result

// Correlated sub-query: user's latest article
val usersWithLatestArticle =
  UsersTable.query
    .map { user =>
      val latestArticle = ArticlesTable.query
        .filter(_.authorId === user.id)
        .sortBy(_.createdAt.desc)
        .take(1)
        .map(_.title)
      (user, latestArticle.result.headOption)
    }
```

---

## Step 615: Dynamic Queries

```scala
// app/repositories/ArticleRepository.scala

case class ArticleFilter(
  status: Option[String] = None,
  authorId: Option[String] = None,
  tag: Option[String] = None,
  search: Option[String] = None,
  fromDate: Option[java.time.Instant] = None,
  toDate: Option[java.time.Instant] = None
)

case class SortOrder(field: String, direction: String = "desc")

def findWithFilter(
  filter: ArticleFilter,
  sort: SortOrder = SortOrder("publishedAt"),
  limit: Int = 20,
  offset: Int = 0
): Future[Seq[Article]] = {

  var query = ArticlesTable.query.asInstanceOf[Query[ArticlesTable, Article, Seq]]

  // Dynamic filters
  filter.status.foreach(s => query = query.filter(_.status === s))
  filter.authorId.foreach(id => query = query.filter(_.authorId === id))
  filter.tag.foreach(t => query = query.filter(_.tags @> List(t)))
  filter.search.foreach { term =>
    val lower = term.toLowerCase
    query = query.filter(a =>
      (a.title.toLowerCase like s"%$lower%") ||
      (a.content.toLowerCase like s"%$lower%")
    )
  }
  filter.fromDate.foreach(d => query = query.filter(_.createdAt >= d))
  filter.toDate.foreach(d => query = query.filter(_.createdAt <= d))

  // Dynamic sorting
  val sorted = sort match {
    case SortOrder("publishedAt", "asc")  => query.sortBy(_.publishedAt.asc.nullsFirst)
    case SortOrder("publishedAt", _)      => query.sortBy(_.publishedAt.desc.nullsLast)
    case SortOrder("viewCount", "asc")    => query.sortBy(_.viewCount.asc)
    case SortOrder("viewCount", _)        => query.sortBy(_.viewCount.desc)
    case SortOrder("title", "asc")        => query.sortBy(_.title.asc)
    case SortOrder("title", _)            => query.sortBy(_.title.desc)
    case _                                => query.sortBy(_.createdAt.desc)
  }

  db.run(sorted.drop(offset).take(limit).result)
}
```

---

## Step 616: Raw SQL

```scala
import slick.jdbc.PostgresProfile.api.*

// Raw SQL Query (typed)
def findByFullText(searchTerm: String): Future[Seq[Article]] = {
  val query = sql"""
    SELECT id, title, content, summary, author_id, status, tags,
           view_count, published_at, created_at, updated_at
    FROM articles
    WHERE to_tsvector('english', title || ' ' || content) @@
          plainto_tsquery('english', $searchTerm)
    AND status = 'published'
    ORDER BY ts_rank(
      to_tsvector('english', title || ' ' || content),
      plainto_tsquery('english', $searchTerm)
    ) DESC
    LIMIT 20
  """.as[Article]  // ต้องมี implicit GetResult[Article]

  db.run(query)
}

// Raw SQL Update
def bulkUpdateStatus(ids: Seq[String], status: String): Future[Int] = {
  val idsStr = ids.map(id => s"'$id'").mkString(",")
  db.run(
    sqlu"UPDATE articles SET status = $status WHERE id IN (#$idsStr)"
  )
}

// Stored Procedure
def callStoredProcedure(articleId: String): Future[Unit] = {
  db.run(sqlu"CALL update_article_stats($articleId)").map(_ => ())
}

// คำเตือน: ระวัง SQL Injection ใน #$variable (unescaped)
// ใช้ $variable (escaped) เสมอสำหรับ user input
```

---

## Step 617: Bulk Operations

```scala
// Batch Insert
def bulkInsert(articles: Seq[Article]): Future[Int] =
  db.run(ArticlesTable.query ++= articles)

// Batch Update ด้วย CASE expression
def bulkPublish(ids: Seq[String]): Future[Int] = {
  val now = java.time.Instant.now()
  db.run(
    ArticlesTable.query
      .filter(_.id inSet ids)
      .map(a => (a.status, a.publishedAt, a.updatedAt))
      .update(("published", Some(now), now))
  )
}

// Batch Upsert (INSERT ON CONFLICT)
def upsertMany(articles: Seq[Article]): Future[Int] = {
  val actions = articles.map(a => ArticlesTable.query.insertOrUpdate(a))
  db.run(DBIO.sequence(actions).map(_.sum))
}

// Efficient bulk upsert ด้วย Raw SQL
def bulkUpsert(articles: Seq[Article]): Future[Int] = {
  val values = articles.map(a =>
    s"('${a.id}', '${a.title}', '${a.authorId}', '${a.status}')"
  ).mkString(",\n")

  db.run(
    sqlu"""
      INSERT INTO articles (id, title, author_id, status)
      VALUES #$values
      ON CONFLICT (id) DO UPDATE SET
        title  = EXCLUDED.title,
        status = EXCLUDED.status,
        updated_at = NOW()
    """
  )
}
```

---

## Step 618: Window Functions

```scala
// PostgreSQL Window Functions ด้วย Raw SQL

// Rank articles by view count within each author
def rankArticlesByViews(): Future[Seq[(String, String, Int, Int)]] = {
  val query = sql"""
    SELECT
      id,
      title,
      view_count,
      RANK() OVER (PARTITION BY author_id ORDER BY view_count DESC) as rank
    FROM articles
    WHERE status = 'published'
  """.as[(String, String, Int, Int)]

  db.run(query)
}

// Running total of views
def runningTotalViews(): Future[Seq[(String, Int, Long)]] = {
  val query = sql"""
    SELECT
      id,
      view_count,
      SUM(view_count) OVER (ORDER BY published_at) as running_total
    FROM articles
    WHERE status = 'published'
    ORDER BY published_at
  """.as[(String, Int, Long)]

  db.run(query)
}

// Lead/Lag for next/prev article
def getArticleWithNavigation(articleId: String): Future[Option[(Article, Option[String], Option[String])]] = {
  val query = sql"""
    SELECT
      id, title, content, summary, author_id, status, tags,
      view_count, published_at, created_at, updated_at,
      LAG(id) OVER (ORDER BY published_at)  as prev_id,
      LEAD(id) OVER (ORDER BY published_at) as next_id
    FROM articles
    WHERE status = 'published'
  """.as[(Article, Option[String], Option[String])]

  db.run(query).map(_.find(_._1.id == articleId))
}
```

---

## Step 619: Streaming Large Result Sets

```scala
import org.apache.pekko.stream.scaladsl.Source

// Stream large result sets แทน loading ทั้งหมดใน memory
def streamAllArticles(): Source[Article, ?] = {
  val publisher = db.stream(
    ArticlesTable.query
      .filter(_.status === "published")
      .result
      .withStatementParameters(fetchSize = 1000)  // fetch 1000 rows at a time
      .transactionally
  )
  Source.fromPublisher(publisher)
}

// ใช้ใน controller
def exportArticles(): Action[AnyContent] = Action {
  val source = articleRepo.streamAllArticles()
    .map(article => Json.toJson(article).toString + "\n")
    .map(s => org.apache.pekko.util.ByteString(s))

  Ok.chunked(source)
    .withHeaders("Content-Disposition" -> "attachment; filename=articles.jsonl")
}
```

---

## Step 620: Query Optimization Tips

```scala
// 1. Use indexes
// CREATE INDEX articles_author_idx ON articles(author_id);
// CREATE INDEX articles_status_published_at ON articles(status, published_at DESC NULLS LAST);

// 2. Select only needed columns
val titlesOnly = ArticlesTable.query
  .filter(_.status === "published")
  .map(a => (a.id, a.title, a.publishedAt))
  .result

// 3. Limit result size
val limited = ArticlesTable.query.take(100).result

// 4. Avoid N+1: batch load instead
def loadArticlesWithAuthors(ids: Seq[String]): Future[Seq[(Article, User)]] = {
  val q = ArticlesTable.query
    .filter(_.id inSet ids)
    .join(UsersTable.query)
    .on(_.authorId === _.id)
    .result
  db.run(q)
}

// 5. Use EXISTS instead of IN for sub-queries when possible
val usersWithPosts = UsersTable.query
  .filter(u =>
    ArticlesTable.query.filter(_.authorId === u.id).exists
  ).result

// 6. Explain Analyze
def analyzeQuery(query: String): Future[Seq[String]] = {
  db.run(sql"EXPLAIN ANALYZE #$query".as[String])
}
```

---

## สรุป Part 62

| Query Type | Slick API | SQL Equivalent |
|-----------|----------|----------------|
| Filter | `.filter(_.col === val)` | `WHERE col = val` |
| Sort | `.sortBy(_.col.desc)` | `ORDER BY col DESC` |
| Paginate | `.drop(n).take(m)` | `OFFSET n LIMIT m` |
| Inner join | `.join(t2).on(...)` | `INNER JOIN` |
| Left join | `.joinLeft(t2).on(...)` | `LEFT JOIN` |
| Count | `.length.result` | `SELECT COUNT(*)` |
| Group by | `.groupBy(_.col)` | `GROUP BY` |
| Aggregate | `.map(_.sum)` | `SUM(...)` |
| Sub-query | `query in subQuery` | `IN (SELECT ...)` |
| Raw SQL | `sql"..."` / `sqlu"..."` | Direct SQL |
| Bulk insert | `table ++= seq` | `INSERT INTO ... VALUES` |
| Stream | `db.stream(query)` | Cursor-based |

---

## แบบฝึกหัด Part 62

1. **Dashboard Queries**: สร้าง analytics queries สำหรับ admin dashboard: total users, articles per month, top authors, trending tags

2. **Full-Text Search**: Implement full-text search ใน PostgreSQL ด้วย `to_tsvector` และ `plainto_tsquery` พร้อม relevance ranking

3. **Complex Reporting**: สร้าง report query ที่ join 4+ tables, ใช้ aggregates, และ window functions เพื่อ generate monthly statistics

4. **Batch Processing**: Implement data migration script ที่ process records ใน batches of 1000 ด้วย streaming เพื่อหลีกเลี่ยง OOM

5. **Query Optimizer**: Analyze slow queries ด้วย EXPLAIN ANALYZE และ add appropriate indexes เพื่อ reduce query time

---

[→ ไปยัง Part 63: Slick Relationships](part-63-slick-relationships.md)
