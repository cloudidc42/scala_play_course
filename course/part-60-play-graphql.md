# Part 60: Play GraphQL with Sangria

## Steps 591-600: Schema Definition, Queries, Mutations, Subscriptions

---

## Step 591: GraphQL และ Sangria

GraphQL คือ query language สำหรับ API ที่ client กำหนดได้ว่าต้องการข้อมูลอะไร

```
REST vs GraphQL:
REST:
  GET /articles          → ได้ทุก field
  GET /articles/1        → ได้ทุก field
  GET /articles/1/author → request เพิ่ม

GraphQL:
  query {
    article(id: 1) {
      title
      author { name }
    }
  }
  → ได้เฉพาะที่ต้องการใน 1 request
```

### build.sbt

```scala
libraryDependencies ++= Seq(
  "org.sangria-graphql" %% "sangria"             % "4.1.0",
  "org.sangria-graphql" %% "sangria-play-json"   % "2.0.2",
  "org.sangria-graphql" %% "sangria-slowlog"     % "3.0.0"
)
```

---

## Step 592: Schema Definition

```scala
// app/graphql/SchemaDefinition.scala
package graphql

import sangria.schema.*
import sangria.macros.derive.*
import models.*

// GraphQL Types
object SchemaDefinition {

  // Basic scalar types ที่ Sangria มีให้: StringType, IntType, LongType,
  // FloatType, BooleanType, IDType

  // Custom scalar: DateTime
  import sangria.schema.ScalarType
  import java.time.Instant

  val DateTimeType: ScalarType[Instant] = ScalarType[Instant](
    "DateTime",
    description = Some("ISO 8601 DateTime string"),
    coerceOutput = (value, _) => value.toString,
    coerceInput = {
      case sangria.ast.StringValue(s, _, _, _, _) =>
        Right(Instant.parse(s))
      case _ =>
        Left(DateCoercionViolation)
    },
    coerceUserInput = {
      case s: String => Right(Instant.parse(s))
      case _         => Left(DateCoercionViolation)
    }
  )

  case object DateCoercionViolation extends ValueCoercionViolation("Date value expected")

  // User type
  val UserType: ObjectType[Unit, User] = deriveObjectType[Unit, User](
    ObjectTypeDescription("A user in the system"),
    DocumentField("id", "Unique user identifier"),
    DocumentField("name", "User's full name"),
    DocumentField("email", "User's email address"),
    ExcludeFields("passwordHash"),  // ไม่ expose password hash
    AddFields(
      Field("articleCount", IntType,
        description = Some("Number of articles written"),
        resolve = ctx => 0  // resolve ใน context
      )
    )
  )

  // Article type
  val ArticleType: ObjectType[MyContext, Article] = ObjectType(
    "Article",
    "A blog article",
    () => fields[MyContext, Article](
      Field("id",          IDType,       resolve = _.value.id),
      Field("title",       StringType,   resolve = _.value.title),
      Field("content",     StringType,   resolve = _.value.content),
      Field("summary",     OptionType(StringType), resolve = _.value.summary),
      Field("publishedAt", OptionType(DateTimeType), resolve = _.value.publishedAt),
      Field("viewCount",   IntType,      resolve = _.value.viewCount),
      Field("author", UserType,
        resolve = ctx => ctx.ctx.userService.findById(ctx.value.authorId)
          .map(_.getOrElse(throw new Exception("Author not found")))
      ),
      Field("tags", ListType(StringType), resolve = _.value.tags),
      Field("comments", ListType(CommentType),
        arguments = LimitArg :: OffsetArg :: Nil,
        resolve = ctx =>
          ctx.ctx.commentService.findByArticle(
            ctx.value.id,
            ctx.arg(LimitArg),
            ctx.arg(OffsetArg)
          )
      )
    )
  )

  // Comment type (lazy เพราะ reference กันเอง)
  lazy val CommentType: ObjectType[MyContext, Comment] = ObjectType(
    "Comment",
    fields[MyContext, Comment](
      Field("id",      IDType,      resolve = _.value.id),
      Field("content", StringType,  resolve = _.value.content),
      Field("author",  UserType,    resolve = ctx =>
        ctx.ctx.userService.findById(ctx.value.authorId)
          .map(_.getOrElse(throw new Exception("Author not found")))
      ),
      Field("createdAt", DateTimeType, resolve = _.value.createdAt)
    )
  )

  // Pagination arguments
  val LimitArg  = Argument("limit",  IntType,  defaultValue = 10)
  val OffsetArg = Argument("offset", IntType,  defaultValue = 0)
  val IdArg     = Argument("id",     IDType)
}
```

