# Part 70: Akka/Pekko Persistence

## Steps 691-700: PersistentActor, Snapshots, Query Side, Read Journal

---

## Step 691: Pekko Persistence คืออะไร

Pekko Persistence (Akka Persistence) คือ built-in Event Sourcing framework สำหรับ Pekko Actors

```
Pekko Persistence ต่างจาก Manual ES (Part 69):
Manual:
  - เขียน Event Store เอง
  - Control ทุกอย่างด้วยตัวเอง
  - ใช้งานง่ายกว่าสำหรับ simple cases

Pekko Persistence:
  - Built-in persistence (journal + snapshot store)
  - Actor-based (message passing)
  - Cluster Sharding support
  - CQRS/RS (Persistence Query)
  - Production-ready with many plugins
```

### build.sbt

```scala
val pekkoVersion = "1.0.3"

libraryDependencies ++= Seq(
  "org.apache.pekko" %% "pekko-persistence-typed"  % pekkoVersion,
  "org.apache.pekko" %% "pekko-persistence-query"  % pekkoVersion,
  
  // PostgreSQL Journal
  "com.lightbend.akka" %% "akka-persistence-jdbc"  % "5.4.1",
  // หรือ
  "org.apache.pekko"  %% "pekko-persistence-r2dbc" % "1.0.0",
  
  // Serialization
  "org.apache.pekko" %% "pekko-serialization-jackson" % pekkoVersion
)
```

---

## Step 692: Persistent Behavior (Typed)

```scala
// app/actors/ArticleActor.scala
package actors

import org.apache.pekko.actor.typed.*
import org.apache.pekko.actor.typed.scaladsl.*
import org.apache.pekko.persistence.typed.*
import org.apache.pekko.persistence.typed.scaladsl.*
import java.time.Instant

// Commands (messages to actor)
sealed trait ArticleCommand
case class CreateArticle(title: String, content: String, authorId: String,
                         replyTo: ActorRef[ArticleReply]) extends ArticleCommand
case class PublishArticle(replyTo: ActorRef[ArticleReply])     extends ArticleCommand
case class UpdateArticle(title: Option[String], content: Option[String],
                         replyTo: ActorRef[ArticleReply])      extends ArticleCommand
case class GetArticleState(replyTo: ActorRef[Option[ArticleState]]) extends ArticleCommand
case class AddView(userId: Option[String])                     extends ArticleCommand

// Replies
sealed trait ArticleReply
case object ArticleSuccess                  extends ArticleReply
case class ArticleError(message: String)    extends ArticleReply

// Events (persisted)
sealed trait ArticleEvent
case class ArticleCreatedEv(title: String, content: String, authorId: String,
                             at: Instant = Instant.now()) extends ArticleEvent
case class ArticlePublishedEv(at: Instant = Instant.now()) extends ArticleEvent
case class ArticleUpdatedEv(title: Option[String], content: Option[String],
                             at: Instant = Instant.now()) extends ArticleEvent
case class ArticleViewedEv(userId: Option[String], at: Instant = Instant.now()) extends ArticleEvent

// State
case class ArticleState(
  id: String,
  title: String = "",
  content: String = "",
  authorId: String = "",
  status: String = "created",
  viewCount: Int = 0,
  publishedAt: Option[Instant] = None,
  isDeleted: Boolean = false
)

object ArticleActor {

  val EntityKey: EntityTypeKey[ArticleCommand] =
    EntityTypeKey[ArticleCommand]("Article")

  def apply(articleId: String): Behavior[ArticleCommand] =
    EventSourcedBehavior[ArticleCommand, ArticleEvent, ArticleState](
      persistenceId = PersistenceId.ofUniqueId(articleId),
      emptyState    = ArticleState(id = articleId),
      commandHandler = commandHandler,
      eventHandler  = eventHandler
    )
    .withRetention(RetentionCriteria.snapshotEvery(
      numberOfEvents = 100,
      keepNSnapshots = 2
    ))

  // Command Handler: Command → Effect (persist events)
  private val commandHandler: (ArticleState, ArticleCommand) => Effect[ArticleEvent, ArticleState] = {
    (state, command) =>
      command match {
        case CreateArticle(title, content, authorId, replyTo) =>
          if (state.title.nonEmpty)
            Effect.reply(replyTo)(ArticleError("Article already exists"))
          else if (title.trim.isEmpty)
            Effect.reply(replyTo)(ArticleError("Title cannot be empty"))
          else
            Effect
              .persist(ArticleCreatedEv(title, content, authorId))
              .thenReply(replyTo)(_ => ArticleSuccess)

        case PublishArticle(replyTo) =>
          if (state.title.isEmpty)
            Effect.reply(replyTo)(ArticleError("Article not found"))
          else if (state.status == "published")
            Effect.reply(replyTo)(ArticleError("Already published"))
          else
            Effect
              .persist(ArticlePublishedEv())
              .thenReply(replyTo)(_ => ArticleSuccess)

        case UpdateArticle(title, content, replyTo) =>
          if (state.title.isEmpty)
            Effect.reply(replyTo)(ArticleError("Article not found"))
          else
            Effect
              .persist(ArticleUpdatedEv(title, content))
              .thenReply(replyTo)(_ => ArticleSuccess)

        case GetArticleState(replyTo) =>
          if (state.title.isEmpty)
            Effect.reply(replyTo)(None)
          else
            Effect.reply(replyTo)(Some(state))

        case AddView(userId) =>
          Effect.persist(ArticleViewedEv(userId))
      }
  }

  // Event Handler: (State, Event) → State (pure function)
  private val eventHandler: (ArticleState, ArticleEvent) => ArticleState = {
    (state, event) =>
      event match {
        case ArticleCreatedEv(title, content, authorId, _) =>
          state.copy(title = title, content = content, authorId = authorId, status = "draft")

        case ArticlePublishedEv(at) =>
          state.copy(status = "published", publishedAt = Some(at))

        case ArticleUpdatedEv(title, content, _) =>
          state.copy(
            title   = title.getOrElse(state.title),
            content = content.getOrElse(state.content)
          )

        case ArticleViewedEv(_, _) =>
          state.copy(viewCount = state.viewCount + 1)
      }
  }
}
```

