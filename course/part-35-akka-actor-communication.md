# Part 35: Akka Actor Communication

## Steps 341-350: Message Protocols, Ask Pattern, Tell Pattern, Stash, Adapters

---

## Step 341: Message Protocol Design

```scala
import akka.actor.typed._
import akka.actor.typed.scaladsl._
import scala.concurrent.duration._

// Well-designed message protocols
object MessageProtocols {
  
  // ===== User Service Protocol =====
  object UserService {
    // Commands (requests)
    sealed trait Command
    case class CreateUser(
      name: String,
      email: String,
      replyTo: ActorRef[CreateUserResponse]
    ) extends Command
    
    case class FindUser(
      id: String,
      replyTo: ActorRef[FindUserResponse]
    ) extends Command
    
    case class UpdateUser(
      id: String,
      name: Option[String] = None,
      email: Option[String] = None,
      replyTo: ActorRef[UpdateUserResponse]
    ) extends Command
    
    case class DeleteUser(
      id: String,
      replyTo: ActorRef[DeleteUserResponse]
    ) extends Command
    
    case class ListUsers(
      page: Int = 1,
      pageSize: Int = 20,
      replyTo: ActorRef[ListUsersResponse]
    ) extends Command
    
    // Responses
    sealed trait CreateUserResponse
    case class UserCreated(id: String, name: String, email: String) extends CreateUserResponse
    case class UserAlreadyExists(email: String) extends CreateUserResponse
    
    sealed trait FindUserResponse
    case class UserFound(id: String, name: String, email: String) extends FindUserResponse
    case class UserNotFound(id: String) extends FindUserResponse
    
    sealed trait UpdateUserResponse
    case class UserUpdated(id: String) extends UpdateUserResponse
    case class UpdateFailed(id: String, reason: String) extends UpdateUserResponse
    
    sealed trait DeleteUserResponse
    case class UserDeleted(id: String) extends DeleteUserResponse
    case class DeleteFailed(id: String, reason: String) extends DeleteUserResponse
    
    case class ListUsersResponse(users: List[UserSummary], total: Int, page: Int, pageSize: Int)
    case class UserSummary(id: String, name: String, email: String)
    
    // Internal messages (private to actor)
    private sealed trait InternalCommand extends Command
    private case class UserPersisted(user: UserSummary) extends InternalCommand
    
    // Behavior
    def apply(): Behavior[Command] = managing(Map.empty, 0)
    
    private def managing(
      users: Map[String, UserSummary],
      nextId: Int
    ): Behavior[Command] = Behaviors.receive { (context, message) =>
      message match {
        case CreateUser(name, email, replyTo) =>
          if (users.values.exists(_.email == email)) {
            replyTo ! UserAlreadyExists(email)
            Behaviors.same
          } else {
            val id = s"user-${nextId + 1}"
            val user = UserSummary(id, name, email)
            replyTo ! UserCreated(id, name, email)
            managing(users + (id -> user), nextId + 1)
          }
          
        case FindUser(id, replyTo) =>
          users.get(id) match {
            case Some(u) => replyTo ! UserFound(u.id, u.name, u.email)
            case None    => replyTo ! UserNotFound(id)
          }
          Behaviors.same
          
        case UpdateUser(id, name, email, replyTo) =>
          users.get(id) match {
            case None => replyTo ! UpdateFailed(id, "Not found")
            case Some(u) =>
              val updated = u.copy(
                name = name.getOrElse(u.name),
                email = email.getOrElse(u.email)
              )
              replyTo ! UserUpdated(id)
              managing(users + (id -> updated), nextId)
          }
          
        case DeleteUser(id, replyTo) =>
          if (users.contains(id)) {
            replyTo ! UserDeleted(id)
            managing(users - id, nextId)
          } else {
            replyTo ! DeleteFailed(id, "Not found")
            Behaviors.same
          }
          
        case ListUsers(page, pageSize, replyTo) =>
          val all = users.values.toList.sortBy(_.id)
          val paged = all.drop((page - 1) * pageSize).take(pageSize)
          replyTo ! ListUsersResponse(paged, all.size, page, pageSize)
          Behaviors.same
      }
    }
  }
}
```

