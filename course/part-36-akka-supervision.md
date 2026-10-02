# Part 36: Akka Supervision

## Steps 351-360: Supervision Strategies, Fault Tolerance, Death Watch, Error Kernel

---

## Step 351: Supervision Strategies

```scala
import akka.actor.typed._
import akka.actor.typed.scaladsl._
import scala.concurrent.duration._

object SupervisionStrategiesDemo {
  
  // ===== Failure types =====
  class TransientException(msg: String) extends RuntimeException(msg)
  class PermanentException(msg: String) extends RuntimeException(msg)
  class CriticalException(msg: String) extends RuntimeException(msg)
  
  // ===== Worker that can fail =====
  object FailableWorker {
    sealed trait Command
    case class Process(id: Int, failWith: Option[String], replyTo: ActorRef[String]) extends Command
    
    def apply(name: String): Behavior[Command] = Behaviors.receiveMessage {
      case Process(id, Some("transient"), replyTo) =>
        throw new TransientException(s"Transient failure on job $id")
      case Process(id, Some("permanent"), replyTo) =>
        throw new PermanentException(s"Permanent failure on job $id")
      case Process(id, Some("critical"), replyTo) =>
        throw new CriticalException(s"Critical failure on job $id")
      case Process(id, None, replyTo) =>
        replyTo ! s"$name completed job $id"
        Behaviors.same
    }
  }
  
  // ===== Supervisor with different strategies =====
  object SmartSupervisor {
    sealed trait Command
    case class DoWork(id: Int, failMode: Option[String]) extends Command
    
    def apply(): Behavior[Command] = Behaviors.setup { context =>
      
      // Strategy 1: Restart on transient errors (max 5 in 10 seconds)
      val restartWorker = context.spawn(
        Behaviors.supervise(FailableWorker("restart-worker"))
          .onFailure[TransientException](
            SupervisorStrategy.restart
              .withLimit(5, 10.seconds)
          ),
        "restart-worker"
      )
      
      // Strategy 2: Resume on transient errors (keep state)
      val resumeWorker = context.spawn(
        Behaviors.supervise(FailableWorker("resume-worker"))
          .onFailure[TransientException](SupervisorStrategy.resume),
        "resume-worker"
      )
      
      // Strategy 3: Stop on permanent errors
      val stopWorker = context.spawn(
        Behaviors.supervise(FailableWorker("stop-worker"))
          .onFailure[PermanentException](SupervisorStrategy.stop),
        "stop-worker"
      )
      
      // Strategy 4: Nested supervision
      val nestedWorker = context.spawn(
        Behaviors.supervise(
          Behaviors.supervise(FailableWorker("nested-worker"))
            .onFailure[TransientException](SupervisorStrategy.restart.withLimit(3, 5.seconds))
        ).onFailure[PermanentException](SupervisorStrategy.stop),
        "nested-worker"
      )
      
      // Watch workers
      context.watch(stopWorker)
      
      Behaviors.receiveMessagePartial[Command] {
        case DoWork(id, failMode) =>
          val replyRef = context.spawnAnonymous(
            Behaviors.receiveMessage[String] { msg =>
              println(s"  Result: $msg")
              Behaviors.stopped
            }
          )
          restartWorker ! FailableWorker.Process(id, failMode, replyRef)
          Behaviors.same
      }.receiveSignal {
        case (context, Terminated(ref)) =>
          println(s"  Worker terminated: ${ref.path.name}")
          Behaviors.same
      }
    }
  }
}
```

---

## Step 352: Death Watch — Monitoring Actor Lifecycle