---

## Step 693: Configuration

```hocon
# conf/application.conf
pekko {
  actor {
    serialization-bindings {
      "actors.ArticleEvent" = jackson-json
    }
  }
  
  persistence {
    journal {
      plugin = "jdbc-journal"
      auto-start-journals = ["jdbc-journal"]
    }
    snapshot-store {
      plugin = "jdbc-snapshot-store"
      auto-start-snapshot-stores = ["jdbc-snapshot-store"]
    }
  }
}

jdbc-journal {
  slick = ${slick}
}

jdbc-snapshot-store {
  slick = ${slick}
}

slick {
  profile = "slick.jdbc.PostgresProfile$"
  db {
    url      = "jdbc:postgresql://localhost:5432/myapp"
    user     = "myapp"
    password = "secret"
    driver   = "org.postgresql.Driver"
    numThreads = 5
    maxConnections = 10
    minConnections = 2
  }
}
```

---

## Step 694: Actor System Setup

```scala
// app/modules/PersistenceModule.scala
package modules

import actors.ArticleActor
import com.google.inject.AbstractModule
import org.apache.pekko.actor.typed.ActorSystem
import org.apache.pekko.cluster.sharding.typed.scaladsl.ClusterSharding
import play.api.libs.concurrent.PekkoGuiceSupport

class PersistenceModule extends AbstractModule with PekkoGuiceSupport {
  override def configure(): Unit = {
    bind(classOf[ArticleActorManager]).asEagerSingleton()
  }
}

// Manager: handles actor creation and interaction
@Singleton
class ArticleActorManager @Inject()(
  system: ActorSystem[?]
)(implicit ec: ExecutionContext) {
  import org.apache.pekko.actor.typed.scaladsl.AskPattern.*
  import org.apache.pekko.util.Timeout
  import scala.concurrent.duration.*

  implicit val timeout: Timeout = 5.seconds

  private def articleActor(id: String): ActorRef[ArticleCommand] =
    system.systemActorOf(ArticleActor(id), s"article-$id")

  def createArticle(id: String, title: String, content: String, authorId: String): Future[ArticleReply] =
    articleActor(id).ask(CreateArticle(title, content, authorId, _))

  def publishArticle(id: String): Future[ArticleReply] =
    articleActor(id).ask(PublishArticle(_))

  def getArticle(id: String): Future[Option[ArticleState]] =
    articleActor(id).ask(GetArticleState(_))
}
```

