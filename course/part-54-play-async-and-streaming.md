# Part 54: Play Async and Streaming

## Steps 531-540: Async Actions, Chunked Responses, Server-Sent Events, Streaming Body

---

## Step 531: Async Programming ใน Play

Play ใช้ non-blocking I/O โดยพื้นฐาน - ทุก Action ควรเป็น async เพื่อ performance สูงสุด

```scala
// build.sbt
libraryDependencies ++= Seq(
  guice,
  "org.apache.pekko" %% "pekko-stream"       % "1.0.2",
  "org.apache.pekko" %% "pekko-stream-typed"  % "1.0.2"
)
```

### Blocking vs Non-Blocking

```scala
// ❌ Blocking - ใช้ Thread ค้างไว้
def blockingAction(): Action[AnyContent] = Action { _ =>
  val result = database.query()  // Thread blocked!
  Ok(result)
}

// ✅ Non-Blocking - Thread ไม่ค้าง
def nonBlockingAction(): Action[AnyContent] = Action.async { _ =>
  database.queryAsync().map { result =>  // Future-based
    Ok(result)
  }
}
```

---

## Step 532: Action.async

```scala
// app/controllers/AsyncController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import scala.concurrent.*
import scala.concurrent.duration.*

@Singleton
class AsyncController @Inject()(
  val controllerComponents: ControllerComponents,
  userService: services.UserService,
  articleService: services.ArticleService,
  implicit val ec: ExecutionContext
) extends BaseController {

  // Basic async action
  def getUser(id: Long): Action[AnyContent] = Action.async { implicit request =>
    userService.findById(id).map {
      case Some(user) => Ok(play.api.libs.json.Json.toJson(user))
      case None       => NotFound
    }
  }

  // รวม multiple futures
  def dashboard(): Action[AnyContent] = Action.async { implicit request =>
    val userId = request.session.get("userId").map(_.toLong).getOrElse(0L)

    // รัน parallel
    val userFuture     = userService.findById(userId)
    val articlesFuture = articleService.list(1, 5)
    val statsFuture    = userService.getStats(userId)

    // รอ ทั้ง 3 futures
    for {
      userOpt   <- userFuture
      (articles, _) <- articlesFuture
      stats     <- statsFuture
    } yield {
      userOpt match {
        case None    => Unauthorized("Not logged in")
        case Some(u) => Ok(views.html.dashboard(u, articles, stats))
      }
    }
  }

  // Async ด้วย timeout
  def withTimeout(): Action[AnyContent] = Action.async { implicit request =>
    import org.apache.pekko.pattern.after
    import org.apache.pekko.actor.ActorSystem
    implicit val system: ActorSystem = ???  // inject

    val operation = userService.findById(1L)
    val timeout   = after(5.seconds, system.scheduler)(Future.failed(new TimeoutException("Timeout!")))

    Future.firstCompletedOf(Seq(operation, timeout)).map {
      case Some(user) => Ok(user.username)
      case None       => NotFound
    }.recover {
      case _: TimeoutException => GatewayTimeout("Operation timed out")
      case ex => InternalServerError(ex.getMessage)
    }
  }

  // Retry ที่ fail
  def withRetry(): Action[AnyContent] = Action.async { implicit request =>
    def attempt(retries: Int): Future[String] = {
      userService.callExternalApi()
        .recoverWith {
          case ex if retries > 0 =>
            play.api.Logger("retry").warn(s"Retrying... ($retries left): ${ex.getMessage}")
            after(1.second, scala.concurrent.ExecutionContext.Implicits.global)(attempt(retries - 1))
          case ex =>
            Future.failed(ex)
        }
    }

    attempt(3).map(Ok(_)).recover {
      case ex => ServiceUnavailable(s"Failed after retries: ${ex.getMessage}")
    }
  }
}
```

---

## Step 533: Execution Context

```scala
// app/executors/CustomExecutors.scala
package executors

import javax.inject.*
import play.api.libs.concurrent.CustomExecutionContext
import org.apache.pekko.actor.ActorSystem

// Custom executor สำหรับ blocking operations (DB, file I/O)
@Singleton
class DatabaseExecutionContext @Inject()(system: ActorSystem)
  extends CustomExecutionContext(system, "database-dispatcher")

// Custom executor สำหรับ CPU-intensive tasks
@Singleton
class CpuExecutionContext @Inject()(system: ActorSystem)
  extends CustomExecutionContext(system, "cpu-dispatcher")
```