```scala
import akka.actor.typed._
import akka.actor.typed.scaladsl._
import scala.concurrent.duration._

object DeathWatchDemo {
  
  // Service that others depend on
  object DataService {
    sealed trait Command
    case class GetData(key: String, replyTo: ActorRef[String]) extends Command
    case object Shutdown extends Command
    
    def apply(): Behavior[Command] = Behaviors.receiveMessage {
      case GetData(key, replyTo) =>
        replyTo ! s"data-for-$key"
        Behaviors.same
      case Shutdown =>
        println("  DataService: shutting down")
        Behaviors.stopped
    }
  }
  
  // Client that watches the service
  object ServiceClient {
    sealed trait Command
    case class UseService(key: String) extends Command
    case object ServiceAvailable extends Command
    case object ServiceDown extends Command
    
    def apply(service: ActorRef[DataService.Command]): Behavior[Command] =
      Behaviors.setup { context =>
        context.watchWith(service, ServiceDown)
        connected(service)
      }
    
    private def connected(service: ActorRef[DataService.Command]): Behavior[Command] =
      Behaviors.receiveMessage {
        case UseService(key) =>
          val adapter = ???  // Would use message adapter
          service ! DataService.GetData(key, ???)
          Behaviors.same
          
        case ServiceDown =>
          println("  Client: Service went down! Switching to degraded mode.")
          degraded()
          
        case ServiceAvailable =>
          Behaviors.same
      }
    
    private def degraded(): Behavior[Command] =
      Behaviors.receiveMessage {
        case UseService(key) =>
          println(s"  Client: Serving from cache for key=$key")
          Behaviors.same
        case ServiceAvailable =>
          println("  Client: Service back online!")
          Behaviors.same
        case _ =>
          Behaviors.same
      }
  }
  
  // Dependency tracking
  object DependencyGraph {
    sealed trait Command
    case class Register(name: String, actor: ActorRef[Nothing]) extends Command
    case class Deregister(name: String) extends Command
    case class GetStatus(replyTo: ActorRef[Map[String, Boolean]]) extends Command
    
    def apply(): Behavior[Command] = managing(Map.empty, Map.empty)
    
    private def managing(
      deps: Map[String, ActorRef[Nothing]],
      status: Map[String, Boolean]
    ): Behavior[Command] = Behaviors.receive { (context, message) =>
      message match {
        case Register(name, actor) =>
          context.watchWith(actor, DependencyDown(name))
          managing(deps + (name -> actor), status + (name -> true))
          
        case Deregister(name) =>
          deps.get(name).foreach(context.unwatch)
          managing(deps - name, status - name)
          
        case GetStatus(replyTo) =>
          replyTo ! status
          Behaviors.same
          
        case DependencyDown(name) =>
          println(s"  Dependency '$name' went down!")
          managing(deps - name, status + (name -> false))
      }
    }
    
    private case class DependencyDown(name: String) extends Command
  }
}
```

---

## Step 353: Error Kernel Pattern

```scala
import akka.actor.typed._
import akka.actor.typed.scaladsl._
import scala.concurrent.duration._

// Error Kernel: risky operations at leaves, stable core at top
object ErrorKernelPattern {
  
  // Core (stable, never crashes)
  object OrderService {
    sealed trait Command
    case class PlaceOrder(
      customerId: String,
      items: List[String],
      replyTo: ActorRef[OrderResult]
    ) extends Command
    
    sealed trait OrderResult
    case class OrderAccepted(orderId: String) extends OrderResult
    case class OrderRejected(reason: String) extends OrderResult
    
    def apply(): Behavior[Command] = Behaviors.setup { context =>
      val validator = context.spawn(
        Behaviors.supervise(OrderValidator())
          .onFailure[ValidationException](SupervisorStrategy.restart.withLimit(10, 30.seconds)),
        "validator"
      )
      
      val paymentProcessor = context.spawn(
        Behaviors.supervise(PaymentProcessor())
          .onFailure[PaymentException](SupervisorStrategy.restart.withLimit(5, 10.seconds)),
        "payment"
      )
      
      Behaviors.receiveMessage {
        case PlaceOrder(customerId, items, replyTo) =>
          // Delegate to child actors (risky operations)
          context.spawnAnonymous(
            OrderCoordinator(customerId, items, replyTo, validator, paymentProcessor)
          )
          Behaviors.same
      }
    }
  }
  
  class ValidationException(msg: String) extends RuntimeException(msg)
  class PaymentException(msg: String) extends RuntimeException(msg)
  
  object OrderValidator {
    sealed trait Command
    case class Validate(
      customerId: String,
      items: List[String],
      replyTo: ActorRef[ValidationResult]
    ) extends Command
    
    sealed trait ValidationResult
    case class Valid(customerId: String, items: List[String]) extends ValidationResult
    case class Invalid(errors: List[String]) extends ValidationResult
    
    def apply(): Behavior[Command] = Behaviors.receiveMessage {
      case Validate(customerId, items, replyTo) =>
        val errors = List(
          if (customerId.isEmpty) Some("Customer ID required") else None,
          if (items.isEmpty) Some("At least one item required") else None,
          if (items.size > 50) Some("Max 50 items per order") else None
        ).flatten
        
        if (errors.isEmpty) replyTo ! Valid(customerId, items)
        else replyTo ! Invalid(errors)
        
        Behaviors.same
    }
  }
  
  object PaymentProcessor {
    sealed trait Command
    case class ProcessPayment(
      customerId: String,
      amount: Double,
      replyTo: ActorRef[PaymentResult]
    ) extends Command
    
    sealed trait PaymentResult
    case class PaymentApproved(transactionId: String) extends PaymentResult
    case class PaymentDeclined(reason: String) extends PaymentResult
    
    def apply(): Behavior[Command] = Behaviors.receiveMessage {
      case ProcessPayment(customerId, amount, replyTo) =>
        // Simulate payment (risky, can throw)
        if (amount > 1000000) throw new PaymentException("Amount exceeds limit")
        
        if (scala.util.Random.nextDouble() > 0.1) {
          replyTo ! PaymentApproved(s"TXN-${System.currentTimeMillis()}")
        } else {
          replyTo ! PaymentDeclined("Insufficient funds")
        }
        Behaviors.same
    }
  }
  
  // Coordinator — orchestrates the workflow
  object OrderCoordinator {
    sealed trait Internal
    case class ValidationComplete(result: OrderValidator.ValidationResult) extends Internal
    case class PaymentComplete(result: PaymentProcessor.PaymentResult) extends Internal
    
    def apply(
      customerId: String,
      items: List[String],
      replyTo: ActorRef[OrderService.OrderResult],
      validator: ActorRef[OrderValidator.Command],
      payment: ActorRef[PaymentProcessor.Command]
    ): Behavior[Internal] = Behaviors.setup { context =>
      
      val validationAdapter: ActorRef[OrderValidator.ValidationResult] =
        context.messageAdapter(ValidationComplete.apply)
      
      val paymentAdapter: ActorRef[PaymentProcessor.PaymentResult] =
        context.messageAdapter(PaymentComplete.apply)
      
      validator ! OrderValidator.Validate(customerId, items, validationAdapter)
      
      waitForValidation(replyTo, items, payment, paymentAdapter)
    }
    
    private def waitForValidation(
      replyTo: ActorRef[OrderService.OrderResult],
      items: List[String],
      payment: ActorRef[PaymentProcessor.Command],
      paymentAdapter: ActorRef[PaymentProcessor.PaymentResult]
    ): Behavior[Internal] = Behaviors.receiveMessage {
      case ValidationComplete(OrderValidator.Valid(customerId, _)) =>
        val amount = items.size * 100.0  // Simplified pricing
        payment ! PaymentProcessor.ProcessPayment(customerId, amount, paymentAdapter)
        waitForPayment(replyTo)
        
      case ValidationComplete(OrderValidator.Invalid(errors)) =>
        replyTo ! OrderService.OrderRejected(errors.mkString(", "))
        Behaviors.stopped
        
      case _ => Behaviors.same
    }
    
    private def waitForPayment(
      replyTo: ActorRef[OrderService.OrderResult]
    ): Behavior[Internal] = Behaviors.receiveMessage {
      case PaymentComplete(PaymentProcessor.PaymentApproved(txnId)) =>
        val orderId = s"ORD-${System.currentTimeMillis()}"
        println(s"  Order created: $orderId (txn: $txnId)")
        replyTo ! OrderService.OrderAccepted(orderId)
        Behaviors.stopped
        
      case PaymentComplete(PaymentProcessor.PaymentDeclined(reason)) =>
        replyTo ! OrderService.OrderRejected(s"Payment declined: $reason")
        Behaviors.stopped
        
      case _ => Behaviors.same
    }
  }
}
```