---

## Step 593: Query Type

```scala
// app/graphql/QueryType.scala
package graphql

import sangria.schema.*
import sangria.execution.deferred.*
import models.*
import SchemaDefinition.*

object QueryType {

  // Arguments
  val ArticleIdArg = Argument("id", IDType)
  val SearchArg    = Argument("search", OptionInputType(StringType))
  val TagArg       = Argument("tag", OptionInputType(StringType))

  val Query: ObjectType[MyContext, Unit] = ObjectType(
    "Query",
    fields[MyContext, Unit](
      // ดึง article เดียว
      Field("article", OptionType(ArticleType),
        arguments = ArticleIdArg :: Nil,
        resolve = ctx =>
          ctx.ctx.articleService.findById(ctx.arg(ArticleIdArg))
      ),

      // ดึง articles ทั้งหมด พร้อม pagination
      Field("articles", ListType(ArticleType),
        arguments = LimitArg :: OffsetArg :: SearchArg :: TagArg :: Nil,
        resolve = ctx =>
          ctx.ctx.articleService.findAll(
            limit  = ctx.arg(LimitArg),
            offset = ctx.arg(OffsetArg),
            search = ctx.arg(SearchArg),
            tag    = ctx.arg(TagArg)
          )
      ),

      // ดึง user
      Field("user", OptionType(UserType),
        arguments = Argument("id", IDType) :: Nil,
        resolve = ctx =>
          ctx.ctx.userService.findById(ctx.arg(Argument("id", IDType)))
      ),

      // ดึง current user
      Field("me", OptionType(UserType),
        resolve = ctx =>
          ctx.ctx.currentUserId.flatMap(id =>
            // return Future[Option[User]]
            Some(ctx.ctx.userService.findById(id))
          ).getOrElse(scala.concurrent.Future.successful(None))
      ),

      // Search articles
      Field("searchArticles", ListType(ArticleType),
        arguments = Argument("query", StringType) :: LimitArg :: Nil,
        resolve = ctx =>
          ctx.ctx.articleService.search(
            query = ctx.arg(Argument("query", StringType)),
            limit = ctx.arg(LimitArg)
          )
      )
    )
  )
}
```

---

## Step 594: Mutation Type

```scala
// app/graphql/MutationType.scala
package graphql

import sangria.schema.*
import sangria.macros.derive.*
import models.*

object MutationType {

  // Input types สำหรับ mutations
  val CreateArticleInput: InputObjectType[CreateArticleData] =
    deriveInputObjectType[CreateArticleData](
      InputObjectTypeDescription("Input for creating a new article")
    )

  val UpdateArticleInput: InputObjectType[UpdateArticleData] =
    deriveInputObjectType[UpdateArticleData]()

  val CreateCommentInput: InputObjectType[CreateCommentData] =
    deriveInputObjectType[CreateCommentData]()

  // Result types
  val ArticleResultType: ObjectType[MyContext, ArticleResult] = ObjectType(
    "ArticleResult",
    fields[MyContext, ArticleResult](
      Field("article", OptionType(ArticleType), resolve = _.value.article),
      Field("errors", ListType(StringType),     resolve = _.value.errors)
    )
  )

  val Mutation: ObjectType[MyContext, Unit] = ObjectType(
    "Mutation",
    fields[MyContext, Unit](
      // Create article
      Field("createArticle", ArticleResultType,
        arguments = Argument("input", CreateArticleInput) :: Nil,
        resolve = ctx => {
          ctx.ctx.requireAuth()
          val input = ctx.arg(Argument("input", CreateArticleInput))
          ctx.ctx.articleService
            .create(input, ctx.ctx.currentUserId.get)
            .map(article => ArticleResult(Some(article), Nil))
            .recover { case ex => ArticleResult(None, List(ex.getMessage)) }
        }
      ),

      // Update article
      Field("updateArticle", ArticleResultType,
        arguments = Argument("id", IDType) :: Argument("input", UpdateArticleInput) :: Nil,
        resolve = ctx => {
          ctx.ctx.requireAuth()
          val id    = ctx.arg(Argument("id", IDType))
          val input = ctx.arg(Argument("input", UpdateArticleInput))
          ctx.ctx.articleService
            .update(id, input, ctx.ctx.currentUserId.get)
            .map(article => ArticleResult(Some(article), Nil))
        }
      ),

      // Delete article
      Field("deleteArticle", BooleanType,
        arguments = Argument("id", IDType) :: Nil,
        resolve = ctx => {
          ctx.ctx.requireAuth()
          val id = ctx.arg(Argument("id", IDType))
          ctx.ctx.articleService.delete(id, ctx.ctx.currentUserId.get)
        }
      ),

      // Add comment
      Field("addComment", CommentType,
        arguments = Argument("input", CreateCommentInput) :: Nil,
        resolve = ctx => {
          ctx.ctx.requireAuth()
          val input = ctx.arg(Argument("input", CreateCommentInput))
          ctx.ctx.commentService.create(input, ctx.ctx.currentUserId.get)
        }
      )
    )
  )
}
```