---

## Step 342: Ask Pattern ลึก

```scala
import akka.actor.typed._
import akka.actor.typed.scaladsl._
import akka.actor.typed.scaladsl.AskPattern._
import akka.util.Timeout
import scala.concurrent._
import scala.concurrent.duration._

object AskPatternDeep extends App {
  
  // Actor that answers requests
  object CalculatorActor {
    sealed trait Command
    case class Calculate(
      expr: String,
      replyTo: ActorRef[Either[String, Double]]
    ) extends Command
    
    def apply(): Behavior[Command] = Behaviors.receiveMessage {
      case Calculate(expr, replyTo) =>
        val result = try {
          // Simple expression evaluator
          val parts = expr.split(" ")
          if (parts.length == 3) {
            val (a, op, b) = (parts(0).toDouble, parts(1), parts(2).toDouble)
            op match {
              case "+" => Right(a + b)
              case "-" => Right(a - b)
              case "*" => Right(a * b)
              case "/" if b != 0 => Right(a / b)
              case "/" => Left("Division by zero")
              case _ => Left(s"Unknown operator: $op")
            }
          } else Left(s"Invalid expression: $expr")
        } catch {
          case e: NumberFormatException => Left(s"Parse error: ${e.getMessage}")
        }
        replyTo ! result
        Behaviors.same
    }
  }
  
  // Actor that uses ask to communicate with another actor
  object PipelineActor {
    sealed trait Command
    case class Process(exprs: List[String], replyTo: ActorRef[List[Double]]) extends Command
    
    def apply(calculator: ActorRef[CalculatorActor.Command]): Behavior[Command] =
      Behaviors.setup { context =>
        implicit val timeout: Timeout = 5.seconds
        implicit val ec = context.executionContext
        implicit val scheduler = context.system.scheduler
        
        Behaviors.receiveMessage {
          case Process(exprs, replyTo) =>
            // Ask multiple times
            val futures: List[Future[Double]] = exprs.map { expr =>
              calculator
                .ask[Either[String, Double]](ref => CalculatorActor.Calculate(expr, ref))
                .map {
                  case Right(v) => v
                  case Left(e)  => Double.NaN
                }
            }
            
            Future.sequence(futures).foreach { results =>
              replyTo ! results
            }
            
            Behaviors.same
        }
      }
  }
  
  // Demonstrate ask pattern
  val system = ActorSystem(
    Behaviors.setup[Nothing] { context =>
      val calc = context.spawn(CalculatorActor(), "calculator")
      val pipeline = context.spawn(PipelineActor(calc), "pipeline")
      
      implicit val timeout: Timeout = 10.seconds
      implicit val scheduler = context.system.scheduler
      implicit val ec = context.executionContext
      
      // Ask pipeline for results
      val exprs = List("10 + 5", "20 * 3", "100 / 4", "7 - 2", "1 / 0")
      
      pipeline.ask[List[Double]](ref => PipelineActor.Process(exprs, ref))
        .foreach { results =>
          println("=== Ask Pattern Results ===")
          exprs.zip(results).foreach { case (expr, result) =>
            println(s"  $expr = $result")
          }
        }
      
      Behaviors.empty
    },
    "ask-system"
  )
  
  Thread.sleep(2000)
  system.terminate()
  Await.ready(system.whenTerminated, 5.seconds)(scala.concurrent.ExecutionContext.global)
}
```

---

## Step 343: Stash — Buffering Messages

