# Part 63: Slick Relationships

## Steps 621-630: One-to-Many, Many-to-Many, Joins, Eager/Lazy Loading, Foreign Keys

---

## Step 621: One-to-One Relationship

```scala
// User ↔ UserProfile (1:1)

case class UserProfile(
  userId: String,      // FK to users.id
  bio: Option[String],
  avatarUrl: Option[String],
  website: Option[String],
  location: Option[String],
  twitterHandle: Option[String]
)

class UserProfilesTable(tag: Tag) extends Table[UserProfile](tag, "user_profiles") {
  def userId        = column[String]("user_id", O.PrimaryKey)
  def bio           = column[Option[String]]("bio")
  def avatarUrl     = column[Option[String]]("avatar_url")
  def website       = column[Option[String]]("website")
  def location      = column[Option[String]]("location")
  def twitterHandle = column[Option[String]]("twitter_handle")

  // FK constraint
  def userFk = foreignKey("user_profiles_user_fk", userId, UsersTable.query)(
    _.id,
    onDelete = ForeignKeyAction.Cascade  // ลบ profile เมื่อ user ถูกลบ
  )

  def * = (userId, bio, avatarUrl, website, location, twitterHandle).mapTo[UserProfile]
}

object UserProfilesTable {
  val query = TableQuery[UserProfilesTable]
}

// Query: User พร้อม Profile
def findUserWithProfile(userId: String): Future[Option[(User, Option[UserProfile])]] =
  db.run(
    UsersTable.query
      .filter(_.id === userId)
      .joinLeft(UserProfilesTable.query)
      .on(_.id === _.userId)
      .result.headOption
  )
```

---

## Step 622: One-to-Many Relationship

```scala
// User ↔ Articles (1:N)

// Query: user's articles
def findArticlesByUser(userId: String): Future[Seq[Article]] =
  db.run(
    ArticlesTable.query
      .filter(_.authorId === userId)
      .sortBy(_.createdAt.desc)
      .result
  )

// Query: user with article count
def findUsersWithArticleCount(): Future[Seq[(User, Int)]] =
  db.run(
    UsersTable.query
      .joinLeft(ArticlesTable.query)
      .on(_.id === _.authorId)
      .groupBy { case (user, _) => user }
      .map { case (user, group) =>
        (user, group.map(_._2).length)
      }
      .result
  )

// Query: articles with author (N:1 — many articles, one author)
case class ArticleWithAuthor(article: Article, author: User)

def findPublishedWithAuthor(limit: Int = 20, offset: Int = 0): Future[Seq[ArticleWithAuthor]] =
  db.run(
    ArticlesTable.query
      .filter(_.status === "published")
      .join(UsersTable.query)
      .on(_.authorId === _.id)
      .sortBy { case (a, _) => a.publishedAt.desc.nullsLast }
      .drop(offset)
      .take(limit)
      .result
  ).map(_.map { case (a, u) => ArticleWithAuthor(a, u) })
```

---

## Step 623: One-to-Many (Comments)

```scala
// Article ↔ Comments (1:N)

case class Comment(
  id: String,
  articleId: String,
  authorId: String,
  content: String,
  parentId: Option[String] = None,  // Self-referential: nested comments
  createdAt: java.time.Instant = java.time.Instant.now()
)

class CommentsTable(tag: Tag) extends Table[Comment](tag, "comments") {
  def id        = column[String]("id", O.PrimaryKey)
  def articleId = column[String]("article_id")
  def authorId  = column[String]("author_id")
  def content   = column[String]("content")
  def parentId  = column[Option[String]]("parent_id")
  def createdAt = column[java.time.Instant]("created_at")

  def articleFk = foreignKey("comments_article_fk", articleId, ArticlesTable.query)(
    _.id, onDelete = ForeignKeyAction.Cascade
  )
  def authorFk = foreignKey("comments_author_fk", authorId, UsersTable.query)(
    _.id, onDelete = ForeignKeyAction.Restrict
  )
  def parentFk = foreignKey("comments_parent_fk", parentId, CommentsTable.query)(
    _.id.?, onDelete = ForeignKeyAction.Cascade
  )

  def articleIdx = index("comments_article_idx", articleId)

  def * = (id, articleId, authorId, content, parentId, createdAt).mapTo[Comment]
}

object CommentsTable {
  val query = TableQuery[CommentsTable]
}

// Article with comment count
def findArticleWithCommentCount(id: String): Future[Option[(Article, Int)]] =
  db.run(
    ArticlesTable.query
      .filter(_.id === id)
      .joinLeft(CommentsTable.query.filter(_.parentId.isEmpty))  // top-level only
      .on(_.id === _.articleId)
      .groupBy { case (a, _) => a }
      .map { case (a, group) => (a, group.map(_._2).length) }
      .result.headOption
  )
```

