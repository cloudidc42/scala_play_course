# Part 69: CQRS and Event Sourcing

## Steps 681-690: CQRS Pattern, Event Store, Projections, Snapshots, Event Replay

---

## Step 681: CQRS และ Event Sourcing คืออะไร

```
CQRS (Command Query Responsibility Segregation):
  แยก Read (Query) และ Write (Command) model ออกจากกัน

Event Sourcing:
  แทนที่จะ save "current state", save "sequence of events" ที่นำไปสู่ state นั้น

ประโยชน์:
├── Complete audit trail
├── Time-travel (replay events ไป state ใดก็ได้)
├── Eventual consistency
├── Better performance (read/write scale แยกกัน)
└── Event-driven architecture
```

---

## Step 682: Domain Events

```scala
// app/domain/events/ArticleEvents.scala
package domain.events

import java.time.Instant

// Base event trait
sealed trait ArticleEvent {
  def articleId: String
  def occurredAt: Instant
  def version: Long
}

// Article events
case class ArticleCreated(
  articleId: String,
  title: String,
  content: String,
  authorId: String,
  occurredAt: Instant = Instant.now(),
  version: Long = 1
) extends ArticleEvent

case class ArticlePublished(
  articleId: String,
  publishedAt: Instant,
  occurredAt: Instant = Instant.now(),
  version: Long
) extends ArticleEvent

case class ArticleUpdated(
  articleId: String,
  title: Option[String],
  content: Option[String],
  summary: Option[String],
  occurredAt: Instant = Instant.now(),
  version: Long
) extends ArticleEvent

case class ArticleTagged(
  articleId: String,
  tags: List[String],
  occurredAt: Instant = Instant.now(),
  version: Long
) extends ArticleEvent

case class ArticleViewed(
  articleId: String,
  userId: Option[String],
  ipAddress: String,
  occurredAt: Instant = Instant.now(),
  version: Long
) extends ArticleEvent

case class ArticleDeleted(
  articleId: String,
  deletedBy: String,
  occurredAt: Instant = Instant.now(),
  version: Long
) extends ArticleEvent
```

---

## Step 683: Aggregate (Write Model)

```scala
// app/domain/Article.scala
package domain

import domain.events.*
import java.time.Instant

// Article Aggregate — represents current state
case class ArticleState(
  id: String,
  title: String,
  content: String,
  summary: Option[String] = None,
  authorId: String,
  tags: List[String] = Nil,
  status: String = "draft",
  viewCount: Int = 0,
  publishedAt: Option[Instant] = None,
  version: Long = 0,
  isDeleted: Boolean = false
)

object Article {
  // Apply event to state (pure function)
  def apply(state: ArticleState, event: ArticleEvent): ArticleState = event match {
    case e: ArticleCreated =>
      ArticleState(
        id       = e.articleId,
        title    = e.title,
        content  = e.content,
        authorId = e.authorId,
        version  = e.version
      )

    case e: ArticleUpdated =>
      state.copy(
        title   = e.title.getOrElse(state.title),
        content = e.content.getOrElse(state.content),
        summary = e.summary.orElse(state.summary),
        version = e.version
      )

    case e: ArticlePublished =>
      state.copy(
        status      = "published",
        publishedAt = Some(e.publishedAt),
        version     = e.version
      )

    case e: ArticleTagged =>
      state.copy(tags = e.tags, version = e.version)

    case e: ArticleViewed =>
      state.copy(viewCount = state.viewCount + 1, version = e.version)

    case e: ArticleDeleted =>
      state.copy(isDeleted = true, version = e.version)
  }

  // Rebuild state from events
  def rebuild(events: Seq[ArticleEvent]): Option[ArticleState] =
    events.foldLeft(Option.empty[ArticleState]) {
      case (None, e: ArticleCreated) => Some(apply(ArticleState(
        id = "", title = "", content = "", authorId = "", version = 0
      ), e))
      case (Some(state), event) => Some(apply(state, event))
      case (None, _) => None
    }

  // Commands → Events
  def create(id: String, title: String, content: String, authorId: String): Either[String, ArticleCreated] =
    if (title.trim.isEmpty) Left("Title cannot be empty")
    else if (content.trim.isEmpty) Left("Content cannot be empty")
    else Right(ArticleCreated(id, title, content, authorId))

  def publish(state: ArticleState): Either[String, ArticlePublished] =
    if (state.isDeleted) Left("Article is deleted")
    else if (state.status == "published") Left("Article is already published")
    else Right(ArticlePublished(state.id, Instant.now(), version = state.version + 1))

  def update(
    state: ArticleState,
    title: Option[String],
    content: Option[String],
    summary: Option[String]
  ): Either[String, ArticleUpdated] =
    if (state.isDeleted) Left("Article is deleted")
    else Right(ArticleUpdated(
      state.id, title, content, summary, version = state.version + 1
    ))
}
```