---

## Step 354-360: Resilience Patterns

```scala
import akka.actor.typed._
import akka.actor.typed.scaladsl._
import scala.concurrent.duration._

// Bulkhead pattern — isolate failures
object BulkheadPattern {
  
  sealed trait ServiceCommand
  case class HandleRequest(id: Int, data: String, replyTo: ActorRef[String]) extends ServiceCommand
  
  object ResilientService {
    def apply(): Behavior[ServiceCommand] = Behaviors.setup { context =>
      // Separate pools for different operations
      val criticalPool = (1 to 5).map { i =>
        context.spawn(
          Behaviors.supervise(WorkerActor(s"critical-$i"))
            .onFailure[Exception](SupervisorStrategy.restart),
          s"critical-worker-$i"
        )
      }.toList
      
      val standardPool = (1 to 3).map { i =>
        context.spawn(
          Behaviors.supervise(WorkerActor(s"standard-$i"))
            .onFailure[Exception](SupervisorStrategy.restart),
          s"standard-worker-$i"
        )
      }.toList
      
      var criticalIdx = 0
      var standardIdx = 0
      
      Behaviors.receiveMessage {
        case req @ HandleRequest(id, data, _) if id % 5 == 0 =>  // Critical 20%
          val worker = criticalPool(criticalIdx % criticalPool.size)
          worker ! req
          criticalIdx += 1
          Behaviors.same
          
        case req @ HandleRequest(_, _, _) =>
          val worker = standardPool(standardIdx % standardPool.size)
          worker ! req
          standardIdx += 1
          Behaviors.same
      }
    }
  }
  
  object WorkerActor {
    def apply(name: String): Behavior[ServiceCommand] = Behaviors.receiveMessage {
      case HandleRequest(id, data, replyTo) =>
        replyTo ! s"$name processed request $id: $data"
        Behaviors.same
    }
  }
}

// Exponential backoff restart
object ExponentialBackoffDemo {
  
  object UnstableActor {
    sealed trait Command
    case class Process(data: String) extends Command
    
    def apply(): Behavior[Command] = Behaviors.setup { context =>
      context.log.info(s"${context.self.path.name}: Starting fresh")
      
      Behaviors.receiveMessage {
        case Process(data) if data == "crash" =>
          throw new RuntimeException("Simulated crash")
        case Process(data) =>
          context.log.info(s"Processing: $data")
          Behaviors.same
      }
    }
  }
  
  def supervisedActor(): Behavior[UnstableActor.Command] =
    Behaviors.supervise(UnstableActor())
      .onFailure[RuntimeException](
        SupervisorStrategy.restartWithBackoff(
          minBackoff = 200.millis,
          maxBackoff = 10.seconds,
          randomFactor = 0.2
        ).withMaxRestarts(5)
      )
}

// Health check pattern
object HealthCheckPattern {
  
  case class HealthStatus(
    name: String,
    healthy: Boolean,
    details: Map[String, String]
  )
  
  object HealthCheckActor {
    sealed trait Command
    case class CheckHealth(replyTo: ActorRef[HealthStatus]) extends Command
    private case class HeartbeatResponse(from: String, healthy: Boolean) extends Command
    
    def apply(name: String, dependencies: List[String]): Behavior[Command] =
      Behaviors.setup { context =>
        Behaviors.receiveMessage {
          case CheckHealth(replyTo) =>
            val startTime = System.currentTimeMillis()
            val status = HealthStatus(
              name = name,
              healthy = true,
              details = Map(
                "uptime" -> s"${(System.currentTimeMillis() - startTime)}ms",
                "dependencies" -> dependencies.mkString(", "),
                "version" -> "1.0.0"
              )
            )
            replyTo ! status
            Behaviors.same
        }
      }
  }
  
  object HealthMonitor {
    sealed trait Command
    case object Check extends Command
    case class GetReport(replyTo: ActorRef[List[HealthStatus]]) extends Command
    private case class StatusReceived(status: HealthStatus) extends Command
    
    def apply(
      actors: Map[String, ActorRef[HealthCheckActor.Command]]
    ): Behavior[Command] = Behaviors.withTimers { timers =>
      timers.startTimerWithFixedDelay("health-check", Check, 30.seconds)
      monitoring(actors, Map.empty)
    }
    
    private def monitoring(
      actors: Map[String, ActorRef[HealthCheckActor.Command]],
      lastStatuses: Map[String, HealthStatus]
    ): Behavior[Command] = Behaviors.receive { (context, message) =>
      message match {
        case Check =>
          actors.values.foreach { actor =>
            implicit val timeout: akka.util.Timeout = 5.seconds
            implicit val scheduler = context.system.scheduler
            implicit val ec = context.executionContext
            import akka.actor.typed.scaladsl.AskPattern._
            
            actor.ask[HealthStatus](ref => HealthCheckActor.CheckHealth(ref))
              .foreach(status => context.self ! StatusReceived(status))
          }
          Behaviors.same
          
        case StatusReceived(status) =>
          if (!status.healthy) context.log.warn(s"UNHEALTHY: ${status.name}")
          monitoring(actors, lastStatuses + (status.name -> status))
          
        case GetReport(replyTo) =>
          replyTo ! lastStatuses.values.toList
          Behaviors.same
      }
    }
  }
}
```

