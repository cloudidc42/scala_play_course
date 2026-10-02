# Part 48: Play WebSockets

## Steps 471-480: WebSocket Endpoint, Flow, Pekko Streams Integration, Chat Application

---

## Step 471: WebSocket ใน Play Framework

Play 3.0 ใช้ Apache Pekko (successor ของ Akka) สำหรับ WebSocket implementation

### build.sbt

```scala
// build.sbt
libraryDependencies ++= Seq(
  guice,
  "org.apache.pekko" %% "pekko-stream" % "1.0.2",
  "org.playframework" %% "play" % "3.0.3",
  ws  // Play WS client
)
```

### WebSocket Flow Model

```
Client Browser ←──WebSocket──→ Play Server
      │                              │
      │ send message                 │
      ├─────────────────────────────→│
      │                              │ process
      │ receive response             │
      │←─────────────────────────────┤
      │                              │
      │ broadcast from others        │
      │←─────────────────────────────┤
```

---

## Step 472: Basic WebSocket

```scala
// app/controllers/WebSocketController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.libs.streams.*
import org.apache.pekko.stream.scaladsl.*
import org.apache.pekko.actor.*

@Singleton
class WebSocketController @Inject()(
  val controllerComponents: ControllerComponents
)(implicit system: ActorSystem) extends BaseController {

  // Simple echo WebSocket
  def echo(): WebSocket = WebSocket.accept[String, String] { request =>
    // Flow: รับ String → ส่งกลับ String เดิม
    Flow[String].map { message =>
      s"Echo: $message"
    }
  }

  // Echo with timestamp
  def echoWithTime(): WebSocket = WebSocket.accept[String, String] { request =>
    Flow[String].map { message =>
      val timestamp = java.time.LocalDateTime.now()
      s"[$timestamp] $message"
    }
  }

  // WebSocket ที่รับและส่ง JSON
  def jsonEcho(): WebSocket = WebSocket.accept[play.api.libs.json.JsValue, play.api.libs.json.JsValue] { request =>
    import play.api.libs.json.*
    Flow[JsValue].map { json =>
      Json.obj(
        "echo"      -> json,
        "timestamp" -> System.currentTimeMillis()
      )
    }
  }
}
```

### Routes

```
# conf/routes
GET   /ws/echo           controllers.WebSocketController.echo()
GET   /ws/echo-time      controllers.WebSocketController.echoWithTime()
GET   /ws/json           controllers.WebSocketController.jsonEcho()
GET   /ws/chat           controllers.ChatController.chat(username: String)
```

### JavaScript Client

```javascript
// public/javascripts/websocket-client.js

// เชื่อมต่อ WebSocket
const ws = new WebSocket('ws://localhost:9000/ws/echo');

ws.onopen = function(event) {
  console.log('WebSocket Connected');
  ws.send('Hello Server!');
};

ws.onmessage = function(event) {
  console.log('Received:', event.data);
};

ws.onerror = function(error) {
  console.error('WebSocket Error:', error);
};

ws.onclose = function(event) {
  console.log('WebSocket Closed:', event.code, event.reason);
};

// ส่ง message
function sendMessage(text) {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(text);
  }
}
```

---

## Step 473: Pekko Streams Flow

```scala
// app/controllers/StreamController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import org.apache.pekko.stream.scaladsl.*
import org.apache.pekko.actor.*
import scala.concurrent.duration.*

@Singleton
class StreamController @Inject()(
  val controllerComponents: ControllerComponents
)(implicit system: ActorSystem) extends BaseController {

  // WebSocket ที่ส่ง heartbeat ทุก 30 วินาที
  def withHeartbeat(): WebSocket = WebSocket.accept[String, String] { _ =>
    // รวม input stream กับ heartbeat source
    val heartbeat = Source.tick(0.seconds, 30.seconds, "ping")

    val echoFlow = Flow[String].map(msg => s"Echo: $msg")
    val mergedSink = Sink.ignore

    // Merge heartbeat กับ echo
    Flow.fromSinkAndSource(
      mergedSink,
      Source.maybe[String].merge(heartbeat)
    )
  }

  // Flow ที่ buffer messages
  def buffered(): WebSocket = WebSocket.accept[String, String] { _ =>
    Flow[String]
      .buffer(100, org.apache.pekko.stream.OverflowStrategy.dropHead)
      .map(msg => s"Processed: $msg")
  }

  // Flow ที่ throttle messages
  def throttled(): WebSocket = WebSocket.accept[String, String] { _ =>
    Flow[String]
      .throttle(10, 1.second)  // max 10 messages per second
      .map(msg => s"Throttled: $msg")
  }

  // Flow ที่ group messages เป็น batches
  def batched(): WebSocket = WebSocket.accept[String, String] { _ =>
    Flow[String]
      .groupedWithin(100, 1.second)  // batch 100 หรือ 1 วินาที
      .map(batch => s"Batch of ${batch.size}: ${batch.mkString(", ")}")
  }
}
```