---

## Step 684: Event Store

```scala
// app/infrastructure/EventStore.scala
package infrastructure

import domain.events.ArticleEvent
import javax.inject.*
import play.api.db.slick.DatabaseConfigProvider
import play.api.libs.json.*
import slick.jdbc.PostgresProfile.api.*
import scala.concurrent.*
import java.time.Instant

// Event Store row
case class EventRow(
  id: Long = 0,
  aggregateId: String,
  aggregateType: String,
  eventType: String,
  payload: JsValue,
  version: Long,
  occurredAt: Instant
)

class EventsTable(tag: Tag) extends Table[EventRow](tag, "events") {
  def id            = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def aggregateId   = column[String]("aggregate_id")
  def aggregateType = column[String]("aggregate_type")
  def eventType     = column[String]("event_type")
  def payload       = column[JsValue]("payload")
  def version       = column[Long]("version")
  def occurredAt    = column[Instant]("occurred_at")

  def aggregateIdx  = index("events_aggregate_idx", (aggregateId, aggregateType))
  def versionIdx    = index("events_version_idx", (aggregateId, version), unique = true)

  def * = (id, aggregateId, aggregateType, eventType, payload, version, occurredAt).mapTo[EventRow]
}

@Singleton
class EventStore @Inject()(
  dbConfigProvider: DatabaseConfigProvider
)(implicit ec: ExecutionContext) {

  private val db     = dbConfigProvider.get[slick.jdbc.JdbcProfile].db
  private val events = TableQuery[EventsTable]

  // Append event (Optimistic Concurrency Control)
  def append(
    aggregateId: String,
    aggregateType: String,
    event: ArticleEvent,
    expectedVersion: Long
  ): Future[Unit] = {
    val action = for {
      // Check version (prevent concurrent writes)
      currentVersion <- events
        .filter(e => e.aggregateId === aggregateId && e.aggregateType === aggregateType)
        .map(_.version)
        .max
        .result

      _ <- currentVersion match {
        case Some(v) if v != expectedVersion =>
          DBIO.failed(new Exception(
            s"Concurrency conflict: expected version $expectedVersion, got $v"
          ))
        case _ =>
          events += EventRow(
            aggregateId   = aggregateId,
            aggregateType = aggregateType,
            eventType     = event.getClass.getSimpleName,
            payload       = serializeEvent(event),
            version       = event.version,
            occurredAt    = event.occurredAt
          )
      }
    } yield ()

    db.run(action.transactionally)
  }

  // Load events for aggregate
  def loadEvents(aggregateId: String, aggregateType: String): Future[Seq[ArticleEvent]] =
    db.run(
      events
        .filter(e => e.aggregateId === aggregateId && e.aggregateType === aggregateType)
        .sortBy(_.version)
        .result
    ).map(_.flatMap(row => deserializeEvent(row)))

  // Load events after version (for incremental rebuilds)
  def loadEventsAfter(
    aggregateId: String,
    aggregateType: String,
    afterVersion: Long
  ): Future[Seq[ArticleEvent]] =
    db.run(
      events
        .filter(e =>
          e.aggregateId === aggregateId &&
          e.aggregateType === aggregateType &&
          e.version > afterVersion
        )
        .sortBy(_.version)
        .result
    ).map(_.flatMap(row => deserializeEvent(row)))

  // All events for projection rebuild
  def loadAllEvents(aggregateType: String, afterPosition: Long = 0): Future[Seq[(Long, ArticleEvent)]] =
    db.run(
      events
        .filter(e => e.aggregateType === aggregateType && e.id > afterPosition)
        .sortBy(_.id)
        .result
    ).map(_.flatMap(row => deserializeEvent(row).map(e => (row.id, e))))

  private def serializeEvent(event: ArticleEvent): JsValue = {
    import domain.events.EventJsonFormats.*
    Json.toJson(event)
  }

  private def deserializeEvent(row: EventRow): Option[ArticleEvent] = {
    import domain.events.EventJsonFormats.*
    row.eventType match {
      case "ArticleCreated"   => row.payload.asOpt[ArticleCreated]
      case "ArticlePublished" => row.payload.asOpt[ArticlePublished]
      case "ArticleUpdated"   => row.payload.asOpt[ArticleUpdated]
      case "ArticleTagged"    => row.payload.asOpt[ArticleTagged]
      case "ArticleViewed"    => row.payload.asOpt[ArticleViewed]
      case "ArticleDeleted"   => row.payload.asOpt[ArticleDeleted]
      case _                  => None
    }
  }
}
```