---

## Step 624: Many-to-Many Relationship

```scala
// Article ↔ Tags (M:N ผ่าน junction table)

case class Tag(
  id: String,
  name: String,
  slug: String
)

case class ArticleTag(
  articleId: String,
  tagId: String
)

class TagsTable(tag: Tag) extends Table[Tag](tag, "tags") {
  def id   = column[String]("id", O.PrimaryKey)
  def name = column[String]("name")
  def slug = column[String]("slug", O.Unique)

  def slugIdx = index("tags_slug_idx", slug, unique = true)

  def * = (id, name, slug).mapTo[Tag]
}

class ArticleTagsTable(tag: Tag) extends Table[ArticleTag](tag, "article_tags") {
  def articleId = column[String]("article_id")
  def tagId     = column[String]("tag_id")

  def pk = primaryKey("article_tags_pk", (articleId, tagId))

  def articleFk = foreignKey("at_article_fk", articleId, ArticlesTable.query)(
    _.id, onDelete = ForeignKeyAction.Cascade
  )
  def tagFk = foreignKey("at_tag_fk", tagId, TagsTable.query)(
    _.id, onDelete = ForeignKeyAction.Cascade
  )

  def * = (articleId, tagId).mapTo[ArticleTag]
}

object TagsTable       { val query = TableQuery[TagsTable] }
object ArticleTagsTable { val query = TableQuery[ArticleTagsTable] }

// Queries สำหรับ M:N
def findTagsForArticle(articleId: String): Future[Seq[Tag]] =
  db.run(
    ArticleTagsTable.query
      .filter(_.articleId === articleId)
      .join(TagsTable.query)
      .on(_.tagId === _.id)
      .map { case (_, tag) => tag }
      .result
  )

def findArticlesByTag(tagSlug: String): Future[Seq[Article]] =
  db.run(
    TagsTable.query
      .filter(_.slug === tagSlug)
      .join(ArticleTagsTable.query).on(_.id === _.tagId)
      .join(ArticlesTable.query).on(_._2.articleId === _.id)
      .filter { case ((_, _), article) => article.status === "published" }
      .map { case (_, article) => article }
      .result
  )

// Add tags to article (transaction)
def setArticleTags(articleId: String, tagIds: Seq[String]): Future[Unit] = {
  val action = for {
    _ <- ArticleTagsTable.query.filter(_.articleId === articleId).delete
    _ <- ArticleTagsTable.query ++= tagIds.map(ArticleTag(articleId, _))
  } yield ()

  db.run(action.transactionally)
}
```

---

## Step 625: Self-Referential Relationship

```scala
// Category ↔ Category (tree structure)
case class Category(
  id: String,
  name: String,
  slug: String,
  parentId: Option[String] = None,
  sortOrder: Int = 0
)

class CategoriesTable(tag: Tag) extends Table[Category](tag, "categories") {
  def id        = column[String]("id", O.PrimaryKey)
  def name      = column[String]("name")
  def slug      = column[String]("slug")
  def parentId  = column[Option[String]]("parent_id")
  def sortOrder = column[Int]("sort_order", O.Default(0))

  def parentFk = foreignKey("categories_parent_fk", parentId, CategoriesTable.query)(
    _.id.?, onDelete = ForeignKeyAction.Restrict
  )

  def * = (id, name, slug, parentId, sortOrder).mapTo[Category]
}

object CategoriesTable { val query = TableQuery[CategoriesTable] }

// Top-level categories
def findRootCategories(): Future[Seq[Category]] =
  db.run(
    CategoriesTable.query
      .filter(_.parentId.isEmpty)
      .sortBy(_.sortOrder)
      .result
  )

// Children of a category
def findChildren(parentId: String): Future[Seq[Category]] =
  db.run(
    CategoriesTable.query
      .filter(_.parentId === parentId)
      .sortBy(_.sortOrder)
      .result
  )

// Full tree ด้วย Raw SQL (recursive CTE)
def findFullTree(): Future[Seq[(Category, Int)]] = {
  val query = sql"""
    WITH RECURSIVE category_tree AS (
      SELECT id, name, slug, parent_id, sort_order, 0 as depth
      FROM categories
      WHERE parent_id IS NULL
      UNION ALL
      SELECT c.id, c.name, c.slug, c.parent_id, c.sort_order, ct.depth + 1
      FROM categories c
      JOIN category_tree ct ON c.parent_id = ct.id
    )
    SELECT id, name, slug, parent_id, sort_order, depth
    FROM category_tree
    ORDER BY depth, sort_order
  """.as[(String, String, String, Option[String], Int, Int)]

  db.run(query).map(_.map { case (id, name, slug, parentId, sortOrder, depth) =>
    (Category(id, name, slug, parentId, sortOrder), depth)
  })
}
```