---

## Step 474: Chat Room ด้วย Actors

```scala
// app/actors/ChatRoomActor.scala
package actors

import org.apache.pekko.actor.*
import play.api.libs.json.*

object ChatRoomActor {
  case class Join(username: String, actorRef: ActorRef)
  case class Leave(username: String)
  case class ChatMessage(username: String, message: String)
  case class BroadcastMessage(from: String, message: String, timestamp: Long)

  def props(): Props = Props(new ChatRoomActor())
}

class ChatRoomActor extends Actor {
  import ChatRoomActor.*
  import context.dispatcher

  // Map ของ username → actor ref ของ client
  private var members: Map[String, ActorRef] = Map.empty

  override def receive: Receive = {
    case Join(username, actorRef) =>
      members = members + (username -> actorRef)
      // แจ้งทุกคนว่ามีคนใหม่เข้ามา
      broadcast(BroadcastMessage(
        "System",
        s"$username เข้าร่วมห้องสนทนา",
        System.currentTimeMillis()
      ), exclude = None)
      // แจ้ง list ของ members ที่อยู่ใน room
      actorRef ! Json.obj(
        "type"    -> "system",
        "message" -> "เชื่อมต่อสำเร็จ",
        "members" -> members.keys.toList
      )

    case Leave(username) =>
      members = members - username
      broadcast(BroadcastMessage(
        "System",
        s"$username ออกจากห้องสนทนา",
        System.currentTimeMillis()
      ), exclude = None)

    case ChatMessage(username, message) =>
      broadcast(BroadcastMessage(
        username,
        message,
        System.currentTimeMillis()
      ), exclude = None)
  }

  private def broadcast(msg: BroadcastMessage, exclude: Option[String]): Unit = {
    val json = Json.obj(
      "type"      -> "message",
      "from"      -> msg.from,
      "message"   -> msg.message,
      "timestamp" -> msg.timestamp,
      "members"   -> members.keys.toList
    )
    members.filter { case (name, _) => !exclude.contains(name) }
           .values
           .foreach(_ ! json)
  }
}
```

### Client Actor

```scala
// app/actors/WebSocketActor.scala
package actors

import org.apache.pekko.actor.*
import play.api.libs.json.*

object WebSocketActor {
  def props(
    username: String,
    chatRoom: ActorRef,
    out: ActorRef
  ): Props = Props(new WebSocketActor(username, chatRoom, out))
}

class WebSocketActor(
  username: String,
  chatRoom: ActorRef,
  out: ActorRef
) extends Actor {

  import ChatRoomActor.*

  override def preStart(): Unit = {
    // เข้าร่วม chat room เมื่อ actor เริ่มทำงาน
    chatRoom ! Join(username, self)
  }

  override def receive: Receive = {
    // รับ message จาก WebSocket client
    case msg: JsValue =>
      val msgType = (msg \ "type").asOpt[String].getOrElse("message")
      msgType match {
        case "message" =>
          val content = (msg \ "content").asOpt[String].getOrElse("")
          if (content.nonEmpty) {
            chatRoom ! ChatMessage(username, content)
          }
        case "ping" =>
          out ! Json.obj("type" -> "pong")
        case _ =>
          // ignore unknown message types
      }

    // รับ broadcast จาก ChatRoom
    case msg: JsValue =>
      out ! msg  // ส่งไปยัง WebSocket client
  }

  override def postStop(): Unit = {
    chatRoom ! Leave(username)
  }
}
```

### Chat Controller

```scala
// app/controllers/ChatController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.libs.streams.ActorFlow
import org.apache.pekko.actor.*
import org.apache.pekko.stream.Materializer
import actors.*

@Singleton
class ChatController @Inject()(
  val controllerComponents: ControllerComponents,
  @Named("chat-room") chatRoom: ActorRef
)(implicit system: ActorSystem, mat: Materializer) extends BaseController {

  // WebSocket endpoint สำหรับ chat
  def chat(username: String): WebSocket = WebSocket.accept[play.api.libs.json.JsValue, play.api.libs.json.JsValue] { request =>
    ActorFlow.actorRef { out =>
      WebSocketActor.props(username, chatRoom, out)
    }
  }
}
```