---

## Step 695: Cluster Sharding

```scala
// app/modules/ClusterShardingSetup.scala — scale horizontally
package modules

import actors.ArticleActor
import org.apache.pekko.actor.typed.ActorSystem
import org.apache.pekko.cluster.sharding.typed.scaladsl.*
import javax.inject.*
import scala.concurrent.*

@Singleton
class ShardedArticleManager @Inject()(
  system: ActorSystem[?]
)(implicit ec: ExecutionContext) {

  import org.apache.pekko.actor.typed.scaladsl.AskPattern.*
  import org.apache.pekko.util.Timeout
  import scala.concurrent.duration.*

  implicit val timeout: Timeout = 5.seconds
  implicit val scheduler = system.scheduler

  // Initialize cluster sharding
  private val sharding = ClusterSharding(system)

  sharding.init(Entity(ArticleActor.EntityKey) { ctx =>
    ArticleActor(ctx.entityId)
  })

  // Get actor via sharding (routes to correct shard automatically)
  private def articleRef(id: String): EntityRef[ArticleCommand] =
    sharding.entityRefFor(ArticleActor.EntityKey, id)

  def createArticle(id: String, title: String, content: String, authorId: String): Future[ArticleReply] =
    articleRef(id).ask(CreateArticle(title, content, authorId, _))

  def publishArticle(id: String): Future[ArticleReply] =
    articleRef(id).ask(PublishArticle(_))

  def addView(id: String, userId: Option[String]): Unit =
    articleRef(id) ! AddView(userId)

  def getArticle(id: String): Future[Option[ArticleState]] =
    articleRef(id).ask(GetArticleState(_))
}
```

---

## Step 696: Persistence Query (Read Journal)

```scala
// app/infrastructure/ArticleReadJournal.scala
package infrastructure

import org.apache.pekko.actor.typed.ActorSystem
import org.apache.pekko.persistence.query.*
import org.apache.pekko.persistence.jdbc.query.scaladsl.JdbcReadJournal
import org.apache.pekko.stream.scaladsl.Source
import javax.inject.*
import scala.concurrent.*

@Singleton
class ArticleReadJournal @Inject()(
  system: ActorSystem[?]
)(implicit ec: ExecutionContext) {

  private val readJournal =
    PersistenceQuery(system).readJournalFor[JdbcReadJournal](JdbcReadJournal.Identifier)

  // Stream all events for an article
  def eventsForArticle(id: String): Source[EventEnvelope, ?] =
    readJournal.eventsByPersistenceId(
      persistenceId = id,
      fromSequenceNr = 0L,
      toSequenceNr = Long.MaxValue
    )

  // Stream all Article events (for projection)
  def allArticleEvents(offset: Offset = NoOffset): Source[EventEnvelope, ?] =
    readJournal.eventsByTag("article", offset)

  // Current events (not live)
  def currentEvents(id: String): Source[EventEnvelope, ?] =
    readJournal.currentEventsByPersistenceId(id, 0L, Long.MaxValue)
}
```

---

## Step 697: Projection from Read Journal

```scala
// app/infrastructure/ArticleProjectionWorker.scala
package infrastructure

import actors.ArticleEvent
import org.apache.pekko.actor.typed.ActorSystem
import org.apache.pekko.stream.scaladsl.*
import javax.inject.*
import scala.concurrent.*

@Singleton
class ArticleProjectionWorker @Inject()(
  readJournal: ArticleReadJournal,
  projection: ArticleProjection,
  system: ActorSystem[?]
)(implicit ec: ExecutionContext) {

  // Start projection (call on app startup)
  def start(): Unit = {
    readJournal.allArticleEvents()
      .mapAsync(parallelism = 4) {
        case EventEnvelope(offset, _, _, event: ArticleEvent) =>
          projection.projectPekkoEvent(event)
        case _ =>
          Future.successful(())
      }
      .runWith(Sink.ignore)(org.apache.pekko.stream.Materializer(system))
  }
}
```