```hocon
# conf/application.conf

# Database dispatcher - larger pool สำหรับ blocking I/O
database-dispatcher {
  executor = "thread-pool-executor"
  throughput = 1
  thread-pool-executor {
    fixed-pool-size = 50  # สำหรับ JDBC connections
  }
}

# CPU dispatcher - smaller pool สำหรับ CPU work
cpu-dispatcher {
  executor = "fork-join-executor"
  throughput = 1
  fork-join-executor {
    parallelism-min    = 4
    parallelism-factor = 2.0
    parallelism-max    = 8
  }
}
```

```scala
// ใช้ custom executor
@Singleton
class ArticleRepository @Inject()(
  db: slick.jdbc.JdbcBackend.Database,
  dbEc: DatabaseExecutionContext
) {
  // ใช้ dbEc สำหรับ database operations
  def findAll(): Future[List[models.Article]] = {
    implicit val ec = dbEc
    Future {
      // blocking database call - run ใน database thread pool
      db.withSession { session =>
        // query...
        List.empty
      }
    }
  }
}
```

---

## Step 534: Chunked Response

```scala
// app/controllers/StreamController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import org.apache.pekko.stream.scaladsl.*
import org.apache.pekko.util.ByteString
import scala.concurrent.duration.*

@Singleton
class StreamController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  // Chunked text response
  def chunkedText(): Action[AnyContent] = Action { implicit request =>
    // Source ที่ emit values ทีละชิ้น
    val source = Source(1 to 100)
      .map(i => ByteString(s"Line $i\n"))
      .throttle(10, 1.second)  // 10 lines per second

    Ok.chunked(source).as("text/plain")
  }

  // Stream large CSV
  def exportCsv(): Action[AnyContent] = Action { implicit request =>
    // สร้าง CSV header
    val header = Source.single(ByteString("id,name,email,created_at\n"))

    // Stream rows จาก database
    val rows = Source(1 to 10000)
      .map { i =>
        ByteString(s"$i,User$i,user$i@example.com,${java.time.LocalDate.now()}\n")
      }

    val csvSource = header.concat(rows)

    Ok.chunked(csvSource)
      .as("text/csv")
      .withHeaders(
        "Content-Disposition" -> "attachment; filename=\"users.csv\""
      )
  }

  // Stream JSON array
  def streamJson(): Action[AnyContent] = Action { implicit request =>
    import play.api.libs.json.*

    val items = Source(1 to 1000)
      .map(i => Json.obj("id" -> i, "name" -> s"Item $i"))
      .map(Json.stringify)
      .map(ByteString(_))

    // Wrap ใน JSON array brackets
    val jsonStream = Source.single(ByteString("["))
      .concat(items.intersperse(ByteString(","), ByteString(","), ByteString("]")))

    Ok.chunked(jsonStream).as("application/json")
  }
}
```

---

## Step 535: Server-Sent Events (SSE)

```scala
// app/controllers/SseController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import org.apache.pekko.stream.scaladsl.*
import org.apache.pekko.util.ByteString
import scala.concurrent.duration.*

@Singleton
class SseController @Inject()(
  val controllerComponents: ControllerComponents,
  notificationService: services.NotificationService
) extends BaseController {

  // Basic SSE
  def events(): Action[AnyContent] = Action { implicit request =>
    val source = Source.tick(0.seconds, 1.second, ())
      .zipWithIndex
      .map { case (_, i) =>
        val data = play.api.libs.json.Json.obj("index" -> i, "time" -> System.currentTimeMillis())
        ByteString(s"data: ${play.api.libs.json.Json.stringify(data)}\n\n")
      }
      .keepAlive(30.seconds, () => ByteString(": heartbeat\n\n"))

    Ok.chunked(source)
      .as("text/event-stream")
      .withHeaders(
        "Cache-Control"     -> "no-cache",
        "X-Accel-Buffering" -> "no"
      )
  }

  // Typed SSE events
  def typedEvents(): Action[AnyContent] = Action { implicit request =>
    def sseEvent(eventType: String, data: String, id: Option[Long] = None): ByteString = {
      val idPart = id.map(i => s"id: $i\n").getOrElse("")
      ByteString(s"${idPart}event: $eventType\ndata: $data\n\n")
    }

    var eventId = 0L

    val source = Source.tick(0.seconds, 2.seconds, ())
      .map { _ =>
        eventId += 1
        val event = eventId % 3 match {
          case 0 => sseEvent("userJoined", s"""{"userId": $eventId}""", Some(eventId))
          case 1 => sseEvent("messageReceived", s"""{"text": "Hello $eventId"}""", Some(eventId))
          case _ => sseEvent("statusUpdate", s"""{"status": "active"}""", Some(eventId))
        }
        event
      }

    Ok.chunked(source).as("text/event-stream")
  }

  // SSE จาก real-time data
  def notifications(userId: Long): Action[AnyContent] = Action { implicit request =>
    val notificationSource = notificationService.getNotificationStream(userId)
      .map { notification =>
        ByteString(s"event: notification\ndata: ${play.api.libs.json.Json.stringify(play.api.libs.json.Json.toJson(notification))}\n\n")
      }

    Ok.chunked(notificationSource).as("text/event-stream")
  }
}
```