---

## Step 685: Command Handler

```scala
// app/application/ArticleCommandHandler.scala
package application

import domain.*
import domain.events.*
import infrastructure.EventStore
import javax.inject.*
import scala.concurrent.*
import java.util.UUID

case class CreateArticleCommand(title: String, content: String, authorId: String)
case class PublishArticleCommand(articleId: String, userId: String)
case class UpdateArticleCommand(articleId: String, title: Option[String], content: Option[String], summary: Option[String])

@Singleton
class ArticleCommandHandler @Inject()(
  eventStore: EventStore
)(implicit ec: ExecutionContext) {

  def handle(cmd: CreateArticleCommand): Future[String] = {
    val articleId = UUID.randomUUID().toString

    Article.create(articleId, cmd.title, cmd.content, cmd.authorId) match {
      case Left(error) => Future.failed(new IllegalArgumentException(error))
      case Right(event) =>
        eventStore.append(articleId, "Article", event, 0).map(_ => articleId)
    }
  }

  def handle(cmd: PublishArticleCommand): Future[Unit] = {
    for {
      events <- eventStore.loadEvents(cmd.articleId, "Article")
      state  <- Article.rebuild(events) match {
        case None        => Future.failed(new NoSuchElementException(s"Article ${cmd.articleId} not found"))
        case Some(state) => Future.successful(state)
      }
      event <- Article.publish(state) match {
        case Left(error)  => Future.failed(new IllegalStateException(error))
        case Right(event) => Future.successful(event)
      }
      _ <- eventStore.append(cmd.articleId, "Article", event, state.version)
    } yield ()
  }

  def handle(cmd: UpdateArticleCommand): Future[Unit] = {
    for {
      events <- eventStore.loadEvents(cmd.articleId, "Article")
      state  <- Article.rebuild(events) match {
        case None        => Future.failed(new NoSuchElementException(s"Article ${cmd.articleId} not found"))
        case Some(state) => Future.successful(state)
      }
      event <- Article.update(state, cmd.title, cmd.content, cmd.summary) match {
        case Left(error)  => Future.failed(new IllegalStateException(error))
        case Right(event) => Future.successful(event)
      }
      _ <- eventStore.append(cmd.articleId, "Article", event, state.version)
    } yield ()
  }
}
```

---

## Step 686: Read Model (Projections)