---

## Step 595: Context และ Schema

```scala
// app/graphql/MyContext.scala
package graphql

import services.*
import scala.concurrent.Future

case class MyContext(
  articleService: ArticleService,
  userService: UserService,
  commentService: CommentService,
  currentUserId: Option[String]
) {
  def requireAuth(): Unit =
    if (currentUserId.isEmpty)
      throw new Exception("Authentication required")
}

// app/graphql/GraphQLSchema.scala
package graphql

import sangria.schema.*

object GraphQLSchema {
  val schema: Schema[MyContext, Unit] = Schema(
    query    = QueryType.Query,
    mutation = Some(MutationType.Mutation)
  )
}
```

---

## Step 596: GraphQL Controller

```scala
// app/controllers/GraphQLController.scala
package controllers

import graphql.*
import javax.inject.*
import play.api.libs.json.*
import play.api.mvc.*
import sangria.execution.*
import sangria.parser.QueryParser
import sangria.marshalling.playJson.*
import services.*
import scala.concurrent.*
import scala.util.*

@Singleton
class GraphQLController @Inject()(
  val controllerComponents: ControllerComponents,
  articleService: ArticleService,
  userService: UserService,
  commentService: CommentService,
  implicit val ec: ExecutionContext
) extends BaseController {

  def graphql(): Action[JsValue] = Action.async(parse.json) { request =>
    val query         = (request.body \ "query").as[String]
    val variables     = (request.body \ "variables").asOpt[JsObject].getOrElse(Json.obj())
    val operationName = (request.body \ "operationName").asOpt[String]
    val userId        = request.session.get("userId")

    executeQuery(query, variables, operationName, userId)
  }

  // GraphQL Playground (dev only)
  def playground(): Action[AnyContent] = Action {
    Ok(views.html.graphqlPlayground())
  }

  private def executeQuery(
    query: String,
    variables: JsObject,
    operationName: Option[String],
    userId: Option[String]
  ): Future[Result] = {

    QueryParser.parse(query) match {
      case Failure(error) =>
        Future.successful(BadRequest(Json.obj("error" -> error.getMessage)))

      case Success(queryAst) =>
        val ctx = MyContext(articleService, userService, commentService, userId)

        Executor.execute(
          schema          = GraphQLSchema.schema,
          queryAst        = queryAst,
          userContext     = ctx,
          variables       = variables,
          operationName   = operationName,
          exceptionHandler = ExceptionHandler {
            case (_, e: Exception) =>
              HandledException(e.getMessage)
          }
        )
        .map(Ok(_))
        .recover {
          case error: QueryAnalysisError =>
            BadRequest(error.resolveError)
          case error: ErrorWithResolver =>
            InternalServerError(error.resolveError)
        }
    }
  }
}
```

---

## Step 597: DataLoader (N+1 Problem)

```scala
// app/graphql/Fetchers.scala
package graphql

import sangria.execution.deferred.*
import models.User
import scala.concurrent.ExecutionContext

// แก้ N+1 problem ด้วย Deferred Values (DataLoader pattern)
object Fetchers {

  val usersFetcher: Fetcher[MyContext, User, User, String] =
    Fetcher.caching { (ctx: MyContext, ids: Seq[String]) =>
      ctx.userService.findByIds(ids)
    }(HasId(_.id))

  val deferredResolvers = DeferredResolver.fetchers(usersFetcher)
}

// ใช้ใน ArticleType
// Field("author", UserType,
//   resolve = ctx => Fetchers.usersFetcher.defer(ctx.value.authorId)
// )
```

---

## Step 598: GraphQL Subscription