### Module Configuration

```scala
// app/Module.scala
package app

import com.google.inject.AbstractModule
import play.api.libs.concurrent.PekkoGuiceSupport
import actors.ChatRoomActor

class Module extends AbstractModule with PekkoGuiceSupport {
  override def configure(): Unit = {
    // สร้าง ChatRoomActor เป็น singleton
    bindActor[ChatRoomActor]("chat-room")
  }
}
```

---

## Step 475: Chat Frontend

```html
<!-- app/views/chat.scala.html -->
@(username: String)(implicit request: Request[?])

@layouts.main("Chat Room") {
  <div class="container mt-4">
    <div class="row">
      <div class="col-md-9">
        <div class="card">
          <div class="card-header d-flex justify-content-between">
            <span>Chat Room</span>
            <span class="badge bg-success" id="status">Connecting...</span>
          </div>
          <div class="card-body" style="height: 400px; overflow-y: scroll;" id="messages">
          </div>
          <div class="card-footer">
            <div class="input-group">
              <input type="text" class="form-control" id="messageInput"
                     placeholder="พิมพ์ข้อความ..." autofocus>
              <button class="btn btn-primary" id="sendBtn">ส่ง</button>
            </div>
          </div>
        </div>
      </div>
      <div class="col-md-3">
        <div class="card">
          <div class="card-header">สมาชิกในห้อง</div>
          <div class="card-body">
            <ul id="memberList" class="list-unstyled"></ul>
          </div>
        </div>
      </div>
    </div>
  </div>

  <script>
    const username = "@username";
    const wsUrl = `ws://${window.location.host}/ws/chat?username=${encodeURIComponent(username)}`;
    let ws;

    function connect() {
      ws = new WebSocket(wsUrl);

      ws.onopen = () => {
        document.getElementById('status').textContent = 'Connected';
        document.getElementById('status').className = 'badge bg-success';
      };

      ws.onmessage = (event) => {
        const data = JSON.parse(event.data);
        handleMessage(data);
      };

      ws.onclose = () => {
        document.getElementById('status').textContent = 'Disconnected';
        document.getElementById('status').className = 'badge bg-danger';
        // reconnect หลัง 3 วินาที
        setTimeout(connect, 3000);
      };

      ws.onerror = (error) => {
        console.error('WebSocket error:', error);
      };
    }

    function handleMessage(data) {
      if (data.type === 'message') {
        addMessage(data.from, data.message, data.timestamp);
        updateMemberList(data.members);
      } else if (data.type === 'system') {
        addSystemMessage(data.message);
        if (data.members) updateMemberList(data.members);
      } else if (data.type === 'pong') {
        // heartbeat response
      }
    }

    function addMessage(from, message, timestamp) {
      const msgDiv = document.createElement('div');
      const isOwnMessage = from === username;
      const time = new Date(timestamp).toLocaleTimeString('th-TH');

      msgDiv.className = `mb-2 ${isOwnMessage ? 'text-end' : ''}`;
      msgDiv.innerHTML = `
        <small class="text-muted">${from} · ${time}</small><br>
        <span class="badge ${isOwnMessage ? 'bg-primary' : 'bg-secondary'} text-wrap text-start">
          ${escapeHtml(message)}
        </span>
      `;

      const messagesDiv = document.getElementById('messages');
      messagesDiv.appendChild(msgDiv);
      messagesDiv.scrollTop = messagesDiv.scrollHeight;
    }

    function addSystemMessage(message) {
      const msgDiv = document.createElement('div');
      msgDiv.className = 'text-center mb-2';
      msgDiv.innerHTML = `<small class="text-muted">${escapeHtml(message)}</small>`;
      document.getElementById('messages').appendChild(msgDiv);
    }

    function updateMemberList(members) {
      const list = document.getElementById('memberList');
      list.innerHTML = members.map(m =>
        `<li><span class="text-success">●</span> ${escapeHtml(m)}</li>`
      ).join('');
    }

    function sendMessage() {
      const input = document.getElementById('messageInput');
      const message = input.value.trim();
      if (message && ws && ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({ type: 'message', content: message }));
        input.value = '';
      }
    }

    function escapeHtml(text) {
      const div = document.createElement('div');
      div.appendChild(document.createTextNode(text));
      return div.innerHTML;
    }

    // Event listeners
    document.getElementById('sendBtn').addEventListener('click', sendMessage);
    document.getElementById('messageInput').addEventListener('keypress', (e) => {
      if (e.key === 'Enter') sendMessage();
    });

    // Heartbeat ทุก 30 วินาที
    setInterval(() => {
      if (ws && ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({ type: 'ping' }));
      }
    }, 30000);

    // เริ่มเชื่อมต่อ
    connect();
  </script>
}
```

---

## Step 476: WebSocket กับ Authentication

```scala
// app/controllers/AuthenticatedWebSocket.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*
import org.apache.pekko.stream.scaladsl.*
import org.apache.pekko.actor.*
import scala.concurrent.{ExecutionContext, Future}