```scala
// app/infrastructure/ArticleProjection.scala
package infrastructure

import domain.events.*
import javax.inject.*
import play.api.db.slick.DatabaseConfigProvider
import slick.jdbc.PostgresProfile.api.*
import scala.concurrent.*

// Read Model — optimized สำหรับ queries
case class ArticleReadModel(
  id: String,
  title: String,
  summary: Option[String],
  authorId: String,
  authorName: String,
  tags: List[String],
  status: String,
  viewCount: Int,
  commentCount: Int,
  publishedAt: Option[java.time.Instant],
  updatedAt: java.time.Instant
)

@Singleton
class ArticleProjection @Inject()(
  dbConfigProvider: DatabaseConfigProvider
)(implicit ec: ExecutionContext) {

  private val db = dbConfigProvider.get[slick.jdbc.JdbcProfile].db

  // Project events ไปยัง Read Model
  def project(event: ArticleEvent): Future[Unit] = event match {
    case e: ArticleCreated =>
      db.run(
        sqlu"""
          INSERT INTO article_read_models (id, title, author_id, status, view_count, comment_count, updated_at)
          VALUES (${e.articleId}, ${e.title}, ${e.authorId}, 'draft', 0, 0, NOW())
          ON CONFLICT (id) DO NOTHING
        """
      ).map(_ => ())

    case e: ArticleUpdated =>
      val updates = Seq(
        e.title.map(t => s"title = '${t.replace("'", "''")}'"),
        e.content.map(_ => ""),  // content ไม่อยู่ใน read model
        e.summary.map(s => s"summary = '${s.replace("'", "''")}'")
      ).flatten.filter(_.nonEmpty)

      if (updates.isEmpty) Future.successful(())
      else db.run(
        sqlu"UPDATE article_read_models SET #${updates.mkString(", ")}, updated_at = NOW() WHERE id = ${e.articleId}"
      ).map(_ => ())

    case e: ArticlePublished =>
      db.run(
        sqlu"UPDATE article_read_models SET status = 'published', published_at = ${e.publishedAt}, updated_at = NOW() WHERE id = ${e.articleId}"
      ).map(_ => ())

    case e: ArticleTagged =>
      val tagsJson = e.tags.mkString("{", ",", "}")
      db.run(
        sqlu"UPDATE article_read_models SET tags = $tagsJson::text[], updated_at = NOW() WHERE id = ${e.articleId}"
      ).map(_ => ())

    case e: ArticleViewed =>
      db.run(
        sqlu"UPDATE article_read_models SET view_count = view_count + 1 WHERE id = ${e.articleId}"
      ).map(_ => ())

    case e: ArticleDeleted =>
      db.run(
        sqlu"DELETE FROM article_read_models WHERE id = ${e.articleId}"
      ).map(_ => ())
  }

  // Rebuild projection จาก all events
  def rebuildAll(events: Seq[(Long, ArticleEvent)]): Future[Unit] = {
    val actions = events.map { case (_, event) => project(event) }
    Future.sequence(actions).map(_ => ())
  }
}
```

---

## Step 687: Snapshots