```scala
import akka.actor.typed._
import akka.actor.typed.scaladsl._
import scala.concurrent.duration._

object StashDemo {
  
  // Database actor that must initialize before serving requests
  object DatabaseActor {
    sealed trait Command
    case class Query(sql: String, replyTo: ActorRef[List[String]]) extends Command
    case class Write(sql: String, replyTo: ActorRef[Boolean]) extends Command
    private case object InitializationComplete extends Command
    private case class InitializationFailed(cause: Throwable) extends Command
    
    def apply(): Behavior[Command] = Behaviors.withStash(100) { stash =>
      
      // Initializing state — stash all incoming requests
      Behaviors.setup { context =>
        context.log.info("Database: initializing...")
        
        // Simulate async initialization
        context.pipeToSelf(
          scala.concurrent.Future {
            Thread.sleep(500)  // Simulate connection establishment
            "Connection established"
          }(context.executionContext)
        ) {
          case scala.util.Success(_) => InitializationComplete
          case scala.util.Failure(e) => InitializationFailed(e)
        }
        
        initializing(stash)
      }
    }
    
    private def initializing(stash: StashBuffer[Command]): Behavior[Command] =
      Behaviors.receiveMessage {
        case InitializationComplete =>
          println("  [DB] Initialized! Processing stashed messages...")
          stash.unstashAll(ready)
          
        case InitializationFailed(cause) =>
          throw cause  // Propagate to supervisor
          
        case other =>
          println(s"  [DB] Stashing: $other")
          stash.stash(other)
          Behaviors.same
      }
    
    private val ready: Behavior[Command] = Behaviors.receiveMessage {
      case Query(sql, replyTo) =>
        println(s"  [DB] Executing query: $sql")
        replyTo ! List(s"row1-${sql.take(10)}", s"row2-${sql.take(10)}")
        Behaviors.same
        
      case Write(sql, replyTo) =>
        println(s"  [DB] Executing write: $sql")
        replyTo ! true
        Behaviors.same
        
      case _ => Behaviors.same
    }
  }
}
```

---

## Step 344-350: Message Adapters and Protocol Composition