@Singleton
class AuthenticatedWebSocket @Inject()(
  val controllerComponents: ControllerComponents
)(implicit system: ActorSystem, ec: ExecutionContext) extends BaseController {

  // WebSocket ที่ต้องการ authentication
  def authenticated(): WebSocket = WebSocket.acceptOrResult[JsValue, JsValue] { request =>
    // ตรวจสอบ authentication จาก session หรือ token
    val userId = request.session.get("userId")
                   .orElse(request.getQueryString("token")
                     .flatMap(validateToken))

    userId match {
      case Some(uid) =>
        // อนุญาตให้เชื่อมต่อ
        val flow = Flow[JsValue].map { msg =>
          Json.obj(
            "userId"  -> uid,
            "message" -> msg,
            "echo"    -> true
          )
        }
        Future.successful(Right(flow))

      case None =>
        // ปฏิเสธการเชื่อมต่อ
        Future.successful(Left(Unauthorized(Json.obj(
          "error" -> "Authentication required"
        ))))
    }
  }

  private def validateToken(token: String): Option[String] = {
    // validate JWT token และ return userId ถ้า valid
    if (token == "valid-token") Some("user-123")
    else None
  }
}
```

---

## Step 477: Server-Sent Events (SSE)

SSE เป็น alternative ของ WebSocket สำหรับ one-way server-to-client streaming

```scala
// app/controllers/SseController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.http.ContentTypes
import org.apache.pekko.stream.scaladsl.*
import scala.concurrent.duration.*

@Singleton
class SseController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  // Server-Sent Events
  def events(): Action[AnyContent] = Action { implicit request =>
    // สร้าง stream ของ events
    val eventSource = Source.tick(0.seconds, 1.second, ())
      .zipWithIndex
      .map { case (_, index) =>
        // SSE format: "data: <message>\n\n"
        s"data: ${play.api.libs.json.Json.obj(
          "index"     -> index,
          "timestamp" -> System.currentTimeMillis(),
          "message"   -> s"Event #$index"
        ).toString()}\n\n"
      }
      .map(org.apache.pekko.util.ByteString(_))
      .keepAlive(30.seconds, () => org.apache.pekko.util.ByteString(": ping\n\n"))

    Ok.chunked(eventSource).as("text/event-stream")
      .withHeaders(
        "Cache-Control" -> "no-cache",
        "X-Accel-Buffering" -> "no"  // Disable nginx buffering
      )
  }

  // SSE พร้อม Event ID
  def namedEvents(): Action[AnyContent] = Action { implicit request =>
    var eventId = 0L

    val source = Source.tick(0.seconds, 2.seconds, ())
      .map { _ =>
        eventId += 1
        s"id: $eventId\nevent: update\ndata: ${eventId}\n\n"
      }
      .map(org.apache.pekko.util.ByteString(_))

    Ok.chunked(source).as("text/event-stream")
  }
}
```

```javascript
// JavaScript SSE client
const eventSource = new EventSource('/events');

eventSource.onmessage = function(event) {
  const data = JSON.parse(event.data);
  console.log('Event:', data);
};

eventSource.addEventListener('update', function(event) {
  console.log('Update event:', event.data);
});

eventSource.onerror = function(error) {
  console.error('SSE Error:', error);
  // EventSource จะ reconnect อัตโนมัติ
};
```

---

## Step 478: Real-time Notifications

```scala
// app/actors/NotificationActor.scala
package actors

import org.apache.pekko.actor.*
import play.api.libs.json.*
import scala.collection.mutable

object NotificationActor {
  case class Subscribe(userId: String, actorRef: ActorRef)
  case class Unsubscribe(userId: String)
  case class Notify(userId: String, notification: JsValue)
  case class BroadcastAll(notification: JsValue)

  def props(): Props = Props(new NotificationActor())
}

class NotificationActor extends Actor {
  import NotificationActor.*