```scala
// app/infrastructure/SnapshotStore.scala
package infrastructure

import domain.ArticleState
import javax.inject.*
import play.api.db.slick.DatabaseConfigProvider
import play.api.libs.json.*
import slick.jdbc.PostgresProfile.api.*
import scala.concurrent.*

@Singleton
class SnapshotStore @Inject()(
  dbConfigProvider: DatabaseConfigProvider
)(implicit ec: ExecutionContext) {

  private val db = dbConfigProvider.get[slick.jdbc.JdbcProfile].db

  implicit val articleStateFormat: OFormat[ArticleState] = Json.format[ArticleState]

  def save(aggregateId: String, state: ArticleState): Future[Unit] =
    db.run(
      sqlu"""
        INSERT INTO snapshots (aggregate_id, aggregate_type, version, payload, created_at)
        VALUES ($aggregateId, 'Article', ${state.version}, ${Json.stringify(Json.toJson(state))}::jsonb, NOW())
        ON CONFLICT (aggregate_id, aggregate_type) DO UPDATE SET
          version = EXCLUDED.version,
          payload = EXCLUDED.payload,
          created_at = EXCLUDED.created_at
      """
    ).map(_ => ())

  def load(aggregateId: String): Future[Option[(Long, ArticleState)]] =
    db.run(
      sql"""
        SELECT version, payload FROM snapshots
        WHERE aggregate_id = $aggregateId AND aggregate_type = 'Article'
      """.as[(Long, String)]
      .headOption
    ).map(_.flatMap { case (version, payloadStr) =>
      Json.parse(payloadStr).asOpt[ArticleState].map(state => (version, state))
    })
}

// ใช้ snapshot ใน command handler
def loadWithSnapshot(articleId: String): Future[ArticleState] = {
  for {
    snapshotOpt <- snapshotStore.load(articleId)
    state <- snapshotOpt match {
      case Some((version, snapshot)) =>
        // Load only events after snapshot
        eventStore.loadEventsAfter(articleId, "Article", version).map { events =>
          events.foldLeft(snapshot)(Article.apply)
        }
      case None =>
        // Load all events
        eventStore.loadEvents(articleId, "Article").flatMap {
          case Nil => Future.failed(new NoSuchElementException(s"Article $articleId not found"))
          case events =>
            val state = Article.rebuild(events).getOrElse(throw new Exception("Cannot rebuild"))
            Future.successful(state)
        }
    }
    // Save snapshot every 100 events
    _ <- if (state.version % 100 == 0) snapshotStore.save(articleId, state)
         else Future.successful(())
  } yield state
}
```

---

## Step 688: Event Replay

```scala
// app/infrastructure/EventReplayer.scala
package infrastructure

import javax.inject.*
import scala.concurrent.*

@Singleton
class EventReplayer @Inject()(
  eventStore: EventStore,
  projection: ArticleProjection
)(implicit ec: ExecutionContext) {

  // Replay all events to rebuild projection
  def replayAll(): Future[Long] = {
    var processed = 0L

    def processChunk(afterPosition: Long): Future[Unit] = {
      eventStore.loadAllEvents("Article", afterPosition).flatMap { events =>
        if (events.isEmpty) Future.successful(())
        else {
          processed += events.size
          Future.sequence(events.map { case (_, event) => projection.project(event) })
            .flatMap { _ =>
              val lastPosition = events.last._1
              processChunk(lastPosition)
            }
        }
      }
    }

    // Clear existing projection
    processChunk(0).map(_ => processed)
  }

  // Time-travel: rebuild state at specific point in time
  def rebuildAtTime(
    articleId: String,
    asOf: java.time.Instant
  ): Future[Option[domain.ArticleState]] = {
    eventStore.loadEvents(articleId, "Article").map { events =>
      val eventsUpToTime = events.filter(_.occurredAt.isBefore(asOf))
      domain.Article.rebuild(eventsUpToTime)
    }
  }
}
```

---

## Step 689: Query Side Controller