---

## Step 698: Controller Integration

```scala
// app/controllers/PersistentArticleController.scala
package controllers

import javax.inject.*
import modules.ShardedArticleManager
import play.api.libs.json.*
import play.api.mvc.*
import scala.concurrent.*

@Singleton
class PersistentArticleController @Inject()(
  val controllerComponents: ControllerComponents,
  articleManager: ShardedArticleManager,
  implicit val ec: ExecutionContext
) extends BaseController {

  def create(): Action[JsValue] = Action.async(parse.json) { request =>
    val id       = java.util.UUID.randomUUID().toString
    val title    = (request.body \ "title").as[String]
    val content  = (request.body \ "content").as[String]
    val authorId = request.session.get("userId").getOrElse("anonymous")

    articleManager.createArticle(id, title, content, authorId).map {
      case ArticleSuccess    => Created(Json.obj("id" -> id))
      case ArticleError(msg) => BadRequest(Json.obj("error" -> msg))
    }
  }

  def publish(id: String): Action[AnyContent] = Action.async {
    articleManager.publishArticle(id).map {
      case ArticleSuccess    => Ok(Json.obj("status" -> "published"))
      case ArticleError(msg) => BadRequest(Json.obj("error" -> msg))
    }
  }

  def get(id: String): Action[AnyContent] = Action.async {
    articleManager.getArticle(id).map {
      case None        => NotFound(Json.obj("error" -> "Not found"))
      case Some(state) => Ok(Json.obj(
        "id"          -> state.id,
        "title"       -> state.title,
        "status"      -> state.status,
        "viewCount"   -> state.viewCount,
        "publishedAt" -> state.publishedAt.map(_.toString)
      ))
    }
  }

  def view(id: String): Action[AnyContent] = Action { request =>
    val userId = request.session.get("userId")
    articleManager.addView(id, userId)
    Ok(Json.obj("status" -> "ok"))
  }
}
```

---

## Step 699: Testing Persistent Actors

```scala
// test/actors/ArticleActorSpec.scala
package actors

import org.apache.pekko.actor.testkit.typed.scaladsl.*
import org.apache.pekko.persistence.testkit.scaladsl.EventSourcedBehaviorTestKit
import org.scalatest.*
import org.scalatest.wordspec.AnyWordSpecLike

class ArticleActorSpec extends ScalaTestWithActorTestKit(
  EventSourcedBehaviorTestKit.config
) with AnyWordSpecLike {

  private val testKit = EventSourcedBehaviorTestKit[ArticleCommand, ArticleEvent, ArticleState](
    system,
    ArticleActor("test-article-1")
  )

  override protected def beforeEach(): Unit = {
    super.beforeEach()
    testKit.clear()
  }

  "ArticleActor" should {
    "create article successfully" in {
      val result = testKit.runCommand(CreateArticle("Hello", "Content", "author-1", _))

      result.reply shouldBe ArticleSuccess
      result.event shouldBe an[ArticleCreatedEv]
      result.stateOfType[ArticleState].title shouldBe "Hello"
      result.stateOfType[ArticleState].status shouldBe "draft"
    }

    "publish article" in {
      testKit.runCommand(CreateArticle("Hello", "Content", "author-1", _))

      val result = testKit.runCommand(PublishArticle(_))
      result.reply shouldBe ArticleSuccess
      result.stateOfType[ArticleState].status shouldBe "published"
    }

    "reject publishing already published article" in {
      testKit.runCommand(CreateArticle("Hello", "Content", "author-1", _))
      testKit.runCommand(PublishArticle(_))

      val result = testKit.runCommand(PublishArticle(_))
      result.reply shouldBe ArticleError("Already published")
    }

    "track view count" in {
      testKit.runCommand(CreateArticle("Hello", "Content", "author-1", _))

      testKit.runCommand[Nothing](_ => AddView(Some("user-1")))
      testKit.runCommand[Nothing](_ => AddView(Some("user-2")))

      val state = testKit.runCommand(GetArticleState(_))
      state.reply.get.viewCount shouldBe 2
    }
  }
}
```