---

## Step 626: Eager Loading (Avoiding N+1)

```scala
// ❌ Bad: N+1 problem
def badLoadArticles(limit: Int): Future[Seq[(Article, User)]] =
  db.run(ArticlesTable.query.take(limit).result).flatMap { articles =>
    Future.sequence(articles.map { article =>
      db.run(UsersTable.query.filter(_.id === article.authorId).result.head)
        .map(user => (article, user))
    })
  }
  // N+1 queries: 1 for articles + N for users

// ✅ Good: Single query with join
def goodLoadArticles(limit: Int): Future[Seq[(Article, User)]] =
  db.run(
    ArticlesTable.query
      .join(UsersTable.query)
      .on(_.authorId === _.id)
      .take(limit)
      .result
  )
  // 1 query with JOIN

// ✅ Good: Batch load
def batchLoadArticles(limit: Int): Future[Seq[(Article, User)]] = {
  for {
    articles <- db.run(ArticlesTable.query.take(limit).result)
    authorIds = articles.map(_.authorId).distinct
    authors  <- db.run(UsersTable.query.filter(_.id inSet authorIds).result)
    authorMap = authors.map(u => u.id -> u).toMap
  } yield articles.flatMap(a => authorMap.get(a.authorId).map(u => (a, u)))
}
// 2 queries total instead of N+1
```

---

## Step 627: Lazy Loading Pattern

```scala
// Lazy loading ใน Scala ด้วย Future
case class ArticleWithLazyAuthor(
  article: Article,
  private val loadAuthor: () => Future[User]
) {
  lazy val author: Future[User] = loadAuthor()
}

def findArticleWithLazyAuthor(id: String)(implicit ec: ExecutionContext): Future[ArticleWithLazyAuthor] = {
  db.run(ArticlesTable.query.filter(_.id === id).result.head).map { article =>
    ArticleWithLazyAuthor(
      article = article,
      loadAuthor = () => db.run(UsersTable.query.filter(_.id === article.authorId).result.head)
    )
  }
}

// หรือใช้ Future.lazy (จะ execute เมื่อ access เท่านั้น)
// แต่ในการใช้งานจริง prefer explicit loading ชัดเจนกว่า
```

---

## Step 628: Complex Domain Model

```scala
// Rich domain model: Article + Author + Tags + Comment count

case class ArticleSummary(
  id: String,
  title: String,
  summary: Option[String],
  authorName: String,
  authorAvatarUrl: Option[String],
  tags: Seq[String],
  commentCount: Int,
  viewCount: Int,
  publishedAt: Option[java.time.Instant]
)

def findArticleSummaries(limit: Int = 20, offset: Int = 0): Future[Seq[ArticleSummary]] = {
  val query = sql"""
    SELECT
      a.id,
      a.title,
      a.summary,
      u.name         AS author_name,
      p.avatar_url   AS author_avatar,
      a.tags,
      COUNT(c.id)    AS comment_count,
      a.view_count,
      a.published_at
    FROM articles a
    JOIN users u ON a.author_id = u.id
    LEFT JOIN user_profiles p ON u.id = p.user_id
    LEFT JOIN comments c ON a.id = c.article_id AND c.parent_id IS NULL
    WHERE a.status = 'published'
    GROUP BY a.id, u.name, p.avatar_url
    ORDER BY a.published_at DESC NULLS LAST
    LIMIT $limit OFFSET $offset
  """.as[(String, String, Option[String], String, Option[String], List[String], Int, Int, Option[java.time.Instant])]

  db.run(query).map(_.map { case (id, title, summary, authorName, avatar, tags, commentCount, viewCount, publishedAt) =>
    ArticleSummary(id, title, summary, authorName, avatar, tags, commentCount, viewCount, publishedAt)
  })
}
```