### JavaScript SSE Client

```javascript
// public/javascripts/sse-client.js

const userId = 42;  // current user
const eventSource = new EventSource(`/sse/notifications/${userId}`);

// Default message handler
eventSource.onmessage = function(event) {
  const data = JSON.parse(event.data);
  console.log('Generic event:', data);
};

// Typed event handler
eventSource.addEventListener('notification', function(event) {
  const notification = JSON.parse(event.data);
  showNotification(notification);
});

eventSource.addEventListener('userJoined', function(event) {
  const data = JSON.parse(event.data);
  console.log('User joined:', data.userId);
});

// Connection events
eventSource.onopen = function() {
  console.log('SSE Connected');
};

eventSource.onerror = function(error) {
  if (eventSource.readyState === EventSource.CLOSED) {
    console.log('SSE Connection closed, reconnecting...');
    setTimeout(() => location.reload(), 3000);
  }
};

function showNotification(notification) {
  // แสดง notification ใน UI
  const toast = document.createElement('div');
  toast.className = 'toast';
  toast.innerHTML = notification.message;
  document.getElementById('notifications').appendChild(toast);
  setTimeout(() => toast.remove(), 5000);
}
```

---

## Step 536: Streaming File Downloads

```scala
// app/controllers/FileController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import org.apache.pekko.stream.scaladsl.*
import org.apache.pekko.util.ByteString
import java.nio.file.*

@Singleton
class FileController @Inject()(
  val controllerComponents: ControllerComponents,
  implicit val mat: org.apache.pekko.stream.Materializer
) extends BaseController {

  // Stream file download
  def download(filename: String): Action[AnyContent] = Action { implicit request =>
    val filePath = Paths.get(s"/var/files/$filename")

    if (!Files.exists(filePath)) {
      NotFound(s"File not found: $filename")
    } else {
      val fileSource = FileIO.fromPath(filePath)
      val fileSize = Files.size(filePath)
      val contentType = detectContentType(filename)

      Ok.streamed(
        content       = fileSource,
        contentLength = Some(fileSize),
        contentType   = Some(contentType)
      ).withHeaders(
        "Content-Disposition" -> s"""attachment; filename="$filename""""
      )
    }
  }

  // Stream กับ Range header (สำหรับ video streaming)
  def streamVideo(filename: String): Action[AnyContent] = Action { implicit request =>
    val filePath = Paths.get(s"/var/videos/$filename")
    val fileSize = Files.size(filePath)

    // Parse Range header
    val rangeHeader = request.headers.get("Range")
    val (start, end) = rangeHeader.flatMap(parseRange(_, fileSize)) match {
      case Some((s, e)) => (s, e)
      case None         => (0L, fileSize - 1)
    }

    val length = end - start + 1
    val fileSource = FileIO.fromPath(filePath, chunkSize = 8192, startPosition = start)
      .take(length)

    if (rangeHeader.isDefined) {
      // Partial content response
      Status(206)(fileSource)
        .as("video/mp4")
        .withHeaders(
          "Content-Range"  -> s"bytes $start-$end/$fileSize",
          "Content-Length" -> length.toString,
          "Accept-Ranges"  -> "bytes"
        )
    } else {
      Ok.streamed(fileSource, Some(fileSize), Some("video/mp4"))
        .withHeaders("Accept-Ranges" -> "bytes")
    }
  }

  private def parseRange(header: String, fileSize: Long): Option[(Long, Long)] = {
    val rangePattern = """bytes=(\d+)-(\d*)""".r
    header match {
      case rangePattern(start, end) =>
        val s = start.toLong
        val e = if (end.isEmpty) fileSize - 1 else end.toLong
        Some((s, Math.min(e, fileSize - 1)))
      case _ => None
    }
  }

  private def detectContentType(filename: String): String =
    filename.split("\\.").lastOption.getOrElse("").toLowerCase match {
      case "pdf"  => "application/pdf"
      case "jpg" | "jpeg" => "image/jpeg"
      case "png"  => "image/png"
      case "mp4"  => "video/mp4"
      case "zip"  => "application/zip"
      case _      => "application/octet-stream"
    }
}
```