```scala
import akka.actor.typed._
import akka.actor.typed.scaladsl._
import scala.concurrent.duration._

// Message Adapter — adapt external protocol to internal
object MessageAdapterDemo {
  
  // External protocol (e.g., from another actor system)
  object External {
    sealed trait Response
    case class Success(id: String, data: String) extends Response
    case class Failure(id: String, error: String) extends Response
    case class Timeout(id: String) extends Response
  }
  
  // Internal actor protocol
  object InternalActor {
    sealed trait Command
    // Adapted message
    case class ExternalResult(result: Either[String, String]) extends Command
    // Request message
    case class FetchData(key: String) extends Command
    
    def apply(externalService: ActorRef[FetchRequest]): Behavior[Command] =
      Behaviors.setup { context =>
        
        // Create adapter: External.Response → InternalActor.Command
        val adapter: ActorRef[External.Response] = context.messageAdapter {
          case External.Success(_, data) => ExternalResult(Right(data))
          case External.Failure(_, err)  => ExternalResult(Left(err))
          case External.Timeout(id)      => ExternalResult(Left(s"Timeout for $id"))
        }
        
        Behaviors.receiveMessage {
          case FetchData(key) =>
            externalService ! FetchRequest(key, adapter)
            Behaviors.same
            
          case ExternalResult(Right(data)) =>
            println(s"  Got data: $data")
            Behaviors.same
            
          case ExternalResult(Left(err)) =>
            println(s"  Error: $err")
            Behaviors.same
        }
      }
  }
  
  case class FetchRequest(key: String, replyTo: ActorRef[External.Response])
  
  object ExternalService {
    def apply(): Behavior[FetchRequest] = Behaviors.receiveMessage { req =>
      // Simulate response
      if (req.key.startsWith("valid")) {
        req.replyTo ! External.Success(req.key, s"Data for ${req.key}")
      } else {
        req.replyTo ! External.Failure(req.key, "Key not found")
      }
      Behaviors.same
    }
  }
}

// Interaction patterns: Tell vs Ask vs Pipe
object InteractionPatterns {
  
  // Tell: Fire and forget (one-way)
  object TellExample {
    object Logger {
      case class Log(level: String, message: String)
      
      def apply(): Behavior[Log] = Behaviors.receiveMessage { case Log(level, msg) =>
        println(s"  [$level] $msg")
        Behaviors.same
      }
    }
    
    object Service {
      case class Process(data: String)
      
      def apply(logger: ActorRef[Logger.Log]): Behavior[Process] =
        Behaviors.receiveMessage { case Process(data) =>
          logger ! Logger.Log("INFO", s"Processing: $data")  // Tell, no reply needed
          Behaviors.same
        }
    }
  }
  
  // Pipe to self: async result as message
  object PipeToSelfExample {
    sealed trait Command
    case class LoadConfig(path: String) extends Command
    private case class ConfigLoaded(config: Map[String, String]) extends Command
    private case class ConfigFailed(cause: Throwable) extends Command
    
    def apply(): Behavior[Command] = Behaviors.receive { (context, message) =>
      message match {
        case LoadConfig(path) =>
          // Pipe Future result back to self
          context.pipeToSelf(
            scala.concurrent.Future {
              // Simulate async config loading
              Thread.sleep(100)
              Map("host" -> "localhost", "port" -> "8080")
            }(context.executionContext)
          ) {
            case scala.util.Success(config) => ConfigLoaded(config)
            case scala.util.Failure(e)      => ConfigFailed(e)
          }
          Behaviors.same
          
        case ConfigLoaded(config) =>
          println(s"  Config loaded: $config")
          Behaviors.same
          
        case ConfigFailed(e) =>
          println(s"  Config failed: ${e.getMessage}")
          Behaviors.same
      }
    }
  }
}

// Aggregation pattern — collect responses from multiple actors
object AggregatorPattern {
  
  object ReportAggregator {
    sealed trait Command
    case class RequestReport(
      sources: List[String],
      replyTo: ActorRef[Report]
    ) extends Command
    
    private case class DataReceived(source: String, data: String) extends Command
    private case class Timeout(requestId: String) extends Command
    
    case class Report(data: Map[String, String], missing: List[String])
    
    def apply(): Behavior[Command] = Behaviors.setup { context =>
      // Spawn child data sources
      val sources = Map(
        "source-1" -> context.spawn(DataSource("source-1"), "ds1"),
        "source-2" -> context.spawn(DataSource("source-2"), "ds2"),
        "source-3" -> context.spawn(DataSource("source-3"), "ds3")
      )
      
      Behaviors.receiveMessage {
        case RequestReport(requestedSources, replyTo) =>
          // Adapter for responses
          // In real code, would use request-per-source with correlation IDs
          val available = requestedSources.filter(sources.contains)
          val missing = requestedSources.filterNot(sources.contains)
          
          // Collect from available sources
          val data = available.map { src =>
            src -> s"data-from-$src-${System.currentTimeMillis()}"
          }.toMap
          
          replyTo ! Report(data, missing)
          Behaviors.same
      }
    }
  }
  
  object DataSource {
    case class Fetch(replyTo: ActorRef[String])
    
    def apply(name: String): Behavior[Fetch] = Behaviors.receiveMessage { case Fetch(replyTo) =>
      Thread.sleep(50)
      replyTo ! s"Data from $name"
      Behaviors.same
    }
  }
}
```

---

## สรุป Part 35

| Pattern | เมื่อใช้ | ตัวอย่าง |
|---------|---------|---------|
| Tell (`!`) | Fire-and-forget | Logging, events |
| Ask (`.ask`) | Request-response | Query, command with reply |
| PipeToSelf | Async result → message | DB calls, HTTP calls |
| Message Adapter | Protocol translation | Bridging actor protocols |
| Stash | Buffer during initialization | DB init, config loading |
| Aggregator | Collect multiple responses | Fan-out/fan-in |
| Router | Distribute load | Worker pool |
| Correlator | Match request/response | Async request tracking |

---

## แบบฝึกหัด Part 35

**ข้อ 1:** Implement `SagaOrchestrator` ที่ coordinate multi-step transaction ด้วย compensating transactions

**ข้อ 2:** สร้าง `RequestTracker` actor ที่ track in-flight requests พร้อม timeout handling

**ข้อ 3:** Implement `BroadcastGroup` ที่ fan-out และ aggregate responses

**ข้อ 4:** สร้าง `ContentionManager` ที่ serialize access to shared resource ผ่าน actor mailbox

**ข้อ 5:** Implement conversation protocol ระหว่าง buyer/seller agents ที่ negotiate price

---

➡️ ต่อไป: [Part 36 — Akka Supervision](part-36-akka-supervision.md)