---

## Step 629: Cascading Operations

```scala
// Delete User + cascade ไปยัง Articles, Comments (ด้วย FK Cascade)
// กำหนดใน FK:
// def authorFk = foreignKey(...)(_.id, onDelete = ForeignKeyAction.Cascade)

// Application-level cascade (ถ้า FK ไม่ได้ตั้ง)
def deleteUserWithData(userId: String): Future[Unit] = {
  val action = for {
    // ลำดับสำคัญ: ลบ children ก่อน parent
    commentIds <- CommentsTable.query.filter(_.authorId === userId).map(_.id).result
    _          <- CommentsTable.query.filter(_.id inSet commentIds).delete
    _          <- CommentsTable.query.filter(_.authorId === userId).delete
    articleIds <- ArticlesTable.query.filter(_.authorId === userId).map(_.id).result
    _          <- ArticleTagsTable.query.filter(_.articleId inSet articleIds).delete
    _          <- ArticlesTable.query.filter(_.authorId === userId).delete
    _          <- UserProfilesTable.query.filter(_.userId === userId).delete
    _          <- UsersTable.query.filter(_.id === userId).delete
  } yield ()

  db.run(action.transactionally)
}
```

---

## Step 630: Testing Relationships

```scala
// test/repositories/RelationshipSpec.scala
class ArticleRelationshipSpec extends PlaySpec with GuiceOneAppPerTest {

  "Article-Tag M:N relationship" should {
    "correctly associate tags with articles" in {
      val articleRepo = app.injector.instanceOf[ArticleRepository]
      val tagRepo     = app.injector.instanceOf[TagRepository]

      // สร้าง test data
      val result = for {
        tag1    <- tagRepo.create(Tag(UUID.randomUUID().toString, "Scala", "scala"))
        tag2    <- tagRepo.create(Tag(UUID.randomUUID().toString, "Play", "play"))
        article <- articleRepo.create(Article(UUID.randomUUID().toString, "Test Article", "Content", None, "user-1"))
        _       <- articleRepo.setTags(article.id, Seq(tag1.id, tag2.id))
        tags    <- articleRepo.findTags(article.id)
      } yield tags

      val tags = Await.result(result, 5.seconds)
      tags must have size 2
      tags.map(_.slug) must contain allOf ("scala", "play")
    }
  }
}
```

---

## สรุป Part 63

| Relationship | Table Structure | Slick Query |
|-------------|----------------|-------------|
| One-to-One | FK in one table | `joinLeft` |
| One-to-Many | FK in "many" side | `join`, `filter(_.fk === parent.id)` |
| Many-to-One | FK in this table | `join` parent table |
| Many-to-Many | Junction table | Double `join` |
| Self-Referential | `parent_id` column | Self-join or recursive CTE |
| Eager Loading | JOIN | Single query |
| Batch Loading | `inSet` | 2 queries max |
| Cascade Delete | FK `onDelete` | DB-level or app-level |

---

## แบบฝึกหัด Part 63

1. **E-commerce Schema**: Design M:N relationships สำหรับ e-commerce: Products ↔ Categories, Orders ↔ Products (with quantity), Users ↔ Roles

2. **Forum Thread**: Implement threaded comments ด้วย self-referential relationship ที่ support infinite nesting

3. **Permission System**: สร้าง RBAC system ด้วย Users ↔ Roles (M:N) และ Roles ↔ Permissions (M:N) พร้อม efficient permission checking

4. **Batch Eager Load**: Refactor N+1 queries ใน existing code ให้ใช้ batch loading strategy เพื่อ reduce DB calls จาก 100+ เป็น 2-3 queries

5. **Graph Queries**: Implement "follower/following" social network ด้วย self-referential M:N และ query "feed" (articles by followed users)

---

[→ ไปยัง Part 64: Doobie](part-64-doobie.md)