---

## Step 700: Production Considerations

```scala
// Graceful shutdown
import org.apache.pekko.actor.CoordinatedShutdown

CoordinatedShutdown(system).addTask(
  CoordinatedShutdown.PhaseBeforeServiceUnbind,
  "drain-actors"
) { () =>
  // รอให้ actors finish processing
  Future.successful(Done)
}

// Passivation (save memory สำหรับ inactive actors)
private def withPassivation(behavior: Behavior[ArticleCommand]): Behavior[ArticleCommand] =
  Behaviors.withTimers[ArticleCommand] { timers =>
    timers.startSingleTimer("passivate", StopArticleActor, 30.minutes)
    behavior
  }

// Dead letter handling
system.deadLetters // monitor ใน production

// Metrics
import org.apache.pekko.actor.ActorSystem
val metrics = Kamon.metrics.actorSystem(system.name)
```

---

## สรุป Part 70

| Concept | Pekko API | จุดประสงค์ |
|---------|----------|-----------|
| EventSourcedBehavior | Core persistence behavior | Actor + event sourcing |
| PersistenceId | Unique ID for actor | Identity |
| commandHandler | Command → Effect | Process commands |
| eventHandler | (State, Event) → State | Update state |
| Effect.persist | Persist events | Durable state |
| Effect.reply | Reply to sender | Responses |
| RetentionCriteria | Snapshot frequency | Performance |
| EntityTypeKey | Cluster sharding key | Scale out |
| EventsByTag | Read journal | Projections |
| EventSourcedBehaviorTestKit | Testing | Unit tests |

---

## สรุปภาพรวม Phase 7: Database & Persistence

```
Part 61: Slick Intro      — FRM, table definitions, basic CRUD
Part 62: Slick Queries    — Complex queries, aggregates, raw SQL
Part 63: Slick Relations  — One-to-many, many-to-many, joins
Part 64: Doobie           — Functional JDBC, SQL fragments, Cats Effect
Part 65: MongoDB          — Document DB, ReactiveMongo, aggregations
Part 66: Redis            — Caching, pub/sub, sorted sets, sessions
Part 67: Elasticsearch    — Full-text search, indexing, aggregations
Part 68: Migrations       — Flyway, versioned migrations, CI
Part 69: CQRS/ES          — Command/Query separation, event store
Part 70: Pekko Persistence — Actor persistence, snapshots, read journal
```

---

## แบบฝึกหัด Part 70

1. **Blog with Pekko**: Rebuild blog backend ด้วย Pekko Persistence ที่ Article, User, Comment แต่ละอันเป็น PersistentActor

2. **Cluster Deployment**: Deploy Pekko Persistence app ใน 3-node cluster ด้วย Cluster Sharding และ verify ว่า events ถูก route ไปถูก node

3. **CQRS with Journal**: สร้าง projection ที่ subscribe Pekko Read Journal และ build Read Model ใน PostgreSQL แบบ real-time

4. **Snapshot Strategy**: Implement adaptive snapshot strategy ที่ take snapshot เมื่อ event count > threshold หรือ เมื่อ state size > limit

5. **Migration**: Migrate existing CRUD service เป็น Event Sourcing ด้วย Pekko Persistence โดยที่ existing data ยังคง accessible

---

**🎉 ยินดีด้วย! คุณเรียนจบ Phase 7: Database & Persistence แล้ว!**

ความรู้ที่ได้รับในหลักสูตรนี้ครอบคลุม:
- Play Framework 3.0 (Phase 5-6)
- Database & Persistence patterns (Phase 7)
- Production deployment และ monitoring

ต่อไปใน Phase 8-10:
- Microservices Architecture
- Cloud & DevOps
- Advanced Patterns