---

## Step 537: Streaming Request Body

```scala
// app/controllers/UploadController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import org.apache.pekko.stream.scaladsl.*
import org.apache.pekko.util.ByteString
import java.nio.file.*
import scala.concurrent.*

@Singleton
class UploadController @Inject()(
  val controllerComponents: ControllerComponents,
  implicit val mat: org.apache.pekko.stream.Materializer,
  implicit val ec: ExecutionContext
) extends BaseController {

  // Stream large file upload
  def upload(): Action[Source[ByteString, ?]] = Action(
    parse.byteString.map(Source.single)
      .orElse(parse.raw.map(raw => FileIO.fromPath(raw.asFile.toPath)))
  ) { implicit request =>
    // Process upload stream
    val filename = s"upload_${System.currentTimeMillis()}.bin"
    val outputPath = Paths.get(s"/tmp/$filename")

    // Write stream to file
    val sink = FileIO.toPath(outputPath)
    val result = request.body.runWith(sink)

    Ok(s"File uploaded to: $filename")
  }

  // Process JSON stream
  def processJsonStream(): Action[AnyContent] = Action.async { implicit request =>
    // อ่าน body เป็น stream
    request.body.asRaw.map { rawBuffer =>
      val bytes = rawBuffer.asBytes().getOrElse(ByteString.empty)
      import play.api.libs.json.*
      val json = Json.parse(bytes.toArray)
      Future.successful(Ok(s"Processed ${(json \ "items").as[JsArray].value.length} items"))
    }.getOrElse(Future.successful(BadRequest("No body")))
  }
}
```

---

## Step 538: Backpressure และ Flow Control

```scala
// app/streams/DataProcessor.scala
package streams

import org.apache.pekko.stream.scaladsl.*
import org.apache.pekko.util.ByteString
import scala.concurrent.*
import scala.concurrent.duration.*

object DataProcessor {

  // Process data ด้วย backpressure
  def processWithBackpressure[T](
    source: Source[T, ?],
    processOne: T => Future[String],
    parallelism: Int = 4
  )(implicit ec: ExecutionContext, mat: org.apache.pekko.stream.Materializer): Future[Int] = {

    source
      .mapAsync(parallelism)(processOne)  // process n items ต่อกัน
      .runFold(0)((count, _) => count + 1)
  }

  // Buffer ก่อน process
  def bufferedProcessing(source: Source[String, ?])(
    implicit ec: ExecutionContext,
    mat: org.apache.pekko.stream.Materializer
  ): Future[Seq[String]] = {

    source
      .buffer(100, org.apache.pekko.stream.OverflowStrategy.backpressure)
      .groupedWithin(50, 1.second)  // batch 50 items หรือ 1 วินาที
      .mapAsync(2) { batch =>
        Future {
          batch.map(item => s"Processed: $item")
        }
      }
      .mapConcat(identity)
      .runWith(Sink.seq)
  }

  // Merge multiple streams
  def mergeStreams(
    sources: List[Source[String, ?]]
  )(implicit ec: ExecutionContext, mat: org.apache.pekko.stream.Materializer): Future[Seq[String]] = {

    val merged = sources.foldLeft(Source.empty[String]) { (acc, src) =>
      acc.merge(src)
    }

    merged.runWith(Sink.seq)
  }
}
```

---

## Step 539: Long-Polling