---

## สรุป Part 36

| Strategy | เมื่อใช้ | ผลลัพธ์ |
|---------|---------|---------|
| `restart` | Transient errors | Actor restarts, state lost |
| `resume` | Ignorable errors | Continues, state preserved |
| `stop` | Permanent errors | Actor stops permanently |
| `restartWithBackoff` | Service unavailable | Restart with increasing delay |
| `.withLimit(n, d)` | Limit restarts | Stop after n restarts in d duration |
| `watchWith(actor, msg)` | Monitor actor | Receive msg when actor terminates |
| Death watch | Dependency tracking | React to actor termination |
| Error Kernel | Isolate risky operations | Core stable, leaves fail safely |
| Bulkhead | Isolate failure domains | Separate pools for operations |

---

## แบบฝึกหัด Part 36

**ข้อ 1:** Implement `HealthMonitor` ที่ track actor health และ alert on failures

**ข้อ 2:** สร้าง self-healing actor ที่ restart และ recover state from persistent storage

**ข้อ 3:** Implement `CascadeShutdown` ที่ gracefully stop actor hierarchy

**ข้อ 4:** สร้าง fault injection framework สำหรับ testing actor supervision

**ข้อ 5:** Implement `AdaptiveCircuitBreaker` actor ที่ adjust thresholds based on error rate

---

➡️ ต่อไป: [Part 37 — Akka Streams](part-37-akka-streams.md)