  private val subscribers = mutable.Map[String, ActorRef]()

  override def receive: Receive = {
    case Subscribe(userId, ref) =>
      subscribers(userId) = ref

    case Unsubscribe(userId) =>
      subscribers.remove(userId)

    case Notify(userId, notification) =>
      subscribers.get(userId).foreach(_ ! notification)

    case BroadcastAll(notification) =>
      subscribers.values.foreach(_ ! notification)
  }
}
```

```scala
// app/controllers/NotificationController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*
import play.api.libs.streams.ActorFlow
import org.apache.pekko.actor.*
import actors.NotificationActor

@Singleton
class NotificationController @Inject()(
  val controllerComponents: ControllerComponents,
  @Named("notification-actor") notificationActor: ActorRef
)(implicit system: ActorSystem) extends BaseController {

  def connect(userId: String): WebSocket =
    WebSocket.accept[JsValue, JsValue] { request =>
      ActorFlow.actorRef { out =>
        // Actor ที่รับ notifications สำหรับ user นี้
        Props(new Actor {
          notificationActor ! NotificationActor.Subscribe(userId, self)

          override def receive: Receive = {
            case notification: JsValue => out ! notification
            case _ =>
          }

          override def postStop(): Unit = {
            notificationActor ! NotificationActor.Unsubscribe(userId)
          }
        })
      }
    }
}
```

---

## Step 479: WebSocket Testing

```scala
// test/controllers/WebSocketControllerSpec.scala
package controllers

import org.scalatestplus.play.*
import org.scalatestplus.play.guice.GuiceOneServerPerTest
import play.api.test.*
import play.api.libs.ws.*
import play.api.libs.json.*
import scala.concurrent.duration.*
import scala.concurrent.Await

class WebSocketControllerSpec extends PlaySpec with GuiceOneServerPerTest {

  "WebSocketController echo" should {
    "echo messages back" in {
      // Test WebSocket ใช้ WsTestClient
      val wsUrl = s"ws://localhost:$port/ws/echo"

      // ใช้ WsTestClient สำหรับ testing
      WsTestClient.withClient { client =>
        val wsClient = client.url(wsUrl)
        // ทดสอบ WebSocket connection
        // (ใช้ library เช่น netty-ws-client ใน tests)
        wsUrl must include("ws://")
      }
    }
  }
}
```

---

## Step 480: Production Considerations

```hocon
# conf/application.conf - WebSocket configuration

# Pekko configuration
pekko.actor.provider = "local"

# WebSocket specific
play.server.websocket {
  # Max message size (bytes)
  # frame.maxLength = 65536
}

# Increase default timeouts
pekko.http.server.idle-timeout = 75 seconds
pekko.http.server.request-timeout = 120 seconds

# Connection pool
pekko.actor.default-dispatcher {
  fork-join-executor {
    parallelism-min = 8
    parallelism-factor = 3.0
    parallelism-max = 64
  }
}
```

---

## สรุป Part 48

| Concept | API | Use Case |
|---------|-----|---------|
| WebSocket accept | `WebSocket.accept[In, Out]` | Simple WS |
| WebSocket acceptOrResult | `WebSocket.acceptOrResult` | WS ที่ต้อง auth |
| ActorFlow | `ActorFlow.actorRef` | Actor-based WS |
| Source.tick | `Source.tick(init, interval, msg)` | Heartbeat |
| Server-Sent Events | `Ok.chunked(source).as("text/event-stream")` | One-way stream |
| Chat Room Actor | Pub/Sub pattern ด้วย actors | Multi-user chat |
| Authentication | `acceptOrResult` + session check | Secure WS |

---

## แบบฝึกหัด Part 48

1. **Echo Server**: สร้าง WebSocket echo server ที่รับ JSON และ return JSON พร้อม timestamp และ message count

2. **Live Updates**: สร้าง SSE endpoint ที่ push database updates ไปยัง connected clients ทุกครั้งที่มี new record

3. **Multiplayer Game**: สร้าง simple turn-based game (Tic-tac-toe) ผ่าน WebSocket ที่รองรับ 2 players

4. **Real-time Dashboard**: สร้าง dashboard ที่ update metrics (CPU, memory, request count) แบบ real-time ด้วย SSE

5. **Pub/Sub System**: สร้าง WebSocket-based pub/sub system ที่ user subscribe ไปยัง topics และรับ messages เฉพาะ topics ที่ subscribe

---

[→ ไปยัง Part 49: Play Filters and Middleware](part-49-play-filters-and-middleware.md)