```scala
// app/controllers/LongPollingController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*
import scala.concurrent.*
import scala.concurrent.duration.*
import org.apache.pekko.actor.ActorSystem

@Singleton
class LongPollingController @Inject()(
  val controllerComponents: ControllerComponents,
  system: ActorSystem,
  implicit val ec: ExecutionContext
) extends BaseController {

  private val longPollTimeout = 25.seconds

  // Long polling endpoint
  def poll(lastEventId: Option[Long] = None): Action[AnyContent] = Action.async { implicit request =>
    val timeoutFuture = org.apache.pekko.pattern.after(
      longPollTimeout,
      system.scheduler
    )(Future.successful(NoContent))  // 204 = no new events

    val eventFuture = waitForEvents(lastEventId).map { events =>
      Ok(Json.toJson(events))
    }

    Future.firstCompletedOf(Seq(eventFuture, timeoutFuture))
  }

  private def waitForEvents(afterId: Option[Long]): Future[List[JsObject]] = {
    // Poll database ทุก 500ms สูงสุด 25 วินาที
    def checkForEvents(attempts: Int): Future[List[JsObject]] = {
      val events = getNewEvents(afterId)
      if (events.nonEmpty || attempts <= 0) {
        Future.successful(events)
      } else {
        org.apache.pekko.pattern.after(500.milliseconds, system.scheduler)(
          checkForEvents(attempts - 1)
        )
      }
    }

    checkForEvents(50)  // 50 attempts * 500ms = 25s
  }

  private def getNewEvents(afterId: Option[Long]): List[JsObject] = {
    // query database สำหรับ events ใหม่
    List.empty  // placeholder
  }
}
```

---

## Step 540: Async Best Practices

```scala
// ✅ Best Practices สำหรับ Async ใน Play

// 1. ใช้ proper execution context
class MyService @Inject()(
  dbEc: DatabaseExecutionContext,  // สำหรับ DB
  implicit val defaultEc: ExecutionContext  // สำหรับ business logic
) {
  def processAndSave(data: String): Future[String] = {
    implicit val ec = dbEc  // ใช้ DB executor สำหรับ DB operations
    Future { /* database operation */ "saved" }
  }
}

// 2. ไม่ block ใน Future
// ❌ ผิด
Future { Thread.sleep(1000); "done" }(defaultExecutionContext)  // blocks play thread!

// ✅ ถูก
Future { Thread.sleep(1000); "done" }(blockingExecutionContext)  // ใช้ blocking pool

// 3. Handle errors properly
def safeAsync(): Future[Result] =
  riskyOperation()
    .map(result => Ok(result))
    .recover {
      case ex: java.sql.SQLException => ServiceUnavailable("Database error")
      case ex: java.util.concurrent.TimeoutException => GatewayTimeout("Timeout")
      case ex => InternalServerError(ex.getMessage)
    }

// 4. ไม่ ignore Future
// ❌ ผิด
def badAction(): Action[AnyContent] = Action { _ =>
  someService.asyncOperation()  // Future ignored!
  Ok("done")
}

// ✅ ถูก
def goodAction(): Action[AnyContent] = Action.async { _ =>
  someService.asyncOperation().map(_ => Ok("done"))
}

// Placeholder
def riskyOperation(): Future[String] = Future.successful("ok")
def someService = new { def asyncOperation() = Future.successful(()) }
```

---

## สรุป Part 54

| Pattern | API | Use Case |
|---------|-----|---------|
| Async Action | `Action.async { Future { } }` | DB queries, API calls |
| Parallel futures | `for { a <- futA; b <- futB }` | Multiple independent operations |
| Chunked response | `Ok.chunked(source)` | Large responses |
| Server-Sent Events | `Ok.chunked(src).as("text/event-stream")` | Real-time push |
| File streaming | `Ok.streamed(fileSource, length)` | File downloads |
| Backpressure | `mapAsync(n)` | Rate-limited processing |
| Custom executor | `CustomExecutionContext` | Blocking operations |

---

## แบบฝึกหัด Part 54

1. **Async Data Pipeline**: สร้าง pipeline ที่ fetch data จาก 3 external APIs concurrently และ merge ผลลัพธ์ใน response

2. **Progress Stream**: สร้าง SSE endpoint ที่ report progress ของ long-running job (เช่น data import) แบบ real-time

3. **Large File Export**: สร้าง CSV export endpoint ที่ stream 1 million rows จาก database โดยไม่ load ทั้งหมดเข้า memory

4. **Retry with Backoff**: Implement exponential backoff retry logic สำหรับ external API calls ที่ fail

5. **Request Coalescing**: Implement request coalescing ที่รวม multiple concurrent requests สำหรับข้อมูลเดียวกัน ให้เป็น DB query เดียว

---

[→ ไปยัง Part 55: Play Caching](part-55-play-caching.md)