```scala
// app/graphql/SubscriptionType.scala
package graphql

import sangria.schema.*
import org.apache.pekko.stream.scaladsl.Source

object SubscriptionType {

  val Subscription: ObjectType[MyContext, Unit] = ObjectType(
    "Subscription",
    fields[MyContext, Unit](
      Field.subs("articleAdded", ArticleType,
        description = Some("Subscribe to new articles"),
        resolve = ctx =>
          ctx.ctx.articleService.newArticleStream().map(Action(_))
      ),

      Field.subs("commentAdded", CommentType,
        arguments = Argument("articleId", IDType) :: Nil,
        resolve = ctx => {
          val articleId = ctx.arg(Argument("articleId", IDType))
          ctx.ctx.commentService.newCommentStream(articleId).map(Action(_))
        }
      )
    )
  )
}
```

---

## Step 599: Introspection และ GraphiQL

```html
@* app/views/graphqlPlayground.scala.html *@
<!DOCTYPE html>
<html>
<head>
  <title>GraphQL Playground</title>
  <link href="https://unpkg.com/graphiql/graphiql.min.css" rel="stylesheet" />
</head>
<body style="margin: 0;">
  <div id="graphiql" style="height: 100vh;"></div>
  <script src="https://unpkg.com/react/umd/react.production.min.js"></script>
  <script src="https://unpkg.com/react-dom/umd/react-dom.production.min.js"></script>
  <script src="https://unpkg.com/graphiql/graphiql.min.js"></script>
  <script>
    const fetcher = GraphiQL.createFetcher({
      url: '/graphql',
      headers: { 'Content-Type': 'application/json' }
    });

    ReactDOM.render(
      React.createElement(GraphiQL, { fetcher }),
      document.getElementById('graphiql')
    );
  </script>
</body>
</html>
```

---

## Step 600: Testing GraphQL

```scala
// test/graphql/GraphQLSpec.scala
package graphql

import org.scalatestplus.play.*
import org.scalatestplus.play.guice.GuiceOneAppPerTest
import play.api.libs.json.*
import play.api.test.*
import play.api.test.Helpers.*

class GraphQLSpec extends PlaySpec with GuiceOneAppPerTest {

  "GraphQL API" should {
    "execute a simple query" in {
      val query = Json.obj(
        "query" ->
          """{
            articles(limit: 5) {
              id
              title
            }
          }"""
      )

      val request = FakeRequest(POST, "/graphql")
        .withJsonBody(query)
        .withHeaders("Content-Type" -> "application/json")

      val result = route(app, request).get

      status(result) mustBe OK
      val json = contentAsJson(result)
      (json \ "data" \ "articles").isDefined mustBe true
    }

    "return error for unauthorized mutation" in {
      val mutation = Json.obj(
        "query" ->
          """mutation {
              createArticle(input: {
                title: "Test"
                content: "Test content"
              }) {
                article { id }
                errors
              }
            }"""
      )

      val request = FakeRequest(POST, "/graphql").withJsonBody(mutation)
      val result  = route(app, request).get

      val json = contentAsJson(result)
      (json \ "data" \ "createArticle" \ "errors").as[Seq[String]] must not be empty
    }
  }
}
```

---

## สรุป Part 60

| Concept | Sangria API | ตัวอย่าง |
|---------|------------|---------|
| Object type | `ObjectType(...)` | `ArticleType` |
| Derive type | `deriveObjectType[Ctx, T]` | Auto from case class |
| Input type | `InputObjectType(...)` | `CreateArticleInput` |
| Query | `Field("fieldName", Type, resolve = ...)` | `article(id: "1")` |
| Mutation | In `Mutation` ObjectType | `createArticle(input: ...)` |
| Arguments | `Argument("name", Type)` | `Argument("id", IDType)` |
| N+1 fix | `Fetcher.caching` | DataLoader pattern |
| Subscriptions | `Field.subs(...)` | Real-time events |
| Context | `MyContext` case class | Services, auth |
| Error handling | `ExceptionHandler` | Structured errors |

---

## แบบฝึกหัด Part 60

1. **Full Blog API**: สร้าง GraphQL API สมบูรณ์สำหรับ blog ที่มี users, articles, comments, tags พร้อม authentication และ authorization

2. **DataLoader**: Implement DataLoader pattern สำหรับ nested queries เพื่อแก้ N+1 problem ด้วย `Fetcher.caching`

3. **File Upload**: Implement GraphQL file upload ด้วย `multipart/form-data` spec สำหรับ upload avatar images

4. **Real-time Chat**: สร้าง real-time chat ด้วย GraphQL subscriptions ที่ใช้ WebSocket และ Pekko Streams

5. **Schema Stitching**: Combine หลาย GraphQL schemas เข้าด้วยกัน สร้าง unified API gateway สำหรับ microservices

---

[→ ไปยัง Part 61: Slick Intro](part-61-slick-intro.md)