```scala
// app/controllers/ArticleReadController.scala — uses Read Model
package controllers

import javax.inject.*
import play.api.libs.json.*
import play.api.mvc.*
import infrastructure.ArticleReadRepository
import scala.concurrent.*

@Singleton
class ArticleReadController @Inject()(
  val controllerComponents: ControllerComponents,
  readRepo: ArticleReadRepository,
  implicit val ec: ExecutionContext
) extends BaseController {

  def list(page: Int = 1): Action[AnyContent] = Action.async {
    readRepo.findPublished(page).map { articles =>
      Ok(Json.toJson(articles))
    }
  }

  def get(id: String): Action[AnyContent] = Action.async {
    readRepo.findById(id).map {
      case Some(article) => Ok(Json.toJson(article))
      case None          => NotFound(Json.obj("error" -> "Not found"))
    }
  }
}

// Write Side Controller
@Singleton
class ArticleWriteController @Inject()(
  val controllerComponents: ControllerComponents,
  commandHandler: application.ArticleCommandHandler,
  implicit val ec: ExecutionContext
) extends BaseController {

  def create(): Action[JsValue] = Action.async(parse.json) { request =>
    val cmd = application.CreateArticleCommand(
      title    = (request.body \ "title").as[String],
      content  = (request.body \ "content").as[String],
      authorId = request.session.get("userId").getOrElse("anonymous")
    )

    commandHandler.handle(cmd)
      .map(id => Created(Json.obj("id" -> id)))
      .recover { case ex => BadRequest(Json.obj("error" -> ex.getMessage)) }
  }

  def publish(id: String): Action[AnyContent] = Action.async { request =>
    val cmd = application.PublishArticleCommand(
      articleId = id,
      userId    = request.session.get("userId").getOrElse("anonymous")
    )

    commandHandler.handle(cmd)
      .map(_ => Ok(Json.obj("status" -> "published")))
      .recover { case ex => BadRequest(Json.obj("error" -> ex.getMessage)) }
  }
}
```

---

## Step 690: Testing CQRS

```scala
// test/domain/ArticleSpec.scala
class ArticleSpec extends AnyFlatSpec with Matchers {

  "Article" should "create successfully" in {
    val result = Article.create("id-1", "Hello World", "Content...", "author-1")
    result.isRight shouldBe true
    result.foreach(e => e.title shouldBe "Hello World")
  }

  it should "reject empty title" in {
    val result = Article.create("id-1", "", "Content...", "author-1")
    result.isLeft shouldBe true
  }

  it should "rebuild state from events" in {
    val events = Seq(
      ArticleCreated("id-1", "Title", "Content", "author-1", version = 1),
      ArticlePublished("id-1", Instant.now(), version = 2),
      ArticleTagged("id-1", List("scala"), version = 3)
    )

    val state = Article.rebuild(events)
    state shouldBe defined
    state.get.status shouldBe "published"
    state.get.tags shouldBe List("scala")
    state.get.version shouldBe 3
  }

  it should "prevent publishing deleted article" in {
    val state = ArticleState("id-1", "T", "C", None, "author-1", version = 2, isDeleted = true)
    val result = Article.publish(state)
    result.isLeft shouldBe true
  }
}
```

---

## สรุป Part 69

| Pattern | Concept | Implementation |
|---------|---------|---------------|
| CQRS | Separate read/write | Two controllers, two models |
| Command | Intent to change state | `CreateArticleCommand` |
| Event | State change that happened | `ArticleCreated` |
| Aggregate | Domain object | `ArticleState` + `Article` |
| Event Store | Append-only log | `events` table |
| Projection | Build read model | `ArticleProjection` |
| Snapshot | Cached state | Every 100 events |
| Replay | Rebuild from events | Full or partial |
| Optimistic locking | Version checking | Expected version |
| Time-travel | Historical state | Events up to time |

---

## แบบฝึกหัด Part 69

1. **Shopping Cart**: Implement CQRS+ES สำหรับ shopping cart ที่ track AddItem, RemoveItem, ApplyDiscount, PlaceOrder events

2. **Event Bus**: สร้าง in-process event bus ที่ dispatch events ไปยัง multiple projections concurrently หลังจาก command handle

3. **Saga Pattern**: Implement order processing saga ที่ coordinate OrderCreated → PaymentProcessed → InventoryReserved → OrderConfirmed

4. **Audit via Events**: เพิ่ม audit log ที่ subscribe ทุก event และ build audit trail โดยไม่ต้องแก้ domain code

5. **Event Versioning**: Handle event schema evolution เมื่อ event payload เปลี่ยน (เช่น เพิ่ม field) โดย backward compatible

---

[→ ไปยัง Part 70: Akka Persistence](part-70-akka-persistence.md)
